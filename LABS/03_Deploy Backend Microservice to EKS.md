## PHASE 3 — Deploy Backend Microservice to EKS 

Goal: Deploy our Flask backend (already pushed to ECR) on EKS, expose it via LoadBalancer, validate using `curl`, and view logs.

---

#  Prerequisites (Before Phase 3)

You must have:

* EKS cluster running (2 nodes Ready)
* `kubectl` installed on your EC2 machine
* kubeconfig set for the cluster
* Backend image available in ECR, example:

  * `12345678.dkr.ecr.ap-south-2.amazonaws.com/aiops/backend:v2`

---

# STEP 0 — Confirm Cluster Access (Very Important)

### 0.1 Check nodes

```bash
kubectl get nodes
```

Expected:

* 2 nodes
* STATUS = `Ready`

### 0.2 Check Kubernetes connectivity

```bash
kubectl cluster-info
```

If this works, you’re connected to the right cluster.

---

# STEP 1 — Create Namespace (Isolated Environment)

We keep everything in a separate namespace called `aiops`.

```bash
kubectl create namespace aiops
```

Verify:

```bash
kubectl get ns
```

You should see:

* `aiops  Active`

---

# STEP 2 — Create Deployment YAML

Create a folder (optional but clean):

```bash
mkdir -p k8s && cd k8s
```

Create file:

```bash
nano backend-deployment.yaml
```

Paste this YAML (IMPORTANT: replace image with your ECR URL):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: aiops
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: 803103365620.dkr.ecr.ap-south-2.amazonaws.com/aiops/backend:v2
        ports:
        - containerPort: 5000
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Why probes are important :**

* `readinessProbe` decides when Pod should receive traffic
* `livenessProbe` restarts Pod if it becomes unhealthy

---

# STEP 3 — Apply Deployment

```bash
kubectl apply -f backend-deployment.yaml
```

Check pods:

```bash
kubectl get pods -n aiops
```

Expected:

* 2 pods
* STATUS = `Running`
* READY = `1/1`

### Optional (shows which node each pod is running on)

```bash
kubectl get pods -n aiops -o wide
```

---

# STEP 4 — Create Service YAML (Expose App)

Create file:

```bash
nano backend-service.yaml
```

Paste:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: aiops
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 5000
```

Apply:

```bash
kubectl apply -f backend-service.yaml
```

---

# STEP 5 — Get LoadBalancer URL

```bash
kubectl get svc -n aiops
```

Example output:

* `EXTERNAL-IP = a716d58e....elb.amazonaws.com`

Copy that `EXTERNAL-IP` (DNS name).

---

# STEP 6 — Verify Application Using curl (Most Important)

Let’s store the URL in variable (recommended):

```bash
export APP_URL="http://a716d58ec973741aeb903089598a50a2-203950.ap-south-2.elb.amazonaws.com"
```

### 6.1 Check root `/`

```bash
curl $APP_URL/
```

Expected:

```json
{"message":"Welcome to AIOps Backend API","try_these_routes":["/api","/health"]}
```

### 6.2 Check health

```bash
curl $APP_URL/health
```

Expected:

```json
{"status":"healthy"}
```

### 6.3 Check API endpoint

```bash
curl $APP_URL/api
```

Expected (most of the time):

```json
{"message":"Backend response successful"}
```

Sometimes (because we simulated failure):

```json
{"error":"Something went wrong"}
```

This is expected and useful for logs + AIOps.

---

# STEP 7 — Generate Traffic (To Create Logs)

Run 20 calls:

```bash
for i in {1..20}; do
  curl -s $APP_URL/api
  echo
done
```

You will see some success and some failures (500).

---

# STEP 8 — Check Logs (Kubernetes Logs)

### 8.1 Check logs from all backend pods

```bash
kubectl logs -n aiops -l app=backend
```

You should see:

* `INFO API endpoint called`
* `ERROR Simulated backend failure`
* `GET /api ... 200`
* `GET /api ... 500`

### 8.2 Check logs from a specific pod

```bash
kubectl get pods -n aiops
```

Pick a pod name and run:

```bash
kubectl logs -n aiops <pod-name>
```

---

#  Phase 3 Success Checklist

You are done with Phase 3 if:

* `kubectl get nodes` shows Ready nodes
* `kubectl get pods -n aiops` shows 2 Running pods
* `kubectl get svc -n aiops` shows external LoadBalancer DNS
* `curl $APP_URL/health` returns `{"status":"healthy"}`
* `curl $APP_URL/api` returns success sometimes and failure sometimes
* `kubectl logs -n aiops -l app=backend` shows INFO and ERROR logs

---

