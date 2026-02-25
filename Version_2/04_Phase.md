

#  AIOps on EKS — Phase 4 RunBook v2

## Title: Deploy Backend Microservice to EKS and Expose via LoadBalancer

---

##  Objective

Deploy the backend container from ECR into EKS and expose it externally using a LoadBalancer service.

---

##  Architecture After Phase 4

```
User
  ↓
AWS LoadBalancer (ELB)
  ↓
Kubernetes Service (NodePort)
  ↓
Deployment (2 Pods)
  ↓
Backend Container
```

---

##  Step 1 — Create Namespace

```bash
kubectl create namespace aiops
```

Verify:

```bash
kubectl get ns
```

---

##  Step 2 — Create Deployment

Create file:

```bash
cat > ~/aiops-lab/phase4-k8s/backend-deployment.yaml <<'EOF'
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
          image: 123456789012.dkr.ecr.ap-south-2.amazonaws.com/aiops/backend:v1
          ports:
            - containerPort: 5000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
EOF
```

Apply:

```bash
kubectl apply -f ~/aiops-lab/phase4-k8s/backend-deployment.yaml
```

Verify pods:

```bash
kubectl get pods -n aiops -o wide
```

Expected:

* 2 pods
* STATUS = Running

---

##  Step 3 — Create Service (LoadBalancer)

Create file:

```bash
cat > ~/aiops-lab/phase4-k8s/backend-service.yaml <<'EOF'
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
EOF
```

Apply:

```bash
kubectl apply -f ~/aiops-lab/phase4-k8s/backend-service.yaml
```

Check service:

```bash
kubectl get svc -n aiops
```

Wait until EXTERNAL-IP is assigned.

---

##  Step 4 — Test Application

```bash
curl http://<EXTERNAL-DNS>/
curl http://<EXTERNAL-DNS>/error
```

Expected:

```
{"message":"Backend service running"}
{"error":"Something went wrong"}
```

---

##  Phase 4 Learning

1. EKS pulls images directly from ECR using node IAM role.
2. LoadBalancer service creates AWS ELB automatically.
3. Kubernetes handles pod scaling transparently.
4. Structured logs are now generated inside pods.
5. This is the foundation for AIOps log analysis.

---

##  Phase 4 Completion Criteria

Pods Running
Service Created
LoadBalancer DNS assigned
200 endpoint works
500 endpoint works

---

Next:

# Phase 5 — Enable Logging & Generate AI-Meaningful Data

High-level:

We must ensure logs reach CloudWatch.

