

#  **AIOps on EKS – From Microservices to Intelligence**

---

#  PHASE 1 — Build Our Own Microservice (From Scratch)

We will build:

* 1 Backend API service (Flask)
* 1 Simple Frontend service (optional but recommended)
* Structured logging
* Intentional error simulation

Keep it clean.
Keep it small.
Keep it production-thinking.

---

#  What We Want From Phase 1

Our backend service must:

* Expose `/health`
* Expose `/api`
* Log every request
* Simulate error (to generate meaningful logs)
* Return JSON

This gives us:

* Logs for AI
* Healthy vs failing behavior
* Real debugging scenario

---

#  Step 1 — Create Backend Service

Create a new folder:

```bash
mkdir aiops-eks-lab
cd aiops-eks-lab
mkdir backend
cd backend
```

---

#  Step 2 — Write `app.py`

Create `app.py`:

```python
from flask import Flask, jsonify, request
import logging
import random
import time

app = Flask(__name__)

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

@app.route("/api")
def api():
    logging.info("API endpoint called")

    # Simulate random failure
    if random.randint(1, 5) == 3:
        logging.error("Simulated backend failure")
        return jsonify({"error": "Something went wrong"}), 500

    time.sleep(0.5)
    return jsonify({"message": "Backend response successful"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

---

#  Why This Design?

This gives us:

* INFO logs
* ERROR logs
* Random failure
* Real-world debugging scenario
* Health endpoint (Kubernetes best practice)

Perfect for AIOps.

---

#  Step 3 — Create `requirements.txt`

Create file:

```
Flask==2.3.3
```

---

#  Step 4 — Create Dockerfile

Create `Dockerfile`:

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

---

#  Step 5 — Test Locally Before Kubernetes

Build:

```bash
docker build -t aiops-backend .
```

Run:

```bash
docker run -p 5000:5000 aiops-backend
```

Test:

```
http://localhost:5000/api
```

You should see:

* Successful JSON
* Sometimes 500 error
* Logs printed in terminal

---

#  FAQ



> Why did we create a health endpoint?

* For readiness/liveness probes



> Why structured logging?

* So AI can parse logs easily

---

# Phase 1 Outcome

We now have:

* A working backend microservice
* Error simulation
* Logs generated
* Dockerized app

---

