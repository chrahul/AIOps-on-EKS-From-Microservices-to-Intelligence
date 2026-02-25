

#  AIOps on EKS — Phase 0 RunBook

## Title: Architecture Blueprint & Environment Preparation

---

#  Objective

Before building anything, we must:

* Validate AWS access
* Lock region
* Verify toolchain
* Define naming standards
* Ensure a clean AWS account
* Prepare structured working directory

This prevents unpredictable behavior later.

---

#  Target Architecture (High-Level)

```text
User
 ↓
LoadBalancer (EKS Service)
 ↓
Backend Pod
 ↓
CloudWatch Logs
 ↓
Lambda (Log Analyzer)
 ↓
Amazon Bedrock
 ↓
AI Log Summary
```

This lab demonstrates:

> "We deployed a microservice on EKS and used Bedrock to analyze its logs."

---

#  Environment Standards

All resources in this lab will use:

| Item         | Value              |
| ------------ | ------------------ |
| Region       | ap-south-2         |
| Cluster Name | aiops-cluster      |
| Nodegroup    | aiops-nodes        |
| Namespace    | aiops              |
| App Name     | backend            |
| ECR Repo     | aiops/backend      |
| Lambda       | aiops-log-analyzer |

---

#  Step 1 — Verify AWS Identity

```bash
aws sts get-caller-identity
```

Expected Output (masked example):

```json
{
  "UserId": "AROA************",
  "Account": "12345678",
  "Arn": "arn:aws:sts::12345678:assumed-role/lab-role/i-xxxxxxxx"
}
```

✔ Confirms:

* You are authenticated
* You are using the correct account

---

#  Step 2 — Set Default Region

Check current region:

```bash
aws configure get region
```

If empty, set region:

```bash
aws configure set region ap-south-2
```

Verify again:

```bash
aws configure get region
```

Expected:

```text
ap-south-2
```

 Ensures all AWS CLI calls use correct region.

---

#  Step 3 — Verify Toolchain

### Check kubectl

```bash
kubectl version --client --output=yaml
```

Expected:

* Client version installed

---

### Check eksctl

```bash
eksctl version
```

Expected:

* eksctl version printed

 Ensures EKS cluster creation will work.

---

#  Step 4 — Confirm Clean AWS State

Ensure no clusters exist:

```bash
aws eks list-clusters
```

Expected:

```json
{
  "clusters": []
}
```

 Prevents accidental conflicts.

---

#  Cost Guardrails (Important)

For this lab:

* Managed Node Group
* 1 node only
* Instance type: t3.small
* Log retention: 3 days
* Delete cluster after demo

This keeps lab affordable.

---

#  Step 5 — Create Lab Folder Structure

```bash
mkdir -p ~/aiops-lab/{phase1-app,phase2-container,phase4-k8s,phase6-lambda,docs}
cd ~/aiops-lab
ls -R
```

Expected structure:

```text
aiops-lab/
 ├── docs
 ├── phase1-app
 ├── phase2-container
 ├── phase4-k8s
 └── phase6-lambda
```

 Provides organized workspace.

---

#  Phase 0 — Key Learnings

1. Always lock region first.
2. Always verify AWS identity before creating infrastructure.
3. Standardize naming early.
4. Plan cost strategy before cluster creation.
5. Build a reproducible folder structure.

---

#  Phase 0 Complete

Environment is now ready for:

 

---


