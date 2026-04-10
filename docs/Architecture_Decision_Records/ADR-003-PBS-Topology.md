# ADR-003-PBS-Topology: Primary/Secondary Mirror Topology for PBS Instances

## Status

Accepted

## Context

With two PBS instances deployed on separate hosts (see ADR-PBS-as-VMs), a sync relationship between them must be defined. Two topologies were evaluated:

**Option A: Primary/secondary mirror**
One PBS instance (pbsfront on lala100) receives all PVE backups from the entire cluster. The second instance (pbsback on lala150) performs a nightly pull sync from pbsfront. pbsback holds a replica of the entire primary datastore.

**Option B: Cross-backup topology**
Each PBS instance is the backup target for VMs on the opposite PVE host. pbsfront (on lala100) receives backups from lala150 VMs. pbsback (on lala150) receives backups from lala100 VMs. Each instance also holds a sync replica of its mate's datastore. Each host's recovery image lives on the opposite host.

Option B was raised as a potential improvement after Option A was already implemented and operational. The argument for Option B is that single-host failure recovery is more direct: lala100 fails, its backups are already on lala150's PBS instance without needing to navigate a mirror copy.

## Decision

Retain the primary/secondary mirror topology (Option A).

The cross-backup topology (Option B) was evaluated and rejected for the following reasons:

**Recovery outcome is equivalent at homelab scale.** When lala100 fails, Option A requires pointing PVE at pbsback and restoring from the sync copy. Option B would point PVE at pbsback and restore from the primary copy. The operational difference is one configuration step, not a fundamentally different recovery procedure. With a single administrator and no recovery time contractual obligation, this difference does not justify a refactor.

**Option B adds inventory and operational complexity.** Two datastores per PBS instance, split backup job targets per VMID, and bidirectional sync relationships increase the Ansible inventory surface and the number of jobs to monitor. Option A is simpler to reason about and maintain.

**The existing recovery runbook covers the Option A gap.** The identified weakness — no explicit runbook for "lala100 is dead, restore from pbsback" — is addressed by documentation, not by topology change. A topology refactor to close a runbook gap is the wrong tool.

**Operational risk of refactoring an operational system.** Option B requires renaming datastores, restructuring Ansible inventory, reconfiguring PVE backup jobs cluster-wide, and re-syncing data. The risk of that refactor outweighs the marginal recovery improvement at this scale.

## Consequences

**Positive:**
- Single backup target for the entire PVE cluster — simpler PVE configuration
- Single sync relationship — one job to monitor on pbsback
- Ansible inventory remains straightforward — one datastore per host
- No disruption to operational system

**Negative:**
- Single-host failure recovery requires an extra manual step to redirect PVE storage target to pbsback
- The backup of lala100's VMs is on lala100's PBS instance — co-location risk acknowledged in ADR-PBS-as-VMs applies here

**Accepted tradeoff:**
Operational simplicity over recovery locality. The mirror copy on a separate host is sufficient protection. The extra manual step on recovery is an acceptable cost at homelab scale.

**Revisit if:**
- The cluster grows to a scale where split backup targets per host become operationally justified
- Actual recovery experience reveals that the extra redirection step causes meaningful delay or error
- A PBS version introduces cluster-aware backup routing that makes Option B easier to manage
