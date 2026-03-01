# Run Book: configure-ansible.md

## Purpose

Install and configure Ansible on the control host in a repeatable, auditable manner.
This is a one-time setup. Once complete, the control host is ready to run all project playbooks.

This document is written for a future operator with **no prior context**.
Follow the steps exactly and in order.

Errors are not expected at any step.
Any deviation is a stop condition.

---

## Scope

This run book covers:

- WSL2 and Ubuntu setup on Windows 11
- Ansible installation
- Project directory and configuration
- Inventory structure and validation
- Ansible collections installation
- Ansible Vault setup for secrets
- SSH key generation

It intentionally stops **before**:
- VM creation (covered in `pve-vm-create`)
- PBS bootstrap (covered in `pbs-bootstrap`)
- PBS service configuration (covered in future `pbs-configure` run book)

---

## Preconditions

- Windows 11 with internet access
- Homelab intranet access

---

## Step 1: Install WSL2 and Ubuntu

From PowerShell running as Administrator:

```powershell
wsl --install -d Ubuntu-22.04
```

Reboot when prompted. When the Ubuntu terminal opens, set a username and password.

All remaining steps run inside the WSL2 Ubuntu shell.

---

## Step 2: Install Ansible

```bash
sudo apt update && sudo apt install -y ansible
```

Verify:

```bash
ansible --version
```

Expected output (versions may differ):

```
ansible [core 2.17.14]
  config file = /home/ted/home-lab/ansible-pbs/ansible.cfg
  configured module search path = ['/home/ted/.ansible/plugins/modules', ...]
  python version = 3.10.12
  jinja version = 3.0.3
  libyaml = True
```

---

## Step 3: Create the Project Directory

If the project is already in a git repo:

```bash
git clone https://github.com/tuxhobo/ansible-pbs ~/home-lab/ansible-pbs
cd ~/home-lab/ansible-pbs
```

All subsequent steps assume the working directory is `~/home-lab/ansible-pbs`.


Reference layout:

```
├── LICENSE
├── README.md
├── ansible.cfg
├── collections
│   ├── ansible_collections
│   │   ├── community
│   │   └── community.general-12.4.0.info
│   │       └── GALAXY.yml
│   └── requirements.yml
├── docs
│   └── run
│       ├── configure-ansible.md
│       ├── pbs-bootstrap.md
│       └── pve-vm-create.md
├── inventories
│   ├── group_vars
│   │   ├── pbs_hosts
│   │   │   ├── all.yml
│   │   │   └── vars.yml
│   │   └── pve_hosts
│   │       ├── vars.yml
│   │       └── vault.yml
│   ├── host_vars
│   │   ├── lala100.yml
│   │   ├── lala150.yml
│   │   ├── pbsback.yml
│   │   └── pbsfront.yml
│   └── hosts.yml
├── pbs-context.txt
├── playbooks
│   ├── pbs-bootstrap.yml
│   ├── pbs-bootstrap.yml.sav
│   └── pve-vm-create.yml
└── roles
    ├── pbs_hardening
    │   ├── handlers
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── pbs_packages
    │   └── tasks
    │       └── main.yml
    ├── pbs_ping_test
    │   └── tasks
    │       └── main.yml
    ├── pbs_update
    │   ├── handlers
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── pbs_validate_identity
    │   └── tasks
    │       └── main.yml
    ├── pbs_zfs_datastore
    │   └── tasks
    │       └── main.yml
    └── pve-vm-create
        ├── README.md
        ├── defaults
        │   └── main.yml
        └── tasks
            └── main.yml

30 directories, 35 files
```

---

## Step 4: Create ansible.cfg

Verify `ansible.cfg` in the project root:

```bash
[defaults]
remote_user = ansible
private_key_file = /home/ted/.ssh/ansible_pbs_key
inventory = inventories
roles_path = roles
collections_path = ./collections:~/.ansible/collections:/usr/share/ansible/collections:/usr/lib/python3/dist-packages/ansible_collections

host_key_checking = True
retry_files_enabled = False
deprecation_warnings = False
interpreter_python = auto_silent

stdout_callback = default
result_format = yaml
bin_ansible_callbacks = True
timeout = 30
forks = 10

[inventory]
enable_plugins = host_list, script, auto, yaml, ini

[privilege_escalation]
become = False
become_method = sudo
become_ask_pass = True
become_flags = -H -S -E
```
---

## Step 5: Verify the inventory graph

```bash
ansible-inventory --graph
```

Expected:

```
@all:
  |--@ungrouped:
  |--@pve_hosts:
  |  |--lala100
  |  |--lala150
  |--@pbs_hosts:
  |  |--pbsfront
  |  |--pbsback
```

Any YAML errors or missing hosts are a stop condition. Fix before proceeding.

---

## Step 6: Verify Ansible Collections

```yaml
collections:
  - name: community.general
  - name: community.proxmox
```

Install:

```bash
ansible-galaxy collection install -r collections/requirements.yml
```

Collections install to `~/.ansible/collections`, which is included in `collections_path` in `ansible.cfg`.

Verify:

```bash
ansible-galaxy collection list | grep -E "community\.(general|proxmox)"
```

---

## Step 7: Generate the Ansible SSH Key

Generate the key pair on the control host. The key path must match `private_key_file` in `ansible.cfg`.

```bash
ssh-keygen -t ed25519 -C "ansible_pbs" -f ~/.ssh/ansible_pbs_key
```

Do not set a passphrase. Ansible requires unattended key-based auth.

Confirm the key exists:

```bash
ls -la ~/.ssh/ansible_pbs_key*
```

Expected:

```
~/.ssh/ansible_pbs_key
~/.ssh/ansible_pbs_key.pub
```

Key distribution to target hosts is covered in `pbs-bootstrap`.

---

## Stop Point

At this stage:
- Ansible is installed and configured
- Inventory is defined and validated
- Collections are installed
- Vault is initialized
- SSH key is generated on the control host

The control host is ready to run playbooks. Proceed to `pve-vm-create`.

---

## Notes

- This run book covers control host setup only
- Do not add VM, PBS, or service configuration steps here
- Update when the Ansible version, WSL distribution, or project structure changes