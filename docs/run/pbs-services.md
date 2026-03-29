# Run Book: pbs-services

## Purpose

Configure PBS services on multiple PBS instances in a repeatable, auditable, low-risk manner.

All steps in the `pbs-bootstrap.md` run book must be comlete without error before starting this run book

This document is written for a future operator with **no prior context**.
Follow the steps exactly and in order.

Errors are not expected at any step.
Any deviation is a stop condition.

---

## Scope

This run book covers:

- Manual pre-flight steps: collect fingerprints, Gotify tokens, and verify vault passwords
- Execute the `pbs-service-config` playbook
- Post-run verification of all configured services

It intentionally stops before:
- Proxmox VE storage configuration (covered separately)
- Restore testing

---

## Preconditions

- `pbs-bootstrap.md` run book is complete without error
- Both PBS instances are reachable at:
  - https://10.0.0.109:8007 (pbsfront)
  - https://10.0.0.159:8007 (pbsback)
- Ansible vault file is accessible at `~/.ansible_vault_pass`
- Ansible control host is operational

---

## Step 1: Collect PBS Fingerprints

The sync job on pbsback authenticates to pbsfront using its TLS fingerprint. This value must be in the vault before the playbook runs.

### Get the pbsfront fingerprint

Login to pbsfront:
```
https://10.0.0.109:8007
```

Navigate to:
```
Datastore -> lalaland-backups -> Summary -> Show Connection Information
```

Copy the fingerprint. It looks like:
```
05:AB:...:xx
```

### Get the pbsback fingerprint

Login to pbsback:
```
https://10.0.0.159:8007
```

Navigate to:
```
Datastore -> secondary-backups -> Summary -> Show Connection Information
```

Copy the fingerprint.

### Write fingerprints to the vault

Open the vault file:
```bash
ansible-vault edit inventories/group_vars/pbs_hosts/vault.yml
```

Set the fingerprint values:
```yaml
vault_pbs_fingerprint:
  pbsfront: "05:AB:..."   # paste full fingerprint
  pbsback:  "c4:..."      # paste full fingerprint
```

Save and close.

Verify contents of the vault:
```bash
ansible-vault view inventories/group_vars/pbs_hosts/vault.yml
```

---

## Step 2: Create Gotify Applications and Collect Tokens

The playbook configures a Gotify notification endpoint on each PBS instance. Each instance needs its own application and token in Gotify.

Login to the Gotify UI at `http://10.0.0.6`.

Create two applications:

| Name | Description |
| --- | --- |
| Proxmox Backup Server Front - 109 | PBS instance on lala100 |
| Proxmox Backup Server Back - 159 | PBS instance on lala150 |

For each application, copy the generated token immediately. Gotify does not show it again.

### Write tokens to the vault

```bash
ansible-vault edit inventories/group_vars/pbs_hosts/vault.yml
```

Set the token values:
```yaml
vault_gotify_token_pbsfront: "XP..."
vault_gotify_token_pbsback:  "XX..."
```

Save and close.

---

## Step 3: Verify Vault Passwords

The playbook creates PBS users `backup@pbs` and `sync@pbs`. Their passwords must be in the vault before the playbook runs.

Open the vault and confirm these keys are present and set:

```bash
ansible-vault view inventories/group_vars/pbs_hosts/vault.yml
```

Confirm the following keys exist and are not empty or placeholder values:

```yaml
vault_pbs_user_password: "..."    # password for backup@pbs and sync@pbs
vault_pbs_fingerprint:
  pbsfront: "..."
  pbsback:  "..."
vault_gotify_token_pbsfront: "..."
vault_gotify_token_pbsback:  "..."
```

Stop if any value is missing or placeholder. Fix before proceeding.

---

## Step 4: Playbook Dry Run

Run the playbook in check mode first.

```bash
ansible-playbook playbooks/pbs-service-config.yml --check
```

### Expected result

- No fatal errors
- Tasks report intended changes
- No assertion failures

Any unexpected fatal error is a stop condition. Investigate before proceeding.

---

## Step 5: Execute the Playbook

### Run for all PBS hosts 

```bash
ansible-playbook playbooks/pbs-service-config.yml
```

### Run for a single host (recommended first time)

```bash
ansible-playbook playbooks/pbs-service-config.yml --limit pbsfront
ansible-playbook playbooks/pbs-service-config.yml --limit pbsback
```
Using --limit on first run controls the blast radius.

### Expected result

- All tasks complete with `ok` or `changed`
- No `failed` tasks
- No `unreachable` hosts

Stop and investigate any failure.

---

## Step 6: Verify Configuration

### 6.1 Verify datastore

On each PBS instance, confirm the datastore exists:

**pbsfront** — `https://10.0.0.109:8007`
- Datastore `lalaland-backups` is listed
- Backing path: `/mnt/datastore/dal`

**pbsback** — `https://10.0.0.159:8007`
- Datastore `secondary-backups` is listed
- Backing path: `/mnt/datastore/dfw`

