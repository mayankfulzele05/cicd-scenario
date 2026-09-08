# 🚨 Troubleshooting Container Image Registry Push Failures

This technical manual details the triage sequence and architectural fixes required when a container image compiles successfully inside a local workspace but fails to push to an upstream registry.

---

## 🔍 Incident Diagnostics & Common Errors
When a pipeline logs a successful `docker build` operation but crashes immediately during the `docker push` lifecycle phase, it indicates a breakdown in transport layer security, resource permissions, or naming configurations.

### 🚨 The 4 Common Failure Signposts

#### 1. `unauthorized: Authentication required`
* **Root Cause:** The authentication credentials provided to the runner have either expired or were never passed correctly. 
* **The Solution:** Implement automated credential generation directly preceding the push action. For AWS ECR, leverage OpenID Connect (OIDC) or invoke `aws ecr get-login-password`.

#### 2. `denied: requested access to the resource is denied`
* **Root Cause:** The image was not tagged with the target registry's URL prefix, or the runner's service account lacks write/upload IAM access scopes to the target repository.
* **The Solution:** Tag the image explicitly with the destination endpoint (`registry-url/repo-name:tag`) and verify that your registry's access control policies (RBAC) permit push actions.

#### 3. `Repository not found`
* **Root Cause:** Many modern registries require repository footprints to be initialized before they accept image pushes.
* **The Solution:** Add a pipeline initialization step that executes `aws ecr create-repository` or configure your registry's system options to dynamically create repositories on demand.

#### 4. `Retrying in X seconds... EOF / Connection reset`
* **Root Cause:** Network-level packet dropping caused by corporate firewall upload limits, slow internet uplinks on self-hosted instances, or a reverse proxy timeout.
* **The Solution:** Optimize runner network pathways or configure smaller layer sizes via multi-stage builds.

---

## 🛠️ Automated Solution Blueprint (GitHub Actions)

This optimized configuration leverages modern BuildKit engine features to handle dynamic authentication, correct tagging, and remote registry layer caching seamlessly.

```yaml
name: Secure Image Distribution Workflow
on:
  push:
    branches: [main]

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Source Code
        uses: actions/checkout@v4

      - name: Initialize Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Authenticate to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: \${{ github.actor }}
          password: \${{ secrets.GITHUB_TOKEN }} # ✅ Dynamically generated single-use credential token

      - name: Compile and Push Container Image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile
          push: true # ✅ Pushes automatically upon compilation success
          # ✅ Explicit registry endpoint tagging prevents routing failure
          tags: ghcr.io/ github.repository /my-app:{{ github.sha }}
          # ✅ BuildKit Remote Caching prevents slow layer re-uploads
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 💬 How to Explain This to an Interviewer (STAR Format)
* **Situation:** A continuous integration deployment workflow builds an application image successfully, but the push process errors out with access denied errors, blocking release delivery.
* **Task:** Identify the authentication or naming configuration bottleneck preventing image upload and restore the pipeline release cycle.
* **Action:** Audit the image namespace and rewrite the tagging directives to include the explicit registry domain URI. Next, enforce fresh, automated token refreshing using OIDC or runtime injection methods to ensure credentials do not expire during the build cycle.
* **Result:** Achieved 100% reliable image distribution workflows, lowered registry access vectors via dynamic credentials, and reduced upload times using BuildKit registry caching.
