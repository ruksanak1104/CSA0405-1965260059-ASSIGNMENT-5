### 1. Docker Compose Version

Add the following LMS service to the existing Docker Compose file from B2:

```yaml
lms:
  image: nginx:1.27-alpine
  ports: ["8081:80"]
  deploy:
    resources:
      limits: { cpus: "1.00", memory: 512M }
```

### 2. Kubernetes Version

Create a file named `lms.yaml` with its own resource requests and limits:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lms
spec:
  replicas: 2
  selector:
    matchLabels: { app: lms }
  template:
    metadata:
      labels: { app: lms }
    spec:
      containers:
        - name: lms
          image: nginx:1.27-alpine
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 256Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: lms
spec:
  selector: { app: lms }
  ports: [{ port: 80, targetPort: 80 }]
```

### 3. Deploy the LMS

```bash
kubectl apply -f lms.yaml
```

### 4. Verify Both Services

```bash
kubectl get deploy,pods,svc
```

**Expected Output:**
- Portal: 3 pods in `Running` status.
- LMS: 2 pods in `Running` status.
