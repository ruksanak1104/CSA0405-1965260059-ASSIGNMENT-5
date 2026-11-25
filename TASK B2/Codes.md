
### Complete the Code – Dockerfile (TODO 1 – TODO 3 Completed)

**Dockerfile:**

```dockerfile
# Container image for the campus result-portal web service (a small Flask app).

FROM python:3.12-slim

# TODO 1 (done): working directory
WORKDIR /app

# Install the dependencies first so that this layer is cached between builds.
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# TODO 2 (done): copy the source and run as non-root user "portal"
COPY . .
RUN useradd --create-home --shell /usr/sbin/nologin portal \
    && chown -R portal:portal /app

USER portal

# TODO 3 (done): document the port and start with gunicorn
EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

**.dockerignore:**

```text
venv/
__pycache__/
.git/
*.db
```

The `.dockerignore` file excludes unnecessary files and folders so that the Docker build context remains small.

### Step 1 – Build and Run, Mapping the Port

**Commands:**

```bash
cd app

# Build the Docker image
docker build -t result-portal .

# Display the image and record its size
docker images result-portal
# [SCREENSHOT: Docker image and SIZE]

# Run the container and map port 8000
docker run -d --name portal1 -p 8000:8000 result-portal

# Verify that the container is running
docker ps

# Test the health endpoint
curl http://localhost:8000/health
# Expected: {"status":"ok",...}

# Verify that the application runs as the non-root user
docker exec portal1 whoami
# Expected: portal
```

**Expected Evidence:**
- Docker image `result-portal` is built successfully.
- Image size is displayed using `docker images result-portal`.
- Container `portal1` appears as running in `docker ps`.
- Health endpoint returns `{"status":"ok",...}`.
- `docker exec portal1 whoami` displays `portal`.


## Step 2 – Docker Compose with a Volume for Database Data

### 1. Docker Compose Configuration

Create a file named `docker-compose.yml`:

```yaml
# docker-compose.yml
services:
  portal:
    build: ./app
    ports: ["8000:8000"]
    environment:
      PORTAL_DB: /data/results.db
    volumes:
      - portal-data:/data            # SQLite file survives container deletion
    depends_on: [db]
    deploy:
      resources:
        limits: { cpus: "0.50", memory: 256M }

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: example_change_me
      POSTGRES_DB: results
    volumes:
      - pgdata:/var/lib/postgresql/data   # database files persist
    deploy:
      resources:
        limits: { cpus: "0.50", memory: 256M }

volumes:
  portal-data:
  pgdata:
```

### 2. Build and Run the Containers

```bash
docker compose up -d --build
docker compose ps
```

### 3. Seed the Database

```bash
curl http://localhost:8000/seed
```

### 4. Stop and Remove the Containers

```bash
docker compose down
```

The containers are removed, but the named volumes are kept.

### 5. Restart the Containers

```bash
docker compose up -d
```

### 6. Verify Volume Persistence

```bash
curl http://localhost:8000/result/RA000007
```

The data is still available, proving that the volume preserves data even after the containers are removed and recreated.

### 7. Output

The demo application stores its results in the SQLite file on the `portal-data` volume. The Postgres service, with its own `pgdata` volume, demonstrates the production pattern of using a separate database container whose data outlive the container.


## Step 3 – Limit CPU and Memory and Show the Effect

### 1. Run the Container with CPU and Memory Limits

```bash
docker run -d --name limited --cpus 0.5 --memory 256m -p 8001:8000 result-portal
```

### 2. Generate Load on the Application

```bash
ab -n 20000 -c 50 http://localhost:8001/health &
```

### 3. Monitor Container Resource Usage

```bash
docker stats --no-stream limited
```

This command displays the CPU and memory usage of the running container.

### 4. Demonstrate the Memory Limit (OOM Kill)

```bash
docker run --rm --memory 50m python:3.12-slim python -c "x = bytearray(200*1024*1024)"
```

Check the exit code:

```bash
echo $?
```

**Expected output:** `137` indicates that the process was killed by `SIGKILL`, typically due to an out-of-memory (OOM) event.

### 5. Verify the CPU Limit Using cgroups

```bash
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' limited).scope/cpu.max
```

**Expected output:**

```text
50000 100000
```

This represents a CPU quota of 50,000 microseconds per 100,000-microsecond period, equivalent to a limit of 0.5 CPU.




