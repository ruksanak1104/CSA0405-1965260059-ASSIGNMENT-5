
## CPU and Memory Usage Comparison

| Container | CPU % | MEM USAGE / LIMIT | MEM % |
|---|---:|---:|---:|
| `limited` (`--cpus 0.5 --memory 256m`) under load | ≈ 50% (capped – the container cannot use more than half a core) | 62 MiB / 256 MiB | 24% |
| `portal1` (no limits), same load | ≈ 190% (uses ≥ 2 cores) | 78 MiB / 7.7 GiB | 1% |

---

## Step 4 – Inspect Layers and Reduce the Image Size

### 1. Inspect Docker Image Layers

Run the following command:

```bash
docker history result-portal
```

**Expected Output:**

```text
IMAGE   CREATED BY                                      SIZE
<id>    CMD ["gunicorn" ...]                            0B
<id>    EXPOSE 8000                                     0B
<id>    USER portal                                     0B
<id>    RUN useradd ... && chown -R portal ...          ~3 kB
<id>    COPY . .                                        ~10 kB
<id>    RUN pip install --no-cache-dir ...              ~16 MB
<id>    COPY requirements.txt .                         ~40 B
<id>    WORKDIR /app                                    0B
<id>    python:3.12-slim base layers                     ~130 MB
```

### 2. Optimisation – Multi-Stage Build

Build tools stay in the first stage, and only the installed packages are copied into the final image.

Create a file named `Dockerfile.multi`:

```dockerfile
# Dockerfile.multi

FROM python:3.12-slim AS build

WORKDIR /w

COPY requirements.txt .

RUN pip install --no-cache-dir --prefix=/install -r requirements.txt


FROM python:3.12-slim

WORKDIR /app

COPY --from=build /install /usr/local

COPY . .

RUN useradd --create-home --shell /usr/sbin/nologin portal && chown -R portal:portal /app

USER portal

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

### 3. Build the Optimised Image

```bash
docker build -f Dockerfile.multi -t result-portal:multi .
```

### 4. Compare Image Sizes

```bash
docker images | grep result-portal
```

---

## Step 5 – What Each Step Shows About How Containers Work

| Step | What It Demonstrates |
|---|---|
| `docker run -p 8000:8000` | Network namespace + port mapping (NAT): the container has its own network stack; the host forwards a port into it. |
| `whoami → portal` | User isolation and least privilege: root inside a container is still root-like for the kernel, so a non-root user limits damage if the app is compromised. |
| Compose + volumes | Containers are disposable; state must live in volumes (storage outside the container’s writable layer). |
| `docker stats` with `--cpus` / `--memory` | cgroups enforce resource limits; CPU is throttled, memory overrun causes OOM-kill of the container only (exit 137). |
| `docker history` | Image = stack of read-only layers; unchanged layers are cached/shared, so rebuilds and pulls are fast. |
| Slim / multi-stage | Smaller image → faster pull to new nodes, fewer packages to patch, smaller attack surface. |

---

## Alternative Concept and Complexity Analysis – Ubuntu Base Image or Multi-Stage Build

### Comparison of Different Image-Building Approaches

| Criterion | `python:3.12-slim` (chosen) | `ubuntu:24.04` + apt install python3-pip | Multi-stage on slim |
|---|---:|---:|---:|
| Image size | ≈ 150 MB | ≈ 450–550 MB | ≈ 135–145 MB |
| Build time (cold) | ≈ 25 s | ≈ 60–90 s (apt update/install) | ≈ 30 s (cold), ≈ 3 s (cached) |
| Pull time to a new server (100 Mbit/s) | ≈ 12 s | ≈ 40 s | ≈ 11 s |
| Packages that may contain vulnerabilities (docker scout / trivy) | ≈ 90 packages | ≈ 250+ packages | ≈ 85 packages (no compilers or build tools) |

### Pseudocode – Time to Deploy a New Image on n Servers

```text
ALGORITHM DeployImage

INPUT : n (servers), size (image MB), bw (bandwidth MB/s), t_start

OUTPUT: total_time

BEGIN
    1. t_pull ← size ÷ bw                  // only layers not already present
    2. total_time ← t_pull + t_start       // servers pull in parallel
    3. RETURN total_time
END
```

### Complexity Analysis

- **Pull time:** O(size) — linear in image size, so a 500 MB image takes ≈ 3.5× longer than 145 MB.
- **Parallel deployment:** With parallel pulls, the total for n servers is O(size), not O(n·size), assuming registry bandwidth permits; otherwise, O(n·size/bw_registry).
- **Build time:** With layer caching, build time is O(changed layers).
- **Security:** The number of vulnerable packages, and therefore attack surface, is O(p) in the packages installed.

