# Lab 8-1 — Generate /etc/hosts via Template

RHCE (EX294) practice exercise.

## Requirement

Write a playbook that generates an `/etc/hosts` file on all managed hosts,
containing every host defined in inventory.

## Key concept

A template task only has access to the facts of the hosts the *current play*
targets. If the play's `hosts:` scope is narrower than what the template
loops over (e.g. `hosts: web` but the template loops `groups['all']`), hosts
outside that scope have empty `hostvars` and the template renders blank
lines for them.


## Structure

- Play 1: `hosts: all`, `gather_facts: yes` — no tasks, just populates
  `hostvars` for every inventory host.
- Play 2: `hosts: all` — deploys `hosts.j2` to `/etc/hosts`, looping
  `groups['all']` and pulling `ansible_default_ipv4.address`,
  `ansible_fqdn`, and `ansible_hostname` from `hostvars[host]`.

The localhost/`::1` lines are kept at the top of the template — overwriting
`/etc/hosts` without them can break local name resolution on some systems.

## Files

- `lab8-1.yaml` — playbook
- `templates/hosts.j2` — Jinja2 template
