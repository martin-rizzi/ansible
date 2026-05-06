# Ansible Speedtest + Maintenance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Configurar un servidor Debian 12 en 192.168.1.99 con Ansible para ejecutar un monitor de velocidad de internet cada 5 minutos y un script de actualización de paquetes semanal.

**Architecture:** Dos playbooks independientes (`speedtest.yml` y `maintenance.yml`) con inventario y config compartidos. El speedtest corre en un virtualenv Python bajo el usuario martin; el mantenimiento corre como root vía cron.

**Tech Stack:** Ansible, Python 3 (venv), apt, cron, GitHub

---

## File Map

| Archivo | Acción | Responsabilidad |
|---|---|---|
| `ansible.cfg` | Crear | Config global: inventario, usuario, host_key_checking |
| `inventory.ini` | Crear | Define el host 192.168.1.99 y sus vars de conexión |
| `speedtest.yml` | Crear | Instala deps, clona repo, crea venv, configura cron |
| `maintenance.yml` | Crear | Crea script de update, configura cron semanal root |

---

### Task 1: Configuración base (ansible.cfg + inventory.ini)

**Files:**
- Create: `ansible.cfg`
- Create: `inventory.ini`

- [ ] **Step 1: Crear ansible.cfg**

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = martin
```

- [ ] **Step 2: Crear inventory.ini**

```ini
[servers]
192.168.1.99

[servers:vars]
ansible_user=martin
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

> Si tu clave SSH tiene otro nombre (ej. `id_ed25519`), ajustá el path.

- [ ] **Step 3: Verificar conectividad con el host**

```bash
ansible servers -m ping
```

Salida esperada:
```
192.168.1.99 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Si falla con "permission denied", verificar que la clave pública esté en `~/.ssh/authorized_keys` del servidor.

- [ ] **Step 4: Commit**

```bash
git init
git add ansible.cfg inventory.ini
git commit -m "feat: add ansible base config and inventory"
```

---

### Task 2: Playbook speedtest.yml

**Files:**
- Create: `speedtest.yml`

- [ ] **Step 1: Crear speedtest.yml**

```yaml
---
- name: Deploy speedtest monitor
  hosts: servers
  vars:
    repo_url: https://github.com/martin-rizzi/speedtest-monitor
    repo_dest: /home/martin/speedtest-monitor
    venv_path: /home/martin/speedtest-monitor/.venv

  tasks:
    - name: Install system packages
      become: yes
      apt:
        name:
          - git
          - python3
          - python3-pip
          - python3-venv
        state: present
        update_cache: yes

    - name: Clone speedtest repository
      become: no
      git:
        repo: "{{ repo_url }}"
        dest: "{{ repo_dest }}"
        version: main
        force: no

    - name: Install Python dependencies in virtualenv
      become: no
      pip:
        requirements: "{{ repo_dest }}/requirements.txt"
        virtualenv: "{{ venv_path }}"
        virtualenv_command: python3 -m venv

    - name: Configure cron job for speedtest (every 5 minutes)
      become: no
      cron:
        name: "speedtest monitor"
        minute: "*/5"
        job: "{{ venv_path }}/bin/python {{ repo_dest }}/speedtest_monitor.py"
        user: martin
        state: present
