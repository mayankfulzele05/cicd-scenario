# 🚨 Security Incident Response Playbook: Secret Log Leakage (Q87)

This operational manual details the immediate mitigation protocols, post-breach log-purging workflows, and automated pipeline guardrails required when a secret or access credential is leaked in plain text inside execution logs.

---

## 🚒 Active Firefighter Mode: The Triage Checklist
When a cryptographic key, database string, or access token is exposed in plain text within the CI console history, follow this rigid response sequence without delay:

*   **Step 1: Invalidate the Target Key:** Instantly revoke the compromised credential at the target resource provider (e.g., rotate AWS keys, regenerate API tokens, alter database passwords). 
*   **Step 2: Isolate Central Logs:** Temporarily freeze public visibility of the failing build execution window.
*   **Step 3: Purge the Workload Run:** Leverage the administrative platform API to completely drop the execution log history.
*   **Step 4: Sweep External SIEM Indexes:** If log forwarding is enabled, issue targeted data retention removal updates inside Splunk/Datadog to wipe out the cached data segments.

---

## 🛠️ Automated Defenses & Prevention Blueprints

To enforce strict protection standards and make it impossible to casually print sensitive items, pipelines must combine **automated token masking** with **static scan validation steps**.

### 1. Enforce Masking Injection (GitHub Actions Standard)
Modern continuous integration engines naturally attempt to intercept known vault values and overwrite them with mask layers (`***`). Ensure your injection format maps vault keys to execution steps correctly:

```yaml
jobs:
  secure-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # ❌ Dangerous Anti-Pattern: Will print the secret in plain text logs
      - name: Broken Plain Text Debug
        run: echo "The token is \${{ secrets.API_TOKEN }}" 

      # ✅ Secure Production Practice: Injected via env block (Masked automatically)
      - name: Correct Inline Variable Execution
        run: npm run deploy
        env:
          MAPPED_SECRET_TOKEN: \${{ secrets.API_TOKEN }}
```

### 2. Shift to Short-Lived Dynamic Identities (OIDC)
The ultimate defense against secret leaks is eliminating long-lived credentials entirely. Transition infrastructure configurations to utilize **OpenID Connect (OIDC)**:
* Instead of storing a permanent AWS Access Key in your platform variables, the runner requests a dynamic, cryptographic identity token that expires automatically within minutes.
* **Result:** Even if an engineering accident exposes the running token in your logs, the credential becomes entirely useless to an outside attacker before they can scan and extract it.

### 3. Pre-Commit and In-Pipeline Gitleaks Scanning
Integrate an explicit credential-scanning node directly into your parallel validation stages to catch committed keys or configuration leaks instantly:

```yaml
  security-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # Required to scan full commit history depth
      - name: Execute Gitleaks Scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: \${{ secrets.GITHUB_TOKEN }}
```

---

## 💬 How to Explain This to an Interviewer (STAR Format)
* **Situation:** An engineering debugging process accidentally prints a high-privilege production API credential in plain text within public pipeline execution logs, creating a major security risk.
* **Task:** Execute a comprehensive security response playbook to invalidate the compromised assets immediately, purge logs completely across all tracking storage systems, and implement safety guardrails.
* **Action:** Acted instantly by revoking the leaked token at the vendor engine before attempting log modification. Used platform administrative APIs to wipe out the run history and coordinated with the security team to scrub indices inside centralized logging platforms. Finally, shifted the pipeline architecture to use short-lived OIDC keys and integrated automated Gitleaks secret scanners to block plain text pushes.
* **Result:** Successfully contained the threat within minutes, eliminated permanent database credential footprints from automation logs, and established automated gate checks.
