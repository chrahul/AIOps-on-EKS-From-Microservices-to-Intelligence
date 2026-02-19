

#  PHASE 4 – Container Logs to CloudWatch (Fluent Bit)

This phase ensures:

> Application logs + node logs from EKS → CloudWatch Logs
> No AccessDenied
> No broken IRSA
> No missing permissions

This version includes **all the learnings from our debugging session**

---

#  PHASE 4 – Enable Container Logs 

---

#  Objective

Send logs from:

* Application Pods
* Kubelet / Container runtime
* Node host logs

To:

```
CloudWatch Logs → /aws/containerinsights/<cluster-name>/*
```

---

#  Step 1 – Verify OIDC Provider Exists (Mandatory for IRSA)

```bash
aws eks describe-cluster \
  --name aiops-cluster \
  --region ap-south-2 \
  --query "cluster.identity.oidc.issuer" \
  --output text
```

You must see something like:

```
https://oidc.eks.ap-south-2.amazonaws.com/id/XXXXXXXX
```

If not — associate OIDC first:

```bash
eksctl utils associate-iam-oidc-provider \
  --cluster aiops-cluster \
  --region ap-south-2 \
  --approve
```

---

#  Step 2 – Create IAM Policy for CloudWatch Logs

Create file:

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

Create policy:

```bash
aws iam create-policy \
  --policy-name EKS-CloudWatch-Logs-Policy \
  --policy-document file://cw-observability-policy.json
```

Save the ARN.

---

#  Step 3 – Attach Policy to Node Instance Role

 IMPORTANT LEARNING FROM OUR ISSUE

Fluent Bit was using the **NodeInstanceRole**, not the IRSA role.

So we must attach permission to:

```
eksctl-aiops-cluster-nodegroup-*-NodeInstanceRole
```

Find the role:

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'NodeInstanceRole')].RoleName"
```

Attach:

```bash
aws iam attach-role-policy \
  --role-name <NodeInstanceRoleName> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

 This prevents AccessDenied for:

```
logs:CreateLogStream
```

---

#  Step 4 – Install CloudWatch Observability Addon

```bash
aws eks create-addon \
  --cluster-name aiops-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region ap-south-2 \
  --resolve-conflicts OVERWRITE
```

Wait until:

```bash
aws eks describe-addon \
  --cluster-name aiops-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region ap-south-2 \
  --query "addon.status"
```

Returns:

```
ACTIVE
```

---

#  Step 5 – Verify Fluent Bit Pods

```bash
kubectl get pods -n amazon-cloudwatch
```

You must see:

```
fluent-bit-xxxxx   Running
```

---

#  Step 6 – Restart Fluent Bit (Force Permission Refresh)

```bash
kubectl delete pod -n amazon-cloudwatch -l k8s-app=fluent-bit
```

Pods will auto recreate.

---

#  Step 7 – Generate Application Logs

```bash
APP_URL="http://<your-elb>"

for i in {1..30}; do
  curl -s $APP_URL/api
  echo
done
```

---

#  Step 8 – Verify Log Groups Created

```bash
aws logs describe-log-groups \
  --region ap-south-2 \
  --log-group-name-prefix "/aws/containerinsights"
```

You should now see:

```
/aws/containerinsights/aiops-cluster/application
/aws/containerinsights/aiops-cluster/dataplane
/aws/containerinsights/aiops-cluster/host
```

---

# What We Learned (Very Important)

Initially:

* Fluent Bit was running
* But using NodeInstanceRole
* That role had no CloudWatch permission
* So CreateLogStream failed
* Log groups never created

CloudTrail revealed:

```
AccessDenied
User: assumed-role/NodeInstanceRole
```

Solution:

Attach CloudWatch permission to NodeInstanceRole
Restart Fluent Bit

Then log groups were created automatically.

---

#  Phase 4 Complete When:

* Log groups exist
* No AccessDenied in fluent-bit logs
* Logs visible in CloudWatch

---


