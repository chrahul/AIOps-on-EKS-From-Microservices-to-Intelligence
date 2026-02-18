# AIOps-on-EKS-From-Microservices-to-Intelligence
AIOps on EKS – From Microservices to Intelligence


#   GOAL



1. Deploy a microservice app on EKS (from scratch)
2. Generate logs
3. Query those logs using Bedrock
4. Show AI-powered DevOps troubleshooting

---

#  The Story

> “We built a microservice.
> We deployed it on Kubernetes (EKS).
> We generated logs.
> Then we used AI to analyze those logs.”


---

# HIGH-LEVEL ARCHITECTURE 

```
User
  ↓
LoadBalancer Service
  ↓
Frontend Pod
  ↓
Backend API Pod
  ↓
CloudWatch Logs
  ↓
Python AI Analyzer
  ↓
AWS Bedrock
  ↓
Natural Language Diagnosis
```

---

#  HIGH LEVEL PLAN

We will build this in 5 clean phases.

---

# PHASE 1 — Write Our Own Microservice

Keep it simple.

### App 1: Frontend (Flask or Node)

* Returns simple HTML
* Calls backend API

### App 2: Backend API

* Has /health endpoint
* Logs every request
* Can simulate error

This gives:

* Microservices
* Inter-service communication
* Logs

---

# PHASE 2 — Containerize

* Write Dockerfile for both
* Build images
* Push to ECR

Now you control the images.

---

# PHASE 3 — Deploy to EKS

* Namespace
* Deployment YAML
* Service YAML
* LoadBalancer type
* Replicas = 2
* Resource limits set

Then verify:

* Pods running
* Services exposed
* Logs generated

---

# PHASE 4 — Enable EKS Logging

* Enable control plane logging
* Or log app logs to CloudWatch

We must deliberately generate:

* Error logs
* 500 responses
* Crash loops (optional)

This gives AI something to analyze.

---

# PHASE 5 — Build Our Own AI Log Analyzer

Instead of copying GitHub:

We write our own minimal Python script:

1. Fetch logs from CloudWatch
2. Format logs into prompt
3. Call Bedrock
4. Print summary

That’s it.

No magic.



