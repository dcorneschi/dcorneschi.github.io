# AIX PowerVM Virtualization Concepts

Conceptual overview of the IBM PowerVM I/O virtualization building blocks on Power/AIX — virtual SCSI, virtual Ethernet, the Shared Ethernet Adapter (SEA), and Integrated Virtual Ethernet (IVE). This is the "why and what" behind the commands in the [AIX VIOS Cheatsheet](articles/aix-vios-cheatsheet.md) and [AIX HMC Cheatsheet](articles/aix-hmc-cheatsheet.md).

> Some of these features require a **Virtual I/O Server (VIOS)** partition that owns the physical adapters and serves them to clients; others (virtual Ethernet, IVE) are provided directly by the hypervisor/hardware and need no VIOS.

## The Building Blocks at a Glance

| Feature | Needs VIOS? | Purpose |
|---------|-------------|---------|
| **Virtual SCSI** | Yes | VIOS backs client disks/optical from files, LVs, whole disks, or optical devices |
| **Virtual Ethernet** | No | In-system, hypervisor-provided networking between partitions |
| **Shared Ethernet Adapter (SEA)** | Yes | Layer-2 bridge from internal virtual Ethernet to the external physical network |
| **Integrated Virtual Ethernet (IVE)** | No | Hardware feature letting partitions reach the external network without a SEA |

## Virtual SCSI

Virtual SCSI lets partitions share storage without each owning a physical adapter. It requires a VIOS.

- The VIOS partition uses **files, logical volumes, whole disks, or optical devices** as the backing storage for the virtual SCSI devices it presents to clients.
- The **client** simply sees a standard SCSI device — an `hdisk#` or `cd#` — and can even **boot** from it.
- It is a **client/server** relationship: the VIOS owns the physical resource; the client sees standard SCSI devices; the **hypervisor** provides the inter-partition communication.

### Supported backing device types (VIOS 1.5 and later)

- Disk backed by a **physical volume**
- Disk backed by a **logical volume**
- Disk backed by a **file**
- Optical: DVD-ROM, DVD-RAM, CD-ROM
- Optical: DVD-RAM / DVD-ROM backed by files

## Virtual Ethernet

Virtual Ethernet lets partitions **on the same system** communicate without any physical Ethernet adapter — the connectivity is provided by the hypervisor, so **no VIOS is required**.

Characteristics and limits:

- A partition can have up to **65536 virtual adapter slots**, but the AIX 5.3/6.1 virtual Ethernet device driver controls at most **256 virtual Ethernet devices**.
- Large MTUs up to **65280 bytes** are supported.
- Interfaces can run both **IPv4 and IPv6**.
- By default a partition has **10 virtual adapter slots**; **two** are reserved for virtual serial adapters that implement the **virtual console** facility.
- The slot count can be raised (up to 65536), but increasing it requires **reactivating** the partition — it can't be changed dynamically.

### MAC addresses and VLANs

- The HMC generates a **locally administered** MAC for each virtual Ethernet adapter so it won't collide with physical adapter MACs.
- Uniqueness is derived from the **system serial number, LPAR ID, and virtual slot number**.
- Besides the port VLAN ID (**PVID**), each virtual Ethernet adapter can carry up to **20 additional VID** values — so a single adapter can reach **21 networks**.

## Shared Ethernet Adapter (SEA)

A SEA is a **Layer-2 bridge** running in the VIOS that connects the internal virtual Ethernet to an **external physical network**, so client partitions with only virtual Ethernet adapters can reach the outside world. It requires a VIOS (and is where SEA failover/load-sharing between two VIOS comes in — see the VIOS cheatsheet).

## Integrated Virtual Ethernet (IVE)

IVE (also called Host Ethernet Adapter, HEA) offers **similar function to virtual Ethernet** but as a **hardware** feature: partitions can communicate with the external physical network **without going through a SEA** (and therefore without a VIOS bridging their traffic).

## How They Fit Together

- Need shared **storage** for many LPARs from a few physical adapters → **Virtual SCSI** (VIOS) or NPIV.
- Need partitions to **talk to each other** inside the box → **Virtual Ethernet** (no VIOS).
- Need those virtual-only partitions to reach the **external LAN** → **SEA** (VIOS) or **IVE** (hardware).

## Related

- [AIX VIOS Cheatsheet](articles/aix-vios-cheatsheet.md) — the commands to create vSCSI/NPIV mappings and SEAs
- [AIX HMC Cheatsheet](articles/aix-hmc-cheatsheet.md) — creating virtual adapters and managing LPAR resources
- [AIX LVM Cheatsheet](articles/aix-lvm-cheatsheet.md) — logical volumes used as vSCSI backing storage
