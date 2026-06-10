# Ubuntu Server Patching Playbook

An Ansible playbook to patch Ubuntu servers with **`apt upgrade`**, reboot them
**only when a reboot is actually required** (i.e. when `/var/run/reboot-required`
is present after the upgrade), and print a **final summary** at the end showing
which servers succeeded and which failed.

A failing host does **not** abort the whole run — its patching is wrapped in
`block`/`rescue`, the outcome is recorded per host, and every host is listed in
the summary regardless of result.

## Files

| File               | Purpose                                              |
| ------------------ | ---------------------------------------------------- |
| `patch-ubuntu.yml` | The patching playbook.                               |
| `inventory.ini`    | Example inventory. Replace with your real hosts.     |

## What it does

1. Asserts the target is a Debian/Ubuntu host.
2. Updates the apt cache.
3. Runs `apt upgrade` (`safe` upgrade by default; configurable).
4. Optionally autoremoves unused dependencies and cleans the cache.
5. Checks for `/var/run/reboot-required`.
6. Reboots and waits for the host to return — **only if required**.
7. Records SUCCESS/FAILED per host and prints a final summary play listing
   succeeded, failed, unreachable/skipped, and rebooted servers.

Example summary output:

```
================== PATCH SUMMARY ==================
SUCCESS              (2): ['web01.example.com', 'web02.example.com']
FAILED               (1): ['db01.example.com']
UNREACHABLE/SKIPPED  (0): []
REBOOTED             (1): ['web01.example.com']
==================================================
```

## Requirements

- Ansible 2.10+ on the control node.
- SSH access to targets with a user able to `sudo`.
- Python 3 on the targets.

## Usage

```bash
# Patch everything in the "ubuntu" group
ansible-playbook -i inventory.ini patch-ubuntu.yml

# Dry run — report what would change, without applying or rebooting
ansible-playbook -i inventory.ini patch-ubuntu.yml --check

# Limit to a subset
ansible-playbook -i inventory.ini patch-ubuntu.yml --limit web
```

## Tunable variables

Set these via `-e` on the command line or in your inventory/group vars:

| Variable               | Default  | Description                                       |
| ---------------------- | -------- | ------------------------------------------------- |
| `apt_upgrade_type`     | `safe`   | `safe`, `dist`, or `full`.                        |
| `apt_autoremove`       | `true`   | Remove unused dependency packages after upgrade.  |
| `apt_autoclean`        | `true`   | Clean obsolete packages from the local cache.     |
| `reboot_timeout`       | `600`    | Seconds to wait for a host to return after reboot.|
| `apt_cache_valid_time` | `3600`   | Skip cache update if newer than this many seconds.|

## Safety notes

- `--check` mode reports updates and skips the reboot.
- A failure on one host is caught (`rescue`) and reported in the summary
  instead of aborting the run, so you always get a full picture.
