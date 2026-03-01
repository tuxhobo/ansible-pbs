# 1. Problem statement

Previous backups were written directly from Proxmox VE to a TrueNAS‑hosted NFS share. Intermittent NFS failures caused backup jobs to fail and left VMs or containers locked on the Proxmox host, requiring manual intervention and skipping subsequent backups.

This design replaces NFS‑based backups with Proxmox Backup Server (PBS) to move failure handling from filesystem semantics to backup semantics.

---

## 1.1 Why Proxmox Backup Server (PBS) Instead of NFS

Using NFS as a Proxmox backup target introduces failure modes that Proxmox Backup Server is explicitly designed to avoid.
NFS-backed backups rely on:
- A remote filesystem
- Client-side locking
- Best-effort write semantics
- Minimal end-to-end verification

When an NFS interruption occurs during backup, Proxmox may leave VMs or containers locked and skip subsequent backups until manual intervention.

Proxmox Backup Server provides:
- Chunk-based, content-addressed storage
- End-to-end checksumming
- Atomic snapshot handling
- Built-in verification and garbage collection
- Network interruption tolerance without guest lock persistence

PBS moves failure handling from “filesystem semantics” to “backup semantics,” which is more reliable and observable. 

The negative arguments against using PBS for backup:
- Complexity of additional virtual appliance to manage just PVE backups
	- The PBS VMs consume, RAM, Disk and CPU resources of the PVE host x 2
- Separate backup strategies for TrueNAS ZFS backups and PVE backups
	- TrueNAS/ZFS uses snapsshots and replication while PBS is purpose built for PVE backups
- By design Proxmox does not provide a method to backup hosts
	- Requires scripting to recover the PVE hosts with subsiquent recovery of the PVE backups

The last point about recovering a PVE host is not a new problem. Virtualizing TrueNAS has simplfied its recovery but the PVE host recovery relies on a text based recovery procedure to stand up a replacement PVE host.
 
My Home Lab has the resources available within existing servers to support deployment of two PBS VM instances on separate servers. The advantages of PBS seem to outweigh the NFS solution. Having said this, the home lab can evolve again in the future should a better solution become available.

---

# 2. Design goals

- Eliminate NFS as a backup target for Proxmox VE
- Avoid VM/LXC lock persistence on backup failure
- Provide verifiable, testable backups
- Maintain a second independent copy of backups
- Keep the design simple enough for a home lab
- Ansible automation assists Proxmox Backup Server installation
- Virtualize the solution, use existing Proxmox hosts within Home Lab cluster 

---

# 3. Existing Home Lab
One architecture objective I follow in my Home Lab is to maximize virtualization and minimize additional equipment. This objective leads to interesting architecture decisions that a commercial data center with a fleet of servers would not face. A side effect for me is that I get to learn more about the internals of the software packages deployed!

##  3.1 Existing cluster nodes
The Proxmox cluster contains three nodes:
1. lala100 is a Dell R530 server supporting  
	- Primary TrueNAS VM  
	- Docker VM  
	- PlexMediaServer  
	- Graylog  
	- 3 more lxc containers  
2. lala008 is a low power Kamrui GK3 Pro MiniPC  
	- Home assistant  
	- PiHole (DNS server)  
	- tailscale-router  
3. lala110 is a Dell R720 server  
	- SecondaryTrueNAS VM

The applications listed above all perform nightly backups which will now target a PBS server VM.

---

# 4. Architecture
<img width="881" height="569" alt="image" src="https://github.com/user-attachments/assets/5cb9e7e2-49bd-4634-a604-dd4fc6659222" />

## 4.1 Physical and virtual layout

Two instances of Proxmox Backup Server (PBS) are deployed as Virtual Machines
-  lala100
	-  Primary Proxmox VE host
	-  Runs primary PBS VM (pbs, VMID 109)
- lala150
	- Secondary Proxmox VE host
	- Runs secondary PBS VM (pbs2, VMID 159)

## 4.2 Storage design
Each PBS instance is deployed as a virtual machine with:
- HDD scsi0 30G ext4 root file system and boot disk
- HDD scsi1 400G zfs pbs datastore

Both disk allocations are from the local-lvm ssd of the PVE host.

## 4.3 Network design

PBS traffic is split logically:
-  **Management network (vmbr0)** – UI access, administration
- **Replication network (vmbr1)** – PBS sync traffic

The replication network is a dedicated point to point link between PVE hosts  lala100 and lala150. This link is a bond of 3 x 1G ethernet ports wired directly between machines without switch fabric between. 

PBS‑to‑PBS sync uses a dedicated replication network to:
- Avoid saturating the management LAN
- Isolate large backup traffic
- Improve predictability and observability

## 4.4 Risks of PBS as a VM
This architecture places the primary PBS VM on the primary PVE server. The risk is that the PBS server will fail when the PVE host fails causing loss of backup images at the time they are most needed.

The architecture includes a backup PBS server on a separate PVE system that synchronizes the primary PBS server nightly.  The assumption is that a failure of both systems at the same time is unlikely and that the primary server can be reconstructed from the secondary backup images when needed. 

Recovery of the primary server means restoring backups from the secondary server.  Workable but not the best. It serves my Home Lab maxium to minimize the number of machines by virtualizing to the maximum extent possisble.

