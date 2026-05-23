# QRMI Secure Credential Setup

This guide covers deploying the secure credential file model introduced in
`spank_qrmi` v2.0. Raw credentials are no longer injected as environment
variables into the job. Instead the SPANK prolog writes a per-job credential
file and injects only its path.

## How it works

1. **Prolog** (`slurm_spank_init_post_opt`):
   - Reads credentials from `qrmi_config.json` (admin-owned, not visible to jobs)
   - For IBM backends: exchanges the long-lived API key for a short-lived (1hr)
     IAM bearer token
   - Writes token + non-secret config to `/run/qrmi/<job_id>/<resource>.json`
     (mode 0400, owned by the job user) using an atomic write
   - Injects only `${NAME}_QRMI_CREDENTIAL_FILE=/run/qrmi/<job_id>/<resource>.json`
     into the job environment

2. **Job**: reads credentials from the file via the QRMI Rust library

3. **Epilog** (`slurm_spank_exit`):
   - Deletes `/run/qrmi/<job_id>/` and all credential files in it

## Token expiry

IBM IAM bearer tokens are valid for **1 hour** from issue time. Jobs running
longer than 1 hour will receive 401 errors. Per-call token refresh is planned
for Phase 2. Design jobs to fit within the 1-hour window, or use the
`QRMI_LEGACY_CREDENTIALS` build flag during a phased migration.

## Config file permissions

The config file contains long-lived API keys and must be protected:

```bash
chmod 640 /etc/slurm/qrmi_config.json
chown root:slurm /etc/slurm/qrmi_config.json
```

Only `slurmd` (running as root or slurm) reads this file at prolog time. Job
processes never have access to it.

## tmpfiles.d

Install the provided tmpfiles rule to clean up orphaned credential dirs:

```bash
cp lib/systemd/tmpfiles.d/qrmi.conf /etc/tmpfiles.d/
systemd-tmpfiles --create /etc/tmpfiles.d/qrmi.conf
```

Dirs under `/run/qrmi` older than 7 days are removed automatically.
The SPANK epilog handles normal cleanup; this is a safety net for jobs
that were OOM-killed or left orphaned dirs on node failure.

## Phased migration

If you need time to transition existing scripts, build the plugin with:

```bash
cmake -DQRMI_LEGACY_CREDENTIALS=ON ...
```

This re-enables the old env-var injection alongside the new credential file.
Remove the flag once all job scripts are updated to use
`${NAME}_QRMI_CREDENTIAL_FILE`.

## Known Phase 1 limitations

- **IBM auth**: the bearer token in the credential file is validated but IBM
  job auth still requires `AuthMethod::BearerToken` support in
  `quantum_system_client` (planned for Phase 2). Until then, use
  `QRMI_LEGACY_CREDENTIALS` for IBM jobs.
- **PASQAL**: fully supported — `pasqal-cloud` credential files work end-to-end.
- **S3**: HMAC credentials are in the credential file (not in the job environment),
  but are still long-lived. STS short-lived credential exchange is Phase 2.
- **`pasqal-local`**: excluded from Phase 1 (auth requirements unconfirmed for
  the local Pulser simulator). Track as Phase 1B.
- **Jobs > 1hr**: will 401 at token expiry. Per-call refresh is Phase 2.
