# 🚨 Post-Deployment Application Outage Playbook

This technical guide provides the systematic incident response sequence, logging commands, and remediation strategies required when a CI/CD pipeline logs a successful deployment, but the live application environment goes down.

---

## 🔍 Incident Discovery & Diagnostics
When the continuous delivery automation executes successfully but monitoring tools fire high-priority health alerts (`HTTP 502 / 503`), the engineer must immediately separate application runtime crashes from cluster configuration errors.

### 🚨 The 4 Critical Failure Vectors

#### 1. Configuration Variable Drift
* **Failure Mode:** The codebase requires new environmental context factors (`ConfigMaps`/`Secrets`) that were omitted from the target namespace during the pipeline run, leading to fatal runtime crashes.
* **The Triage:** Execute `kubectl logs <pod-name> --previous` to dump runtime stack traces before execution drops.

#### 2. Service Endpoints Misalignment
* **Failure Mode:** Label selector strings inside the Kubernetes Service configuration drift from the metadata labels baked into the new deployment template. The Service fails to route incoming load balancer traffic to the pod array.
* **The Triage:** Run `kubectl get endpoints <service-name>`. An output returning `<none>` confirms a mapping selector breakdown.

#### 3. Probe Deficit Deadlocks (Liveness / Readiness)
* **Failure Mode:** The application server starts up perfectly, but the deployment configuration targets an incorrect health path or network port. The orchestrator flags the environment as dead and blocks all routing access.
* **The Triage:** Run `kubectl describe pod <pod-name>` and look at the `Events` log block for `Readiness probe failed` warnings.

---

## 🛠️ Systematic Triage Terminal Script

When debugging a post-deployment application drop, execute this exact command string in sequence to isolate the failure layer:

```bash
echo "=== 1. Reviewing Live Pod Status Matrix ==="
kubectl get pods -l app=my-application-label -n production

echo "=== 2. Checking Service Endpoint Mappings ==="
# If this list is blank, network traffic cannot reach your application pods
kubectl get endpoints -l app=my-application-label -n production

echo "=== 3. Inspecting Orchestrator Controller Events ==="
# Reviews capacity errors, mounting dropouts, or probe tracking failures
kubectl describe pods -l app=my-application-label -n production | grep -A 10 Events

echo "=== 4. Fetching Application Stack Trace Prior to Crash ==="
# Captures runtime errors, missing database configs, or uncaught exceptions
kubectl logs deployment/my-app-deployment --previous --tail=50 -n production
```

---

## 💬 How to Explain This to an Interviewer (STAR Format)
* **Situation:** A continuous deployment pipeline executes perfectly and reports success, but the targeted live application goes down immediately, returning HTTP 502 errors to users.
* **Task:** Isolate the application lifecycle failure instantly, route traffic to a stable environment, and fix the root execution bottleneck.
* **Action:** Checked the cluster tracking nodes using logs (`--previous`) to target application boot logic. Identified a failing readiness probe caused by an un-synced environment database variable. Executed an immediate manual rollback to the last known-good container image tag (`kubectl rollout undo`) to restore public service while the configuration parameters were corrected inside the platform configuration workspace.
* **Result:** Restored system availability under 3 minutes, established strict automated fallback verification boundaries within the deployment pipeline, and achieved 100% service uptime compliance.
