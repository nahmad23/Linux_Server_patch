# Ubuntu Server Patching Playbook

An Ansible playbook to patch Ubuntu servers with **`apt upgrade`**, fix any
broken **dpkg** state, **reboot every server that was actually patched**, and
print a **final patching status summary** on screen.

A failing host does **not** abort the whole run — its patching is wrapped in
`block`/`rescue`, the outcome is recorded per host, and every host is listed in
the summary regardless of result.

> **Note on architecture:** Ansible is agentless and push-based. Run this from
> a single **control node**; the target servers only need SSH access and
> Python 3. Do **not** clone this repo onto the servers being patched, and do
> not list the control node itself in the `[ubuntu]` group (rebooting it would
> kill the run).

## Files

| File                    | Purpose                                              |
| ----------------------- | ---------------------------------------------------- |
| `patch-ubuntu.yml`      | The patching playbook (you don't need to edit this). |
| `inventory.ini`         | Example inventory. Replace with your real hosts.     |
| `group_vars/ubuntu.yml` | **Settings** — upgrade type, reboot timeout.         |

## What it does (per host)

1. **Fixes dpkg state** with `dpkg --configure -a`.
2. Updates the apt cache.
3. Runs **`apt upgrade`** (`safe` upgrade by default; configurable).
4. Autoremoves unused dependencies and cleans the cache.
5. **Mandatory reboot if the host was patched** (i.e. `apt upgrade` changed
   anything). Hosts with no updates are not rebooted.
6. Records SUCCESS / FAILED for the summary.

Then a final play (runs once) **prints the summary**:

```
================== PATCH SUMMARY ==================
SUCCESS              (2): ['web01', 'web02']
FAILED               (1): ['db01']
UNREACHABLE/SKIPPED  (0): []
REBOOTED             (2): ['web01', 'web02']
==================================================
```

## Requirements

- Ansible 2.10+ on the control node.
- SSH access to targets with a user able to `sudo`; Python 3 on the targets.

## Usage

```bash
# Patch everything in the "ubuntu" group
ansible-playbook -i inventory.ini patch-ubuntu.yml

# Dry run — reports actions, skips the reboot
ansible-playbook -i inventory.ini patch-ubuntu.yml --check

# Limit to a subset
ansible-playbook -i inventory.ini patch-ubuntu.yml --limit web

# If sudo needs a password
ansible-playbook -i inventory.ini patch-ubuntu.yml --ask-become-pass
```

## Settings

Edit **`group_vars/ubuntu.yml`** — no need to touch the playbook. These apply to
every host in the `[ubuntu]` group and can also be overridden per run with `-e`.

| Variable           | Default | Description                              |
| ------------------ | ------- | ---------------------------------------- |
| `apt_upgrade_type` | `safe`  | `safe`, `dist`, or `full`.               |
| `reboot_timeout`   | `600`   | Seconds to wait for a host after reboot. |

Override for a single run, for example:

```bash
ansible-playbook -i inventory.ini patch-ubuntu.yml -e apt_upgrade_type=dist
```

## Safety notes

- `--check` mode reports actions and skips the reboot.
- A failure on one host is caught (`rescue`) and reported in the summary
  instead of aborting the run, so you always get a full picture.
