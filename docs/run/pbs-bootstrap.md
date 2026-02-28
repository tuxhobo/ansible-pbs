# run book pbs-bootstrap

# Purpose
Perform initial configuration of a freshly installed Proxmox Backup Server (PBS) virtual machines in a repeatable, auditable, low-risk manner.

This run book follows the steps in pve-vm-install.md. Follow the steps in this run book only after the procedures in that run book are complete without error.

This document is written for a future operator with **no prior context**.
Follow the steps exactly and in order.

Errors are not expected at any step.
Any deviation is a stop condition.

## Scope

This run book covers:
- Manual setup preparing for Ansible playbook operation
    - Add no-subscription repository
    - install sudo 
    - Add ansible user
    - Configure SSH keys
    - Test SSH operation for Ansible
- Run ansible playbook pbs-bootstrap
    - Harden user access rights
    - Install needed packages
    - ZFS pool configuration
    - Replication network configuration
    - Configure Graylog reporting
- Post run validation of PBS configuration

This run book intentionally stops before pbs servuce configuration 
---

## Preconditions

The procedures described in pve-vm-create.md are complete without error.

# Manual PBS configuration steps
Prepare for running playbook pbs-bootstrap.yml

## Add No-Subscription repository

From the PBS UI
Administration->Repositories  
Highlight https://enterprise.proxmox.com/debian/pbs  
click: disable

Next click the Add button  
Select No-Subscription repository  
Click add

Go to the PBS shell as root user and run upgrade:
```bash
root@host:~# apt update && apt full-upgrade
```

## SSH Key Creation
- On the ansible control host create an ssh key exclusively for ansible
```bash
# ssh-keygen -t ed25519 -f ~/.ssh/ansible_pbs_key
```
  - skip ssh-keygen step if the key exists already

- Push the key to the PBS server
```bash
ssh-copy-id -i ~/.ssh/ansible_pbs_key.pub ansible@<pbs-ip>
```
Verify the key functions:
```bash
# ansible <pbs_host> -m ping
```
Use the inventory pbs_host names: pbsfront or pbsback
Expect the following result:
```bash
<pbs_host> | SUCCESS => {
    "changed": false,
    "ping": "pong"
}

```

## Install sudo package

```bash
# apt update && apt install sudo -y
```


