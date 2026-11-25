
# TASK B4 – Final Engineering Decision (5 Marks)

## B4.1 Implement a Practical Extension (2 Marks)

The university will also host its Learning Management System (LMS) and online examinations on the same infrastructure and wants result day handled without buying new servers.



### 5. Verify LMS Resource Limits

```bash
kubectl describe pod -l app=lms | grep -A6 Limits
```

### 6. Check Docker Compose Resource Usage

```bash
docker stats --no-stream
```

Each service should show its own resource limit.

### 7. Output and Evidence

Both services run at the same time, each with its own limits. Hitting the portal with ApacheBench (`ab`) does not exceed the portal's CPU limit, and the LMS stays responsive. This demonstrates resource isolation between services.

**Evidence:** Both services running simultaneously with independent resource limits.



---

## B4.2 Estimate the Capacity (1 Mark)

Using the per-container throughput measured at the 0.5-CPU limit (`r ≈ 150 req/s per portal pod`, reference value – substitute your measured value) and the peak requirement from the case study:

### 1. Capacity Calculation

| Step | Calculation | Result |
|---|---|---:|
| Total requests | 150,000 students × 10 requests each | 1,500,000 |
| Average rate over 2 hours | 1,500,000 ÷ 7,200 s | ≈ 208 req/s |
| Peak rate (burst factor 5) | 208 × 5 | ≈ 1,040 req/s |
| Usable throughput per pod (70% target utilisation) | 150 × 0.7 | 105 req/s |
| Pods required | 1,040 ÷ 105 = 9.9 → round up | 10 pods |
| Safety spare (N+2) | 10 + 2 | 12 pods |
| Resources for 12 pods | 12 × 0.5 CPU, 12 × 512 MiB (limits) | 6 vCPU, 6 GiB |
| If virtual machines were used | VM (2 vCPU) ≈ 560 req/s → 1,040 ÷ (560 × 0.7) = 2.65 → 3 VMs (+1 spare = 4) | 4 VMs: 8 vCPU, ≈ 8 GB + 4 guest OSs (≈ 2.4 GB) |

### 2. Pseudocode – Capacity Estimate

```text
ALGORITHM EstimateCapacity

INPUT : students, req_per_student, window_s, burst, r (req/s per unit), util, spare

OUTPUT: units

BEGIN
    1. avg  ← (students × req_per_student) ÷ window_s
    2. peak ← avg × burst
    3. units ← CEILING(peak ÷ (r × util)) + spare
    4. RETURN units
END
```

### 3. Complexity Analysis

- **Arithmetic complexity:** O(1) — a fixed number of arithmetic operations.
- **Capacity growth:** The number of units grows linearly with the number of users, O(students).
- **Throughput relationship:** Required units grow inversely with per-unit throughput `r`.

**Conclusion:** 12 container pods (6 CPU, 6 GiB) fit easily inside one modest server (e.g. 16 cores / 64 GB) together with the LMS, which is why no new servers are required.

---

## B4.3 Production Decision (2 Marks)

### 1. Illustrative Yearly Cost Comparison

**Assumptions:**
- One result day per semester = 2 per year.
- Cloud VM with 4 vCPU / 16 GB ≈ ₹18/hour.
- Existing servers are already paid for.
- Check real quotations before use.

| Option | Capacity / Speed | Yearly Cost (Illustrative) | Data Control | Verdict |
|---|---|---|---|---|
| **1. Own servers + VMs** | 4 VMs fixed; new VM takes minutes; no autoscaling; ≈ 8 GB+ RAM overhead. | ≈ ₹0 new hardware; admin effort high. | Full | Works, but slow to scale and wasteful. |
| **2. Own servers + containers + Kubernetes** | 12 pods ≈ 6 vCPU; scaling in ≈ 30–60 s; self-healing. | ≈ ₹0 new hardware; electricity + staff time. | Full | Meets measured need; limited by the server room if load is 3–5× higher than estimated. |
| **3. Public cloud only (autoscaled)** | Practically unlimited; nodes in ≈ 2–5 min. | ≈ ₹3.2 lakh baseline + control plane + database ≈ ₹4–5 lakh/year (fixed VMs ≈ ₹4.7 lakh). | Provider holds data (contracts, region, compliance). | Expensive for 4 busy hours a year. |
| **4. Hybrid: own Kubernetes baseline + cloud burst / disaster recovery** | Own 12 pods + cloud node group scaled to 12–24 pods only on result day. | ≈ ₹650 compute for 2 × 6 h × 3 nodes + load balancer/transfer/DR storage ≈ ₹10,000–25,000. | Student records stay on-premises; only stateless pods and cached content go to the cloud. | Recommended. |

