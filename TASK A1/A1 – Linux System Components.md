
# TASK A1 – Evaluate the Linux System Components (7 Marks)

## A1.1 Linux Architecture (2 Marks)

Linux is a layered system. Applications talk to the kernel only through system calls; everything else is built around that boundary.

| **Layer** | **What it is** | **Role for the result portal** |
|---|---|---|
| Hardware | CPU, RAM, disks, NIC | The physical server being shared |
| **Kernel** (monolithic, loadable modules) | Process scheduler, memory manager, VFS, device drivers, network stack, security (LSM) | Runs Gunicorn workers, caches files, handles TCP connections |
| **System libraries** | glibc (system-call wrappers), libssl, libpthread | Python and Nginx call `open()`, `read()`, `socket()` through glibc |
| **Shell** | bash, sh, zsh – command interpreter and scripting language | Runs administration and cron scripts (B1) |
| **Utilities** | `ps`, `top`, `grep`, `awk`, `systemctl`, `ip`, `ss`, `tar`, `ufw` | Observe, automate and repair the server |
| User applications | Nginx, Gunicorn/Flask, database | The portal itself |

### How the kernel manages each resource

- **Processes:** Creates them with `fork()`/`clone()` and `exec()`; the scheduler decides which task runs on each CPU; signals and IPC (pipes, sockets, shared memory) let them cooperate. Each process has a `task_struct` holding its state, PID, memory map and open files.
- **Memory:** Gives every process a private virtual address space; the MMU and page tables map pages to RAM; demand paging loads pages only when touched; the page cache keeps recently used file data; swap holds rarely used anonymous pages.
- **Files:** The Virtual File System (VFS) presents one tree over ext4, XFS, tmpfs, NFS, `/proc`; file descriptors index the per-process open-file table; the page cache and I/O scheduler sit below.
- **Devices:** Everything is a file under `/dev`; drivers (built in or loadable modules) turn `read()`/`write()` into hardware operations; udev creates device nodes.
- **Network:** Sockets API → TCP/UDP → IP routing → netfilter (firewall/NAT) → NIC driver; `epoll` lets one process serve thousands of connections.

## A1.2 Process and Memory Management in Linux (3 Marks)

**Completely Fair Scheduler (CFS).**

CFS gives each runnable task a fair share of CPU time. For every task it tracks **vruntime** (virtual runtime = actual CPU time scaled by priority weight). Runnable tasks are kept in a **red-black tree** ordered by vruntime; the scheduler always runs the left-most node (smallest vruntime), i.e. the task that has had the least CPU. Nice values (−20 to +19) change a task’s weight. (Recent kernels, 6.6 onward, replace CFS with the EEVDF scheduler, which keeps the same fairness idea and adds latency-based deadlines.)

**Evaluation for the portal:**

CFS is excellent for fairness and for mixed workloads – Gunicorn workers, Nginx and cron jobs all get CPU, so a heavy log-analysis script cannot starve the web service completely. Cost: it is not real-time; on result day, if 500 runnable threads compete for 4 cores, every task slows down proportionally and tail latency rises. Picking the next task is `O(log n)` (tree operations); with cgroups (A4) CFS can enforce hard CPU shares per container.

**Page cache and swapping.**

Linux uses almost all free RAM as page cache, so repeated reads of static result pages and PDFs come from memory, not disk – this is why “free” memory looks low on a healthy server. Under pressure the kernel first reclaims clean cache pages (cheap), then swaps out cold anonymous pages (expensive, disk speed). The `vm.swappiness` setting (default 60) controls the balance.

**Evaluation:** The cache makes the portal fast cheaply; swapping is a warning sign – if the portal swaps during result day, response times jump from milliseconds to seconds, so the design sets swappiness low (10), sizes RAM for the peak, and sets container memory limits so that one runaway process is killed by the OOM killer instead of freezing the host.

**`/proc` and `/sys` as observation tools.**

`/proc` is a virtual file system exposing process and kernel data (`/proc/<pid>/status`, `/proc/meminfo`, `/proc/cpuinfo`, `/proc/loadavg`, `/proc/net/`); tools like `ps`, `top` and `free` just read it. `/sys` exposes the device and driver model (`/sys/class/net`, `/sys/block`, `/sys/fs/cgroup`) and allows tuning.

