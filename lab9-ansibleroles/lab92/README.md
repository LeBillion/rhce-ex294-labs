# Lab 9-2 — RHEL SELinux System Role

RHCE (EX294) practice exercise.

## Requirements

- A boolean is set to allow SELinux relabeling to be automated via cron.
- `/var/ftp/uploads` is created, permissions set to `0777`, context label
  set to `public_content_rw_t`.
- SELinux allows web servers to use port 82 instead of port 80.
- SELinux is in enforcing state.

## Key concept: system roles are narrow by design

`rhel-system-roles.selinux` manages SELinux state only — booleans, file
contexts, port contexts, enforcing mode. It does **not** create directories
or set filesystem permissions. That's why `/var/ftp/uploads` is created via
a plain `file` module `pre_task` before the role applies its context — the
role would otherwise have nothing to label, or would apply context to a
directory that doesn't exist yet in the intended state.

## Verifying values instead of guessing

- `getsebool -a | grep -i relabel` — confirms the actual boolean name
  (`auto_relabel_on_cron`) rather than assuming it.
- `semanage port -l | grep http` — confirms `http_port_t` already covers
  port 80, so adding port 82 to the same type is consistent with existing
  policy rather than inventing a new type.

## Variables used

- `selinux_state` — enforcing/permissive/disabled
- `selinux_booleans` — list of `{name, state}` dicts
- `selinux_fcontexts` — list of `{target, setype}` dicts
- `selinux_ports` — list of `{ports, proto, setype}` dicts

These are the role's "API" — read from its `README.md`/`defaults/main.yml`,
not invented from memory.

## Files

- `lab9-2.yaml` — playbook