### 2. Final Engineering Decision

**Decision: Option 4 – Hybrid Infrastructure**

The university should run the portal and LMS as containers on Kubernetes on its own servers, with a pre-arranged public-cloud burst/backup (hybrid).

### 3. Justification Based on Measurements

1. **Containers, not VMs, for the web tier:** The container reached ≈ 97% of bare-metal throughput and starts in ≈ 2 s against ≈ 48 s for a VM. Each VM costs ≈ 600 MB of guest-OS memory, while 12 pods fit in 6 GiB.

2. **Own servers:** The capacity estimate shows result day needs only ≈ 6 vCPU / 6 GiB for the portal – well within existing hardware – so no new purchase is needed, as the university requires. Student data and examination data remain under university control.

3. **Kubernetes:** Self-healing (pod replaced in ≈ 5 s) and HPA scaling from 3 → 12 pods address the original crash-on-result-day problem.

4. **Cloud burst:** Provides insurance against a mis-estimated peak (the burst factor is an assumption) and against a failure of the server room. It costs only about ₹650 of compute per year when used for two result days, compared with ₹3–5 lakh for running the cloud all year.

5. **Database isolation:** Keep the database in a VM (or managed service) with snapshots and backups for stronger isolation; stateless pods connect to it.

### 4. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Unexpected high traffic | Load test two weeks before result day. |
| Sudden traffic spike | Pre-scale 24 hours before. |
| Static PDF traffic | Use a CDN for static PDFs. |
| DNS changes during failover | Lower DNS TTL. |
| Faulty application release | Keep rollback image 1.0 retained. |
| Performance or application errors | Monitor with Prometheus alerts (CPU, 5xx rate, pod restarts). |

---

## Alternative Concept and Complexity Analysis – Public Cloud Only, Paying Per Hour

### 1. Comparison: Hybrid vs Public Cloud Only

| Criterion | Own Servers + Kubernetes (Hybrid) | Public Cloud Only | Evaluation |
|---|---|---|---|
| Cost over a year (2 result days) | ≈ ₹10,000–25,000 variable + existing fixed costs. | Autoscaled: baseline 2 nodes × 8,760 h × ₹18 = ₹3,15,360 + burst ₹650 + control plane/DB; or 3 fixed VMs ≈ ₹4,73,000. Pure pay-per-use is only cheap if you can scale to zero (not possible for a database). | Hybrid much cheaper because the baseline hardware already exists. |
| Time to add capacity | Own: seconds for pods up to the server’s limit; new hardware takes weeks. | Minutes (cloud nodes), effectively unlimited. | Cloud is superior for sudden large growth. |
| Who controls the student data | The university. | Cloud provider holds it (shared-responsibility model; data-protection law, region, vendor lock-in). | Hybrid keeps sensitive data in-house. |

### 2. Pseudocode – Yearly Cost of Pay-Per-Hour Cloud

```text
ALGORITHM CloudYearCost

INPUT : price, base_nodes, burst_nodes, burst_hours_per_year

OUTPUT: yearly_cost

BEGIN
    1. yearly_cost ← price × (base_nodes × 8760 + burst_nodes × burst_hours_per_year)
    2. RETURN yearly_cost
END
```

### 3. Complexity Analysis

- **Yearly cloud cost:** O(base × T + burst × t_peak) — dominated by the always-on baseline, so it is linear in time even when the real load exists for only 4 hours.
- **Own hardware cost:** O(1) per year once bought (fixed).
- **Time to add capacity:** Cloud O(1) rounds (minutes); own hardware O(procurement time).

### 4. Final Conclusion

The hybrid approach provides the best balance of performance, cost, scalability, recovery, and data control. The university can use its existing servers for normal operations and planned result-day capacity, while public cloud resources provide additional capacity and disaster recovery when required.

