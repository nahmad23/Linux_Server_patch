# Ubuntu Server Patching Playbook

An Ansible playbook to patch Ubuntu servers and reboot them **only when a
reboot is actually required** (i.e. when `/var/run/reboot-required` is present
after the upgrade, typically following kernel or core library updates).

## Files

| File               | Purpose                                              |
| ------------------ | ---------------------------------------------------- |
| `patch-ubuntu.yml` | The patching playbook.                               |
| `inventory.ini`    | Example inventory. Replace with your real hosts.     |

## What it does

1. Asserts the target is a Debian/Ubuntu host.
2. Updates the apt cache.
3. Lists and reports upgradable packages.
4. Upgrades all packages (`dist` upgrade by default).
5. Optionally autoremoves unused dependencies and cleans the cache.
6. Checks for `/var/run/reboot-required`.
7. Reboots and waits for the host to return — **only if required**.

Hosts are patched in batches (`serial: 25%` by default) so the entire fleet
never reboots simultaneously.

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

# Patch one host at a time
ansible-playbook -i inventory.ini patch-ubuntu.yml -e patch_serial=1
```

## Tunable variables

Set these via `-e` on the command line or in your inventory/group vars:

| Variable               | Default  | Description                                       |
| ---------------------- | -------- | ------------------------------------------------- |
| `apt_upgrade_type`     | `dist`   | `dist`, `full`, or `safe`.                        |
| `apt_autoremove`       | `true`   | Remove unused dependency packages after upgrade.  |
| `apt_autoclean`        | `true`   | Clean obsolete packages from the local cache.     |
| `reboot_timeout`       | `600`    | Seconds to wait for a host to return after reboot.|
| `apt_cache_valid_time` | `3600`   | Skip cache update if newer than this many seconds.|
| `patch_serial`         | `25%`    | Batch size for rolling patching.                  |

## Safety notes

- `--check` mode reports updates and skips the reboot.
- Rolling batches limit blast radius. For database or stateful tiers,
  consider `-e patch_serial=1` and patch outside peak hours.
