
## TASK A5 – Design the Cloud Deployment and Orchestration (4 Marks)

### A5.1 Cloud Set-up (2 Marks)

| **Model** | **You manage** | **Provider manages** | **Example for the university** |
|---|---|---|---|
| **IaaS** | OS, runtime, application, data | Hardware, virtualisation, network | Cloud VMs (EC2, Azure VM) running Docker or Kubernetes |
| **PaaS** | Application and data | OS, runtime, scaling | Managed Kubernetes, managed database, App Service |
| **SaaS** | Only configuration/data | Everything | Hosted e-mail or LMS (Google Workspace, Moodle Cloud) |

**Result-day architecture (hybrid, container-based):**

```text
Students
   |
   v
DNS (results.university.edu, TTL 60 s)
   |
   v
CDN (caches static pages and PDFs)
   |
   v
Load balancer (HTTPS, health checks)
   |
   +---- On-premises Kubernetes cluster (3 nodes)
   |        Baseline: 3 portal pods
   |
   +---- Cloud managed Kubernetes node group
   |        Burst: autoscaled to 12 pods
   |
   v
Cache (Redis, optional)
   |
   v
Managed database (primary + read replica,
                  daily backups, multi-AZ)
```

- **Compute:** Managed Kubernetes (PaaS) rather than hand-managed VMs, so nodes are replaced and scaled automatically.

- **Load balancer:** Spreads traffic, terminates TLS, removes unhealthy pods using readiness checks.

- **Managed database:** Automatic backups, failover and read replicas; student data encrypted at rest and in transit.

- **CDN + caching:** Result PDFs and static assets are served from the edge – reduces origin load by 50–70%.

- **Operational plan:** Scale up 24 h before result day (pre-warm nodes), load test beforehand, keep rollback image 1.0 available.

### A5.2 Kubernetes (2 Marks)

| **Concept** | **Explanation** | **How it helps the portal** |
|---|---|---|
| **Pod** | Smallest unit: one or more containers sharing network and storage | One portal container (+ optional sidecar) |
| **Deployment** | Declares desired state (image, replicas) and performs rolling updates/rollbacks | Always keeps the declared number of pods running; new version 1.1 rolls out with zero downtime |
| **Service** | Stable virtual IP and DNS name that load-balances to matching pods (selected by labels) | Clients keep one address even when pods die and are replaced |
| **Replicas (ReplicaSet)** | Number of identical pods | Redundancy and capacity: 3 replicas survive 2 failures |
| **Requests and limits** | Request = guaranteed amount used by the scheduler for placement; limit = hard ceiling enforced by cgroups | One runaway pod cannot starve others; scheduler never overcommits nodes |
| **Horizontal Pod Autoscaler (HPA)** | Adds/removes replicas to keep a metric (e.g. average CPU 70% of request) at target; desired = ceil(current × currentMetric ÷ targetMetric) | Scales 3 → 12 pods as login traffic rises, back down afterwards |

**Scaling:** The HPA reads metrics every 15 s and adjusts the Deployment’s replicas; the scheduler places new pods on nodes with free requests; if no node has space the cluster autoscaler adds a cloud node.

**Recovery:** If a container crashes, the kubelet restarts it; if a pod is deleted or a node fails, the ReplicaSet controller creates a replacement pod elsewhere; readiness probes (`/health`) keep traffic away from pods that are not yet ready. This control loop – observe actual state, compare with desired state, correct – is what provides self-healing.

### Alternative Concept and Complexity Analysis – Fixed VMs Sized for Result Day Instead of Kubernetes with Autoscaler

Assume (from B4) that result day needs about 3 VMs of 4 vCPU / 16 GB (including headroom) and that a cloud VM of that size costs about **₹18 per hour** (illustrative price; check the provider’s calculator).

| **Criterion** | **Fixed VMs sized for result day** | **Kubernetes + autoscaler** | **Evaluation** |
|---|---|---|---|
| **Response time on result day** | Good if estimate is right; if traffic exceeds estimate, no automatic reaction – timeouts | HPA adds pods in ≈ 30–60 s; cluster autoscaler adds nodes in ≈ 2–5 min | Autoscaling handles surprises; fixed VMs only handle the planned peak |
| **Idle resources on other 364 days** | ≈ 99.9% idle (2 h of peak in 8,760 h; 2 result days = 4 h of 8,760 h = 0.05% busy) | Baseline 3 small pods; extra nodes released after the peak | Large waste with fixed VMs |
| **Annual cost (3 VMs)** | 3 × 8,760 h × ₹18 ≈ **₹4,73,000** | 2 baseline nodes: 2 × 8,760 × ₹18 = ₹3,15,360; burst 3 extra nodes × 6 h × 2 days × ₹18 ≈ ₹650; plus control-plane fee (~₹79,000 if managed). Using on-premises baseline instead: ≈ **₹650 + small fees** | Autoscaling is cheaper; hybrid with on-premises baseline is cheapest |

**Pseudocode – Annual Cost of Fixed versus Autoscaled Capacity**

```text
ALGORITHM AnnualCost

INPUT: price (per VM-hour), v_fixed, v_base, v_burst, hours_burst (per year)

OUTPUT: cost_fixed, cost_auto

BEGIN
    1. cost_fixed ← v_fixed × 8760 × price
    2. cost_auto  ← v_base × 8760 × price + v_burst × hours_burst × price
    3. RETURN cost_fixed, cost_auto
END
```

**Complexity:** Cost for fixed VMs is **O(V × T)** (peak capacity V for all T hours); autoscaled cost is **O(V_base × T + V_burst × t_peak)** with `t_peak ≪ T`, so it grows with average, not peak, demand.

**Reaction time:** Fixed O(1) (nothing happens); HPA O(log) rounds of doubling – each 15 s evaluation can at most double the replicas, so reaching 12 from 3 takes ⌈log₂(12/3)⌉ = 2 evaluation rounds plus pod start time.

