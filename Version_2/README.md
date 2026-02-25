

## “We deployed a microservice on EKS and used Bedrock to analyze its logs.”

---

#  What This Demo Must Demonstrate

1. How logs flow from container → CloudWatch
2. How to fetch logs programmatically
3. How to call Bedrock
4. How to combine both
5. How AI transforms raw logs into readable diagnosis

That’s it.
---

#  Phase Structure 

---


Before coding:

Draw this clearly:

```text
User
 ↓
LoadBalancer
 ↓
Frontend Pod
 ↓
Backend Pod
 ↓
CloudWatch Logs
 ↓
Python Script
 ↓
Bedrock
 ↓
AI Summary
```

Explain:

* EKS runs containers
* CloudWatch stores logs
* Bedrock analyzes text
* Lambda or Python script glues them together

Students must understand flow first.

---
Good. Clear decisions.

You chose:

* **Lambda from the beginning**
* **Managed Node Group**
* Clean integration story
* Simple objective: *“We deployed a microservice on EKS and used Bedrock to analyze its logs.”*

That is a strong and realistic lab.

Now let me give you the architect-level blueprint before we run anything.

---

#  Final Architecture for This Demo

We are building this:

```text
User
  ↓
LoadBalancer Service
  ↓
Frontend Pod
  ↓
Backend Pod
  ↓
CloudWatch Logs
  ↓
Lambda (AI Analyzer)
  ↓
Amazon Bedrock
  ↓
AI Log Summary
```

No EC2 jump box for AI.
No local scripts.
Everything serverless on analysis side.

This is clean and modern.

---

#  Why Lambda from Beginning is Correct

Because:

* No SSH dependency
* No Python environment setup issues
* No pip issues
* No region mismatch problems
* Fully serverless AI analyzer
* Easy to demo from console

And architecturally correct.

---

# 🏗 Clean Lab Structure (Final Version)

We will build in this order:

---

## Phase 0 — Blueprint + IAM Planning

Before touching EKS:

* Decide naming convention
* Define IAM roles
* Define log group names
* Define cost guardrails
* Define region

---

## Phase 1 — Build Backend Microservice (With Logging)

Very small Flask app:

* `/health`
* `/error`
* Logs structured text
* Simulates 500

---

## Phase 2 — Containerize + Push to ECR

* Dockerfile
* ECR repo
* Push image

---

## Phase 3 — Create EKS Cluster (Managed Node Group)

Cost-optimized config:

* 1 node only
* t3.small (enough for demo)
* min=1 max=1 desired=1
* No overprovisioning

Important:
We do NOT start with 2 nodes.
Students don’t need HA for demo.

---

## Phase 4 — Deploy App to EKS

* Namespace
* Deployment (replicas=2)
* Service (LoadBalancer)
* Resource limits

Generate traffic.
Generate errors.

---

## Phase 5 — Enable CloudWatch Logging

Install:

* amazon-cloudwatch-observability add-on

Verify:

* `/aws/containerinsights/<cluster>/application`

---

## Phase 6 — Build Lambda AI Analyzer

Lambda:

* Fetch logs from CloudWatch
* Build prompt
* Call Bedrock
* Return summary

Test via:

* Lambda console
* API Gateway (optional)

---

