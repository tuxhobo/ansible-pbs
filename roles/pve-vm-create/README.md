# pve-vm-create

Generic role to create Proxmox virtual machines.
Create and maintain Proxmox VE virtual machines from declarative inventory data.

This role is intentionally narrow in scope:
- VM definition and creation only
- No guest OS configuration
- No lifecycle automation beyond `state: present`

It is designed to be **predictable, idempotent, and auditable**.

---

## Design Principles

1. **Declarative input**
    - Inventory expresses intent and policy
    - VM intent lives in inventory
    - Playbook performs translation
    - Role enforces execution and defaults
    - No hidden defaults in tasks

2. **Explicit contracts**
   - The role consumes a single variable: `vm_spec`
   - Structure is validated before execution

3. **Separation of concerns**
   - Inventory: project intent
   - Playbook: mapping and normalization
   - Role: rendering and execution
   - Defaults: system-level assumptions

4. **Fail fast**
   - Schema violations stop execution early
   - UI drift is treated as configuration error

---

## vm_spec

`vm_spec` is the authoritative input contract passed from the playbook into the role.

It is **not** read from inventory directly.

See:
docs/vm_spec.schema.yml

for the full authoritative schema.

### Key characteristics

- Ordered disks and networks
- Explicit disk `backup` flag (required)
- Optional overrides via `vm_spec.options`
- All assumptions centralized in `vm_defaults`

---

## Defaults

System-level defaults live in:
defaults/main.yml


These represent:
- Proxmox-wide conventions
- Safe assumptions
- Settings that rarely change

Inventory overrides should be **intentional and minimal**.

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

## Debugging

Optional debug output is available:

```bash
ansible-playbook playbooks/pve-vm-create.yml \
  -e pve_vm_debug=true


