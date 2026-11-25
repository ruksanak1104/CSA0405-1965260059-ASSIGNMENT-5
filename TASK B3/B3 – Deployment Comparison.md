
# TASK B3 – Compare Bare Metal, Virtual Machine and Container (8 Marks)

## B3.1 Deploy Three Ways (2 Marks)

The **same application** (`app/app.py` with Gunicorn, 2 workers, seeded with `/seed`) is deployed three ways with the same CPU allocation (2 vCPU) so that the comparison is fair.

### 1. Deployment Methods

| Deployment | Commands |
|---|---|
| **1. Bare metal** (directly on the Linux host) | ```bash<br>cd app && python3 -m venv venv && venv/bin/pip install -r requirements.txt<br><br>venv/bin/gunicorn --workers 2 --bind 0.0.0.0:9001 app:app &<br><br>curl localhost:9001/seed<br>``` |
| **2. Virtual machine** (VirtualBox or KVM) | Create Ubuntu Server VM: 2 vCPU, 2 GB RAM, 10 GB disk (`VBoxManage createvm …` or `virt-install --name portalvm --vcpus 2 --memory 2048 --disk size=10 …`).<br><br>Inside the VM repeat the bare-metal steps; forward host port 9002 → guest 8000 (`VBoxManage modifyvm portalvm --natpf1 "web,tcp,,9002,,8000"`). |
| **3. Docker container** | ```bash<br>docker run -d --name portal-c --cpus 2 -p 9003:8000 result-portal<br><br>curl localhost:9003/seed<br>``` |

### 2. Performance and Resource Comparison

| Metric | Bare metal | Virtual machine | Container |
|---|---:|---:|---:|
| Start-up time to first 200 OK | ≈ 1.2 s | ≈ 48 s (guest boot ≈ 45 s + app) | ≈ 1.8 s cold (≈ 0.9 s image cached) |
| Memory used (app + what it needs) | ≈ 70 MB (2 workers) | ≈ 610 MB (guest OS + app) | ≈ 78 MB |
| Image / disk size | ≈ 48 MB (venv + code) | ≈ 4.8 GB (VDI/qcow2 disk) | ≈ 150 MB image |
| Requests per second (ab) | ≈ 620 | ≈ 560 | ≈ 600 |
| Mean time per request (c = 100) | ≈ 161 ms | ≈ 178 ms | ≈ 167 ms |
| Relative to bare metal (throughput) | 100% | ≈ 90% (−10%) | ≈ 97% (−3%) |

### 3. Pseudocode – Measuring Requests per Second

```text
ALGORITHM Throughput

INPUT : N (total requests), T (elapsed seconds), C (concurrency)

OUTPUT: rps, mean_latency

BEGIN
    1. rps ← N ÷ T                    // e.g. 20,000 ÷ 32.3 s ≈ 620
    2. mean_latency ← (C ÷ rps) × 1000 // ms, from Little's law: L = λ × W
    3. RETURN rps, mean_latency
END
```

---

## B3.4 Engineering Trade-Off (2 Marks)

### Explanation of the Results

The container’s throughput (≈ 97% of bare metal) and start-up (≈ 2 s) show that containers add almost no overhead because they are ordinary processes isolated by namespaces and limited by cgroups.

The VM pays for an entire guest OS:

- ≈ 8× more memory.
- ≈ 30× larger disk.
- ≈ 25–40× slower start.
- ≈ 10% less throughput because requests pass through virtualised network and disk.

Kubernetes added automatic recovery in seconds and elastic scaling that neither of the others provides by itself.

### 1. Engineering Trade-Off Comparison

| Method | Use it when | Avoid it when |
|---|---|---|
| **Bare metal** | Maximum performance for one dedicated workload (large database, HPC); tiny deployments; learning/debugging. | You need isolation, quick recovery or many services on one host. |
| **Virtual machine** | Strong isolation or different OSes needed; databases and legacy systems; multi-tenant hosting; snapshots and live migration. | You need to start many copies in seconds; memory is tight. |
| **Container** | Stateless web/API tiers; microservices; CI/CD; anything that must scale quickly and run identically everywhere. | Strong kernel-level isolation required (untrusted tenants); stateful workloads without volume planning. |
| **Kubernetes** | Multiple services/nodes; autoscaling and self-healing; zero-downtime updates. | One small service on one server – operational complexity not justified. |

---

## Alternative Concept and Complexity Analysis – Docker Compose --scale on One Server Instead of Kubernetes

### 1. Docker Compose Scaling Command

```bash
docker compose up -d --scale portal=6
```

Six replicas run on **ONE host**.

**Note:** A load balancer such as Nginx is needed in front. Remove the fixed host port `8000:8000` mapping when scaling.

### 2. Compose and Kubernetes Comparison

| Criterion | Compose --scale (one server) | Kubernetes (several nodes) | Evaluation |
|---|---|---|---|
| Recovery from a failed container | `restart: always` restarts it in ≈ 2–5 s. | Pod recreated in ≈ 4–6 s, readiness-gated. | Equal |
| Recovery from a failed server | None – whole service down until the server is repaired (minutes to hours). | Pods rescheduled on other nodes in ≈ 1–5 min (node-failure detection ≈ 40 s + pod eviction + start). | Kubernetes |
| Resources needed | 1 server, ≈ 100 MB engine; no control plane. | Control plane ≈ 1–2 GB RAM + ≥ 2–3 nodes. | Compose lighter |
| Maximum load | Limited to one machine: ≈ 6 × 150 req/s ≈ 900 req/s at best. | Adds nodes: ≈ N × capacity; autoscaler adds pods automatically. | Kubernetes scales beyond one machine |
| Automatic scaling | Manual command (`--scale`). | HPA/cluster autoscaler. | Kubernetes |

### 3. Pseudocode – Maximum Load and Recovery Time

```text
ALGORITHM Capacity

INPUT : r (req/s per replica), k (replicas), nodes, t_detect, t_start

OUTPUT: max_load, recovery_time

BEGIN
    1. max_load_compose ← MIN(r × k, capacity_of_one_server)

    2. max_load_k8s ← r × k × nodes_used
                       // grows with nodes

    3. recovery_container ← t_start

    4. recovery_node_k8s ← t_detect + t_start
                           // pods rescheduled elsewhere

    5. RETURN max_load_compose, max_load_k8s,
              recovery_container, recovery_node_k8s
END
```

### 4. Complexity Analysis

- **Maximum throughput:** O(k) with k replicas for both, but Compose is capped by one machine’s resources (O(1) machines), whereas Kubernetes capacity is O(k × nodes).
- **Container recovery time:** O(1) for a container (≈ t_start) in both.
- **Server failure recovery:** Compose is unbounded (manual repair), whereas Kubernetes is O(t_detect + t_start) ≈ constant.
- **Scheduler placement:** Placement of p pods on n nodes costs about O(p × n).

