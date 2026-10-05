# Ansible role - log_retention
[![Maintainer](https://img.shields.io/badge/maintained%20by-bouola-e00000?style=flat-square)](https://github.com/bouola)
[![License](https://img.shields.io/github/license/bouola/ansible-role-log_retention?style=flat-square)](LICENSE)
[![Release](https://img.shields.io/github/v/release/bouola/ansible-role-log_retention?style=flat-square)](https://github.com/bouola/ansible-role-log_retention/releases)
[![Status](https://img.shields.io/github/actions/workflow/status/bouola/ansible-role-log_retention/ci.yml?style=flat-square&label=tests&branch=main)](https://github.com/bouola/ansible-role-log_retention/actions?query=workflow%3A%22CI%22)
[![Ansible version](https://img.shields.io/badge/ansible-%3E%3D2.15-black.svg?style=flat-square&logo=ansible)](https://github.com/ansible/ansible)
[![Ansible Galaxy](https://img.shields.io/badge/ansible-galaxy-black.svg?style=flat-square&logo=ansible)](https://galaxy.ansible.com/bouola/log_retention)

Keep local logs bounded: rotate and compress log files on a fixed schedule, with a known number of days of history.

Galaxy FQCN: `bouola.log_retention`

Minimal cloud images often ship without logrotate. The rules that packages drop in `/etc/logrotate.d` (nginx, apt,
dpkg) are then never run, and log files grow until the disk is full. This role installs logrotate, sets its global
defaults, adds or replaces rules, and controls when its timer runs.

Local logs are for short-term troubleshooting. Ship logs to a central store (Loki, Elasticsearch) for search, and
archive the ones you need for audits elsewhere.

## Requirements

- Ansible 2.15 or newer
- Debian 12, Debian 13, Ubuntu 22.04, or Ubuntu 24.04, with systemd

## Role Variables

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `log_retention_logrotate_enabled` | boolean | `true` | Install and configure logrotate. `false` leaves logrotate untouched. |
| `log_retention_logrotate_frequency` | string | `daily` | Global rotation frequency: `hourly`, `daily`, `weekly`, `monthly` or `yearly`. |
| `log_retention_logrotate_rotate` | integer | `7` | Rotated files kept by default. With a daily frequency, the days of local history. |
| `log_retention_logrotate_compress` | boolean | `true` | Compress rotated files with gzip, one cycle late (`delaycompress`). |
| `log_retention_logrotate_rules` | list | `[]` | Rules written to `/etc/logrotate.d/<name>`. See [Rule definition](#rule-definition). |
| `log_retention_logrotate_timer_on_calendar` | string | `daily` | When `logrotate.timer` runs, as a systemd [calendar expression](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html#Calendar%20Events), validated with `systemd-analyze calendar`. |

The global defaults apply to every rule that does not set the option itself. Packaged rules often do: the nginx
package, for example, keeps 14 days whatever the global default. Add a rule with the same name to change it.

### Rule definition

| Key | Required | Default | Description |
| --- | --- | --- | --- |
| `name` | yes | | File name in `/etc/logrotate.d`. A rule named like a packaged file replaces it. |
| `paths` | yes, unless `state: absent` | | Absolute paths or globs of the files to rotate. |
| `state` | no | `present` | `absent` removes `/etc/logrotate.d/<name>`. |
| `frequency` | no | global default | `hourly`, `daily`, `weekly`, `monthly` or `yearly`. |
| `rotate` | no | global default | Rotated files kept. |
| `maxsize` | no | | Also rotate when a file grows past this size, for example `100M`. Checked only when the timer runs. |
| `compress` | no | `log_retention_logrotate_compress` | Compress rotated files, one cycle late. |
| `missingok` | no | `true` | Do not fail when no file matches. |
| `notifempty` | no | `true` | Do not rotate empty files. |
| `dateext` | no | `false` | Suffix rotated files with the date instead of a number. |
| `copytruncate` | no | `false` | Copy then truncate the file, for programs that cannot reopen their log. A few lines can be lost. |
| `create` | no | global `create` | Mode, owner and group of the new file, for example `0640 www-data adm`. Ignored with `copytruncate`. |
| `su` | no | | User and group logrotate switches to for these files, for example `www-data adm`. |
| `prerotate` | no | | Shell script run once before rotation. |
| `postrotate` | no | | Shell script run once after rotation, typically to make the program reopen its log. |

Every rule file is checked with `logrotate --debug` before it is installed, then the whole configuration is checked,
so a syntax error or a file claimed by two rules fails the play instead of breaking rotation.

## How it works

```text
logrotate.timer   (OnCalendar=<log_retention_logrotate_timer_on_calendar>)
        |
logrotate.service -> logrotate /etc/logrotate.conf
        |
        |- global defaults: frequency, rotate, create, compress
        '- include /etc/logrotate.d: packaged rules and the rules of this role
```

The timer and the service come from the logrotate package. The role only overrides the timer schedule, in
`/etc/systemd/system/logrotate.timer.d/override.conf`, when it is not `daily`.

Check what logrotate would do, without rotating anything:

```bash
logrotate --debug /etc/logrotate.conf
```

## Dependencies

None.

## Example Playbook

Seven days of compressed history for every log, and the nginx package rule aligned on it:

```yaml
---
- name: "Bound local logs"
  hosts: all
  become: true
  roles:
    - role: "bouola.log_retention"
      vars:
        log_retention_logrotate_rotate: 7
        log_retention_logrotate_rules:
          - name: "nginx"
            paths:
              - "/var/log/nginx/*.log"
            rotate: 7
            create: "0640 www-data adm"
            postrotate: |
              if [ -f /var/run/nginx.pid ]; then
                kill -USR1 "$(cat /var/run/nginx.pid)"
              fi
```

An application that writes a lot: rotate it as soon as it reaches 200 MB, checked every hour.

```yaml
log_retention_logrotate_timer_on_calendar: "hourly"
log_retention_logrotate_rules:
  - name: "myapp"
    paths:
      - "/var/log/myapp/*.log"
    rotate: 5
    maxsize: "200M"
    copytruncate: true
```

## Molecule Scenarios

| Scenario | Coverage |
| --- | --- |
| `default` | logrotate installation, global defaults (including the Ubuntu `su` directive), rule options, removal of an absent rule, hourly timer override, and forced rotations that recreate, compress and run `postrotate`. |

```bash
molecule test -s default
```

## License

MIT
