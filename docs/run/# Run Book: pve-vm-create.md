# Run Book: pve-vm-create

## Purpose

Create and install Proxmox Backup Server (PBS) virtual machines in a
repeatable, auditable, low-risk manner.

This document is written for a future operator with **no prior context**.
Follow the steps exactly and in order.

Errors are not expected at any step.
Any deviation is a stop condition.

---

## Scope

This run book covers:

- Inventory validation
- VM creation via Ansible
- Manual PBS installation
- Post-install validation

It intentionally stops **before**:
- system bootstrap
- PBS configuration
- datastore or service setup

---

## Preconditions

- Proxmox VE cluster is healthy
- Target nodes are reachable
- PBS ISO is already downloaded to Proxmox ISO storage
- Inventory changes are committed
- Playbooks run from the Ansible control host

---

## Step 1: Inventory validation

Validate inventory parsing and variable resolution.

```bash
ansible-inventory -i inventories/pve/hosts.yml --host lala100
ansible-inventory -i inventories/pve/hosts.yml --host lala150
```
### Expected result
- Commands succeed
- Host variables render cleanly
- PBS VM definitions are present
- Disk entries explicitly include:
```yaml
backup: false
```
### Stop if
- YAML errors
- Missing variables
- Unexpected defaults
Fix inventory before proceeding.

---

## Step 2: Playbook dry run (check mode)

Run the VM creation playbook in check mode.
```bash
ansible-playbook playbooks/pve-vm-create.yml --check
```

### Expected result
- No fatal errors
- Schema assertions pass
- Intended VM actions are shown

### Interpretation
- First execution: changes are expected
- Subsequent executions: no changes should appear

Any reported change on an existing VM is a signal.
Investigate before proceeding.

---

## Step 3: Execute VM creation
### Run for all PBS VMs
```bash
ansible-playbook playbooks/pve-vm-create.yml
```
Run for a single VM (recomended)
```bash
ansible-playbook playbooks/pve-vm-create.yml --limit lala100
```
### Expected result
- VM(s) created or confirmed present
- VMs are not automatically started
- Disk backup flags are disabled

---

## Step 4: Manual PBS installation
These steps are intentionally manual.

Precondition
Decide on the right version to download. Current version is 4.1-1.
[PBS Downloads](https://www.proxmox.com/en/downloads/proxmox-backup-server)

Download the ISO to each target host. In the Proxmox UI:
Datacenter->lala100->locak (lala100)-> ISO images->Download from URL

### 4.1 Attach PBS ISO
1. In the Proxmox UI:
2. Select the PBS VM
3. Hardware → Add → CD/DVD Drive
4. Storage: ISO-capable storage (e.g. local)
5. ISO image: PBS installer ISO
6. Confirm

---

### 4.2 Set boot order
1. VM → Options → Boot Order
2. Enable CD-ROM
3. Move CD-ROM to highest priority
4. Save

---

### 4.3 Install PBS
1. Start the VM
2. Open console
3. Follow the PBS installer

### Installer options (authoritative)
- Install Proxmox Backup Server
- Target disk: primary VM disk
- Filesystem: default
- Locale / keyboard: as documented
- Network:
    - Use assigned interface
    - Static IP per inventory
- Root password: vault-defined value
- Email: blank

Proceed until installation completes.

---

### 4.4 Post-install cleanup
After installation:
1. Shut down the VM
2. Remove CD/DVD Drive
3. Reset boot order to disk-only
4. Start the VM

---

## Step 5: Validation
### 5.1 Proxmox validation
- VM is running
- No backup jobs target this VM
- Disk backup is disabled

### 5.2 PBS validation
- Web UI reachable
- Correct hostname
- Correct IP configuration
- No dashboard errors

### 5.3 Consistency check (recommended)
Run from Ansible control host
```bash
ansible-playbook playbooks/pve-vm-create.yml --check
```
Expected:
- No changes reported
Any difference is a stop condition.

---

## Optional: Debug rendered VM configuration

If troubleshooting is required:
```bash
ansible-playbook playbooks/pve-vm-create.yml \
  -e pve_vm_debug=true
```

This prints the rendered disk and network maps
used by the Proxmox API.

---

## Stop point
At this stage:
- VM exists
- PBS is installed
- System is stable
- No services configured

When this run book is complete without error then
proceed to the pbs-botstrap run book

---

## Notes

- This run book is authoritative
- Manual steps are intentional
- Do not “optimize” without updating this document
- Update run book as procedures evolve 
    - Ansible syntax
    - New PVE procedures with upgrade
    - New PBS procedures with upgrade

