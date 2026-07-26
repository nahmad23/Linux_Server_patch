# Ubuntu Server Patching Playbook

An Ansible playbook to patch Ubuntu servers with **`apt upgrade`**, fix any
broken **dpkg** state, **reboot every server that was actually patched**, and
print a **final patching status summary** on screen.

Hosts are patched in **parallel batches of 50% at a time** (`serial`), not one
by one — so a fleet is patched in two waves while never taking the whole
estate down at once.

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
| `ansible.cfg`           | `forks = 50` so a 50% wave runs fully in parallel.   |

## What it does (per host)

1. **Fixes dpkg state** with `dpkg --configure -a` (auto-answers config-file
   prompts via `--force-confdef --force-confold`).
2. **Updates the apt cache — non-fatal.** Retries a few times; if it still
   fails, the run continues and patches with the existing cache instead of
   breaking.
3. Runs **`apt upgrade`** (`safe` upgrade by default; configurable).
4. **Autoremoves** unused/orphaned dependencies, then **autocleans** the cache
   (two separate steps so autoremove always runs).
5. **Mandatory reboot if the host was patched** (i.e. `apt upgrade` changed
   anything). Hosts with no updates are not rebooted.
6. Records SUCCESS / FAILED for the summary.

**Prompts are answered automatically.** Every step runs with
`DEBIAN_FRONTEND=noninteractive` and `force-confdef,force-confold`, so any
"yes/no" or "keep/replace config file?" question during `dpkg --configure -a`
or `apt upgrade` is answered without human input (keeps the current config).

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

### Batch size (parallelism)

By default 50% of the hosts are patched at a time. Change the batch size with
`patch_serial` (a percentage or a fixed count):

```bash
ansible-playbook -i inventory.ini patch-ubuntu.yml -e patch_serial=25%   # 4 waves
ansible-playbook -i inventory.ini patch-ubuntu.yml -e patch_serial=1     # one at a time
ansible-playbook -i inventory.ini patch-ubuntu.yml -e patch_serial=100%  # all at once
```

**Parallelism:** `serial` sets the wave size, but Ansible's default `forks = 5`
would still trickle 5 hosts at a time. `ansible.cfg` raises this to `forks = 50`
so the whole 50% wave runs at once. For a fleet larger than ~100 servers, bump
`forks` in `ansible.cfg` to at least half your total host count (or pass `-f`,
e.g. `-f 100`).

## Safety notes

- `--check` mode reports actions and skips the reboot.
- A failure on one host is caught (`rescue`) and reported in the summary
  instead of aborting the run, so you always get a full picture.
