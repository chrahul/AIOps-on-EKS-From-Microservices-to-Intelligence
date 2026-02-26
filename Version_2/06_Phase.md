
#  Phase 6 Runbook

# AI-Driven Log Intelligence Using Amazon Bedrock

---

#  Objective

Build a minimal working **AIOps intelligence engine** that:

1. Fetches real-time logs from CloudWatch
2. Sends them to Amazon Bedrock
3. Generates an AI-based SRE summary
4. Produces root cause analysis style output

This phase transforms the system from:

```text
Observability
```

To:

```text
Intelligent Observability (AIOps)
```

---

#  Architecture

```text
EKS Cluster
    ↓
CloudWatch Logs
    ↓
Python (boto3)
    ↓
Amazon Bedrock (Claude via Inference Profile)
    ↓
AI RCA Summary
```

---

#  Environment Details

| Component         | Value                                            |
| ----------------- | ------------------------------------------------ |
| Cluster Name      | aiops-cluster                                    |
| Region            | ap-south-2                                       |
| Log Group         | /aws/containerinsights/aiops-cluster/application |
| Model             | Claude Sonnet 4.5                                |
| Inference Profile | global.anthropic.claude-sonnet-4-5-20250929-v1:0 |

---

#  Prerequisites

## 1️ AWS CLI Configured

```bash
aws configure
```

Confirm region:

```bash
aws configure get region
```

Must return:

```text
ap-south-2
```

---

## 2️ Python Installed

```bash
python3 --version
```

Recommended: Python 3.10+

---

## 3️ Install boto3

```bash
python3 -m pip install boto3
```

---

## 4️ Confirm Bedrock Model Availability

```bash
aws bedrock list-inference-profiles --region ap-south-2
```

Expected output includes:

```text
global.anthropic.claude-sonnet-4-5-20250929-v1:0
```

---

#  Step 1 — Create Phase 6 Script

Navigate:

```bash
cd ~/aiops-lab
```

Create file:

```bash
nano phase6-bedrock-summary.py
```

---

#  Full Python Script

```python
import boto3
import json
from datetime import datetime, timedelta

# ===============================
# Configuration
# ===============================

AWS_REGION = "ap-south-2"
CLUSTER_NAME = "aiops-cluster"
LOG_GROUP = f"/aws/containerinsights/{CLUSTER_NAME}/application"

# IMPORTANT: Use inference profile ID
BEDROCK_MODEL_ID = "global.anthropic.claude-sonnet-4-5-20250929-v1:0"

# ===============================
# AWS Clients
# ===============================

logs_client = boto3.client("logs", region_name=AWS_REGION)
bedrock_client = boto3.client("bedrock-runtime", region_name=AWS_REGION)


# ===============================
# Fetch Recent Logs
# ===============================

def fetch_recent_logs(minutes=5, limit=50):
    end_time = int(datetime.utcnow().timestamp() * 1000)
    start_time = int(
        (datetime.utcnow() - timedelta(minutes=minutes)).timestamp() * 1000
    )

    response = logs_client.filter_log_events(
        logGroupName=LOG_GROUP,
        startTime=start_time,
        endTime=end_time,
        limit=limit
    )

    events = response.get("events", [])

    if not events:
        return None

    messages = []
    for event in events:
        messages.append(event["message"])

    return "\n".join(messages)


# ===============================
# Generate AI Summary
# ===============================

def generate_summary(log_text):

    prompt = f"""
You are an expert Site Reliability Engineer (SRE).

Analyze the following Kubernetes application logs and provide:

1. Summary of system behavior
2. Any detected errors or anomalies
3. Observed patterns
4. Possible root cause (if identifiable)
5. Suggested next steps

Logs:
{log_text}
"""

    body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 400,
        "temperature": 0.2,
        "messages": [
            {
                "role": "user",
                "content": prompt
            }
        ]
    }

    response = bedrock_client.invoke_model(
        modelId=BEDROCK_MODEL_ID,
        body=json.dumps(body),
        contentType="application/json",
        accept="application/json"
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]


# ===============================
# Main
# ===============================

if __name__ == "__main__":

    print("========================================")
    print("Phase 6 — AIOps Log Intelligence Engine")
    print("========================================\n")

    print("Fetching recent logs...\n")
    logs = fetch_recent_logs()

    if not logs:
        print("No logs found in the last 5 minutes.")
        exit()

    print("Generating AI summary...\n")
    summary = generate_summary(logs)

    print("============== AI SUMMARY ==============\n")
    print(summary)
    print("\n========================================")
```

---

#  Step 2 — Run the Script

```bash
python3 phase6-bedrock-summary.py
```

---

#  Expected Output

```text
============== AI SUMMARY ==============

SRE Log Analysis Report

Summary of system behavior...
Detected 500 errors...
Observed STS AssumeRoleWithWebIdentity failures...
Suggested fix: Verify IRSA trust policy...
```

---

#  What This Phase Achieves

| Capability             | Status |
| ---------------------- | ------ |
| Real log ingestion     | Yes      |
| AI-based summarization | Yes      |
| Pattern detection      | Yes      |
| Error clustering       | yes      |
| Root cause suggestion  | yes      |

---

#  Troubleshooting

##  Error: Invocation with on-demand throughput not supported

Cause:
Using foundation model ID instead of inference profile.

Fix:
Use:

```text
global.anthropic.claude-sonnet-4-5-20250929-v1:0
```

---

##  Error: AccessDenied

Check:

```bash
aws sts get-caller-identity
```

Ensure IAM role attached to EC2 has:

* Bedrock access
* CloudWatch Logs read permissions

---

#  Success Criteria

Phase 6 is complete when:

* Script runs without exception
* AI summary generated
* Errors correctly interpreted
* Output readable and actionable

---

#  Phase 6 Outcome

You have built a working:

```text
AIOps Log Intelligence Engine
```

System now performs:

```text
Detection → Interpretation → RCA Suggestion
```

Without hardcoded rules.

---

#  What This Enables Next

Future Phases can now build:

* Automated anomaly detection
* Incident trigger mode
* Alerting integration
* Self-healing automation
* AI-driven diagnostics dashboard

---

#  Phase 6 Status

```text
Phase 6 — COMPLETED
```

---


