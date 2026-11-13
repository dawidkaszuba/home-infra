# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Ansible playbooks and roles for managing the user's home server infrastructure: two hosts, `hp` (10.0.0.107) and `fujitsu` (10.0.0.110), defined in `inventory`.

## Common commands

Run playbooks from the repo root (inventory and vault password are picked up automatically via `ansible.cfg`):

```
ansible-playbook bootstrap.yml         # apt update + safe upgrade, all hosts
ansible-playbook fujitsu.yml           # applies rclone_mount role to fujitsu
ansible-playbook home_budget_backup.yml # applies postgres_backup role to fujitsu
```

Target a single host with `-l hp` or `-l fujitsu`. Use `--check --diff` to dry-run before applying.

Vault files (`group_vars/*/vault.yml`) are encrypted with `ansible-vault` using the password file at `~/.vault_pass.txt` (configured in `ansible.cfg`, not present in the repo):

```
ansible-vault encrypt group_vars/fujitsu/vault.yml
ansible-vault decrypt group_vars/fujitsu/vault.yml
ansible-vault edit group_vars/fujitsu/vault.yml
```

There is no lint/test tooling configured (no CI, no `ansible-lint` config). Sanity-check playbooks with `ansible-playbook --syntax-check <playbook>.yml` before applying.

SSH access uses the `dawid` remote user with the `ansible` key pair (`~/.ssh/ansible`); a host's `authorized_keys` is bootstrapped once via `ssh-copy-id -i ~/.ssh/ansible.pub <host-ip>` (see README.md).

## Architecture

- **Inventory/vars layering**: `group_vars/all.yml` applies to every host; `group_vars/<group>/vars.yml` holds non-secret group vars; `group_vars/<group>/vault.yml` holds the encrypted counterparts (e.g. `postgres_db_name`, `postgres_db_user`, `postgres_db_password`). Vars and vault files are split so non-secret config stays readable in plaintext diffs.
- **Playbooks are thin**: each top-level `*.yml` playbook just binds a host group to one role (e.g. `fujitsu.yml` → `rclone_mount`, `home_budget_backup.yml` → `postgres_backup`). Playbook naming reflects intent/purpose, not the role name — check the file itself to see which role it wires up.
- **Roles are self-contained** under `roles/<name>/{tasks,handlers,templates}`. No `defaults/` or `vars/` inside roles — all variables are supplied externally via `group_vars`.
- **`rclone_mount` role**: creates a dedicated `rclone_user`, installs an rclone config from `templates/rclone.conf.j2` (FTP remote), and mounts it via a generated systemd service (`rclone-ftp.service.j2`) with `--vfs-cache-mode writes --allow-other`. Enables `user_allow_other` in `/etc/fuse.conf`.
- **`postgres_backup` role**: installs a backup script (`postgres-backup.sh.j2`) run on a systemd timer (`postgres-backup.timer.j2`/`.service.j2`). The script `pg_dump`s the configured database, gzips it, pushes it to the rclone remote configured by `rclone_mount`, then prunes local backups older than `postgres_backup_keep_days`. Auth to Postgres is via `.pgpass` (`pgpass.j2`), not a CLI password. This role depends on `rclone_mount` having already configured the rclone remote/config file on the same host — apply `fujitsu.yml` before `home_budget_backup.yml` on a fresh host.
- Handlers in both roles just reload systemd (`daemon_reload: true`) after unit/template changes.

## Working with secrets

Never print decrypted vault contents into commits, logs, or command output that gets persisted. When editing a vault file, use `ansible-vault edit` rather than manually decrypting/re-encrypting.
