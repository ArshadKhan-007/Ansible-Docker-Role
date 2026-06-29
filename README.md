# Ansible Role: Automated Docker Installation on Ubuntu

Idempotent Ansible automation that installs and configures Docker on one or more Ubuntu servers using a reusable role.

[![Ansible](https://img.shields.io/badge/Ansible-2.1%2B-EE0000?logo=ansible&logoColor=white)](https://www.ansible.com/)
[![Platform](https://img.shields.io/badge/Platform-Ubuntu-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)

## What This Does

Running this playbook against a target host (or fleet of hosts) will:

- Update the APT package cache
- Install Docker (`docker.io`)
- Ensure the `docker` group exists
- Add a specified user to the `docker` group
- Start the Docker service and enable it on boot
- Push a custom `daemon.json` config and restart Docker only if that config actually changes

Re-running the playbook on an already-configured host is safe — idempotent by design, no unnecessary changes.

## Repository Structure

```text
Ansible-Docker-Role/
├── inventory.ini          # Target hosts
├── playbook.yml           # Entry point — invokes the docker role
└── docker/                # Ansible role
    ├── defaults/
    │   └── main.yml       # Default variables (lowest precedence, easily overridden)
    ├── handlers/
    │   └── main.yml       # Restart handler — runs only when notified
    ├── meta/
    │   └── main.yml       # Role metadata (Galaxy info)
    ├── tasks/
    │   └── main.yml       # Core automation logic
    ├── files/
    │   └── daemon.json    # Docker daemon config — copied to /etc/docker/daemon.json on target
    └── README.md          # Role-level documentation
```

> Note: the role lives at the repo root rather than inside a `roles/` directory. Ansible resolves this correctly because it also searches the playbook's own directory for roles — but if more roles get added later, moving `docker/` into `roles/docker/` is the more conventional layout.

## Requirements

- Ansible **2.1+** on the control node
- Target hosts running **Ubuntu** (uses the `apt` module — won't work as-is on RHEL/CentOS/Amazon Linux)
- SSH access to target hosts with a user that has (or can get) `sudo`/`become` privileges
- Python 3 on target hosts

## Role Variables

Defined in `docker/defaults/main.yml`, overridable per-host or per-group:

| Variable          | Default      | Description                              |
|--------------------|-------------|-------------------------------------------|
| `docker_packages`  | `docker.io` | Package to install                        |
| `docker_service`   | `docker`    | Service name to start/enable               |
| `docker_user`      | `ubuntu`    | User to add to the `docker` group          |

## Inventory Setup

Edit `inventory.ini` with your target hosts:

```ini
[docker_servers]
server1 ansible_host=<IP_ADDRESS> ansible_user=ubuntu
server2 ansible_host=<IP_ADDRESS> ansible_user=ubuntu

[all:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

Host key checking and other CLI behavior belongs in `ansible.cfg`, not in the inventory:

```ini
# ansible.cfg
[defaults]
host_key_checking = False
```

## Usage

```bash
# Dry run — see what would change without applying it
ansible-playbook -i inventory.ini playbook.yml --check

# Apply
ansible-playbook -i inventory.ini playbook.yml

# Override a default variable inline
ansible-playbook -i inventory.ini playbook.yml -e "docker_user=devops"
```

## Verification

```bash
docker --version            # Docker installed
systemctl status docker     # Service running
groups ubuntu                # User added to docker group
cat /etc/docker/daemon.json  # Custom daemon config applied
```

## Key Ansible Concepts Demonstrated

This project is small on purpose — but it touches enough core Ansible mechanics to be worth explaining properly for interview prep.

### 1. Roles

A role is Ansible's unit of reuse — a fixed directory convention (`tasks/`, `handlers/`, `defaults/`, `meta/`, `files/`) that packages automation logic so it can be dropped into any playbook.

**Q: Why not just write one big playbook?**
A: Doesn't scale past a handful of tasks. Roles isolate "install Docker" from "configure nginx" as independent, reusable units.

### 2. Tasks

Tasks are the ordered module calls that do the work — the key distinction is **declarative vs imperative**.

**Q: Is this playbook idempotent — how do you know?**
A: Yes — every task uses a proper module (`apt`, `user`, `group`, `service`, `copy`), not `shell`/`command`. Each one checks real system state before acting.

### 3. Defaults (Variables & Precedence)

`defaults/main.yml` sits at the **lowest** point in Ansible's variable precedence chain — deliberately, so it's easy to override.

**Q: Difference between `defaults/main.yml` and `vars/main.yml`?**
A: `defaults` = lowest precedence, meant to be overridden (group_vars, `-e`, etc). `vars` = high precedence, used for values not meant to be tuned by the caller.

### 4. Handlers

A handler only runs when explicitly `notify`-triggered by a task that reported `changed`:

```text
Copy daemon.json (changed: true) → notify: "Restart Docker" → Handler queued → Docker restarts at end of play
```

**Q: If three tasks notify the same handler, how many times does it run?**
A: Once — handlers are deduplicated and run once at the end of the play, not immediately on notify.

**Q: What if a task fails after notifying a handler?**
A: It won't run. Queued notifications are discarded if the play aborts before reaching the end.

### 5. Idempotency

Running this playbook 1 time or 100 times converges to the **same end state**, with no side effects on repeat runs.

**Q: Why does idempotency matter operationally?**
A: Safe to run on every deploy or cron schedule without worrying about double-installs or needlessly bouncing a running service.

### 6. Privilege Escalation (`become`)

`playbook.yml` sets `become: true` at the play level, since installing packages, managing services, and modifying `/etc/docker/daemon.json` all require root.

**Q: What does `become: true` do, and why is it needed here?**
A: Escalates privileges to root (via `sudo`) before running tasks. Needed because `apt`, `systemctl`, and writing to `/etc/docker/` all require root.

**Q: Difference between `become` and `become_user`?**
A: `become` turns escalation on (defaults to root). `become_user` picks a *different* target user instead of root.

**Q: `sudo` vs `su` as `become_method`?**
A: `sudo` needs sudo rights on the target user; `su` needs the root password directly. `sudo` is the modern default.

**Q: Why set `become` at play level instead of per-task?**
A: Every task here needs root, so setting it once at the play covers all of them — no repetition needed.

## Author

**Arshad Khan** — [GitHub: ArshadKhan-007](https://github.com/ArshadKhan-007)

## License

MIT
