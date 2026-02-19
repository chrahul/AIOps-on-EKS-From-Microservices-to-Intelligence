# Phase 4 — Enable CloudWatch Observability on EKS (Logs + Container Insights) 

## Goal

Enable the **amazon-cloudwatch-observability** EKS add-on so that:

* **Fluent Bit** ships container logs to **CloudWatch Logs**
* Log groups appear under:

  * `/aws/containerinsights/<cluster>/application`
  * `/aws/containerinsights/<cluster>/dataplane`
  * `/aws/containerinsights/<cluster>/host` (depending on config)

---

# Prerequisites

### 1) Tools

* `kubectl`
* `awscli`
* cluster access configured (`aws eks update-kubeconfig ...`)

### 2) Set variables

```bash
export AWS_REGION="ap-south-2"
export CLUSTER_NAME="aiops-cluster"
export ACCOUNT_ID="12345678"
```

---

# Step 1 — Verify OIDC provider exists (IRSA prerequisite)

```bash
aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

Now confirm the provider exists in IAM:

```bash
aws iam list-open-id-connect-providers
```

 You must see an entry like:
`arn:aws:iam::<account>:oidc-provider/oidc.eks.<region>.amazonaws.com/id/<id>`

---

# Step 2 — Create IAM Policy for CloudWatch Logs (Minimal)

Create a **minimal policy** (works for log group auto-creation + streams + puts):

```bash
cat > cw-observability-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    }
  ]
}
EOF
```

Create a managed policy:

```bash
aws iam create-policy \
  --policy-name EKS-CloudWatch-Observability-Logs \
  --policy-document file://cw-observability-policy.json
```

Copy the ARN printed (store it):

```bash
export CW_LOGS_POLICY_ARN="arn:aws:iam::<account>:policy/EKS-CloudWatch-Observability-Logs"
```

---

# Step 3 — Create IRSA Role for Fluent Bit ServiceAccount

We will attach:

* AWS managed `CloudWatchAgentServerPolicy` (common baseline)
* Our custom logs policy

### 3.1 Create trust policy (IRSA)

```bash
OIDC_ISSUER=$(aws eks describe-cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --query "cluster.identity.oidc.issuer" \
  --output text)

OIDC_PROVIDER=$(echo $OIDC_ISSUER | sed -e "s/^https:\/\///")

cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$OIDC_PROVIDER"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$OIDC_PROVIDER:aud": "sts.amazonaws.com",
          "$OIDC_PROVIDER:sub": "system:serviceaccount:amazon-cloudwatch:fluent-bit"
        }
      }
    }
  ]
}
EOF
```

### 3.2 Create role

```bash
aws iam create-role \
  --role-name EKS-FluentBit-IRSA-Role \
  --assume-role-policy-document file://trust-policy.json
```

### 3.3 Attach permissions

```bash
aws iam attach-role-policy \
  --role-name EKS-FluentBit-IRSA-Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name EKS-FluentBit-IRSA-Role \
  --policy-arn $CW_LOGS_POLICY_ARN
```

---

# Step 4 — Install/Enable the EKS Add-on with the IRSA Role

```bash
aws eks create-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name amazon-cloudwatch-observability \
  --region $AWS_REGION \
  --service-account-role-arn arn:aws:iam::$ACCOUNT_ID:role/EKS-FluentBit-IRSA-Role \
  --resolve-conflicts OVERWRITE
```

Wait until ACTIVE:

```bash
aws eks describe-addon \
  --cluster-name $CLUSTER_NAME \
  --addon-name amazon-cloudwatch-observability \
  --region $AWS_REGION \
  --query "addon.status" \
  --output text
