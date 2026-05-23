# QRMI Secure Credential Setup

## tmpfiles.d

Install the provided tmpfiles rule to clean up orphaned credential dirs:

    cp lib/systemd/tmpfiles.d/qrmi.conf /etc/tmpfiles.d/
    systemd-tmpfiles --create /etc/tmpfiles.d/qrmi.conf

Dirs under /run/qrmi older than 7 days are removed automatically.
The SPANK epilog handles normal cleanup; this is a safety net only.
