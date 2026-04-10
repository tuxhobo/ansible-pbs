# ADR-002-PBS-as-VMs: Deploy PBS as Virtual Machines on Existing PVE Hosts

## Status

Accepted

## Context

Having decided to use PBS as the backup target (see ADR-PBS-over-NFS), the deployment model must be chosen. The options are:

1. Dedicated physical hardware running PBS bare metal
2. PBS deployed as VMs on existing PVE hosts

The homelab architecture principle is to maximize virtualization and minimize the number of physical machines. Three PVE hosts already exist: lala100 (Dell R530), lala150 (Dell R720), and lala008 (Kamrui GK3 Pro MiniPC). Both lala100 and lala150 have sufficient resources to host a PBS VM.

The fundamental risk of co-locating a PBS instance on a PVE host is that a failure of the PVE host also takes down the PBS instance running on it — precisely when backup access is most needed. This risk must be explicitly acknowledged and mitigated.

## Decision

Deploy PBS as virtual machines on existing PVE hosts. One PBS instance per host, on two separate hosts (lala100 and lala150).

VM specification per instance:
- 2 vCPU (1 socket, 2 cores, host CPU type)
- 4 GiB RAM
- 30 GiB boot disk (ext4, scsi0, local-lvm)
- 400 GiB datastore disk (ZFS, scsi1, local-lvm)
- Machine type: q35
- QEMU guest agent enabled
- Start on boot enabled

The two instances run on separate physical hosts. Simultaneous failure of both hosts is the only scenario that leaves both PBS instances unavailable at the same time.

## Consequences

**Positive:**
- No additional hardware required
- PBS VMs are disposable — they can be rebuilt from run books and Ansible automation
- Virtualization provides consistent provisioning via Ansible
- Existing infrastructure, networking, and management tooling applies without modification

**Negative:**
- Primary PBS instance on lala100 is unavailable if lala100 fails — exactly when lala100's VM backups are needed most
- PBS VMs consume host resources: RAM, CPU, and SSD capacity on local-lvm
- Both disk allocations (boot and datastore) draw from the same physical SSD on the PVE host. If that SSD fails, both disks fail together

**Mitigation:**
The secondary PBS instance on lala150 holds a nightly sync copy of the primary datastore. Recovery of lala100 means pointing PVE at the lala150 PBS instance, restoring VMs from the sync copy, then rebuilding the lala100 PBS instance via Ansible and re-syncing. This is documented in the recovery run books.

A monthly cold backup to removable storage provides an additional recovery path independent of both PBS instances.

**Accepted tradeoff:**
Co-location risk is accepted in exchange for avoiding additional hardware. The mitigation — a second PBS instance on a separate host — is sufficient for homelab scale where simultaneous dual-host failure is unlikely and recovery time is not contractually bounded.

**Revisit if:**
- A dedicated low-power physical host becomes available that justifies bare-metal PBS deployment
- Resource pressure on PVE hosts makes PBS VM footprint untenable
- Simultaneous host failure occurs and the recovery path proves inadequate
