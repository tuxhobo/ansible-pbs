# Run Book: pve-backup-config

## Purpose

Configure the Proxmox VE cluster to use the primary PBS server as a backup
storage target and create a scheduled backup job for selected VMs and LXC
containers.

This run book is written for a future operator with **no prior context**.
Follow the steps exactly and in order.

Errors are not expected at any step.
Any deviation is a stop condition.

---

## Scope

This run book covers:

- Pre-flight inventory and vault verification
- Alignment check between duplicated inventory values
- Execution of `pve-backup-config.yml` playbook
- Visual verification in the PVE UI
- Immediate post-run validation tests
- Day-after validation after the first scheduled backup completes

This run book intentionally stops before:
- PBS retention tuning
- Notification configuration on the PVE side

---

## Prerequisites

The following run books must be completed in order, without error, before
proceeding:

1. `configure-ansible.md` — Ansible control host ready
2. `pve-vm-create.md` — PBS VMs created and PBS installed
3. `pbs-bootstrap.md` — PBS system configuration complete
4. `pbs-service-config.md` — PBS datastores, users, sync, and verify jobs configured

Do not proceed if any prior run book has an open error or unresolved stop condition.

---

## Pre-flight

### Step 1: Verify vault contents

View the PVE vault and confirm all three required secrets are present and
non-empty:

```bash
ansible-vault view ./inventories/group_vars/pve_hosts/vault.yml --ask-vault-pass
```

Confirm the following keys exist:

| Key | Notes |
| --- | --- |
| `vault_pve_api_token` | PVE API token secret. See note below. |
| `vault_pbs_user_password` | Password for `backup@pbs` on pbsfront. |
| `vault_pbs_fingerprint_pbsfront` | TLS fingerprint from pbsfront datastore summary. |

**`vault_pve_api_token` note**: PVE displays the token value only once at
creation. You cannot retrieve it from the PVE UI afterward. To verify the
stored token is still valid, run the following from the Ansible control host:

```bash
TOKEN=$(ansible-vault view ./inventories/group_vars/pve_hosts/vault.yml --ask-vault-pass \
  | grep vault_pve_api_token | awk '{print $2}' | tr -d '"')

curl -k -s -H 'Authorization: PVEAPIToken=ansible@pam!automation='"$TOKEN" \
 https://10.0.0.100:8006/api2/json/version | python3 -m json.tool

 unset TOKEN

```

Expected result: JSON response containing `"version"` and `"release"` fields.

If the response is `403` or empty `{"data": {}}`:
- The token is invalid or has been deleted from PVE.
- Recreate it as described in `pve-vm-create.md` under "Steps to create the
  PVE api-token".
- Update the vault with the new token value.

Do not proceed with an invalid token. The playbook will fail at every task.

---

### Step 2: Verify duplicated inventory values are aligned

Two inventory files contain values that must match each other:

| Value | Location 1 | Location 2 |
| --- | --- | --- |
| pbsfront IP | `hosts.yml` → `pbsfront.ansible_host` | `pve_hosts/all.yml` → `pve_cluster_storage.pbs_options.server` |
| Datastore name | `host_vars/pbsfront.yml` → `pbs_datastore.name` | `pve_hosts/all.yml` → `pve_cluster_storage.pbs_options.datastore` |
| PBS fingerprint | vault → `vault_pbs_fingerprint_pbsfront` | used in `pve_cluster_storage.pbs_options.fingerprint` |

Run the alignment check playbook to validate these automatically:

```bash
ansible-playbook ./playbooks/pve-backup-config.yml \
  --limit lala100 \
  --tags preflight \
  --ask-vault-pass
```

The preflight tag runs assertions only. No changes are made to any system.

Expected result: All assertions pass. Any failure identifies exactly which
value is mismatched and in which file.

If assertions fail: correct the inventory file indicated in the failure
message. Do not edit the vault to match a wrong inventory value — trace the
discrepancy to its source first.

---

### Step 3: Verify pbsfront is reachable

Confirm the PBS web UI is reachable from the control host before the playbook
attempts API calls against it:

```bash
curl -k -s -o /dev/null -w "%{http_code}\n" https://10.0.0.109:8007/
```

Expected: `200`

If unreachable: verify pbsfront VM is running on lala100 in the PVE UI before
proceeding.

---

## Execution

### Step 4: Run the playbook

```bash
ansible-playbook ./playbooks/pve-backup-config.yml \
  --limit lala100 \
  --ask-vault-pass
```

**Why `--limit lala100`**:
Storage targets and backup jobs are cluster-wide objects in PVE. Any cluster
node can receive the API call — the result is the same regardless of which
node is targeted. Running against both `lala100` and `lala150` is idempotent
but takes twice as long for no benefit. `lala100` is the primary node and the
host of pbsfront. It is the correct single target for this operation.

Expected result:
- `pve_cluster_storage` role: `changed` on first run, `ok` on subsequent runs
- `pve_backup_job` role: `changed` on first run, `ok` on subsequent runs
- No failed tasks

Stop and investigate any failure before proceeding to verification.

---

### Step 5: Consistency check

Run the playbook a second time immediately without making any changes.

