

#  PHASE 2 — Push Backend Image to ECR 

##  Objective

* Verify identity
* Get region securely
* Create ECR repo
* Login to ECR
* Tag image
* Push image

Everything using best practices.

---

#  Step 1 — Verify IAM Role

```bash
aws sts get-caller-identity
```

You should see:

```
"Arn": "arn:aws:sts::ACCOUNT_ID:assumed-role/aiops-eks-role/i-xxxxx"
```

This confirms:

* EC2 IAM role working
* Temporary credentials active
* No hardcoded secrets used

---

#  Step 2 — Get Region Securely (IMDSv2 Compatible)

Modern EC2 uses **IMDSv2**, so we must fetch metadata with a token.

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
-H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

export AWS_REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" \
http://169.254.169.254/latest/meta-data/placement/region)
```

Now verify:

```bash
echo $AWS_REGION
```

Example output:

```
ap-south-2
```

---

##  Explanation


* IMDSv1 allowed direct metadata access
* IMDSv2 requires session token
* Protects against SSRF attacks
* This is modern cloud security best practice

---

#  Step 3 — Get Account ID

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Verify:

```bash
echo $AWS_ACCOUNT_ID
```

---

#  Step 4 — Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name aiops/backend \
  --region $AWS_REGION
```

If already exists, AWS will return a message.

---

#  Step 5 — Login to ECR

```bash
aws ecr get-login-password --region $AWS_REGION \
| docker login --username AWS --password-stdin \
$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

Expected:

```
Login Succeeded
```

 Docker warns about credential storage — acceptable for demo.

---

#  Step 6 — Tag Image

```bash
docker tag aiops-backend:latest \
$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/aiops/backend:latest
```

---

#  Step 7 — Push Image

```bash
docker push \
$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/aiops/backend:latest
```

You already got:

```
latest: digest: sha256:...
```

 Push successful.

---

#  Final Trainer Summary for Phase 2

In your demo, summarize like this:

> “We built the image locally.
> We pushed it to ECR.
> Now it becomes a centralized artifact that any EKS node can pull.”

Then ask:

> Why not deploy directly from my local Docker?

Expected answer:

* Cluster nodes cannot access local images
* Need centralized registry
* Supports CI/CD pipelines
* Enables versioning

---

#  Phase 2 Complete

✔ IAM role validated
✔ Region fetched securely (IMDSv2)
✔ ECR repo created
✔ Docker login successful
✔ Image pushed to ECR

---

Now we move to:

#  PHASE 3 — Deploy to EKS

Where we will:

* Create namespace
* Write Deployment YAML (with probes)
* Add resource limits
* Create Service
* Expose via LoadBalancer
* Test via browser

Ready?