```

- [ ] **Step 2: Validar sintaxis**

```bash
ansible-playbook speedtest.yml --syntax-check
```

Salida esperada:
```
playbook: speedtest.yml
```
(sin errores)

- [ ] **Step 3: Dry run para verificar cambios planeados**

```bash
ansible-playbook speedtest.yml --check --ask-become-pass
```

Revisar que las 4 tareas aparezcan como `changed` o `ok` sin errores. Si alguna falla en `--check`, corregir antes de continuar.

- [ ] **Step 4: Ejecutar el playbook**

```bash
ansible-playbook speedtest.yml --ask-become-pass
```

Salida esperada: todas las tareas en `ok` o `changed`, `failed=0`.

- [ ] **Step 5: Verificar en el servidor**

```bash
ssh martin@192.168.1.99 "crontab -l"
```

Salida esperada (entre otras líneas):
```
*/5 * * * * /home/martin/speedtest-monitor/.venv/bin/python /home/martin/speedtest-monitor/speedtest_monitor.py
```

```bash
ssh martin@192.168.1.99 "ls /home/martin/speedtest-monitor/"
```

Debe listar `speedtest_monitor.py`, `requirements.txt`, `.venv/`, etc.

- [ ] **Step 6: Commit**

```bash
git add speedtest.yml
git commit -m "feat: add speedtest monitor playbook"
```

---

### Task 3: Playbook maintenance.yml

**Files:**
- Create: `maintenance.yml`

- [ ] **Step 1: Crear maintenance.yml**

```yaml
---
- name: Configure system maintenance
  hosts: servers

  tasks:
    - name: Create system update script
      become: yes
      copy:
        dest: /usr/local/bin/update_system.sh
        mode: '0755'
        owner: root
        group: root
        content: |
          #!/bin/bash
          export DEBIAN_FRONTEND=noninteractive
          apt-get update -y
          apt-get upgrade -y
          apt-get dist-upgrade -y
          apt-get autoremove -y
          apt-get autoclean -y

    - name: Configure weekly cron job for system update (Sunday 03:00)
      become: yes
      cron:
        name: "weekly system update"
        minute: "0"
        hour: "3"
        weekday: "0"
        job: "/usr/local/bin/update_system.sh >> /var/log/update_system.log 2>&1"
        user: root
        state: present
```

- [ ] **Step 2: Validar sintaxis**

```bash
ansible-playbook maintenance.yml --syntax-check
```

Salida esperada:
```
playbook: maintenance.yml
```
(sin errores)

- [ ] **Step 3: Dry run**

```bash
ansible-playbook maintenance.yml --check --ask-become-pass
```

Verificar que las 2 tareas aparezcan sin errores.

- [ ] **Step 4: Ejecutar el playbook**

```bash
ansible-playbook maintenance.yml --ask-become-pass
```

Salida esperada: `failed=0`.

- [ ] **Step 5: Verificar en el servidor**

```bash
ssh martin@192.168.1.99 "sudo crontab -l"
```

Salida esperada (entre otras líneas):
```
0 3 * * 0 /usr/local/bin/update_system.sh >> /var/log/update_system.log 2>&1
```

```bash
ssh martin@192.168.1.99 "cat /usr/local/bin/update_system.sh"
```

Debe mostrar el script con los 5 comandos apt.

- [ ] **Step 6: Commit**

```bash
git add maintenance.yml
git commit -m "feat: add weekly system maintenance playbook"
```

---

### Task 4: Verificación end-to-end

- [ ] **Step 1: Correr el speedtest manualmente para verificar que funciona**

```bash
ssh martin@192.168.1.99 "/home/martin/speedtest-monitor/.venv/bin/python /home/martin/speedtest-monitor/speedtest_monitor.py"
```

Salida esperada: el script corre sin errores. Luego verificar que el CSV se generó:

```bash
ssh martin@192.168.1.99 "cat /home/martin/speedtest-monitor/resultados.csv"
```

Debe mostrar al menos una fila con datos de velocidad.

- [ ] **Step 2: Correr el script de update manualmente para verificar permisos**

```bash
ssh martin@192.168.1.99 "sudo /usr/local/bin/update_system.sh"
```

Salida esperada: apt-get corre sin errores (o dice "nothing to upgrade"). El log queda en `/var/log/update_system.log`.

- [ ] **Step 3: Re-ejecutar ambos playbooks para verificar idempotencia**

```bash
ansible-playbook speedtest.yml --ask-become-pass
ansible-playbook maintenance.yml --ask-become-pass
```

Salida esperada: `changed=0` en todas las tareas (nada cambia porque ya está configurado).

- [ ] **Step 4: Commit final**

```bash
git add docs/
git commit -m "docs: add design spec and implementation plan"
```
