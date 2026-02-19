
---

#  PHASE 5 – Enable Container Insights Metrics (EKS → CloudWatch)

This phase enables:

> CPU
> Memory
> Pod metrics
> Node metrics
> Cluster metrics

Visible under:

```
CloudWatch → Container Insights → EKS
```

---

#  Objective

Send **metrics (not just logs)** from:

* Nodes
* Pods
* Containers
* Kubernetes cluster

To:

```
CloudWatch Metrics → Namespace: ContainerInsights
```

---

#  Important Clarification

In Phase 4, we enabled:

 Container Logs (Fluent Bit)
 Metrics (CloudWatch Agent was not running)

That is why:

```bash
aws cloudwatch list-metrics --namespace ContainerInsights
```

Returned empty.

Because:

> CloudWatch Agent (metrics collector) was NOT deployed.

---

#  Architecture (Very Simple)

| Component        | Purpose                  |
| ---------------- | ------------------------ |
| Fluent Bit       | Logs                     |
| CloudWatch Agent | Metrics                  |
| Node IAM Role    | Must allow metrics write |

---

#  STEP 1 – Verify Addon Is Installed

```bash
aws eks list-addons \
  --cluster-name aiops-cluster \
  --region ap-south-2
```

You should see:

```
amazon-cloudwatch-observability
```

If not present → Install again:

```bash
aws eks create-addon \
  --cluster-name aiops-cluster \
  --addon-name amazon-cloudwatch-observability \
  --region ap-south-2 \
  --resolve-conflicts OVERWRITE
```

---

#  STEP 2 – Verify Pods in amazon-cloudwatch Namespace

```bash
kubectl get pods -n amazon-cloudwatch
```

You must see:

* fluent-bit pods
* cloudwatch-agent pods

If you only see fluent-bit → metrics not enabled.

---

#  STEP 3 – Ensure Node Role Has Required Policies

This is CRITICAL.

Find NodeInstanceRole:

```bash
aws iam list-roles \
  --query "Roles[?contains(RoleName, 'NodeInstanceRole')].RoleName"
```

Attach BOTH policies:

```bash
aws iam attach-role-policy \
  --role-name <NodeInstanceRoleName> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
```

AND (if not already attached)

```bash
aws iam attach-role-policy \
  --role-name <NodeInstanceRoleName> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

---

#  STEP 4 – Restart Addon Pods

```bash
kubectl delete pod -n amazon-cloudwatch --all
```

Pods will recreate automatically.

---

#  STEP 5 – Verify CloudWatch Agent Running

```bash
kubectl get pods -n amazon-cloudwatch -l app.kubernetes.io/name=cloudwatch-agent
```

You should now see:

```
cloudwatch-agent-xxxxx   Running
```

---

#  STEP 6 – Generate Workload Traffic

```bash
APP_URL="http://<your-elb>"

for i in {1..50}; do
  curl -s $APP_URL/api
  echo
done
```

---

#  STEP 7 – Verify Metrics Exist

```bash
aws cloudwatch list-metrics \
  --namespace ContainerInsights \
  --region ap-south-2 \
  --max-items 10
```

Now you should see metrics.

Example:

```
pod_cpu_utilization
node_memory_utilization
cluster_node_count
```

---

#  STEP 8 – Check CloudWatch Console

Go to:

```
CloudWatch → Container Insights → Performance Monitoring → EKS
```

Select:

```
Cluster: aiops-cluster
```

You should now see:

* Node CPU
* Node Memory
* Pod CPU
* Pod Memory
* Container stats

---

#  What Actually Happens Internally

1. CloudWatch Agent runs as DaemonSet
2. It scrapes:

   * kubelet
   * cAdvisor
   * cluster APIs
3. It pushes metrics to:

   ```
   CloudWatch → Namespace: ContainerInsights
   ```

---

#  If Metrics Still Not Showing

Check:

```bash
kubectl logs -n amazon-cloudwatch <cloudwatch-agent-pod>
```

If AccessDenied appears → Node role missing policy.

---

#  Phase 5 Complete When:

* cloudwatch-agent pod running
* Metrics visible in namespace ContainerInsights
* CloudWatch console shows EKS performance dashboard

---

#  Big Picture Summary

| Phase   | What We Enabled                                 |
| ------- | ----------------------------------------------- |
| Phase 4 | Logs (Fluent Bit → CloudWatch Logs)             |
| Phase 5 | Metrics (CloudWatch Agent → CloudWatch Metrics) |

---

You now have:

Centralized logs
Cluster metrics
Node metrics
Pod metrics



---



You tell me.
