

# Phase 1 RunBook — Backend Microservice with Structured Logging

---

##  Objective

Build a simple backend microservice that:

* Runs locally
* Produces structured JSON logs
* Has a healthy endpoint
* Has an intentional error endpoint
* Generates clean logs suitable for AI analysis

This service will later be:

* Containerized
* Deployed to EKS
* Logged to CloudWatch
* Analyzed by Bedrock

---

#  Architecture Context

At this stage:

```text
User
 ↓
Backend Service (Local)
 ↓
Structured Logs (Console)
```

No Docker.
No EKS.
No CloudWatch yet.

We validate the application first.

---

#  Step 1 — Prepare Working Directory

```bash
cd ~/aiops-lab/phase1-app
```

Verify Python:

```bash
python3 --version
```

Expected:

```text
Python 3.x.x
```

---

#  Step 2 — Install Flask

```bash
pip3 install flask
```

Confirm successful installation.

---

#  Step 3 — Create Backend Application

Create file:

```bash
nano app.py
```

Paste:

```python
from flask import Flask, request, jsonify
import logging
import time
import json
from datetime import datetime

app = Flask(__name__)

# Disable default Flask access logs
werkzeug_logger = logging.getLogger("werkzeug")
werkzeug_logger.setLevel(logging.ERROR)

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format="%(message)s"
)

def log_request(status_code, start_time):
    latency_ms = int((time.time() - start_time) * 1000)

    log_entry = {
        "timestamp": datetime.utcnow().isoformat() + "Z",
        "level": "ERROR" if status_code >= 500 else "INFO",
        "method": request.method,
        "path": request.path,
        "status": status_code,
        "latency_ms": latency_ms,
        "client_ip": request.remote_addr
    }

    if status_code >= 500:
        logging.error(json.dumps(log_entry))
    else:
        logging.info(json.dumps(log_entry))


@app.route("/")
def home():
    start_time = time.time()
    response = jsonify({"message": "Backend service running"})
    log_request(200, start_time)
    return response, 200


@app.route("/health")
def health():
    start_time = time.time()
    response = jsonify({"status": "healthy"})
    log_request(200, start_time)
    return response, 200


@app.route("/error")
def error():
    start_time = time.time()
    response = jsonify({"error": "Something went wrong"})
    log_request(500, start_time)
    return response, 500


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

Save and exit.

---

#  Step 4 — Run Application

```bash
python3 app.py
```

You should see:

```text
Running on http://127.0.0.1:5000
```

---

#  Step 5 — Generate Traffic

In another terminal:

```bash
curl http://localhost:5000/
curl http://localhost:5000/health
curl http://localhost:5000/error
```

---

#  Expected Log Output

Example:

```json
{"timestamp":"2026-02-25T09:54:11.078035Z","level":"ERROR","method":"GET","path":"/error","status":500,"latency_ms":0,"client_ip":"127.0.0.1"}
```

Logs must:

* Be single-line JSON
* Contain level field
* Show ERROR for 500
* Not contain duplicate Flask logs

---

#  Phase 1 — Key Learning

1. Always use structured logging in microservices.
2. Disable default framework noise.
3. Include severity, latency, and timestamp.
4. Log format should be machine-readable (JSON).
5. AI quality depends on log quality.

---

#  Phase 1 Completion Criteria

 App runs locally
 Endpoints respond correctly
 Structured logs visible
 ERROR logs generated

---

