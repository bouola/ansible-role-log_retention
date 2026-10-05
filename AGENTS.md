# AGENTS.md

This file defines how agents should work in this repository.

These instructions apply repo-wide unless the user gives a more specific request.

## Intent

This is the `bouola.log_retention` Ansible role, published as open source. It keeps local logs bounded on Debian and
Ubuntu hosts. Today it manages logrotate: package, global defaults, rules in `/etc/logrotate.d`, and the timer
schedule.

Local logs are for short-term troubleshooting. The role does not ship or archive logs: that belongs to a log
collector and a backup tool.

## Layout

- `tasks/`: `validate.yml` and `logrotate.yml`, included from `main.yml`.
- `templates/`: `/etc/logrotate.conf`, one rule file per rule, and the `logrotate.timer` drop-in.
- `vars/main.yml`: per-distribution constants (the global `su` directive) and the allowed frequencies.

## Variables

- Public variables start with `log_retention_` and are all documented in `defaults/main.yml` and `README.md`.
- Private variables (loop variables, registered results, facts) start with `_log_retention_`. `.ansible-lint`
  enforces this for loop variables.
- Defaults are typed and truthful: every default must be implemented end to end. Do not add variables for a feature
  that is not implemented yet.
- Use YAML booleans `true` and `false`, never `yes` or `no`.

## Task style

- Every task has a sentence-case `name:` in double quotes and uses FQCNs.
- Use modules instead of `command` whenever one exists. When `command` is needed, set `changed_when` explicitly.
- Validation uses `ansible.builtin.assert` with an explicit multiline `fail_msg: >-`.
- Every loop sets `loop_control.loop_var` and `loop_control.label`.
- Role tags are `role-log-retention-validate` and `role-log-retention-logrotate`, applied through `include_tasks`
  with `apply.tags`.

## Safety

- Validate every rule file with `logrotate --debug` before installing it, and the whole configuration after: a
  broken rule silently stops rotation for every file.
- Never delete log files. The role changes how logs rotate, not the logs already on disk.
- Replacing a packaged rule (same name) is explicit in the user's variables, never implicit.

## Testing and validation

Before considering a change done, run what is available locally:

- `yamllint .`
- `ansible-lint`
- `molecule test -s default` when Docker access is available

Scenario checks belong in Ansible `molecule/*/verify.yml` playbooks. If a scenario cannot run, say so in the final
report.
