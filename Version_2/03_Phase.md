**Phase 3 RunBook ** 

---

#  AIOps on EKS — Phase 3 RunBook 

## Title: Create EKS Cluster (Production-style `cluster.yaml`, Managed Node Group, gp3 20GB)

---

##  Objective

Create an EKS cluster in `ap-south-2` using `eksctl` and a declarative `cluster.yaml` config:

* Cluster name: `aiops-cluster`
* Managed node group: `aiops-nodes`
* Node volume: `gp3`, **20GB**
* Control plane logs enabled (API, audit, auth, controller-manager, scheduler)

---

##  Architecture Context

After Phase 3:

```text
EKS Control Plane (Managed by AWS)  ---> Control Plane Logs (CloudWatch)
              |
              v
Managed Node Group (1 node) ---> Pods will run here (later phases)
```

---

##  Prerequisites

* AWS CLI configured and logged in
* Region set to `ap-south-2`
* `kubectl` installed
* `eksctl` installed

Verify region:

```bash
aws configure get region
```

Expected:

```text
ap-south-2
```

---

##  Step 1 — Create Phase Folder

```bash
mkdir -p ~/aiops-lab/phase3-eks
```

---

##  Step 2 — Create `cluster.yaml`

```bash
cat > ~/aiops-lab/phase3-eks/cluster.yaml <<'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: aiops-cluster
  region: ap-south-2
  version: "1.29"

cloudWatch:
  clusterLogging:
    enableTypes: ["api", "audit", "authenticator", "controllerManager", "scheduler"]

vpc:
  clusterEndpoints:
    publicAccess: true
    privateAccess: true

managedNodeGroups:
  - name: aiops-nodes
    instanceType: t3.medium
    desiredCapacity: 1
    minSize: 1
    maxSize: 2
    volumeType: gp3
    volumeSize: 20
    ssh:
      allow: false
    iam:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
    labels:
      role: backend
    tags:
      Environment: aiops-lab
EOF
```

Verify:

```bash
cat ~/aiops-lab/phase3-eks/cluster.yaml
```

---

##  Step 3 — Create Cluster

```bash
eksctl create cluster -f ~/aiops-lab/phase3-eks/cluster.yaml
```

This step creates:

* VPC + subnets + NAT
* EKS control plane
* Managed node group
* Security groups
* IAM roles

---

##  Verification

### Check Nodes

```bash
kubectl get nodes -o wide
```

Expected:

* 1 node
* Status = `Ready`

Example:

```text
NAME ... STATUS Ready ... VERSION v1.29.x-eks-...
```

### Check Cluster

```bash
kubectl cluster-info
```

---

##  Phase 3 Key Learnings

1. `cluster.yaml` makes cluster creation repeatable and teachable.
2. Control plane logging is critical for an AIOps lab.
3. gp3 20GB is cost-optimized and sufficient for student workloads.
4. Managed node groups reduce operational burden (AWS manages upgrades/health).

---

##  Troubleshooting

### Node not Ready

```bash
kubectl get nodes
kubectl describe node <node-name>
```

### eksctl fails midway

Check CloudFormation stacks:

```bash
aws cloudformation list-stacks --region ap-south-2
```

---

 Phase 3 complete.

---

# Next Phase: Phase 4 — Deploy Backend to EKS

## Phase 4 (high-level)

We will:

1. Create namespace `aiops`
2. Create Deployment (2 replicas) using ECR image `aiops/backend:v1`
3. Create Service (LoadBalancer)
4. Test endpoints and generate logs

---