The complete disaster recovery plan includes a cold backup that makes a monthly copy of the entire system on a hard drive from cold storage.  

There is a future plan to arrange an additional remote storage drive located off site for additional data protection.  

## 4.5 Assumptions
- Single administrator
- Home lab scale (<20 VMs)
- Physical access to hardware
	- 3 PVE host servers
	- 1 Win 11 host BlueIris (plan to replace with Frigate on a Debian machine)
- No multi‑tenant requirements

## 4.6 Explicit tradeoffs

| Decision | Tradeoff       |
| ---- | ------------------- |
| Single primary datastore |	Simplicity vs flexibility |
| Single secondary datastore | Backup sync of primary datastore |
| No redundancy on PBS disk	| Lower cost vs availability, limited by existing hardware capability |
| PBS VM on PVE host | Potential loss of backup with PVE host. Recovery from backup PBS datastore |
| Weekly verify |	Less load vs slower detection. Relies on daily ZFS sweeps |
| Manual restore tests |	Time cost vs assurance. Up time is desirable but not mandatory.  |

---
# 5. Datastore and namespace strategy
## 5.1 Datastore layout
Each PBS instance hosts exactly one datastore located in a zfs pool :
| PBS  | Datastore name      |
| ---- | ------------------- |
| pbs  | `lalaland-backups`  |
| pbs2 | `secondary-backups` |

### 5.2 Namespace strategy

A **single namespace (root)** is used per datastore.

Rationale:

* Single‑tenant home lab
* Minimal operational overhead
* No need for per‑tenant ACLs

Future considerations that might justify namespaces:

* Multiple unrelated Proxmox clusters
* Different retention policies per workload
* Delegated access

Until then, namespaces add complexity without benefit.

---

# 6. VM sizing 
## 6.1 Current sizing

| Resource       | Value       |
| -------------- | ----------- |
| vCPU           | 2           |
| Memory         | 4 GiB       |
| Boot disk      | 30 GiB SSD  |
| Datastore disk | 400 GiB SSD |

***Sizing Justification***
* **Memory**: PBS uses RAM for chunk metadata, verification, and garbage collection. Community experience and Proxmox documentation indicate 2–4 GiB is sufficient for small environments, with 4 GiB providing headroom during verify and GC jobs.
* **CPU**: 2 vCPUs are sufficient for compression, chunk hashing, and parallel tasks at home‑lab scale.
* **Disk**: 400 GiB reflects current usage (~16%) with room for growth. This is expected to change and should be adjustable via automation. This drive is thin provisioned. The actual resource consumption expands as needed.

This sizing plan is intentionally conservative and optimized for stability over density.

Separating the root and data disks allows PBS to keep running even when the data disk is full — a fault tolerance decision independent of the ZFS choice for the data disk. Note that the underlying PVE host uses a single SSD for all VM logical volumes. If that SSD fails, every service on the host likely goes down too. The second PBS instance on a separate host guards against this: simultaneous failures on both hosts are unlikely.

---

# 7. ZFS justification

ZFS is used for the PBS datastore disk. It introduces some complexity because the boot disk and data disks are separate devices. The benefits outweigh the small increase in complexity.

What ZFS provides:

* End‑to‑end block checksums
* Early detection of silent corruption via scrubs
* Compression

What ZFS does **not** provide in this design:

* Redundancy (single‑disk pool)
* Disaster recovery

Primary data protection comes from PBS sync between hosts. ZFS protects against undetected corruption before corrupted data is propagated.
Note that data protection is further enhanced at the hardware level because the PVE host servers use Error Correcting Memory (ECC).

--- 

# 8. Scheduling and housekeeping
## 8.1 Verification

* Schedule: **Saturday 18:15**
* Scope: Entire datastore
* Skip already‑verified backups
* Re‑verify after 30 days

## 8.2 Garbage collection

* Schedule: **Sunday 18:15**
* Runs after verification to avoid I/O contention

## 8.3 Sync

* Direction: The backup or secondary PBS server is `pbsback` pulls from  the primary pbs server `pbsfront`
* Model: Pull‑based
* Schedule: **2am Daily**

## 8.4 Prune
* Both primary and secondary servers prune on the same shcedule
* Schedule: **Monday 2pm**

---

# 9. Trust and credentials

## 9.1 Users

| User         | Purpose                   |
| ------------ | ------------------------- |
| `backup@pbs` | Proxmox VE → PBS backups  |
| `sync@pbs`   | PBS → PBS synchronization |

## 9.2 Credential handling plan

* Credentials stored in Ansible Vault
* No interactive password reuse
* No root or system account access
* Rotation only on compromise

This is sufficient for a single‑admin home lab.

---

# 10. Run Books
I'm an engineer, not an IT professional. I use this home lab as a learning tool, and the implementation details of any deployment inevitably fade over time. These run books are written for future me — someone who needs to rebuild this system without remembering how it was originally put together.

*Run books:*

1. *configure_ansible.md* — Sets up Ansible on a WSL instance running on Windows 11.
2. *pve-vm-create.md* — Creates the PBS virtual machines on two PVE hosts and documents the manual PBS appliance installation steps.
3. *pbs-bootstrap.md* — Configures each PBS VM, including ZFS pool setup and network configuration.
4. *pbs-services.md* — Completes PBS instance configuration to meet the home lab's operational objectives.

Note: These run books mix manual and automated steps throughout.
