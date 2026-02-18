

#  What is `aws sts get-caller-identity`?

It is a command that asks AWS:

> “Who am I right now?”

That’s it.

It tells you:

* Which AWS account you're operating in
* Which IAM role or user you're using
* Whether your credentials are valid

---

#  What Does STS Mean?

STS = **Security Token Service**

STS is the AWS service responsible for:

* Issuing temporary credentials
* Managing assumed roles
* Identity federation
* Cross-account access

---

#  What This Command Actually Does

When you run:

```bash
aws sts get-caller-identity
```

Internally:

1. AWS CLI signs a request using your current credentials
2. It sends the request to AWS STS endpoint
3. STS validates the credentials
4. STS responds with:

   * Account ID
   * UserId
   * ARN

---

#  Why It Worked After Attaching IAM Role

Your output:

```json
{
  "UserId": "XXXXXXXXVQPCCPG:i-xxxxxxc2da333",
  "Account": "12345678",
  "Arn": "arn:aws:sts::12345678:assumed-role/aiops-eks-role/i-xxxxxxc2da333"
}
```

Important line:

```
assumed-role/aiops-eks-role/i-xxxxxxc2da333
```

That means:

* EC2 instance is assuming role `aiops-eks-role`
* STS issued temporary credentials
* Those credentials are being used by AWS CLI

This confirms:
IAM role attached correctly
Temporary credentials active
AWS CLI working

---

#  Why We Use This Command in DevOps

We use it to:

* Verify IAM role is attached
* Confirm which account we’re in
* Debug permission issues
* Avoid deploying in wrong account

Very common mistake in enterprises:
Engineers deploy in wrong AWS account.

This command prevents that.

---

#  “How Does CLI Know Which Credentials to Use?”

AWS CLI checks in this order:

1. Environment variables
2. AWS config files (~/.aws/credentials)
3. IAM role attached to EC2 (via Instance Metadata Service)

Since you are on EC2:

* CLI queries EC2 metadata service
* Metadata service provides temporary credentials
* CLI uses them automatically

---

#  “Where Are These Credentials Stored?”

They are NOT stored permanently.

EC2 retrieves temporary credentials from:

```
http://169.254.169.254
```

This is the Instance Metadata Service (IMDS).

STS rotates them automatically.

That’s why this is secure.

---

#  Advanced Explanation 

Internally:

* CLI creates a signed request using SigV4
* STS verifies signature
* Returns identity without requiring explicit permission
  (This API does not require explicit IAM allow)

Fun fact:

`get-caller-identity` works even if IAM policy denies everything else.

---

# 

> “aws sts get-caller-identity tells me which IAM identity my CLI is currently using. It confirms whether my EC2 role or credentials are working.”

---

# 



> “Before pushing to ECR, always verify identity using STS. This prevents cross-account mistakes.”



---


