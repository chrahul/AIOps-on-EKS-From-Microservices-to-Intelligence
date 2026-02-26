

#  Phase 5 Runbook

## Enable CloudWatch Observability on EKS (Production-Grade IRSA Setup)

---

#  Goal

Enable:

* Container logs → CloudWatch Logs
* Container Insights → Metrics
* Log groups auto-created under:

```
/aws/containerinsights/<cluster>/application
/aws/containerinsights/<cluster>/dataplane
/aws/containerinsights/<cluster>/host
```

---

#  Environment

Cluster: `aiops-cluster`
Region: `ap-south-2`
K8s Version: `1.29`

---

# Step 1 — Verify OIDC Provider

```bash
aws eks describe-cluster \
  --name aiops-cluster \
  --region ap-south-2 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

If missing:

```bash
eksctl utils associate-iam-oidc-provider \
  --region ap-south-2 \
  --cluster aiops-cluster \
  --approve
```

---

# Step 2 — Create Minimal Logs Policy

`cw-observability-policy.json`

```json
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
```

Create policy:

```bash
aws iam create-policy \
  --policy-name EKS-CloudWatch-Observability-Logs \
  --policy-document file://cw-observability-policy.json
```

---

# Step 3 — Create IRSA Role

## 3.1 Trust Policy

Important:

```
system:serviceaccount:amazon-cloudwatch:cloudwatch-agent
```

Create role:

```bash
aws iam create-role \
  --role-name EKS-FluentBit-IRSA-Role \
  --assume-role-policy-document file://trust-policy.json
```

Attach policies:

```bash
aws iam attach-role-policy \
  --role-name EKS-FluentBit-IRSA-Role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

aws iam attach-role-policy \
  --role-name EKS-FluentBit-IRSA-Role \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/EKS-CloudWatch-Observability-Logs
```

---

# Step 4 — Install Add-on

Correct add-on name:

```
amazon-cloudwatch-observability
```

```bash
aws eks create-addon \
  --cluster-name aiops-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region ap-south-2 \
  --service-account-role-arn arn:aws:iam::<ACCOUNT_ID>:role/EKS-FluentBit-IRSA-Role \
  --resolve-conflicts OVERWRITE
```

Verify:

```bash
aws eks describe-addon \
  --cluster-name aiops-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region ap-south-2 \
  --query "addon.status" \
  --output text
```

Must be:

```
ACTIVE
```

---

# Step 5 — Verify Pods

```bash
kubectl get pods -n amazon-cloudwatch
```

Expected:

* amazon-cloudwatch-observability-controller-manager
* cloudwatch-agent
* fluent-bit

---

# Step 6 — Restart Fluent Bit

```bash
kubectl delete pod -n amazon-cloudwatch -l k8s-app=fluent-bit
```

---

# Step 7 — Generate Traffic

```bash
for i in {1..20}; do
  curl <ELB_URL>/
  curl <ELB_URL>/error
done
```

---

# Step 8 — Verify Log Groups

```bash
aws logs describe-log-groups \
  --region ap-south-2 \
  --log-group-name-prefix "/aws/containerinsights"
```

Success Criteria:

```
/aws/containerinsights/aiops-cluster/application
/aws/containerinsights/aiops-cluster/dataplane
/aws/containerinsights/aiops-cluster/host
```

---

#  Critical Learning (Enterprise Level)

### Truth Source = CloudTrail Principal ARN

If logs don’t appear:

Check:

```bash
aws cloudtrail lookup-events \
  --region ap-south-2 \
  --lookup-attributes AttributeKey=EventName,AttributeValue=CreateLogStream \
  --max-results 5
```

If you see:

```
assumed-role/<NodeInstanceRole>/
```

Then node role is being used.

If:

```
assumed-role/EKS-FluentBit-IRSA-Role
```

Then IRSA is working.

---

#  Phase 5 Status

| Component              | Status   |
| ---------------------- | -------- |
| IRSA                   | YES        |
| Add-on                 | Yes ACTIVE |
| Logs Shipping          | Yes        |
| Container Insights     | yes        |
| EKS 1.29 Compatible    | yes        |
| Production-Grade Setup | yes        |

---

You now have:

```
/aws/containerinsights/aiops-cluster/application
/aws/containerinsights/aiops-cluster/dataplane
/aws/containerinsights/aiops-cluster/host
```

That means:

CloudWatch Agent is running
Fluent Bit is shipping logs
IRSA role works
No AccessDenied
Container Insights enabled
EKS 1.29 compatibility confirmed


