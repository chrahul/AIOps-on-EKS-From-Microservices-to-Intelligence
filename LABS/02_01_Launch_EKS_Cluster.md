

#  PHASE 2.5 — Create 2-Node EKS Cluster with eksctl

We will:

1. Install eksctl (if not installed)
2. Create cluster config
3. Launch 2-node managed node group
4. Verify cluster
5. Update kubeconfig

---

#  Step 1 — Install eksctl

Check:

```bash
eksctl version
```

If not installed:

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```

Verify again:

```bash
eksctl version
```

---

#  Step 2 — Confirm AWS Access

Run:

```bash
aws sts get-caller-identity
```

This ensures:

* IAM role attached
* Permissions working

---

#  Step 3 — Create Cluster (Simple Command)

Basic version:

```bash
eksctl create cluster \
--name aiops-cluster \
--region ap-south-2 \
--nodegroup-name aiops-nodes \
--node-type t3.medium \
--nodes 2 \
--nodes-min 2 \
--nodes-max 2 \
--managed
```

---

#  What This Does (Explain in Demo)

This command automatically:

* Creates VPC
* Creates subnets
* Creates control plane
* Creates managed node group
* Attaches IAM roles
* Updates kubeconfig

Time: ~12–18 minutes.

---

#  

> Why t3.medium?

Answer:

* Enough memory for microservices
* Cheap enough for lab
* Avoid t2 (older gen)

---

#  Alternative (Better Production Style — Config File)

Instead of CLI flags, create YAML:

```bash
nano cluster.yaml
```

Paste:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: aiops-cluster
  region: ap-south-2
  version: "1.29"

managedNodeGroups:
  - name: aiops-nodes
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 2
    volumeSize: 20
    ssh:
      allow: true
    labels:
      role: worker
```

Then run:

```bash
eksctl create cluster -f cluster.yaml
```

This is cleaner for teaching.

---

#  Important — Permissions Needed

Your EC2 role must have:

* AmazonEKSClusterPolicy
* AmazonEKSWorkerNodePolicy
* AmazonEC2FullAccess
* IAMFullAccess (for lab simplicity)

If it fails → it’s IAM issue.

---

# Step 4 — Verify Cluster

After creation:

```bash
kubectl get nodes
```

You should see:

```
ip-xxx    Ready
ip-yyy    Ready
```

---

#  Step 5 — Verify Nodegroup

```bash
eksctl get nodegroup --cluster aiops-cluster --region ap-south-2
```

---

#  Explain What Just Happened (Important for Demo)

Explain architecture:

EC2 (you)
↓
eksctl
↓
CloudFormation stack
↓
EKS Control Plane (managed by AWS)
↓
EC2 Worker Nodes (2 instances)

---

#  

Ask:

> Is control plane inside my VPC?

Answer:

* Managed by AWS
* Endpoint exposed
* You don’t manage control plane EC2

---

#  After Cluster Ready

We will:

* Create namespace
* Deploy backend
* Create LoadBalancer
* Generate logs
* Move to Bedrock

---