```bash
ansible-playbook ./playbooks/pve-backup-config.yml \
  --limit lala100 \
  --ask-vault-pass
```

Expected result: All tasks report `ok`. No `changed` tasks.

Any `changed` result on the second run indicates the role is not idempotent.
Stop and investigate before treating the configuration as stable.

Note: `uri` module tasks do not support `--check` mode. A live second run is
the only way to verify idempotency for this playbook.

---

## Verification

### Step 6: Visual check in PVE UI

Open the PVE web UI at `https://10.0.0.100:8006`.

**Verify storage target:**
Datacenter → Storage

Confirm:
- Entry `cluster-backup-server` is present
- Type shows `PBS`
- Status is `active`
- Content shows `Backup`

Click `cluster-backup-server` → verify server IP and datastore name match
inventory values in `pve_hosts/all.yml`.

**Verify backup job:**
Datacenter → Backup

Confirm:
- A backup job with ID `pbs-daily` is present
- Storage shows `cluster-backup-server`
- Schedule shows `02:00`
- The VMID list matches `pve_backup_job.vmids` in inventory
- Mode shows `Snapshot`
- Enabled is checked

---

### Step 7: Immediate API validation

Confirm the storage target is registered and active via the PVE API directly:

```bash
TOKEN=$(ansible-vault view ./inventories/group_vars/pve_hosts/vault.yml --ask-vault-pass \
  | grep vault_pve_api_token | awk '{print $2}' | tr -d '"')
curl -k -s -H 'Authorization: PVEAPIToken=ansible@pam!automation='"$TOKEN" \
  https://10.0.0.100:8006/api2/json/storage/cluster-backup-server \
  | python3 -m json.tool
unset TOKEN
```

Expected: JSON response with `"type": "pbs"` and correct server and datastore values.

Confirm the backup job is registered:

```bash
TOKEN=$(ansible-vault view ./inventories/group_vars/pve_hosts/vault.yml --ask-vault-pass \
  | grep vault_pve_api_token | awk '{print $2}' | tr -d '"')
curl -k -s -H 'Authorization: PVEAPIToken=ansible@pam!automation='"$TOKEN" \
  https://10.0.0.100:8006/api2/json/cluster/backup \
  | python3 -m json.tool
```

Expected: JSON array containing an entry with `"id": "pbs-daily"` and
correct storage, schedule, and vmid values.

---

### Step 8: Trigger a manual test backup

Do not wait for the scheduled run to validate backup connectivity. Run one
manual backup immediately from the PVE UI:

Datacenter → lala100 →  104 (graylog104) → Backup → Run Now 

Do not run all VMIDs on the first
test — if something is wrong you want a small blast radius.

Monitor progress:
Datacenter → Backup → Job Log

Expected result:
- Job starts and completes without error
- No lock warnings in the log
- Duration is reasonable for the VM size

If the job fails: check the job log for the specific error. Common causes at
this stage are fingerprint mismatch, wrong password for `backup@pbs`, or
pbsfront datastore full or unavailable.

---

### Step 9: Verify backup appears in PBS

Login to pbsfront UI at `https://10.0.0.109:8007`.

Datastore → lalaland-backups → Content

Confirm:
- The test backup is present
- Backup type matches the VMID type (vm or ct)
- Size is non-zero
- Timestamp is current

---

## Day-after validation

After the first scheduled backup run completes (check after 02:00):

### Step 10: Confirm scheduled run completed

PVE UI → Datacenter → Backup → select `pbs-daily` → Job Log

Confirm:
- All VMIDs in the job list completed
- No failures or skipped entries
- No locked VM/LXC warnings

### Step 11: Confirm backups appear in PBS

pbsfront UI → Datastore → lalaland-backups → Content

Confirm:
- All backed-up VMIDs have entries with today's timestamp
- No entries are missing from the VMID list

### Step 12: Confirm sync to pbsback completed

pbsback UI at `https://10.0.0.159:8007`

Datastore → secondary-backups → Content

Confirm:
- Entries from the previous night's backup are present on pbsback
- Timestamps match pbsfront

The sync job runs at 02:00 daily. If backup and sync share the same schedule,
sync may not have run yet. Check the sync job log on pbsback:

Datastore → secondary-backups → Sync Jobs → Last Run

Confirm sync completed without error after the backup job finished.

### Step 13: Check Gotify notifications

Confirm notification delivery for the backup job run in the Gotify UI.

Expect notifications from:
- pbsfront: backup job completion
- pbsback: sync job completion (next day if sync runs after backup)

Missing notifications at this stage indicate a PVE-side notification
configuration gap, not a backup failure. Backup and notification are
independent.

---

## Stop point

At this stage:
- PBS storage target is registered with the PVE cluster
- Scheduled backup job is configured and verified
- First backup has been manually validated end-to-end
- Scheduled run has been confirmed
- Secondary copy on pbsback has been confirmed

The automated PBS backup system is operational.

---

## Notes

- This run book is authoritative
- Do not modify the playbook or inventory without updating this document
- Update when PVE or PBS versions change procedures
- `--limit lala100` is intentional — document any change to this decision