### 6.2 Verify scheduled jobs

On each PBS instance, confirm the following jobs exist with the correct schedules:

| Job | Schedule | Location |
| --- | --- | --- |
| Verify | sat 18:15 | Datastore -> {datastore} -> Verify Jobs |
| GC | sun 18:15 | Datastore -> {datastore} -> GC Jobs |
| Prune | mon 14:00 | Datastore -> {datastore} -> Prune Jobs |
| Sync (pbsback only) | 02:00 daily | Datastore -> secondary-backups -> Sync Jobs |

Sync job does not appear on pbsfront. pbsfront is the sync source, not the puller.

### 6.3 Verify remotes

On **pbsback**, confirm the remote is configured:

```
Configuration -> Remotes
```

Expected:
- Remote ID: `pbsfront`
- Host: `172.16.100.5`
- Port: `8007`
- AuthID: `sync@pbs`
- Fingerprint: matches vault value

### 6.4 Verify users and ACLs

On **pbsfront**:
```
Configuration -> Access Control
```

Expected users:
- `backup@pbs` — role `DatastoreBackup` on `lalaland-backups`
- `sync@pbs` — role `DatastoreReader` on `lalaland-backups`

On **pbsback**:
- `sync@pbs` — role `RemoteSyncOperator` on `secondary-backups`

### 6.5 Verify Gotify notification

The scheduled jobs run at off-hours. To confirm Gotify is wired up without waiting:

**Trigger a manual GC run from the PBS UI:**

```
Datastore -> {datastore} -> GC Jobs -> Run Now
```

Then check the Gotify UI at `http://10.0.0.6` for a notification from the matching application.

If no notification arrives within 2 minutes:
- Confirm the Gotify endpoint is configured under `Configuration -> Notifications`
- Confirm `mail-to-root` is disabled
- Confirm `Gotify.006` is enabled in the default-matcher
- Check rsyslog is running: `systemctl status rsyslog`

On **pbsback**, confirm the sync notification match rule is also present:
```
Configuration -> Notifications -> Notification Matchers -> default-matcher
```
Match Rules should include `type=sync`.

---

## Step 7: Consistency Check

Re-run the playbook with no manual changes. The result must be idempotent.

```bash
ansible-playbook playbooks/pbs-service-config.yml
```

### Expected result

- All tasks report `ok`
- Zero `changed`

Any `changed` task is a stop condition. It means playbook state diverged from actual host state. Investigate before proceeding.

---

## Stop Point

At this stage:
- Datastores are created and registered
- Users and ACLs are configured
- Sync, verify, prune, and GC jobs are scheduled
- Gotify notifications are active
- Replication network sync path is confirmed

Proceed to the Proxmox VE storage configuration run book to add pbsfront as the PVE backup target.

---

## Notes

- Retention policy is managed on PBS, not PVE. Do not configure retention in the PVE backup job.
- The sync job on pbsback pulls from pbsfront over the replication network (172.16.x.x), not the management network.
- pbsfront does not have a sync job. Do not add one unless implementing reverse recovery pull.
- The `vault_pbs_user_password` applies to both `backup@pbs` and `sync@pbs`. Both use the same password as provisioned.

---

## Appendix: Validation Playbook Concept

A read-only validation playbook that queries each PBS host and compares actual configuration to inventory values is a reasonable addition to this project.

**What it would do:**

- Query each host using `proxmox-backup-manager` CLI commands
- Compare actual values to inventory variables
- Print actual vs. expected side by side for each check
- Exit non-zero on any mismatch

**What it would cover:**

| Check | Command |
| --- | --- |
| Datastore exists, correct path | `proxmox-backup-manager datastore list` |
| Users exist with correct roles | `proxmox-backup-manager acl list` |
| Verify job schedule | `proxmox-backup-manager verify-job list` |
| Prune job schedule and retention | `proxmox-backup-manager prune-job list` |
| GC job schedule | `proxmox-backup-manager garbage-collection-job list` |
| Sync job remote, schedule (pbsback) | `proxmox-backup-manager sync-job list` |
| Remote fingerprint (pbsback) | `proxmox-backup-manager remote list` |
| ZFS pool name, autotrim, compression | `zpool get all <pool>` |
| Hostname and FQDN | `hostname -f` |
| Replication network reachability | `ping -c1 -W2 <remote_gateway>` |
| Timezone | `timedatectl show -p Timezone --value` |

**Use cases:**

- Post-deployment verification (this run book, Step 7)
- After any manual change to confirm drift
- Periodic health check (monthly cron from the ansible host)
- Before and after a PBS upgrade

**Concerns:**

The PBS CLI outputs JSON with `--output-format json`, which makes parsing straightforward in Ansible. The main risk is CLI output schema changes across PBS versions. Pin the playbook to a tested PBS version and update it at upgrade time.

This is a reasonable next step after the service configuration playbook is stable.
