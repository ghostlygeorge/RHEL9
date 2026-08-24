# RHEL 9 CIS Benchmark Remediation (Ansible)

Automated hardening of RHEL 9 / Rocky 9 / Alma 9 hosts against the CIS Benchmark.
Controls are applied as idempotent tasks, gated by tags and per-rule variables so
you can roll out incrementally instead of all-at-once.

> **Read this first:** several controls can lock you out (SSH ciphers, firewall
> defaults, `pam` stacks, `noexec` on `/tmp`). Always run against a test host in
> `--check` mode before touching anything you care about.

---

## Requirements

```bash
ansible-core >= 2.15
python3 >= 3.9 on the control node
```

Install the collections and the role:

```bash
ansible-galaxy collection install ansible.posix community.general
ansible-galaxy install -r requirements.yml
```

`requirements.yml`:

```yaml
---
roles:
  - name: rhel9-cis
    src: https://github.com/ansible-lockdown/RHEL9-CIS.git
    scm: git
    version: main
```

---

## Layout

```
.
├── ansible.cfg
├── inventory.ini
├── requirements.yml
├── site.yml
└── group_vars/
    └── rhel9.yml
```

---

## Inventory — `inventory.ini`

```ini
[rhel9]
web01.example.com  ansible_host=10.10.20.11
web02.example.com  ansible_host=10.10.20.12
db01.example.com   ansible_host=10.10.30.21

[rhel9:vars]
ansible_user=ansible
ansible_become=true
ansible_become_method=sudo
ansible_python_interpreter=/usr/bin/python3

[workstations]
build01.example.com

[canary]
web01.example.com
```

The `canary` group is just a convenience target so you can pilot a change on a
single host with `--limit canary` before widening the blast radius.

---

## Config — `ansible.cfg`

```ini
[defaults]
inventory = ./inventory.ini
roles_path = ./roles
host_key_checking = True
retry_files_enabled = False
stdout_callback = yaml
callbacks_enabled = profile_tasks, timer
forks = 15
log_path = ./ansible.log

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=120s
```

---

## Playbook — `site.yml`

```yaml
---
- name: Remediate RHEL 9 hosts against the CIS Benchmark
  hosts: rhel9
  become: true
  any_errors_fatal: false
  serial: "25%"          # roll through the fleet in batches

  vars:
    # Profile selection
    rhel9cis_level_1_server: true
    rhel9cis_level_2_server: false
    rhel9cis_level_1_workstation: false

    # Behaviour
    rhel9cis_skip_reboot: true        # never reboot mid-run
    rhel9cis_selinux_disable: false   # keep SELinux enforcing
    rhel9cis_rule_1_1_1_1: true       # example: disable cramfs

    # Things that need a human decision — opt out and handle separately
    rhel9cis_notauto:
      - rule_5_2_20                   # sshd MaxStartups
      - rule_1_3_1                    # AIDE install/init

  pre_tasks:
    - name: Confirm we are on a supported platform
      ansible.builtin.assert:
        that:
          - ansible_facts['os_family'] == "RedHat"
          - ansible_facts['distribution_major_version'] == "9"
        fail_msg: "Host is not RHEL 9 — skipping."

    - name: Snapshot current sshd config
      ansible.builtin.copy:
        src: /etc/ssh/sshd_config
        dest: "/etc/ssh/sshd_config.pre-cis.{{ ansible_date_time.date }}"
        remote_src: true
        mode: "0600"
      changed_when: false

  roles:
    - role: rhel9-cis

  post_tasks:
    - name: Validate sshd config still parses
      ansible.builtin.command: sshd -t
      changed_when: false
```

---

## Usage

```bash
# 1. Dry run — show what would change, change nothing
ansible-playbook site.yml --check --diff --limit canary

# 2. Apply a single section (filesystem controls only)
ansible-playbook site.yml --tags section1 --limit canary

# 3. Apply one specific rule
ansible-playbook site.yml --tags rule_5_2_20

# 4. Apply Level 1 server profile, skipping controls known to break your apps
ansible-playbook site.yml --tags level1-server --skip-tags mount,firewalld

# 5. Full run across the fleet, batched
ansible-playbook site.yml

# 6. Vault-backed credentials
ansible-playbook site.yml --ask-become-pass --vault-id prod@prompt
```

Useful tag families exposed by the role:

| Tag              | Scope                                  |
|------------------|----------------------------------------|
| `level1-server`  | CIS Level 1, Server profile            |
| `level2-server`  | CIS Level 2, Server profile            |
| `section1`…`section6` | Benchmark sections               |
| `rule_x_y_z`     | A single control                       |
| `patch`          | Tasks that change state                |
| `audit`          | Read-only checks                       |

---

## Per-group overrides — `group_vars/rhel9.yml`

```yaml
---
# Disable controls that conflict with local requirements.
# Document the reason — auditors will ask.

rhel9cis_rule_1_1_2_1: false   # /tmp as separate partition — image-level decision
rhel9cis_rule_4_1_1_1: false   # auditd storage handled by central SIEM agent

rhel9cis_time_synchronization: chrony
rhel9cis_time_synchronization_servers:
  - ntp1.internal.example.com
  - ntp2.internal.example.com

rhel9cis_ssh_allow_groups: "sysadmins ansible"
rhel9cis_firewall: firewalld

rhel9cis_bootloader_password_hash: "{{ vault_grub_pbkdf2_hash }}"
```

---

## Suggested rollout

1. Run in `--check --diff` mode and capture the output as your gap report.
2. Pilot Level 1 on the `canary` group; soak for a change window.
3. Widen to the full group with `serial` batching.
4. Re-run with `--tags audit` to produce evidence of the post-state.
5. Only then evaluate Level 2 controls, which are materially more disruptive.

## Verifying independently

Don't grade your own homework — check the result with a separate scanner:

```bash
dnf install -y openscap-scanner scap-security-guide

oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis \
  --results /tmp/cis-results.xml \
  --report /tmp/cis-report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

## Rollback

The role does not ship an uninstall path. Practical options:

- Restore from the VM snapshot taken before the run.
- Flip the offending `rhel9cis_rule_*` variable to `false` and re-run — this
  stops re-application but does not revert an already-applied change.
- Restore the specific config from the `pre_tasks` backup and restart the unit.

Take a snapshot before the first run on any host that matters.
