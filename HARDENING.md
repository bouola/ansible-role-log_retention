# Introduction
In this document, a list of relevant settings for hardening log_retention is provided.
This document is non-exhaustive, however, it provides a solid base of security hardening
measures.

## Rule files

- Rule files in `/etc/logrotate.d` are owned by `root:root` with mode `0644`. logrotate refuses rules that are
  group- or world-writable, because `prerotate` and `postrotate` scripts run as root.
- Treat anyone who can change `log_retention_logrotate_rules` as root on the host: their scripts run as root.

## Rotated files

- Set `create` with the narrowest mode that the program and its readers need, for example `0640 www-data adm`.
  Access logs hold client IP addresses and request paths.
- Use `su` for directories that are writable by a non-root user, so that logrotate does not follow a link planted
  there with root privileges.

## Retention

- Local history is a troubleshooting aid, not an audit trail: a compromised host can rewrite its own logs. Ship logs
  you must keep to a separate store, and archive them from there.
- Keep only the history you need. Logs often hold personal data, such as IP addresses, whose retention must be
  bounded.
