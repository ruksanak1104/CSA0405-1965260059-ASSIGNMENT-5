
## TASK A4 – Evaluate Containerisation (4 Marks)

### A4.1 How Containers Work (2 Marks)

A container is **not** a small VM; it is an ordinary Linux process (or group of processes) that the kernel isolates and limits using two features, with a layered file system on top.

| **Feature** | **What it does** | **Example** |
|---|---|---|
| **Namespaces** (isolation: what a process can see) | `pid` (own process tree – the app is PID 1), `net` (own interfaces/IP/ports), `mnt` (own mount table), `uts` (own hostname), `ipc`, `user` (map root inside to an unprivileged UID outside), `cgroup` | `unshare --pid --fork --mount-proc bash` gives a shell that sees only its own processes; `docker exec c ps` shows only the container’s processes |
| **Control groups (cgroups)** (limits: what a process can use) | Limit and account for CPU, memory, I/O and PIDs per group, under `/sys/fs/cgroup` | `docker run --cpus 0.5 --memory 256m` → CPU quota 50,000/100,000 µs; exceeding memory triggers OOM-kill of that container only |
| **Image layers (union file system – overlayfs)** | An image is a stack of read-only layers, one per Dockerfile instruction (`FROM`, `RUN`, `COPY`…), identified by hashes and shared between images; a thin writable layer is added when a container starts (copy-on-write) | `docker history result-portal` shows each layer and its size |

**Layer caching:** If a layer’s instruction and inputs are unchanged, Docker reuses it – so the Dockerfile copies `requirements.txt` and installs dependencies before copying the source code; changing `app.py` then rebuilds only the last layers. Shared layers are downloaded once per server, which makes pulling and starting fast.

### A4.2 Containers versus Virtual Machines (2 Marks)

| **Criterion** | **Container** | **Virtual machine** | **Winner** |
|---|---|---|---|
| Start-up time | ≈ 1–3 s (start a process) | ≈ 30–90 s (boot guest OS) | Container |
| Memory use | ≈ 20–60 MB (app only; kernel shared) | ≈ 500 MB – 1 GB (full guest OS + app) | Container |
| Disk size | ≈ 50–200 MB image | ≈ 2–10 GB disk image | Container |
| Isolation and security | Shared kernel – a kernel exploit can escape; mitigated by non-root user, seccomp, AppArmor, read-only file system, user namespaces | Separate kernel behind the hypervisor – stronger (hardware-enforced) | VM |
| Portability | “Runs the same everywhere” – same image from laptop to cloud | Image is hypervisor-specific and large | Container |
| Density | Dozens of containers per host | A handful of VMs per host | Container |

**Decision for the portal:** Run the **stateless web/application tier in containers** (fast scaling and recovery, small, dense) and the **database in a VM** (or a managed database) for stronger isolation, simpler backups/snapshots and stable storage. Containers may also run on top of VMs (the way cloud Kubernetes nodes work) to combine both strengths.

### Alternative Concept and Complexity Analysis – Portal and Database Each in Their Own VM

| **Criterion** | **Two containers (chosen)** | **Two VMs (alternative)** | **Evaluation** |
|---|---|---|---|
| Start-up time | ≈ 2 s each | ≈ 45–60 s each (≈ minutes if provisioned from scratch) | Containers ≈ 20–30× faster; critical when scaling during result day |
| Memory per instance | ≈ 50 MB (portal) + ≈ 100 MB (DB cache excluded) | ≈ 600 MB guest OS + app each | VMs ≈ 4–10× more RAM; 6 portal copies = 6 guest OSs |
| Isolation strength | Process-level, shared kernel | Hardware-level, separate kernels | VMs stronger – better for the database holding student records |

**Pseudocode – Time to Add n Instances**

```text
ALGORITHM ScaleOut

INPUT: n (new instances), t_start (start time of one instance), p (instances started in parallel)

OUTPUT: total_time, total_memory

BEGIN
    1. total_time   ← CEILING(n / p) × t_start
    2. total_memory ← n × (mem_app + mem_guest_os)  // mem_guest_os = 0 for containers
    3. RETURN total_time, total_memory
END
```

**Complexity:** Time to scale out is **O(n/p) × t_start**; since `t_start(VM) ≈ 45 s` and `t_start(container) ≈ 2 s`, adding 6 instances in parallel takes ≈ 1 round in each case but the VM round is 20× longer. Memory is **O(n)** in both, with a much larger constant for VMs (guest OS per instance).

