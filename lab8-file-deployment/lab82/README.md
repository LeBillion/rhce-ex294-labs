# Lab 8-2 — Manage a vsftpd Service

RHCE (EX294) practice exercise.

## Requirements

- Install, start, and enable vsftpd; open the firewall for FTP.
- Templatize 4 settings in `/etc/vsftpd/vsftpd.conf`: `anonymous_enable`,
  `local_enable`, `write_enable`, `anon_upload_enable`. Leave everything
  else in the file unmodified.
- Set `/var/ftp/pub` to mode `0777`.
- Enable the `ftpd_anon_write` SELinux boolean.
- Set the `public_content_rw_t` SELinux context on `/var/ftp/pub`.

## Structure — two plays

1. **Install and enable vsftpd service** (`hosts: web`) — package, firewall,
   service start/enable. Pure infrastructure setup, no config content yet.
2. **Configure vsftpd service** (`hosts: web`) — directory/permissions,
   templated config, SELinux boolean + context. Application-level config,
   separated from the infrastructure play above.

## How the template was built

The template is **not** written from scratch. The approach:

1. Install vsftpd on a test host and read the real default file:
   `dnf install -y vsftpd && cat /etc/vsftpd/vsftpd.conf`.
2. Identify only the 4 lines the lab names.
3. Replace only those 4 values with `{{ variable }}` — everything else in
   the file stays untouched.
4. Define the variable values in `vars:` at the play level, **quoted**
   (`"YES"` not `yes`) — unquoted `yes`/`no` in YAML can get parsed as
   booleans and render wrong inside the template.

`anon_upload_enable` isn't present in the default file by default, so it's
added as a new line rather than modified in place.

## Notes on task ordering and gotchas

- The `/var/ftp/pub` directory task uses `state: directory` with `mode:
  '0777'` in one task — this both guarantees the directory exists (in case
  a minimal package build doesn't ship the sample dir) and sets its
  permissions, rather than assuming it already exists and only chmod'ing it.
- `sefcontext` only updates SELinux's *policy database* for the path
  pattern — it does not relabel files already on disk. A `restorecon`
  handler, notified by the `sefcontext` task, is required to actually
  apply the new context to `/var/ftp/pub`.
- The template task's `notify: restart vsftpd` only fires if the rendered
  file actually changed — config changes need a restart to take effect,
  which a plain `template` task alone won't do.
- Worth checking on a real target: is `firewalld` even installed/running?
  If not, the firewall task fails outright rather than silently no-op'ing.

## Files

- `lab8-2.yaml` — playbook
- `templates/vsftpd.conf.j2` — Jinja2 template (default file, 4 lines
  templatized)
