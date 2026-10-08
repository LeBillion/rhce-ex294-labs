# Lab 9-1 — Galaxy Role Version Pinning

RHCE (EX294) practice exercise.

## Requirements

- Use a requirements file to install Nginx via Galaxy role — pinned to the
  version *before* the latest, not latest itself.
- The same requirements file installs the latest version of the PostgreSQL
  Galaxy role.
- The playbook must ensure neither `httpd` nor `mysql`/`mariadb-server` is
  installed.

## Core concept: what a role actually is

A role is pre-written, generic task logic. Variables are the inputs you
supply to parameterize it — same relationship as a function and its
arguments. You never edit a role's task files directly; you read its
`README.md` / `defaults/main.yml` to learn what variables it accepts (its
"API"), then supply values via `vars:` in your own playbook. This lab
doesn't need custom vars, but the mental model matters for reading any
role's docs before using it.

## `requirements.yml` is a manifest, not a playbook

No `hosts:`/`tasks:` — just a declaration of roles/collections and the
versions needed. It's installed once, separately and *before* running the
actual playbook:

```
ansible-galaxy install -r requirements.yml
```

## Finding the right version — don't guess

```
ansible-galaxy role info geerlingguy.nginx
```
lists available versions. Pin to the second-newest listed, not a guessed
number — confirmed as `2.6.9` for this session via the command above. Note
that Galaxy role versions change over time, so re-verify with the same
command rather than assuming this pin still holds later.

## Execution order

Ansible always runs `pre_tasks → roles → tasks → post_tasks → handlers`,
regardless of the order these blocks appear visually in the playbook file.
`pre_tasks` here removes `httpd`/`mariadb-server` before the roles install
and configure Nginx/PostgreSQL.

## Hidden collection dependency

`geerlingguy.postgresql`'s internal tasks call `postgresql_user`, which
isn't bundled with the role itself. This doesn't show up as an install-time
warning — only as a runtime "couldn't resolve module/action" error.
Diagnostic loop when this happens:

1. Find the failing task file (role's `tasks/main.yml` or similar).
2. Find the exact module name it's calling.
3. Look it up with `ansible-doc -t module <name>` or the Ansible module
   index to find which collection owns it.
4. Add that collection to `requirements.yml`.

For this role, the missing piece is `community.postgresql`.

## Files

- `requirements.yml` — Galaxy manifest (roles + collection)
- `lab9-1.yaml` — playbook
