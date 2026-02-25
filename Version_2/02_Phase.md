# Phase 2 — RunBook v2

**Title: Containerizing the Backend and Publishing to Amazon ECR**

---

##  Objective

Convert backend service into a production-ready Docker image and push it to Amazon ECR so it can be deployed on EKS.

---

##  Architecture Context

After Phase 2:

```
Backend Code
    ↓
Docker Image
    ↓
Amazon ECR
```

EKS will later pull this image from ECR.

---

##  Step 1 — Create requirements.txt

```bash
cd ~/aiops-lab/phase1-app

cat > requirements.txt <<'EOF'
flask==3.1.3
EOF
```

---

##  Step 2 — Create Dockerfile

```bash
cat > Dockerfile <<'EOF'
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

---

##  Step 3 — Build Docker Image

```bash
docker build -t aiops-backend:v1 .
```

Verify:

```bash
docker images
```

---

##  Step 4 — Test Container Locally

```bash
docker run -d -p 5000:5000 --name aiops-test aiops-backend:v1
```

Test:

```bash
curl http://localhost:5000/
curl http://localhost:5000/error
docker logs aiops-test
```

---

##  Step 5 — Login to ECR

```bash
aws ecr get-login-password --region ap-south-2 | \
docker login --username AWS --password-stdin \
123456789012.dkr.ecr.ap-south-2.amazonaws.com
```

(Note: Account ID masked in RunBook.)

---

##  Step 6 — Tag Image for ECR

```bash
docker tag aiops-backend:v1 \
123456789012.dkr.ecr.ap-south-2.amazonaws.com/aiops/backend:v1
```

---

##  Step 7 — Push Image

```bash
docker push \
123456789012.dkr.ecr.ap-south-2.amazonaws.com/aiops/backend:v1
```

Verify:

```bash
aws ecr list-images \
  --repository-name aiops/backend \
  --region ap-south-2
```

---

##  Phase 2 Learning

1. Always test container locally before pushing.
2. Use slim base images for smaller attack surface.
3. Keep dependency versions pinned.
4. ECR repository naming matters (`aiops/backend` ≠ `aiops-backend`).
5. Tag images explicitly (never rely only on `latest`).

---

##  Phase 2 Completion Criteria

 Image built
 Container tested
 Image tagged
 Image pushed to ECR
 Verified in ECR

---

#  Next Phase

# PHASE 3 — Deploy to EKS (Managed Node Group)


