# Ubuntu Server Patching Playbook

An Ansible playbook to patch Ubuntu servers with **`apt upgrade`**, fix any
broken **dpkg** state, **reboot every server that was actually patched**, and
**email a final patching status report**.

A failing host does **not** abort the whole run — its patching is wrapped in
`block`/`rescue`, the outcome is recorded per host, and every host is listed in
the report regardless of result.

> **Note on architecture:** Ansible is agentless and push-based. Run this from
> a single **control node**; the target servers only need SSH access and
> Python 3. Do **not** clone this repo onto the servers being patched, and do
> not list the control node itself in the `[ubuntu]` group (rebooting it would
> kill the run).

## Files

| File               | Purpose                                              |
| ------------------ | ---------------------------------------------------- |
| `patch-ubuntu.yml` | The patching playbook.                               |
| `inventory.ini`    | Example inventory. Replace with your real hosts.     |

## What it does (per host)

1. **Fixes dpkg state** with `dpkg --configure -a`.
2. Updates the apt cache.
3. Runs **`apt upgrade`** (`safe` upgrade by default; configurable).
4. Autoremoves unused dependencies and cleans the cache.
5. **Mandatory reboot if the host was patched** (i.e. `apt upgrade` changed
   anything). Hosts with no updates are not rebooted.
6. Records SUCCESS / FAILED for the report.

Then a final play (runs once) **prints and emails** the report:

```
Ubuntu patching report - 2026-06-20

SUCCESS  (2): web01, web02
FAILED   (1): db01
SKIPPED  (0): none
REBOOTED (2): web01, web02
```

The email is sent to **nawazish.ahmad@unitedlex.com** (configurable).

## Requirements

- Ansible 2.10+ on the control node.
- The `community.general` collection (for the email step):
  ```bash
  ansible-galaxy collection install community.general
  ```
- A reachable SMTP relay (set `smtp_host` / `smtp_port` in the playbook).
- SSH access to targets with a user able to `sudo`; Python 3 on the targets.

## Usage

```bash
# Patch everything in the "ubuntu" group
ansible-playbook -i inventory.ini patch-ubuntu.yml

# Dry run — reports actions, skips reboot and email
ansible-playbook -i inventory.ini patch-ubuntu.yml --check

# Limit to a subset
ansible-playbook -i inventory.ini patch-ubuntu.yml --limit web

# If sudo needs a password
ansible-playbook -i inventory.ini patch-ubuntu.yml --ask-become-pass
```

## Tunable variables

Set these via `-e` on the command line, or edit them in the playbook:

| Variable           | Default                          | Description                                  |
| ------------------ | -------------------------------- | -------------------------------------------- |
| `apt_upgrade_type` | `safe`                           | `safe`, `dist`, or `full`.                   |
| `reboot_timeout`   | `600`                            | Seconds to wait for a host after reboot.     |
| `mail_to`          | `nawazish.ahmad@unitedlex.com`   | Report recipient.                            |
| `mail_from`        | `ansible-patching@unitedlex.com` | Sender address.                              |
| `smtp_host`        | `localhost`                      | SMTP relay host.                             |
| `smtp_port`        | `25`                             | SMTP relay port.                             |

## Safety notes

- `--check` mode reports actions and skips both the reboot and the email.
- A failure on one host is caught (`rescue`) and reported instead of aborting
  the run, so you always get a full report and email.
