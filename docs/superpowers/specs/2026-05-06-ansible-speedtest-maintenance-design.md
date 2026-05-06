# Ansible: Speedtest Monitor + System Maintenance

**Date:** 2026-05-06
**Target host:** 192.168.1.99 (Debian 12 minimal)
**Ansible user:** martin (sudo via SSH key)

## Overview

Two independent Ansible playbooks to configure a Debian 12 server:
1. `speedtest.yml` — deploys a Python internet speed monitor from GitHub and schedules it every 5 minutes
2. `maintenance.yml` — creates a full system update script and schedules it weekly

## File Structure

```
ansible/
├── inventory.ini
├── ansible.cfg
├── speedtest.yml
└── maintenance.yml
```

## inventory.ini

Defines a single host group `servers` with:
- Host: `192.168.1.99`
- `ansible_user=martin`
- `ansible_ssh_private_key_file` pointing to the local SSH private key

## ansible.cfg

- `inventory = inventory.ini`
- `host_key_checking = False`
- `remote_user = martin`

## speedtest.yml

Tasks (in order):

1. **Install system packages** (become: yes): `git`, `python3`, `python3-pip`, `python3-venv` via `apt`
2. **Clone repo** (become: no): `https://github.com/martin-rizzi/speedtest-monitor` → `/home/martin/speedtest-monitor/`
3. **Create virtualenv and install dependencies** (become: no): virtualenv at `/home/martin/speedtest-monitor/.venv/`, install from `requirements.txt`
4. **Configure cron** (become: no): runs every 5 minutes as user martin

Cron entry:
```
*/5 * * * * /home/martin/speedtest-monitor/.venv/bin/python /home/martin/speedtest-monitor/speedtest_monitor.py
```

Results are saved by the script itself to `/home/martin/speedtest-monitor/resultados.csv`.

## maintenance.yml

Tasks (in order):

1. **Create update script** (become: yes): `/usr/local/bin/update_system.sh`, permissions 755, owned by root

Script content:
```bash
#!/bin/bash
apt-get update -y
apt-get upgrade -y
apt-get dist-upgrade -y
apt-get autoremove -y
apt-get autoclean -y
```

2. **Configure weekly cron** (become: yes): runs as root every Sunday at 03:00 AM, output logged to `/var/log/update_system.log`

Cron entry:
```
0 3 * * 0 /usr/local/bin/update_system.sh >> /var/log/update_system.log 2>&1
```

## How to Run

```bash
# Deploy speedtest monitor
ansible-playbook speedtest.yml --ask-become-pass

# Deploy system maintenance
ansible-playbook maintenance.yml --ask-become-pass
```

## Privilege Escalation

`become: yes` is scoped only to tasks that require it:
- Installing apt packages
- Creating `/usr/local/bin/update_system.sh`
- Configuring the root cron job

Tasks for the martin user (git clone, virtualenv, personal cron) run without escalation.
