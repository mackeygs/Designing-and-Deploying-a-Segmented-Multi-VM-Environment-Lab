# Virtual Lab: Designing and Deploying a Segmented Multi-VM Environment

![VirtualBox](https://img.shields.io/badge/VirtualBox-7.2.16-183A61?logo=virtualbox&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04.1%20LTS-E95420?logo=ubuntu&logoColor=white)
![Windows 11](https://img.shields.io/badge/Windows%2011-25H2-0078D4?logo=windows11&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Pi--hole-2496ED?logo=docker&logoColor=white)

A homelab project in which I designed and built a small virtual infrastructure on a single Windows host using Oracle VirtualBox. The environment has a Linux server, a Windows workstation, and an isolated Linux test system on a segmented internal network, with the isolated machine reachable only through a jump box. I then used it to test the day-to-day operations that make virtualization valuable: resizing a VM's resources, cloning a VM for rapid provisioning, rolling back changes with snapshots, and looking at how virtual storage actually works on disk. As a bonus, I deployed Pi-hole in a Docker container on the host to compare containers with full VMs.

The project is framed around a fictional company, **BlueSky Analytics**, that needs a flexible, low-cost environment for hosting core services, simulating end-user workstations, and safely testing applications.

---

## Table of Contents

- [Highlights](#highlights)
- [Environment](#environment)
- [Architecture](#architecture)
- [Deploying the Virtual Machines](#deploying-the-virtual-machines)
- [Network Segmentation](#network-segmentation)
- [Scaling a VM's Resources](#scaling-a-vms-resources)
- [Rapid Provisioning with Cloning](#rapid-provisioning-with-cloning)
- [Snapshots and Recovery](#snapshots-and-recovery)
- [How the Storage Works](#how-the-storage-works)
- [Bonus: Pi-hole in Docker](#bonus-pi-hole-in-docker)
- [Challenges and Troubleshooting](#challenges-and-troubleshooting)
- [What I Learned](#what-i-learned)
- [Recommendations for Growth](#recommendations-for-growth)

---

## Highlights

Four VMs run on one physical machine, each with its own static IP on the `192.168.10.0/24` subnet. The test system is deliberately cut off from the internet and can only be reached by SSH through the server acting as a jump box, which I confirmed with ping and SSH tests. I upgraded the Windows client from 4000 MB RAM, 2 vCPUs, and 60 GB of disk to 5000 MB, 3 vCPUs, and 70 GB without rebuilding it. I cloned that client into a fully independent fourth VM in a few minutes, compared with the thirty minutes to an hour a fresh install would take. Restoring a snapshot rolled back a deliberate change in one to two minutes. Every disk is dynamically allocated, so the 70 GB client disk uses only about 43 GB on the host.

---

## Environment

| Component | Details |
|-----------|---------|
| Hypervisor | Oracle VirtualBox 7.2.16 (Type 2, hosted) on Windows |
| Host CPU | AMD Ryzen 7 9800X3D |
| Linux guests | Ubuntu 26.04.1 LTS |
| Windows guest | Windows 11 25H2 |
| Networking | VirtualBox Internal Network + NAT adapters |
| Storage | Dynamically allocated VDI disks |

I chose VirtualBox because it is free and open source, supports a wide range of guest operating systems, and has built-in cloning, snapshots, and flexible virtual networking (NAT, bridged, host-only, and internal), which made it easy to build and iterate on a multi-VM lab without dedicated server hardware. As a Type 2 hypervisor it runs on top of the host OS rather than on bare metal, so it competes with the host for CPU and memory and carries more overhead than Type 1 platforms such as VMware ESXi or Hyper-V. That trade-off is fine for a lab but is why it would not be my choice for production.

| VirtualBox Install | Installation Media |
|:---:|:---:|
| ![VirtualBox install](docs/images/virtualbox-install.png) | ![Installation media](docs/images/install-media.png) |

---

## Architecture

![Network diagram](docs/images/network-diagram.png)

| VM | Hostname | OS | vCPU | RAM | Disk | IP | Adapters | Role |
|----|----------|----|:---:|:---:|:---:|----|----------|------|
| VM1 – Server | `1Server` | Ubuntu 26.04.1 LTS | 2 | 4000 MB | 40 GB | 192.168.10.10 | Internal + NAT | Core server for services such as DNS, DHCP, or file sharing; jump box into the isolated segment |
| VM2 – Client | `2Client` | Windows 11 | 2 → 3 | 4000 → 5000 MB | 60 → 70 GB | 192.168.10.20 | Internal + NAT | End-user workstation for testing client-server connectivity and user-facing access |
| VM2.5 – Client Clone | `2Dot5ClientClone` | Windows 11 | 3 | 5000 MB | 70 GB | 192.168.10.21 | Internal + NAT | Full clone of VM2, used to demonstrate rapid provisioning |
| VM3 – App/Test | `3AppTest` | Ubuntu 26.04.1 LTS | 2 | 4000 MB | 30 GB | 192.168.10.30 | Internal only | Isolated environment for testing applications, scripts, and security tools |

The operating systems were picked to mirror a realistic small office. Ubuntu is lightweight, free, and widely used for servers, so it hosts the core services. Windows 11 represents the typical business workstation. A second Ubuntu system with a full desktop serves as a sandbox where applications and security tools can be tested without risking the server or client.

<details>
<summary>Initial architecture design</summary>

![Planned architecture](docs/images/architecture-planned.png)

</details>

---

## Deploying the Virtual Machines

Each VM was created in VirtualBox with 2 vCPUs, 4000 MB of RAM, and a dynamically allocated disk sized to its role, then given a hostname that identifies it at a glance. These allocations are right for a lab, but a production server handling many users or services would need considerably more CPU, memory, and storage. The Windows client received the largest disk, 60 GB, because Windows, its applications, and user files need the most space.

| | Configuration | Running | System Info |
|---|:---:|:---:|:---:|
| **VM1** | ![VM1 config](docs/images/vm1-config.png) | ![VM1 running](docs/images/vm1-running.png) | ![VM1 info](docs/images/vm1-system-info.png) |
| **VM2** | ![VM2 config](docs/images/vm2-config.png) | ![VM2 running](docs/images/vm2-running.png) | ![VM2 info](docs/images/vm2-system-info.png) |
| **VM3** | ![VM3 config](docs/images/vm3-config.png) | ![VM3 running](docs/images/vm3-running.png) | ![VM3 info](docs/images/vm3-system-info.png) |

---

## Network Segmentation

All VMs share the `192.168.10.0/24` subnet, but access is controlled by which adapters each VM has rather than by separate IP ranges. VM1 and VM2 each have both an **Internal Network** adapter, which lets them talk to the other VMs, and a **NAT** adapter, which lets them reach the internet through the host while staying hidden from outside hosts. VM3 has only the Internal Network adapter, so it has no path to the internet at all and can only be reached by pivoting through another VM on the internal segment. The result is a front-facing server and client pair with an isolated back-end segment behind them.

VirtualBox's Internal Network acts as a private virtual switch: only VMs assigned to the same internal network name can join it, and it has no connection to the host or the outside world. Because the whole topology lives in software, adapters can be added, removed, or reassigned in seconds, so a change like moving VM3 from dual-adapter to internal-only is instant to make and test. The design also leaves room to grow: most of the `/24` is unused, and new systems can simply be placed in the open or isolated segment based on how much exposure they need, the same way an enterprise adds systems to existing security zones.

**Testing.** Ping tests between all three VMs confirmed Layer 3 reachability across the internal segment. I then opened an SSH session from VM1 to VM3, proving the jump-box pattern worked: even with no external access, VM3 remained fully reachable for legitimate administration through VM1.

| Internal Network Config | IP Configuration |
|:---:|:---:|
| ![Internal network config](docs/images/internal-network-config.png) | ![IP configuration](docs/images/ip-config.png) |

| Ping Tests | SSH Through the Jump Box (VM1 → VM3) |
|:---:|:---:|
| ![Ping tests](docs/images/ping-tests.png) | ![SSH jump box](docs/images/ssh-jumpbox.png) |

---

## Scaling a VM's Resources

I scaled VM2 up from 4000 MB RAM, 2 vCPUs, and a 60 GB disk to 5000 MB RAM, 3 vCPUs, and a 70 GB disk. RAM and vCPU changes were made directly in the VM's settings, but they required powering the VM off first because VirtualBox does not support hot-adding memory or processors, so there was a brief period of downtime. Expanding the disk took one more step: VirtualBox raises the VDI's maximum size, but the guest OS partition does not grow on its own, so I extended it from inside Windows before the extra space became usable.

On a physical server, the same change would mean opening the machine, installing memory modules, possibly swapping the processor, and adding or replacing drives, all with scheduled downtime, hands-on access, and the risk of damaging hardware. The flip side of how easy this is in software is over-allocation: giving one VM too much can starve its neighbors, destabilize the host if total allocations exceed physical capacity, and hide inefficient configurations instead of fixing them.

| | Configuration | Inside the OS |
|---|:---:|:---:|
| **Before** | ![Original config](docs/images/resize-original-config.png) | ![Original OS](docs/images/resize-original-os.png) |
| **After** | ![Updated config](docs/images/resize-updated-config.png) | ![Updated OS](docs/images/resize-updated-os.png) |

---

## Rapid Provisioning with Cloning

Using VirtualBox's Clone feature, I made a **full clone** of VM2 called VM2.5 – Client Clone. A full clone copies the entire virtual disk and configuration rather than referencing the original, so the new VM runs completely independently even if VM2 is modified or deleted. The clone took only a few minutes because VirtualBox copies an already configured disk, compared with booting an installer, stepping through setup, and installing updates and software, which can take thirty minutes to over an hour.

The clone did need a new identity to avoid conflicts with the original on the same network. Its MAC address was regenerated during cloning, and I assigned it a new static IP (`192.168.10.21`) and hostname (`2Dot5ClientClone`) inside Windows. Beyond saving time, cloning gives consistency: every copy starts from the same known-good baseline instead of being exposed to human error in repeated manual installs.

| Source VM | Clone Process | New VM | Unique Hostname and IP |
|:---:|:---:|:---:|:---:|
| ![Source VM](docs/images/clone-source-vm.png) | ![Clone process](docs/images/clone-process.png) | ![New VM](docs/images/clone-new-vm.png) | ![Unique identity](docs/images/clone-unique-identity.png) |

---

## Snapshots and Recovery

From the Snapshots tab in VirtualBox Manager, I took a snapshot of VM2 in its baseline state, capturing its disk, memory state, and configuration. I then made a deliberate, easy-to-verify change by creating a new text file, and used **Restore Snapshot** to roll back. The restore took about one to two minutes, after which the VM was exactly as it had been at the snapshot and the text file was gone.

Snapshots make it safe to try updates, new software, or configuration changes, since a bad outcome can be undone almost immediately. They are not backups, though. A snapshot is differential data tied to the original virtual disk and usually lives on the same physical storage, so if that storage fails, the VM and all its snapshots are lost together. Long snapshot chains also hurt performance. A real backup is an independent copy stored somewhere else.

| Snapshot Created | Original State | Change Introduced | Restoring | Restored State |
|:---:|:---:|:---:|:---:|:---:|
| ![Snapshot created](docs/images/snapshot-created.png) | ![Original state](docs/images/snapshot-original-state.png) | ![Change introduced](docs/images/snapshot-change.png) | ![Restore](docs/images/snapshot-restore.png) | ![Restored state](docs/images/snapshot-restored-state.png) |

---

## How the Storage Works

Each VM is backed by its own VDI (Virtual Disk Image) file, managed through VirtualBox's Virtual Media Manager. VM1 has a 40 GB disk, VM2 and its clone each have 70 GB, and VM3 has 30 GB. All are dynamically allocated, so a disk file only takes up as much space on the host as the guest is actually using. The difference shows clearly in the manager: VM2's disk has a virtual size of 70 GB but an actual size of about 43 GB on the host.

The guest has no idea any of this is happening. VM3's Ubuntu install sees an ordinary 30 GB hard drive to partition and format, when in reality it is a single file on my host's drive at a path like `C:\Users\<user>\VirtualBox VMs\VM3 - App_Test\VM3 - App_Test.vdi`. That abstraction is what makes VMs portable: the whole system, OS, software, and data included, can be moved to a different machine with different hardware just by copying the file. It is also what makes cloning and snapshots possible without touching physical disks.

This lab uses local, per-VM disk files. In a larger environment, shared storage such as a SAN or NAS would enable live migration between hosts, centralized backup and management, more efficient use of capacity through deduplication and thin provisioning, and resilience that does not depend on any single host.

| Virtual Disk Files | VM Storage Config | Hypervisor Storage Config |
|:---:|:---:|:---:|
| ![Virtual disk files](docs/images/virtual-disk-files.png) | ![VM storage config](docs/images/vm-storage-config.png) | ![Hypervisor storage config](docs/images/hypervisor-storage-config.png) |

---

## Bonus: Pi-hole in Docker

To compare containers with full VMs, I installed Docker Desktop on the host with the WSL2 backend, which runs the Linux container engine through the Windows Subsystem for Linux instead of a full Linux VM. After a reboot to finish the WSL2 integration, I verified the install with `docker --version` and `docker run hello-world`, then deployed the network-wide ad blocker Pi-hole:

```powershell
docker pull pihole/pihole:latest

docker run -d --name pihole `
  -p 53:53/tcp -p 53:53/udp -p 80:80 `
  -e TZ="America/New_York" `
  -e WEBPASSWORD="<your-admin-password>" `
  --cap-add=NET_ADMIN `
  --restart=unless-stopped `
  pihole/pihole:latest
```

Port 53 handles DNS over TCP and UDP, port 80 serves the admin interface, `NET_ADMIN` gives Pi-hole the network privileges it needs to manage DNS traffic, and `--restart=unless-stopped` keeps it running across host reboots. `docker ps` confirmed the container was up, and I logged in to the dashboard at `http://localhost/admin` to verify DNS queries were being logged.

| Installing Docker | Creating the Container | Pi-hole Dashboard |
|:---:|:---:|:---:|
| ![Docker install](docs/images/docker-install.png) | ![Pi-hole container](docs/images/pihole-container.png) | ![Pi-hole dashboard](docs/images/pihole-dashboard.png) |

**VMs vs. containers.** A VM virtualizes an entire machine and runs a full guest OS on a hypervisor. That makes it heavy, gigabytes in size and tens of seconds to minutes to boot, but strongly isolated behind a hypervisor boundary. A container virtualizes only at the OS level, sharing the host's kernel and relying on namespaces and cgroups for isolation. That makes it light, megabytes in size and seconds to start, but its security boundary is thinner, since a kernel-level exploit could in theory let a container escape to the host.

---

## Challenges and Troubleshooting

Isolating VM3 took careful attention to adapter assignment. A single misconfigured adapter would have quietly given the "isolated" VM internet access and undermined the whole segmentation design, so I verified the isolation through connectivity testing rather than assuming it.

Expanding VM2's disk revealed that raising a VDI's maximum size in VirtualBox does not extend the guest's partition. The added space only became usable after I extended the partition from inside Windows.

Cloning required giving the new VM its own identity. Without a regenerated MAC address, a unique static IP, and a distinct hostname, VM2.5 would have conflicted with the original VM2 on the same internal network.

---

## What I Learned

The biggest takeaway was how much flexibility virtualization adds over physical infrastructure. Resource allocation, network segmentation, and disk provisioning could all be defined, changed, and tested entirely in software, which made rapid iteration possible in a way that would be slow and expensive on real hardware. The jump-box design reinforced the value of layered access control, and working with snapshots and clones showed how they make provisioning fast and experimentation safe. Running everything locally also meant full control, no cloud costs, and a network completely isolated from the internet, which was ideal for safely testing segmentation and recovery. These are the same concepts, hypervisor architecture, virtual networking, storage abstraction, and VM lifecycle management, that underpin real enterprise and security operations environments.

---

## Recommendations for Growth

If BlueSky Analytics were a real company outgrowing this setup, I would recommend the following.

| Area | Recommendation |
|------|----------------|
| Hypervisor | Move from a single Type 2 host to a Type 1 platform such as VMware ESXi or Microsoft Hyper-V for better performance and centralized management across multiple hosts |
| Storage | Add shared storage (SAN or NAS) to enable live migration and centralized backups instead of relying on per-VM files on one host's disk |
| Backups | Back up VMs to independent storage rather than treating snapshots as a recovery plan |
| Network | Introduce a DMZ, a dedicated management network, and VLAN-based segmentation for finer-grained control |
| Access | Formalize the VM1 jump box into a hardened bastion host |

---

## Author

**Geoffrey Mackey**, Cybersecurity Professional & Student at the University of Cincinnati.
