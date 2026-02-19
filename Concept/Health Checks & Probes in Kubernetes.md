# Health Checks & Probes in Kubernetes



---

## 1️ Why Do We Need Health Checks?

In distributed systems, services fail.

* A process may hang.
* A dependency (database, API) may become unreachable.
* Memory may leak.
* Threads may deadlock.
* Network issues may occur.

The important question is:

> How does Kubernetes know whether a container is healthy or not?

Kubernetes does not understand your application logic.
It cannot guess if your Flask, Node, or Java app is functioning properly.

So we must **teach Kubernetes how to verify health**.

That’s where probes come in.

---

## 2️ What Is a Probe?

A **probe** is a periodic health check performed by Kubernetes against a container.

It answers questions like:

* Is the container running correctly?
* Is the application ready to receive traffic?
* Should this pod be restarted?

Kubernetes supports three types of probes:

1. Liveness Probe
2. Readiness Probe
3. Startup Probe

---

## 3️ Liveness Probe — "Should I Restart This?"

The liveness probe checks:

> Is the application alive?

If this probe fails repeatedly:

* Kubernetes assumes the container is unhealthy.
* The container is restarted automatically.

### Example

If your backend hangs due to a deadlock, the liveness probe will fail, and Kubernetes will restart it.

This is automatic self-healing.

---

## 4️ Readiness Probe — "Can I Send Traffic?"

The readiness probe checks:

> Is this pod ready to serve requests?

If readiness fails:

* Kubernetes does NOT restart the container.
* Instead, it removes the pod from the Service load balancing.
* Traffic is temporarily stopped.

This is very important.

Imagine:

* Your app starts but is still connecting to the database.
* You don’t want traffic during initialization.
* Readiness probe prevents early traffic.

---

## 5️ Startup Probe — "Is It Still Booting?"

Some applications take longer to start.

Startup probe tells Kubernetes:

> Don’t apply liveness or readiness checks until startup is complete.

This prevents unnecessary restarts for slow-booting apps.

---

## 6️ How Probes Work Internally

Kubernetes periodically performs checks using:

* HTTP request
* TCP socket check
* Command execution inside container

If the check returns failure enough times (based on thresholds), Kubernetes acts accordingly.

---

## 7️ How We Used Probes in Our Backend

In our Deployment YAML:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5
  periodSeconds: 5
```

What this means:

* Kubernetes calls `GET /health`
* If response is HTTP 200 → healthy
* If not → action taken

Our Flask app:

```python
@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200
```

So Kubernetes checks that endpoint regularly.

---

## 8️ Difference Between Liveness and Readiness

| Probe Type | If It Fails         | What Happens             |
| ---------- | ------------------- | ------------------------ |
| Liveness   | Container unhealthy | Container is restarted   |
| Readiness  | Not ready yet       | Pod removed from traffic |
| Startup    | Still booting       | Delays other probes      |

Understanding this difference is critical in production systems.

---

## 9️ Why Probes Matter in Microservices

In microservices architecture:

* There are many services.
* They depend on each other.
* Failures are normal.

Without probes:

* Users may hit unhealthy pods.
* Applications may hang indefinitely.
* Manual intervention would be required.

With probes:

* Kubernetes self-heals.
* Traffic routing becomes intelligent.
* Downtime reduces significantly.

---

##  What Happens Without Probes?

If probes are not defined:

* Kubernetes assumes container is healthy as long as process is running.
* Even if your app is frozen, traffic continues.
* No automatic restart occurs.

This is dangerous in production.

---

##  Real-World Scenario

Imagine a payment service:

* Service connects to external bank API.
* That API is temporarily unavailable.
* Your service becomes unresponsive.

If liveness probe detects failure:

* Pod restarts automatically.
* Service recovers.

If readiness probe detects dependency failure:

* Traffic stops temporarily.
* Other healthy pods handle requests.

This is resilient architecture.

---

##  Best Practices for Probes

* Always expose `/health` endpoint.
* Keep health check lightweight.
* Avoid expensive DB queries inside health endpoint.
* Set reasonable delays and intervals.
* Use readiness to control traffic.
* Use liveness to handle crashes.

---

## 1️ In Our AIOps Demo Context

We simulated random backend failures.

Now:

* Logs show 500 errors.
* Kubernetes still keeps pods running.
* LoadBalancer distributes traffic.
* Health checks ensure unhealthy pods can be restarted if required.

Later, when we integrate CloudWatch + Bedrock:

* AI can analyze failure patterns.
* Identify instability.
* Suggest improvements.

That’s where DevOps meets AIOps.

---

##  Summary

Health probes allow Kubernetes to:

* Monitor application health
* Restart unhealthy containers
* Control traffic routing
* Improve reliability
* Enable self-healing systems

They are foundational to production-grade Kubernetes deployments.

---

