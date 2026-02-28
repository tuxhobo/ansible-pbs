# run book: configure ansible

```yaml
[defaults]
remote_user = ansible
private_key_file = /home/ted/.ssh/ansible_pbs_key
inventory = inventories
roles_path = roles
collections_path = ~/.ansible/collections:/usr/share/ansible/collections

host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
timeout = 30

interpreter_python = auto_silent
forks = 10

[inventory]
enable_plugins = yaml

[privilege_escalation]
# become = true
# become_method = sudo
# Sbecome_ask_pass = false
```

The inventory path is configured for the pve-vm-create playbook.
May need to adjust this for other playbook operation.

# Configuring collections
Create:
```bash
./collections/requirements.yml
```
With the following content:
```yaml
collections:
- name: community.general
- name: community.proxmox
```

Then run:
```bash
ansible-galaxy collection install -r collections/requirements.yml
```

This command shall populate:
```bash
~/.ansible/collections
```

Ansible version at this time:
```bash
# ~/home-lab/ansible-pbs$ ansible --version
ansible [core 2.17.14]
  config file = /home/ted/home-lab/ansible-pbs/ansible.cfg
  configured module search path = ['/home/ted/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/ted/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.10.12 (main, Jan 26 2026, 14:55:28) [GCC 11.4.0] (/usr/bin/python3)
  jinja version = 3.0.3
  libyaml = True
```

Inventory should look like this:
```bash
# ~/home-lab/ansible-pbs$ ansible-inventory --graph
@all:
  |--@ungrouped:
  |--@pve_hosts:
  |  |--lala100
  |  |--lala150
  |--@pbs_hosts:
  |  |--pbsfront
  |  |--pbsback
```

