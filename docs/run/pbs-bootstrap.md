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

This run book intentionally stops before pbs service configuration 
---

## Preconditions

The procedures described in pve-vm-create.md are complete without error.

# Manual PBS configuration steps
Prepare for running playbook pbs-bootstrap.yml:
- Launch PBS Web UI 
  - https://\<inventory-ip\>:8007

Login with the User name and Password assigned during installation.
Ignore the no-subscription warning for now.

## Step 1: Add No-Subscription repository

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

## Step 2: Install sudo package
```bash
# apt update && apt install sudo -y
```
## Step 3: Add ansible user

```bash
    adduser ansible
    usermod -aG sudo ansible
```

## Step 3: SSH Key Creation
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

Close web browser to pbs instance.

## Step 4: Execute pbs_bootstrap
From the ansible host
### Run for all PBS VMs
```bash
ansible-playbook playbooks/pbs_bootstrap.yml
```
Run for a single VM (recomended)
```bash
ansible-playbook playbooks/pbs-bootstrap.yml --limit pbsback
```
### Expected result 
All tasks complete with OK
Some tasks skipped
No failed tasks.

Stop and investigate failures.



## Step 5: Verify result
Manually verify configuration

1. Check ansible user hardning

From the ansible host:
```bash
# ssh ansible@<pbs_ip_address>
```
Expected result:
```bash
ansible@<pbs_ip_address>: Permission denied (publickey).
```

2. Check PBS configuration

- Launch PBS Web UI 
  - https://\<inventory-ip\>:8007

Login with the User name and Password assigned during installation.

- The no-subscription  warning should not appear
- Storage/Disks shows ZFS pool 
- Datastore name exists as defined in inventory

Verify operation of replication network
From pbs Shell
Ping of both gateways sorks from both PBS instances
```bash
# ping 172.16.100.1
```
```bash
# ping 172.16.150.1
```
Both of the above ping results must succeed

Exit the PBS UI

Login to the Graylog UI
- Filter for pbsfront or pbsback in input messages.
- Find messages from both.
- Error if no input messages



### Step 6:  Consistency check (recommended)
Run from Ansible control host
```bash
ansible-playbook playbooks/pbs-bootstrap.yml
```
Expected:
- No changes reported
Any difference is a stop condition.

Above assumes that both PBS instances are complete. Use --limit pbsfront or pbsback.

---

## Stop point
At this stage:
- VM exists
- PBS is installed
- Replication network is operating
- User hardning is complete
- Graylog reporting is operating 
- No services configured

When this run book is complete without error then
proceed to the pbs-services run book

---

## Notes

- This run book is authoritative
- Manual steps are intentional
- Do not “optimize” without updating this document
- Update run book as procedures evolve 
    - Ansible syntax
    - New PBS procedures with upgrade










