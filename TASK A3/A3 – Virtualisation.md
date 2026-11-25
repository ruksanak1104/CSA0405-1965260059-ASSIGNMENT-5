
# TASK A3 – Evaluate Virtualisation (5 Marks)

## A3.1 Hypervisors (3 Marks)

A hypervisor (virtual machine monitor) lets several guest operating systems share one physical machine, each believing it has its own hardware.

### Types of Hypervisors

| **Criterion** | **Type-1 (Bare-metal)** | **Type-2 (Hosted)** |
|---|---|---|
| Runs on | Directly on hardware | As an application on a host OS |
| Examples | KVM (a Linux kernel module), VMware ESXi, Xen, Microsoft Hyper-V | Oracle VirtualBox, VMware Workstation, Parallels |
| Overhead | Low (≈ 2–5% CPU) | Higher: guest → hypervisor → host OS → hardware |
| Typical use | Data centres, cloud, production | Developer laptops, labs, testing |

### Types of Virtualisation

- **Full Virtualisation:** The guest OS is unmodified; privileged instructions are trapped and emulated (binary translation) or, today, handled by hardware extensions. Maximum compatibility (Windows and Linux guests).

- **Para-virtualisation:** The guest kernel is modified (or has drivers such as virtio) and makes **hypercalls** to the hypervisor instead of executing privileged instructions; faster, especially for I/O (Xen PV, virtio-net, virtio-blk).

- **Hardware Support:** Intel **VT-x** and AMD **AMD-V** add a new CPU mode (VMX root/non-root) so guests run directly on the CPU and only sensitive events cause a VM exit; **EPT / NPT** add hardware nested page tables, removing the cost of shadow page tables; VT-d/IOMMU allows safe device pass-through.

### Checking Hardware Virtualisation Support in Linux

```bash
# Check CPU virtualisation support
grep -E 'vmx|svm' /proc/cpuinfo

# Check whether KVM modules are loaded
lsmod | grep kvm
```

## A3.2 Benefits and Costs (2 Marks)

### Benefits of Virtualisation for the University

| **Benefit** | **Explanation** |
|---|---|
| Isolation | The portal, the learning-management system and the exam system run in separate VMs; a crash or compromise in one does not touch the others |
| Consolidation | Ten under-used servers (≈ 10% CPU each) can be replaced by 1–2 hosts, saving power, space and hardware |
| Snapshots | Take a snapshot before a patch or upgrade and roll back in seconds if it fails; clone a VM to build a test copy of the portal |
| Live migration | Move a running VM to another host (`vMotion` / `virsh migrate --live`) for hardware maintenance with no downtime |

### Costs of Virtualisation

| **Cost** | **Explanation** |
|---|---|
| Overhead | CPU ≈ 2–5% (hardware-assisted), I/O 5–10% unless virtio is used |
| Memory per VM | Each VM carries a full guest OS: ≈ 512 MB – 1 GB before the application starts (see B3 measurement) |
| Disk | Each VM disk image is GBs; snapshots grow over time |
| Licences | KVM and Xen are free; ESXi/vSphere and Windows guests need paid licences |
| Boot time and management | Tens of seconds to minutes; every guest OS needs patching |

### Evaluation

VMs give strong isolation and easy recovery, so they are suitable for the database and for separating university systems, but they are too heavy and too slow to start for scaling the stateless web tier on result day.

## Alternative Concept and Complexity Analysis – VirtualBox (Type-2) Instead of KVM (Type-1)

### Comparison: KVM vs VirtualBox

| **Criterion** | **KVM (Type-1)** | **VirtualBox (Type-2)** | **Evaluation** |
|---|---|---|---|
| CPU-bound work | ≈ 1–3% slower than bare metal | ≈ 3–8% slower | Small difference for both, because VT-x runs guest code natively |
| I/O-bound work (disk and network) | ≈ 3–8% slower with virtio | ≈ 15–30% slower: I/O passes through the host OS and emulated devices | KVM clearly better |
| Memory used by host OS | Linux host ≈ 0.3–0.5 GB | Full desktop OS (Windows/macOS/Linux GUI) ≈ 2–4 GB plus VirtualBox ≈ 0.2 GB | VirtualBox wastes memory that could host VMs |
| Suitability for production | Yes – used by major clouds (AWS Nitro, Google Cloud) | No – no live migration, host updates/reboots disturb VMs, weak management APIs | Use VirtualBox only for development and for this lab |

### Pseudocode – Virtualisation Overhead

```text
ALGORITHM Overhead

INPUT:
    t_bare (time on bare metal)
    t_vm   (time in VM)

OUTPUT:
    overhead_percent

BEGIN
    1. overhead_percent ← ((t_vm − t_bare) ÷ t_bare) × 100

    2. RETURN overhead_percent
END
```

### Complexity Analysis

- The overhead is a constant-time calculation, **O(1)**.
- Hypervisor overhead for CPU-bound work is a constant factor (the work still grows as **O(n)** with the job size n).
- For I/O-bound work each I/O operation pays an extra fixed cost, so overhead grows as **O(k)** for k I/O operations, which is why type-2 (extra layer per I/O) loses more.
- **Memory:** O(m) for m VMs, each with its own guest OS.
