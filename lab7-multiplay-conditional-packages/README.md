# Multi-Play Ansible Lab - Conditional Package Install + Web Server Deployment

RHCE (EX294) practice exercise. Goal: use multiple plays sensibly, variable
inclusion, conditional looping, and error handling (`block`/`rescue`/`always`)
in a single playbook.

## Requirements

1. Install `httpd` and `mod_ssl` on host **ansible2** only.
2. Package names come from a separate vars file, looped over with a conditional.
3. Only install if the OS is CentOS or Red Hat (not Fedora), version 8.0+.
   Otherwise fail with: `Host <hostname> does not meet minimal requirements`.
4. On the control host, create `/tmp/index.html` containing "welcome to my webserver".
5. If that file copies successfully to `/var/www/html`, restart the web server.
   If the copy fails, show an error message.
6. Open the firewall for both `http` and `https`.

## Final structure — three plays

|Play| Hosts      | Purpose 
|    |            |    
| 1  | `localhost`| Create `/tmp/index.html` 
| 2  | `ansible2` | Conditional package install (vars file + `dict2items` loop) 
| 3  | `ansible2` | Copy file → restart service (`block`/`rescue`/`always`) → open firewall 

Files: `multiplaylab7.yaml`, `packages.yaml`.

Splitting into three plays instead of two matters here: package installation
and web-server deployment are logically separate concerns even though they
target the same host, and the requirement explicitly scopes the package
install to `ansible2` only — not the whole inventory.

## Bugs hit and fixes

- **`debug:`/`msg:` as siblings** instead of `msg:` nested under `debug:` →
  "conflicting action statements". `debug:` is the module; `msg:` is one of
  its parameters and belongs underneath it.
- **`yum:`/`fail:` as siblings on one task** — same conflicting-action error.
  A task can only run one module. Fixed by splitting into two tasks with
  complementary `when:` conditions (positive, then negated).
- **Negating the wrong sub-expression.** First attempt at the fail condition was:
  ```
  not (A or B) and C
  ```
  which is *not* the negation of `(A or B) and C` — it's `(not A and not B) and C`.
  A host that matched the OS check but failed the version check would then
  skip *both* branches silently instead of failing. Correct form negates the
  whole expression as one unit:
  ```
  not ( (A or B) and C )
  ```
- **`ansible_facts.ansible_hostname`** — invalid, mixes two different fact
  reference styles. Correct is either `ansible_facts.hostname` or the flat
  `ansible_hostname`, never both combined.
- **`"Red Hat"` vs `"RedHat"`** — the actual `ansible_facts.distribution`
  value has no space. Verify with `ansible -m setup -a "filter=ansible_distribution*"`
  instead of guessing.
- **`vars_files:` misunderstanding** — a vars file's top-level keys become
  the variables directly; there's no automatic variable named after the
  filename. Fixed by nesting package names under a `packages:` key so
  `packages | dict2items` had a real dict to loop over.
- **`firewalld`'s `service:` takes a single string, not a list** — needed
  `loop: [http, https]` to open both.
- **Ran with `-C` (check mode)** at one point — `copy:` doesn't actually
  create the file in check mode, so the next real task failed trying to read
  something that was never written. Good reminder that check mode doesn't
  chain realistically across tasks with file-existence dependencies.
- **Whole deploy block originally under `hosts: localhost`** — but httpd,
  `/var/www/html`, and the firewall all live on the web server host, not the
  control node. `copy:`'s `src:` always reads from the control node
  regardless of which hosts the play targets, which is what made this bug
  easy to miss at first (the source-file part worked fine).
- **`hosts: all` instead of `hosts: ansible2`** — the package-install play
  originally targeted every host in inventory. It didn't error, because
  ansible1 already had httpd installed from an earlier lab — a silent
  scope violation rather than a crash. Requirement said ansible2 only.

## Lessons for next time

- A play "passing" isn't proof of correctness — check the recap `changed=`
  counts against which hosts should have been affected at all.
- When negating a compound condition, negate the whole compound as a unit
  (`not (X and Y)`) rather than manually distributing the negation, unless
  you're confident in De Morgan's law under pressure.
- Always verify fact values (`ansible_facts.distribution`, etc.) against
  `ansible -m setup` output rather than assuming the human-readable OS name.
