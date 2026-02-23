# **Phase 7 — Conversational AIOps: Serverless NLP Query Engine (API Gateway + Lambda + Bedrock + CloudWatch Logs)**

---

#  Phase 7 RUNBOOK (Step-by-Step, Engineer-Friendly)

## Goal

Build a serverless endpoint where engineers can ask NLP questions about EKS logs:

```text
User (NLP Question)
   ↓
API Gateway (HTTP API)
   ↓
Lambda (Python AIOps Engine)
   ↓
CloudWatch Logs (Container Insights / application)
   ↓
Amazon Bedrock (Claude via inference profile)
   ↓
Natural Language Answer
```

**Deliverable:**
A working HTTP endpoint that returns JSON like:

```json
{"analysis":"..."}
```

---

## Prerequisites

1. EKS cluster already sending logs to CloudWatch Log Group:

   * `/aws/containerinsights/<cluster-name>/application`

2. AWS CLI configured on a machine (EC2 or laptop) with permissions for:

   * IAM role creation
   * Lambda create/update
   * API Gateway v2 (HTTP API)
   * CloudWatch Logs read
   * Bedrock invoke

3. Bedrock Anthropic access enabled:

   * Bedrock Console → Model access → Anthropic → submit use case (one-time)

---

## Configuration Values (Use These Consistently)

Set your constants:

* **Region:** `ap-south-2`
* **CloudWatch Log Group:** `/aws/containerinsights/aiops-cluster/application`
* **Lambda Name:** `aiops-query-engine`
* **IAM Role:** `aiops-lambda-role`
* **API Name:** `aiops-api`
* **Model ID:** `global.anthropic.claude-haiku-4-5-20251001-v1:0`
* **Account ID:** `12345678`

---

---

# STEP 1 — Create IAM Role for Lambda 

## 1.1 Create Role

```bash
aws iam create-role \
  --role-name aiops-lambda-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": { "Service": "lambda.amazonaws.com" },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

 Output should include Role ARN like:
`arn:aws:iam::<account-id>:role/aiops-lambda-role`

---

# STEP 2 — Attach Required Policies 

## 2.1 Attach Basic Lambda Logging Policy

```bash
aws iam attach-role-policy \
  --role-name aiops-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

## 2.2 Add Inline Policy for CloudWatch Logs + Bedrock

```bash
aws iam put-role-policy \
  --role-name aiops-lambda-role \
  --policy-name AIOpsLambdaPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "logs:DescribeLogStreams",
          "logs:GetLogEvents",
          "logs:FilterLogEvents"
        ],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "bedrock:InvokeModel"
        ],
        "Resource": "*"
      }
    ]
  }'
```

---

# STEP 3 — Create Lambda Code 

## 3.1 Create Working Directory

```bash
mkdir aiops-lambda
cd aiops-lambda
```

## 3.2 Create `lambda_function.py`

```bash
nano lambda_function.py
```

Paste:

```python
import json
import boto3

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

    if not streams["logStreams"]:
        return []

    log_stream = streams["logStreams"][0]["logStreamName"]

    response = logs_client.get_log_events(
        logGroupName=LOG_GROUP,
        logStreamName=log_stream,
        limit=MAX_EVENTS,
        startFromHead=False
    )

    return [event["message"] for event in response["events"]]

def analyze_with_ai(question, logs):
    logs_text = "\n".join(logs)

    prompt = f"""
You are a senior Kubernetes SRE.

User Question:
{question}

Cluster Logs:
{logs_text}

Provide:
1. Summary
2. Top Issues
3. Probable Root Cause
4. Recommended Debug Steps
"""

    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 400,
        "messages": [{"role": "user", "content": prompt}]
    })

    response = bedrock.invoke_model(
        modelId="global.anthropic.claude-haiku-4-5-20251001-v1:0",
        body=body,
        contentType="application/json",
        accept="application/json"
    )

    result = json.loads(response["body"].read())
    return result["content"][0]["text"]

def lambda_handler(event, context):
    try:
        body = json.loads(event.get("body", "{}"))
        question = body.get("question", "Summarize cluster issues.")

        logs = fetch_logs()
        ai_response = analyze_with_ai(question, logs)

        return {"statusCode": 200, "body": json.dumps({"analysis": ai_response})}

    except Exception as e:
        return {"statusCode": 500, "body": json.dumps({"error": str(e)})}
```

