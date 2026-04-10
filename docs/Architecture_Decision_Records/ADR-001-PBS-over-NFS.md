# ADR-001-PBS-over-NFS: Use Proxmox Backup Server Instead of NFS as PVE Backup Target

## Status

Accepted

## Context

Proxmox VE backups were written directly to a TrueNAS-hosted NFS share. This worked until it didn't.

NFS interruptions during active backup jobs caused Proxmox to leave VMs and LXC containers in a locked state. Locked entities cannot be managed and subsequent backup jobs are skipped until the lock is manually cleared. The failure mode was silent — nothing in the job log indicated which backup failed or why — and recovery required manual intervention at the PVE console for each affected VMID.

The root cause is architectural. NFS exposes a remote filesystem to the Proxmox backup client. Backup semantics are layered on top of filesystem semantics. When the filesystem layer fails mid-operation, the backup layer has no recovery path and leaves state behind on the client.

The homelab has sufficient resources on existing servers to run two PBS VM instances without adding hardware.

## Decision

Replace NFS with Proxmox Backup Server (PBS) as the backup target for all PVE VMs and LXC containers.

PBS is purpose-built for PVE backups and handles network interruptions at the backup layer rather than the filesystem layer. Specifically:

- Chunk-based, content-addressed storage eliminates partial-write corruption
- Atomic snapshot handling prevents guest lock persistence on failure
- End-to-end checksumming detects corruption independently of the transport
- Built-in verification and garbage collection replace manual datastore hygiene
- Network interruption tolerance means a failed backup job does not leave the guest locked

## Consequences

**Positive:**
- VM and LXC lock persistence on backup failure is eliminated
- Backup integrity is verifiable via PBS verify jobs
- Deduplication reduces actual disk consumption below the sum of snapshot sizes
- Operational visibility improves — PBS job logs are specific about failure cause

**Negative:**
- Two additional VMs to manage (PBS instances on each PVE host)
- PBS consumes RAM, CPU, and disk on each PVE host
- Two separate backup strategies now exist: PBS for PVE workloads, ZFS snapshots and replication for TrueNAS
- PVE host recovery remains a manual text-based procedure. PBS does not back up PVE hosts, only the VMs and containers running on them. This is not a new problem — it existed with NFS as well

**Accepted tradeoff:**
The operational complexity of two PBS VMs is justified by eliminating the manual intervention requirement on backup failure. The homelab has available resources. The NFS failure mode was unpredictable and required hands-on recovery that outweighed the simplicity of the NFS approach.

**Revisit if:**
- A simpler solution emerges that provides equivalent backup semantics without the additional VM overhead
- Resource pressure on PVE hosts makes the PBS VM footprint untenable