```

 Must be `ACTIVE`

---

# Step 5 — Verify Pods

```bash
kubectl get pods -n amazon-cloudwatch --show-labels
kubectl get sa -n amazon-cloudwatch
kubectl describe sa fluent-bit -n amazon-cloudwatch
```

You should see annotation:
`eks.amazonaws.com/role-arn: arn:aws:iam::<acct>:role/EKS-FluentBit-IRSA-Role`

---

# Step 6 — **Critical Safety Step** (Prevents our problem)

## Choose ONE pattern

---

##  Pattern A (Recommended for Students / Guaranteed Smooth)

Grant CloudWatch Logs permission to **Node Instance Role** too.

> Why: In real life, Fluent Bit sometimes still uses node role credentials (IMDS). If node role is permitted, lab never breaks.

1. Find node role name:

```bash
# If you already know it, use it. Otherwise:
aws eks describe-nodegroup \
  --cluster-name $CLUSTER_NAME \
  --nodegroup-name <your-nodegroup> \
  --region $AWS_REGION \
  --query "nodegroup.nodeRole" \
  --output text
```

2. Attach CloudWatch Logs permissions:

```bash
aws iam attach-role-policy \
  --role-name <NODE_INSTANCE_ROLE_NAME> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

---

## Pattern B (Strict / Best Practice)

Force Fluent Bit to use IRSA by **disabling IMDS fallback**.

Patch daemonset to disable EC2 metadata credentials:

```bash
kubectl -n amazon-cloudwatch patch daemonset fluent-bit \
  --type='json' \
  -p='[
    {"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"AWS_EC2_METADATA_DISABLED","value":"true"}}
  ]'
```

Restart pods:

```bash
kubectl delete pod -n amazon-cloudwatch -l k8s-app=fluent-bit
```

 This makes Fluent Bit prefer IRSA instead of silently using node role.

---

# Step 7 — Restart Fluent Bit and Generate Logs

Restart fluent-bit:

```bash
kubectl delete pod -n amazon-cloudwatch -l k8s-app=fluent-bit
```

Generate app traffic (example):

```bash
APP_URL="http://<your-elb-dns>/api"
for i in {1..30}; do curl -s $APP_URL; echo; done
```

---

# Step 8 — Verify CloudWatch Log Groups (Success Criteria)

```bash
aws logs describe-log-groups \
  --region $AWS_REGION \
  --log-group-name-prefix "/aws/containerinsights" \
  --output json
```

 You should see log groups like:

* `/aws/containerinsights/<cluster>/application`
* `/aws/containerinsights/<cluster>/dataplane`

---

# Step 9 — If students still see AccessDenied (Bulletproof Debug)

## 9.1 Fluent Bit logs

```bash
kubectl logs -n amazon-cloudwatch -l k8s-app=fluent-bit --tail=50
```

If you see:
`CreateLogStream API responded with error='AccessDeniedException'`

Then do **the real truth check**:

## 9.2 CloudTrail (Identity truth)

```bash
aws cloudtrail lookup-events \
  --region $AWS_REGION \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateLogStream \
  --max-results 5 \
  --query 'Events[].CloudTrailEvent' \
  --output text | head -n 1
```

Look for the `userIdentity.arn`:

### Case A: It shows node role like:

`assumed-role/<NodeInstanceRole>/<instance-id>`
 Fix: attach CloudWatch Logs policy to node role (Pattern A)

### Case B: It shows IRSA role like:

`assumed-role/EKS-FluentBit-IRSA-Role/...`
 Fix: policy missing on IRSA role OR wrong trust `sub`/namespace/SA name

---

## What went wrong in our case (Key Learning)

Even though we configured **IRSA** and the pod showed:

* `AWS_ROLE_ARN=...`
* `AWS_WEB_IDENTITY_TOKEN_FILE=...`

CloudTrail proved Fluent Bit was actually calling CloudWatch Logs using the **Node Instance Role** (IMDS) and not IRSA.

 **Truth source = CloudTrail principal ARN**, not pod env.

Because the node role didn’t have permission (`logs:CreateLogStream`), we got:

* `AccessDeniedException` on CreateLogStream
* No log groups created

We fixed it by attaching `CloudWatchLogsFullAccess` to the **node instance role**, after which log groups immediately appeared.

**To prevent students from facing this:** we will implement one of these safe patterns:

* **Pattern A (Recommended for smooth lab):** Grant node role CloudWatch Logs permissions (least friction)
* **Pattern B (Stricter / best practice):** Force Fluent Bit to use IRSA by disabling IMDS fallback

This runbook includes both, but students can follow Pattern A for guaranteed success.


---