---

# STEP 4 — Package and Deploy Lambda 

## 4.1 Zip Deployment

```bash
zip function.zip lambda_function.py
```

## 4.2 Create Lambda

```bash
aws lambda create-function \
  --function-name aiops-query-engine \
  --runtime python3.12 \
  --role arn:aws:iam::12345678:role/aiops-lambda-role \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --region ap-south-2
```

---

# STEP 5 — Fix Lambda Timeout & Memory (Critical) 

Default timeout is 3 seconds → Bedrock calls usually exceed this.

 Update to 30 seconds:

```bash
aws lambda update-function-configuration \
  --function-name aiops-query-engine \
  --timeout 30 \
  --memory-size 512 \
  --region ap-south-2
```

---

# STEP 6 — Test Lambda Directly (Before API Gateway) 

AWS CLI requires binary-format option:

```bash
aws lambda invoke \
  --function-name aiops-query-engine \
  --payload '{"body":"{\"question\":\"Summarize cluster issues\"}"}' \
  --cli-binary-format raw-in-base64-out \
  response.json \
  --region ap-south-2
```

View response:

```bash
cat response.json
```

 Expected: `statusCode: 200` with `analysis`.

---

# STEP 7 — Create API Gateway HTTP API 

## 7.1 Create HTTP API

```bash
aws apigatewayv2 create-api \
  --name aiops-api \
  --protocol-type HTTP \
  --target arn:aws:lambda:ap-south-2:12345678:function:aiops-query-engine \
  --region ap-south-2
```

Save:

* `ApiId`
* `ApiEndpoint`

Example:

* ApiId: `7ou4njkxgc`
* Endpoint: `https://7ou4njkxgc.execute-api.ap-south-2.amazonaws.com`

---

# STEP 8 — Allow API Gateway to Invoke Lambda 

 For HTTP API, source ARN should be `api-id/*`

Remove old permission (if any):

```bash
aws lambda remove-permission \
  --function-name aiops-query-engine \
  --statement-id apigateway-invoke
```

Add correct permission:

```bash
aws lambda add-permission \
  --function-name aiops-query-engine \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:ap-south-2:12345678:7ou4njkxgc/*"
```

---

# STEP 9 — Test End-to-End with curl 

```bash
curl -X POST https://7ou4njkxgc.execute-api.ap-south-2.amazonaws.com \
  -H "Content-Type: application/json" \
  -d '{"question":"Summarize cluster issues"}'
```

 Expected JSON:

```json
{"analysis":"..."}
```

---

# Troubleshooting Guide (Must Include)

## Issue 1: `Invalid base64` when invoking Lambda

 Fix:

```bash
--cli-binary-format raw-in-base64-out
```

---

## Issue 2: Lambda Timeout (3 seconds)

Symptom:
`Task timed out after 3.00 seconds`

 Fix:

```bash
aws lambda update-function-configuration --timeout 30 --memory-size 512
```

---

## Issue 3: API Gateway returns `{"message":"Internal Server Error"}`

Most common: Lambda permission ARN mismatch

 Fix permission for HTTP API:

```bash
--source-arn "arn:aws:execute-api:ap-south-2:12345678:<api-id>/*"
```

---

## Issue 4: Bedrock error “use case not submitted”

 Fix: Bedrock Console → Anthropic → submit use case details.

---

# Verification Checklist (Definition of Done)

 Lambda direct invoke returns 200 + analysis
 API Gateway curl returns 200 + analysis
 Lambda timeout ≥ 30 seconds
 CloudWatch log group `/aws/lambda/aiops-query-engine` exists
 Bedrock model access approved

---

# Output Example (What CTO/Engineer Sees)

```json
{
  "analysis": "Summary... Top issues... Root cause... Debug steps..."
}
```

---

