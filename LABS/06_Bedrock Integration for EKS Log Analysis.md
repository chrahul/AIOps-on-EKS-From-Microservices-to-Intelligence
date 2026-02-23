

#  PHASE 6 RUNBOOK

# **AIOps on EKS – From Microservices to Intelligence**

---

#  GOAL

Build an AI-powered troubleshooting pipeline:

```
EKS Microservices
        ↓
CloudWatch Logs
        ↓
Python Analyzer
        ↓
Amazon Bedrock
        ↓
Natural Language Diagnosis
```

Deliverable:

Students must be able to:

* Fetch EKS logs from CloudWatch
* Send logs to Bedrock
* Receive structured AI diagnosis
* Debug IAM / IRSA / Bedrock errors
* Secure public exposure

---

#  ARCHITECTURE

```
User
   ↓
LoadBalancer Service
   ↓
Frontend Pod
   ↓
Backend Pod
   ↓
CloudWatch Logs (Container Insights)
   ↓
Python AI Script
   ↓
Bedrock (Claude Haiku)
   ↓
AI Summary (Root Cause + Debug Steps)
```

---

#  STEP 0 — Preconditions

You must already have:

* EKS cluster running
* Backend app deployed
* Service type: LoadBalancer
* CloudWatch Observability Add-on installed

Verify:

```bash
kubectl get pods -n amazon-cloudwatch
```

---

#  STEP 1 — Verify CloudWatch Log Groups

Check log groups:

```bash
aws logs describe-log-groups \
  --query 'logGroups[*].logGroupName'
```

You should see:

```
/aws/containerinsights/<cluster>/application
/aws/containerinsights/<cluster>/dataplane
/aws/containerinsights/<cluster>/host
/aws/containerinsights/<cluster>/performance
```

If not present → Fluent-bit is not working.

---

#  STEP 2 — Common Fluent-bit Failure (IMPORTANT)

### Symptom:

Log groups not created.

Check logs:

```bash
kubectl logs -n amazon-cloudwatch <fluent-bit-pod>
```

Error example:

```
AccessDeniedException: CreateLogStream
```

---

##  Root Cause (Critical Learning)

CloudTrail showed:

```
assumed-role/<NodeInstanceRole>/<EC2 instance-id>
```

Meaning:

Fluent-bit was using **Node Instance Role**, not IRSA role.

Even though:

```
eks.amazonaws.com/role-arn
```

was present on ServiceAccount.

---

##  Fix

Attach CloudWatch permissions to Node Instance Role:

```bash
aws iam attach-role-policy \
  --role-name <NodeInstanceRole> \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

Restart fluent-bit:

```bash
kubectl delete pods -n amazon-cloudwatch --all
```

Verify log groups appear.

---

#  STEP 3 — Install boto3 (EC2 Analyzer Host)

On EC2:

```bash
python3 -m ensurepip --upgrade
python3 -m pip install --upgrade boto3
```

---

#  STEP 4 — Test CloudWatch Log Fetch

Create `fetch_logs.py`

```python
import boto3

logs = boto3.client("logs", region_name="ap-south-2")

response = logs.describe_log_streams(
    logGroupName="/aws/containerinsights/aiops-cluster/application",
    orderBy="LastEventTime",
    descending=True,
    limit=1
)

print(response)
```

Run:

```bash
python3 fetch_logs.py
```

Must return latest stream.

---

#  STEP 5 — Bedrock Model Access (Critical)

If invoking Claude gives:

```
ResourceNotFoundException:
Model use case details have not been submitted
```

### Fix:

AWS Console → Bedrock → Model Access → Anthropic → Submit Use Case

Example:

```
Internal DevOps log summarization for Kubernetes workloads.
```

Wait 10–15 minutes.

---

#  STEP 6 — Test Bedrock Independently

Create `test_bedrock.py`

```python
import boto3
import json

bedrock = boto3.client("bedrock-runtime", region_name="ap-south-2")

body = json.dumps({
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 200,
    "messages": [
        {
            "role": "user",
            "content": "Explain Amazon EKS in one sentence."
        }
    ]
})

response = bedrock.invoke_model(
    modelId="global.anthropic.claude-haiku-4-5-20251001-v1:0",
    body=body,
    contentType="application/json",
    accept="application/json"
)

result = json.loads(response["body"].read())
print(result["content"][0]["text"])
```

Run:

```bash
python3 test_bedrock.py
```

Must return AI output.

---

#  STEP 7 — Final AIOps Engine Script

Create `aiops_engine.py`

(Complete working script)

```python
import boto3
import json

REGION = "ap-south-2"
LOG_GROUP = "/aws/containerinsights/aiops-cluster/application"
MAX_EVENTS = 30

logs_client = boto3.client("logs", region_name=REGION)
bedrock = boto3.client("bedrock-runtime", region_name=REGION)

def fetch_logs():
    streams = logs_client.describe_log_streams(
        logGroupName=LOG_GROUP,
        orderBy="LastEventTime",
        descending=True,
        limit=1
    )

    log_stream = streams["logStreams"][0]["logStreamName"]

    response = logs_client.get_log_events(
        logGroupName=LOG_GROUP,
        logStreamName=log_stream,
        limit=MAX_EVENTS,
        startFromHead=False
    )

    return [event["message"] for event in response["events"]]

def analyze_with_ai(logs):
    logs_text = "\n".join(logs)

    prompt = f"""
You are a senior Kubernetes SRE.

Analyze these logs and provide:
1. Summary
2. Top issues
3. Probable root cause
4. Recommended debug steps

Logs:
{logs_text}
"""

    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 400,
        "messages": [
            {"role": "user", "content": prompt}
        ]
    })

    response = bedrock.invoke_model(
        modelId="global.anthropic.claude-haiku-4-5-20251001-v1:0",
        body=body,
        contentType="application/json",
        accept="application/json"
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]

if __name__ == "__main__":
    logs = fetch_logs()
    analysis = analyze_with_ai(logs)
    print("\n=== AI DIAGNOSIS ===\n")
    print(analysis)
```

Run:

```bash
python3 aiops_engine.py
```

---

#  STEP 8 — Public Exposure & Security Hardening

If service type:

```
LoadBalancer
```

Internet scanners will hit it.

You may see:

```
CONNECT www.baidu.com:443
```

This is open-proxy scanning.

Not compromise.

---

## Secure ELB

Find Security Group attached to ELB.

Change inbound rule from:

```
0.0.0.0/0
```

To:

```
<YOUR_PUBLIC_IP>/32
```

Or use Kubernetes:

```yaml
loadBalancerSourceRanges:
  - <YOUR_PUBLIC_IP>/32
```

---

#  VALIDATION CHECKLIST

Students must confirm:

| Check                      | Expected |
| -------------------------- | -------- |
| Log groups created         | Yes        |
| Fluent-bit no AccessDenied | yes        |
| Bedrock invocation works   | Yes        |
| AI diagnosis structured    | Yes        |
| ELB restricted             | yes        |

---

#  KEY LEARNINGS

1. IRSA does not guarantee AWS API usage — verify with CloudTrail.
2. NodeInstanceRole may be used instead of IRSA.
3. Bedrock requires model access approval (Anthropic).
4. AI reasoning must be validated by engineers.
5. Public LoadBalancers attract scanning traffic.
6. Secure at edge (Security Groups).

---

# END RESULT


```
EKS → Observability → AI → Natural Language Diagnosis
```

This is not a toy lab.

This is production-grade AIOps foundation.

---

