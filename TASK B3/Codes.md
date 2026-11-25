
## B3.2 Measure (2 Marks)

### 1. Start-up Time Measurement

Measure the time from the container launch command to the first successful HTTP 200 response.

```bash
# Start-up time: from launch command to first HTTP 200
start=$(date +%s.%N)

docker run -d --name t1 -p 9003:8000 result-portal >/dev/null

until curl -sf localhost:9003/health >/dev/null; do
    sleep 0.1
done

echo "container start-up: $(echo "$(date +%s.%N) - $start" | bc) s"
```

**Note:** 
- VM: Time from `VBoxManage startvm --type headless` to the first HTTP 200 response.
- Bare metal: Time from Gunicorn launch to the first HTTP 200 response.

### 2. Memory Usage Measurement

```bash
# Bare metal
ps -o rss= -C gunicorn | awk '{s+=$1} END {print s/1024 " MB (bare metal)"}'

# VM
ssh vm 'free -m | awk "/Mem/{print \$3 \" MB used in guest\"}"'

# Container
docker stats --no-stream --format "{{.MemUsage}}" portal-c
```

### 3. Image and Disk Size Measurement

```bash
# Bare metal
du -sh app/venv

# VM
VBoxManage list hdds | grep Capacity
ls -lh ~/VirtualBox\ VMs/portalvm/*.vdi

# Container
docker images result-portal --format "{{.Size}}"
```

### 4. Throughput Measurement Using ApacheBench

Run the benchmark three times for each environment and take the median result.

```bash
# Bare metal
ab -n 20000 -c 100 -k http://localhost:9001/result/RA000007

# VM
ab -n 20000 -c 100 -k http://localhost:9002/result/RA000007

# Container
ab -n 20000 -c 100 -k http://localhost:9003/result/RA000007
```

---

## Complete the Code – deployment.yaml (TODO 1 and TODO 2 Completed)

### 1. Kubernetes Deployment and Service Configuration

Create a file named `deployment.yaml`:

```yaml
# Kubernetes Deployment and Service for the result portal.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-portal
spec:
  # TODO 1 (done): 3 replicas, select pods labelled app: result-portal
  replicas: 3
  selector:
    matchLabels:
      app: result-portal
  template:
    metadata:
      labels:
        app: result-portal
    spec:
      containers:
        - name: portal
          image: simats/result-portal:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          # TODO 2 (done): requests 250m CPU / 256Mi; limits 500m CPU / 512Mi
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
---
apiVersion: v1
kind: Service
metadata:
  name: result-portal
spec:
  type: LoadBalancer
  selector:
    app: result-portal
  ports:
    - port: 80
      targetPort: 8000
```



**Note:** `imagePullPolicy: IfNotPresent` was added so that kind/minikube use the locally loaded image; the rest is exactly as given.

---

## Recovery and Scaling – Commands and Expected Evidence

### 1. Check the Running Pods

```bash
kubectl get pods
```

**Expected Output:**
- 3 pods in `Running` status.
- Each pod should be `READY 1/1`.

### 2. Verify CPU and Memory Requests and Limits

```bash
kubectl describe pod <pod-name> | grep -A6 -E "Limits|Requests"
```

**Expected Output:**

```text
Limits:
  cpu:     500m
  memory:  512Mi
Requests:
  cpu:     250m
  memory:  256Mi
```

### 3. Test Automatic Pod Recovery

Delete one pod:

```bash
kubectl delete pod <pod-name>
```

Watch the pods:

```bash
kubectl get pods -w
```

**Expected Evidence:**
- The old pod enters `Terminating` status.
- A new pod with a different suffix enters `ContainerCreating`.
- The new pod reaches `Running` within approximately 5 seconds.
- The ReplicaSet restores the desired count of 3 pods.

### 4. Scale the Deployment Manually

```bash
kubectl scale deployment result-portal --replicas=6
```

### 5. Configure Automatic Scaling Under Load

```bash
kubectl autoscale deployment result-portal --cpu-percent=70 --min=3 --max=12
```

### 6. Forward the Service Port

```bash
kubectl port-forward svc/result-portal 8080:80 &
```

### 7. Generate Load Using ApacheBench

```bash
ab -n 200000 -c 200 http://localhost:8080/result/RA000007
```

### 8. Monitor Horizontal Pod Autoscaler

```bash
kubectl get hpa -w
```

**Expected Output:**

```text
TARGETS: 180%/70%
REPLICAS: 3 -> 6 -> ...
```

### 9. Monitor Pod Resource Usage

```bash
kubectl top pods
```

### 10. Output

- Kubernetes maintains the desired number of application replicas.
- If one pod is deleted, the ReplicaSet creates a replacement pod.
- Manual scaling increases the deployment to 6 replicas.
- The Horizontal Pod Autoscaler (HPA) monitors CPU utilisation and can scale the deployment between 3 and 12 replicas.
- The `kubectl top pods` command displays the resource usage of individual pods.

