# 🚨 Troubleshooting Missing Secrets / Credentials in CI/CD Pipelines

This technical operational document details the triage sequence, scoping audits, and syntax validations required when automated workflow steps fail to consume tokens injected from secure platform vaults.

---

## 🔍 Incident Discovery & Diagnostics
When a pipeline crashes during validation or infrastructure provisioning stages with `Unauthorized` or `Missing Parameter` errors, it signifies a breakdown in token accessibility, environmental scoping, or runner context handover.

### 🚨 The 4 Critical Failure Vectors

#### 1. Unassigned Environment Scopes
* **Failure Mode:** Storing a secret inside a specific target layer (e.g., `production`) but failing to declare the `environment:` boundary attribute within the execution job block, leaving the runner with zero lookup rights.
* **The Fix:** Explicitly declare environment references in the configuration YAML scripts.

#### 2. Forked Pull Request Restrictions
* **Failure Mode:** External code updates opened via repository forks evaluate with empty secret strings by default to prevent unauthorized token exfiltration through malicious pull request changes.
* **The Fix:** Restrict pipeline production validation runs strictly to internal tracking branches or separate PR reviews from build steps.

#### 3. Execution Context Isolation
* **Failure Mode:** Expecting a credentials configuration applied inside Job A to naturally persist inside an independent runner assigned to Job B.
* **The Fix:** Explicitly pass structural parameters forward using pipeline outputs or leverage central authentication providers (OIDC/Vault keys).

---

## 🛠️ Automated Injection Blueprint (GitHub Actions)

This workflow configuration demonstrates the correct methodology for setting environment contexts, verifying token existence safely, and mapping secret strings to execution runtime parameters.

```yaml
name: Secure Database Deployment Workflow
on:
  push:
    branches: [main]

jobs:
  database-upgrade:
    runs-on: ubuntu-latest
    # ✅ FIX 1: Enforcing explicit environment matching to unlock the vault
    environment: production 
    
    steps:
      - name: Fetch Application Workspace
        uses: actions/checkout@v4

      - name: Verify Secret Availability (Safe Debugging)
        run: |
          if [ -z "\${{ secrets.PROD_DB_URL }}" ]; then
            echo "❌ ERROR: Dynamic secret injection failed. Context is empty."
            exit 1
          else
            echo "✅ Validation Passed: Secret is securely loaded into the runner context."
          fi

      - name: Run Schema Evolution Tasks
        run: npm run db-migrate
        # ✅ FIX 2: Explicitly mapping vault strings to runtime environment blocks
        env:
          DATABASE_URL: \${{ secrets.PROD_DB_URL }}
```

---

## 💬 How to Explain This to an Interviewer (STAR Format)
* **Situation:** A deployment step fails because required API tokens or credentials appear as completely empty values inside the runner context, halting the deployment train.
* **Task:** Identify the secret scoping or injection breakdown immediately, re-establish token availability securely, and enforce guidelines to prevent future credential drift.
* **Action:** Audit environment-level security parameters to ensure matching configurations are mapped to the active job block. Verify the trigger source to rule out fork execution drops, and implement character existence validation steps to test variable assignment cleanly without exposing sensitive information in plain text logs.
* **Result:** Achieved 100% reliable secret injection tracking, maintained absolute log security, and eliminated environment-scoping deployment anomalies.
