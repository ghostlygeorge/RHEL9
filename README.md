# RHEL9-CIS Remediation

Runs a locally held RHEL9-CIS role against a set of hosts.

## Layout

```
.
├── inventory.ini
├── site.yml
└── roles/
    └── RHEL9-CIS/      # the role, as you already have it
```

Ansible picks up any role sitting in `./roles/` next to the playbook. If the role
lives somewhere else, pass `--roles-path` or set `roles_path` in `ansible.cfg`.

## Install

Only `sshpass` is needed, for the SSH password prompt:

```bash
sudo dnf install -y sshpass
```

## Inventory

`inventory.ini`

```ini
[rhel9]
192.168.10.11
192.168.10.12

[rhel9:vars]
ansible_user=sec_audit
ansible_become=true
ansible_become_method=sudo
```

No passwords in the inventory — they are prompted for at run time.

## Playbook

`site.yml`

```yaml
---
- name: Apply CIS hardening
  hosts: rhel9
  become: true
  vars:
    rhel9cis_level_1_server: true
    rhel9cis_skip_reboot: true
  roles:
    - RHEL9-CIS
```

The role name must match the directory name under `roles/` exactly — it is case-sensitive.

## Run

`-k` prompts for the SSH password, `-K` for the sudo password.

```bash
# check connectivity
ansible -i inventory.ini rhel9 -m ping -k -K

# dry run first
ansible-playbook -i inventory.ini site.yml -k -K --check --diff

# apply
ansible-playbook -i inventory.ini site.yml -k -K

# single host
ansible-playbook -i inventory.ini site.yml -k -K --limit 192.168.10.11

# one section only
ansible-playbook -i inventory.ini site.yml -k -K --tags section1
```

Both hosts must share the same `sec_audit` password — you are prompted once per run. If they differ, run them separately with `--limit`.

Reboot the hosts afterwards for kernel-level controls to take effect.
