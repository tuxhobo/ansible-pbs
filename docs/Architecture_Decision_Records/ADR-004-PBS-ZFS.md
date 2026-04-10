# ADR-004-PBS-ZFS: Use ZFS for the PBS Datastore Disk

## Status

Accepted

## Context

Each PBS VM has two disks: a 30 GiB boot disk and a 400 GiB datastore disk. The filesystem choice for the datastore disk must be made independently of the boot disk.

The datastore disk is a virtual disk backed by local-lvm SSD on the PVE host. The underlying hardware uses ECC memory, which provides protection against in-memory bit errors before data reaches disk.

PBS itself provides chunk-level checksumming and end-to-end integrity verification. The question is whether the filesystem layer beneath PBS should add additional integrity guarantees.

Options considered:

1. **ext4** — simple, well-understood, low overhead, no native checksumming
2. **ZFS (single-disk pool)** — block-level checksumming, compression, scrub capability, no redundancy at single disk

## Decision

Use ZFS for the PBS datastore disk, configured as a single-disk pool with compression enabled and autotrim enabled (SSD-backed storage).

ZFS pool configuration per instance:

| Parameter | Value |
|---|---|
| RAID level | Single disk (stripe) |
| ashift | 12 |
| Compression | on |
| Autotrim | on |

The boot disk remains ext4. Separating boot and data disks means a full datastore disk does not prevent PBS from continuing to operate.

## Consequences

**Positive:**
- Block-level checksums detect silent corruption (bit rot) before it propagates to the sync replica on pbsback
- ZFS scrubs provide periodic sweep of all data blocks independent of PBS verify jobs
- Compression reduces actual disk consumption on the datastore disk
- Autotrim maintains SSD performance over time
- Early corruption detection is particularly valuable here: corrupted data that reaches pbsback via sync is harder to recover from than corruption caught on pbsfront before sync

**Negative:**
- ZFS requires a separate disk from the boot disk — adds minor complexity to VM layout and PBS installation
- Single-disk ZFS pool provides no redundancy. Disk failure means datastore loss on that instance
- ZFS memory overhead is modest but nonzero — acceptable at 4 GiB RAM allocation

**What ZFS does not provide in this design:**
- Redundancy — single-disk pool, no mirror or raidz
- Disaster recovery — that role belongs to the PBS sync between hosts

**Accepted tradeoff:**
ZFS corruption detection before sync propagation justifies the added complexity of a separate datastore disk. The lack of redundancy is accepted — disk failure on one PBS instance is covered by the mirror copy on the other host. Both instances losing their datastore disk simultaneously is the same catastrophic scenario as both hosts failing, which is already acknowledged as an accepted risk.

**Revisit if:**
- The PVE host SSD is replaced with hardware that supports multiple physical disks, enabling a mirrored ZFS pool
- ZFS memory overhead becomes a constraint at the allocated VM memory size
