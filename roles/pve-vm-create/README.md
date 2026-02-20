# ./roles/pve-vm-create/tasks/README.md

Generic role to create Proxmox virtual machines.
Create and maintain Proxmox VE virtual machines from declarative inventory data.

This role is a thin, explicit wrapper around
community.general.proxmox_kvm.
- Inventory defines the VM 
- The role applies it.
   - Inventory passed directly to the module
- Nothing more.

The role is intentionally narrow in scope:
- VM definition and creation only
- No guest OS configuration
- No lifecycle automation beyond `state: present`

It is designed to be **predictable, idempotent, and auditable**.

---

## Design Principles

1. **Designed for community.general.proxmox_kvm**
    - Inventory expresses intent and policy
    - Inventory shape guided by community.general.proxmox_kvm
    - VM intent lives in inventory
    - No hidden defaults in tasks
    - No hidden defaults
    - No schema mutation inside the role
    - Compatible with check mode
    - Compatible with API token authentication

2. ## Required Inventory Structure
Each VM must define a dictionary compatible with `community.general.proxmox_kvm`.
- disks must be structured exactly as expected by the module.
- net must follow module syntax.

No transformation occurs in the role.

3. **Authentication Model**

The role supports API authentication
SSH is not used
API token must be first be created on the target
API token is stored in Ansible Vault

---

## vm_spec

`vm_spec` is the authoritative input contract passed from the playbook into the role.
Each VM must define a dictionary compatible with `community.general.proxmox_kvm`.
---

## Defaults

Cluster-wide defaults can be defined in pve_vm_defaults.
The role merges pve_vm_defaults + vm_spec.
vm_spec overrides defaults.
The merge is recursive.


---

## Idempotency

The role is written to be idempotent:

- Re-running the playbook should result in **no changes**
- Any reported change indicates:
  - inventory drift
  - UI/manual modification
  - or a Proxmox API behavior change

This is by design.

---

## What This Role Does
- Creates VM if missing
- Updates VM if configuration differs
- Supports check mode
- Passes parameters directly to the module

It does not manage:
- Cloud-init configuration
- Guest OS configuration
- Post-boot provisioning
- Network configuration inside the guest
- Storage pools
- ISO uploads

It strictly manages Proxmox VM hardware configuration.
---

## State Handling

Supported states:
```yaml
state: present
state: absent
```

If not defined, defaults to ```present```.

---

## Idempotency Notes
- Disk size changes are applied if supported by Proxmox.
- Certain hardware changes may require VM shutdown.
- The module does not automatically handle complex disk migrations.
- Inventory should reflect final desired state.

---

## Role Boundaries
This role:
- Defines VM hardware
- Attaches disks
- Attaches networks
- Enables guest agent
- Sets boot order
This role does not:
- Configure PBS
- Configure ZFS
- Configure networking inside the VM
- Perform OS installation

Those responsibilities belong in separate roles.

---

## Operational Guidance
- Keep VM definitions declarative.
- Avoid dynamic computation inside inventory.
- Prefer explicit disk definitions.
- Treat inventory as the single source of truth.
- Avoid mixing provisioning logic into this role.

