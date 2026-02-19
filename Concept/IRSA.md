
# First: What Is an IAM Role?

An **IAM Role** is simply:

> A permission container in AWS.

It defines:

* What actions are allowed
* On which resources
* Under what conditions

Example:

```
logs:CreateLogStream
logs:PutLogEvents
```

That’s it. Nothing magical.

IAM Role is just a policy wrapper.

---

# Then What Is IRSA?

IRSA = **IAM Roles for Service Accounts**

Important:

> IRSA is NOT a new type of IAM role.

It is a mechanism to allow a Kubernetes pod to assume an IAM role.

So:

| IAM Role              | IRSA                                                       |
| --------------------- | ---------------------------------------------------------- |
| AWS permission object | A method to attach IAM Role to a Kubernetes ServiceAccount |

---

# Without IRSA (Traditional Way)

Before IRSA existed:

All pods in a node used:

```
EC2 Node Instance Role
```

Meaning:

Every pod had the same permissions.

If one pod needed S3 access,
all pods got S3 access.

Security problem.

---

# With IRSA

Now we can say:

Pod A → can use IAM Role A
Pod B → can use IAM Role B
Pod C → no AWS access at all

This is fine-grained access control.

---

# How IRSA Actually Works (Simple Flow)

Step 1
EKS cluster has OIDC provider enabled.

Step 2
You create IAM Role with trust policy like this:

```
"Condition": {
  "StringEquals": {
    "oidc.eks.region.amazonaws.com/id/XYZ:sub":
    "system:serviceaccount:namespace:serviceaccount-name"
  }
}
```

This means:

> Only this Kubernetes ServiceAccount can assume this role.

Step 3
You annotate the ServiceAccount:

```
eks.amazonaws.com/role-arn: arn:aws:iam::123:role/MyRole
```

Step 4
When pod starts:

Kubernetes mounts:

```
/var/run/secrets/eks.amazonaws.com/serviceaccount/token
```

AWS SDK uses that token to call:

```
sts:AssumeRoleWithWebIdentity
```

Then AWS returns temporary credentials.

Now pod has AWS credentials.

---

# Big Difference: Node Role vs IRSA

## Node Role

Attached to EC2 instance.
All pods inherit it via metadata service.

Security boundary = Node.

---

## IRSA Role

Attached to ServiceAccount.
Only that pod can use it.

Security boundary = Pod.

---

# Why Our Issue Happened

Even though IRSA was configured,
Fluent Bit ended up using:

```
Node Instance Role
```

Not IRSA.

CloudTrail proved that.

So permissions on IRSA didn’t matter.

Node role needed permission.

---

# Conceptually

IAM Role = The permission definition
IRSA = The bridge that lets a Kubernetes pod use that IAM Role

---

# Visual Model

Think like this:

IAM Role = ID Card
IRSA = Gate system that gives the ID card to a specific pod

Without IRSA:
Security guard gives same master ID to whole building.

With IRSA:
Each employee gets their own ID.

---

# Final Clean Definition

IAM Role:
An AWS identity with permissions.

IRSA:
A Kubernetes-to-AWS integration mechanism that allows a pod to assume an IAM role using OIDC federation.

---