**Evaluation:** They cost nothing to read, need no extra software and are always consistent with the kernel’s real state, so they are the right base for monitoring scripts (B1.1, B1.2); their limitation is that they show only the instant, so history needs a collector (Prometheus/journald).

## A1.3 Linux File System and Permissions (2 Marks)

| **Directory** | **Purpose** | **Used for the portal** |
|---|---|---|
| `/` | Root of the single tree | |
| `/bin`, `/sbin`, `/usr`, `/lib` | Commands, libraries, system programs | Python, Nginx binaries |
| `/etc` | Configuration | `/etc/nginx`, `/etc/systemd/system/portal.service`, `/etc/ssh` |
| `/var` | Variable data | `/var/log/nginx`, `/var/lib/portal` (database) |
| `/home`, `/root` | User home directories | Admin accounts |
| `/tmp`, `/run` | Temporary files, runtime sockets | Gunicorn socket |
| `/proc`, `/sys`, `/dev` | Virtual file systems and device nodes | Monitoring |
| `/opt`, `/srv` | Optional software, service data | `/opt/portal` application code |

**Virtual File System (VFS).**

VFS is a kernel abstraction layer defining common objects – superblock, inode, dentry and file – and common operations (`open`, `read`, `write`). Each real file system (ext4, XFS, NFS, tmpfs, procfs) registers its own implementation, so applications use the same system calls regardless of where data lives. This is also what allows container image layers (overlayfs) to be stacked.

**Permission model.**

Each file has an owner, a group and “others”, with read (4), write (2) and execute (1) bits: `chmod 640 app.conf` = owner `rw-`, group `r--`, others none.

**setuid:** (`chmod u+s`) makes a program run with its owner’s privileges (e.g. `/usr/bin/passwd` runs as root so users can change their own password) – powerful but a security risk, so the design audits it with `find / -perm -4000`.

**sudo:** Lets named users run specific commands as root, configured in `/etc/sudoers`, with every use logged – so nobody needs to log in as root.

Principle applied: least privilege.

## Alternative Concept and Complexity Analysis – Windows Server instead of Linux

If the portal ran on Windows Server, the same application would run on IIS (or Python behind IIS/Waitress) with the NT kernel, NTFS, Active Directory/ACL permissions, PowerShell for automation and Task Scheduler instead of cron. The solution is workable but costs more and uses more resources.

| **Criterion** | **Linux (Ubuntu Server 24.04)** | **Windows Server 2025** | **Evaluation** |
|---|---|---|---|
| Base memory (idle, no GUI) | ≈ 250–400 MB | ≈ 1.5–2 GB (Core) to 2+ GB (Desktop) | Linux leaves ≈ 1.5 GB more per server for the application |
| Base disk | ≈ 3–5 GB minimal install | ≈ 10–32 GB (Core ≈ 10 GB, Desktop ≈ 32 GB) | Linux images and VMs are 3–6× smaller |
| Time to automate a routine task | Shell script + cron: minutes; text-based, scriptable everywhere | PowerShell + Task Scheduler: similar once learned; GUI-heavy defaults | Roughly equal for experts; Linux tooling is simpler over SSH |
| Licence cost per server | ₹0 (open source); optional support subscription | Server licence ≈ ₹60,000–₹1,20,000 per 16-core edition plus CALs (illustrative; verify with Microsoft pricing) | For 1 server: negligible vs hardware; for 20 servers or cloud VMs the licence dominates |
| Containers | Native (namespaces/cgroups) | Windows containers, heavier images (GBs) | Linux is the natural container host |

### Pseudocode – estimating licence and memory cost for n servers

```text
ALGORITHM CompareOSCost

INPUT:
    n           (number of servers)
    licence_win (Windows licence cost per server)
    ram_win     (GB used by Windows base OS)
    ram_lin     (GB used by Linux base OS)
    ram_price   (cost per GB of RAM)

OUTPUT:
    extra_cost

BEGIN
    1. cost_lin   ← n × ram_lin × ram_price
                   // No licence fee

    2. cost_win   ← n × (licence_win + ram_win × ram_price)

    3. extra_cost ← cost_win − cost_lin

    4. RETURN extra_cost
END
```

**Complexity:**

The cost is linear in the number of servers: **T(n) = O(n)** time to compute and **O(1)** space. The Windows extra cost grows as `n × (licence + extra RAM)`, so at `n = 20` it is 20× the single-server difference, which is why Linux is chosen.

