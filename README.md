# MediMesh CI/CD System — `template-ci`

> **One reusable GitHub Actions workflow for all MediMesh / CareSync microservices.**
> All CI/CD logic lives here. Microservice repos only pass inputs and secrets.

---

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Pipeline Stages](#2-pipeline-stages)
3. [Docker Tagging Strategy](#3-docker-tagging-strategy)
4. [Email Notifications](#4-email-notifications)
5. [Complete Setup Guide](#5-complete-setup-guide)
6. [User Guide — Day-to-Day Usage](#6-user-guide--day-to-day-usage)
7. [Adding a New Microservice](#7-adding-a-new-microservice)
8. [Secrets Reference](#8-secrets-reference)
9. [Inputs Reference](#9-inputs-reference)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│  Each microservice repo  (auth / doctor / patient / …)   │
│                                                          │
│   .github/workflows/<service>.yml  ← CALLER ONLY        │
│   └─ uses: medimesh-org/template-ci/                     │
│            .github/workflows/main-template.yml@main      │
│   └─ with: { service_name, docker_image_name, … }        │
│   └─ secrets: { DOCKERHUB_TOKEN, SNYK_TOKEN, … }         │
└──────────────────────┬───────────────────────────────────┘
                       │  workflow_call
                       ▼
┌──────────────────────────────────────────────────────────┐
│  template-ci  (ALL CI/CD logic)                          │
│  main-template.yml                                        │
│  ├── Job 1: SonarQube analysis                           │
│  ├── Job 2: Install dependencies (npm install)           │
│  ├── Job 3: Snyk security scan                           │
│  ├── Job 4: Docker build  → outputs image_tag            │
│  ├── Job 5: Trivy container scan                         │
│  ├── Job 6: Push to Docker Hub  (skipped for PRs)        │
│  ├── Job 7: Manual approval gate  (production only)      │
│  ├── Job 8: CD — update argocd-manifests/values.yaml     │
│  └── Job 9: Success email notification                   │
│  + Per-job failure email on every job                    │
└──────────────────────┬───────────────────────────────────┘
                       │  git push (values.yaml updated)
                       ▼
┌──────────────────────────────────┐
│  argocd-manifests repo           │
│  values.yaml ← image tag updated │
│  ArgoCD auto-syncs → Kubernetes  │
└──────────────────────────────────┘
```

**Core principle:** Zero CI/CD logic in microservice repos. One template scales to 10+ services.

---

## 2. Pipeline Stages

| # | Job | Depends On | Condition |
|---|-----|-----------|-----------|
| 1 | `sonarqube` | — | Always |
| 2 | `install` | sonarqube | Always |
| 3 | `snyk` | install | Always |
| 4 | `docker-build` | snyk | Always |
| 5 | `trivy` | docker-build | Always |
| 6 | `docker-push` | docker-build + trivy | **Not PRs** |
| 7 | `approval` | docker-push | **Production only** |
| 8 | `cd` | docker-push + approval | **Not PRs** |
| 9 | `notify-success` | all jobs | **Success + Not PRs** |

Each job also runs a **failure email step** with `if: failure()`.

### Job Details

**Job 1 — SonarQube Analysis**
- Full git history (`fetch-depth: 0`)
- Runs `npm install` for coverage context
- Scans with `sonarqube-scan-action`
- Checks Quality Gate result (fails pipeline if RED)

**Job 2 — Install Dependencies**
- Runs `npm install` (no build script — Node.js backend only)
- Verifies package count
- Cached via `actions/setup-node` npm cache

**Job 3 — Snyk Security Scan**
- Scans `package.json` + `package-lock.json`
- Fails pipeline on **CRITICAL** vulnerabilities
- Uploads SARIF report to GitHub Security tab

**Job 4 — Docker Build**
- Computes image tag based on branch/event (see §3)
- Builds with BuildKit + GHA cache for speed
- Saves image as `.tar.gz` artifact for downstream jobs
- Outputs: `image_tag`, `full_image`, `is_pr`

**Job 5 — Trivy Container Scan**
- Loads image from artifact
- Fails on CRITICAL CVEs (`exit-code: 1`)
- Also prints human-readable table summary
- Uploads SARIF to GitHub Security tab

**Job 6 — Docker Push** *(skipped for PRs)*
- Authenticates to Docker Hub
- Pushes versioned image tag
- Also pushes `latest` on production branch
- Creates a Git tag on the source repo (production)

**Job 7 — Manual Approval Gate** *(production branch only)*
- Uses GitHub Environment `production`
- Pipeline pauses until a designated reviewer approves in the GitHub UI
- Configure required reviewers in: `Repo → Settings → Environments → production`

**Job 8 — CD (ArgoCD GitOps)**
- Clones `argocd-manifests` repo via PAT
- Updates the `tag:` field under `values_key` in `values.yaml`
- Commits with `[ci skip]` to prevent loops
- ArgoCD auto-syncs to Kubernetes

**Job 9 — Success Notification**
- Runs only when ALL jobs succeed (`if: success()`)
- Skipped for pull requests
- Sends HTML email with full deployment summary

---

## 3. Docker Tagging Strategy

| Branch | Event | Tag Format | Example |
|--------|-------|------------|---------|
| `dev` | push | `dev-<7-char-sha>` | `dev-a1b2c3d` |
| `production` | push | `v<MAJOR>.<MINOR>.<PATCH+1>` | `v1.0.3` |
| any | pull_request | `pr-<7-char-sha>` | `pr-ff00abc` |

- Production: reads latest `v*.*.*` git tag and auto-increments patch
- PRs: image is built + scanned but **never pushed**
- Production: also pushes `latest` tag

---

## 4. Email Notifications

### Failure Emails (Per Job)
Every job has a step:
```yaml
- name: "📧 Notify — <Stage> Failure"
  if: failure()
  uses: dawidd6/action-send-mail@v3
```
Triggered if that specific job fails. Email includes:
- Service name, stage name that failed
- Branch, commit SHA, author (github.actor)
- Direct link to the failed run

### Success Email (Final Job)
The `notify-success` job runs only when the full pipeline succeeds:
```yaml
if: success() && github.event_name != 'pull_request'
```
HTML email includes: service, image tag, branch, commit, author, run link.

### Email is skipped if SMTP secrets are not set
Jobs won't error if `SMTP_USERNAME` / `SMTP_PASSWORD` / `ALERT_EMAIL` are empty.

---

## 5. Complete Setup Guide

### Step 1 — Create GitHub Organization

1. Go to [github.com/organizations/new](https://github.com/organizations/new)
2. Create org: `medimesh-org` (or your org name)
3. Create these repositories inside the org:
   - `template-ci` ← set to **Public** or **Internal**
   - `argocd-manifests`
   - `auth-service`, `doctor-service`, `patient-service`, `appointment-service`, `frontend`

### Step 2 — Push Code to GitHub

```bash
# template-ci repo
cd caresync-org/template-ci
git init
git remote add origin https://github.com/medimesh-org/template-ci.git
git add .
git commit -m "feat: add reusable CI/CD workflow"
git push -u origin main

# Each microservice (repeat for all)
cd caresync-org/auth-service
git remote set-url origin https://github.com/medimesh-org/auth-service.git
git add .
git commit -m "feat: add CI workflow"
git push -u origin dev

# ArgoCD manifests
cd caresync-org/argoCD-manifests
git init
git remote add origin https://github.com/medimesh-org/argocd-manifests.git
git add values.yaml
git commit -m "feat: initial values.yaml"
git push -u origin main
```

### Step 3 — Set `template-ci` Visibility

`GitHub → template-ci → Settings → General → Change repository visibility → Public`

> The reusable workflow must be accessible to all microservice repos.

### Step 4 — Create Docker Hub Access Token

1. Log in to [hub.docker.com](https://hub.docker.com)
2. **Account Settings → Security → New Access Token**
3. Name: `medimesh-ci` | Permissions: `Read, Write, Delete`
4. Copy the token (you won't see it again)

### Step 5 — Set Up Snyk

1. Sign up at [snyk.io](https://snyk.io)
2. **Account Settings → Auth Token** → copy
3. Add as `SNYK_TOKEN` secret

### Step 6 — Set Up SonarQube

**Option A — SonarCloud (recommended, free for public repos)**
1. Go to [sonarcloud.io](https://sonarcloud.io) → Login with GitHub
2. Import your GitHub org → select each repo
3. **My Account → Security → Generate Token** → copy
4. `SONARQUBE_TOKEN` = token, `SONARQUBE_URL` = `https://sonarcloud.io`

**Option B — Self-hosted**
```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
# Access http://localhost:9000, set admin password
# Administration → Projects → Create project
```

### Step 7 — Create GitHub PAT for ArgoCD Manifest Repo

1. `GitHub → Settings → Developer settings → Personal access tokens → Fine-grained`
2. Repository access: **Only `argocd-manifests`**
3. Permissions: Contents = **Read + Write**
4. Generate → copy token → add as `MANIFEST_REPO_PAT` to all microservice repos

### Step 8 — Add Secrets to Each Microservice Repo

Go to: `GitHub → <service-repo> → Settings → Secrets and variables → Actions → New secret`

| Secret | Value |
|--------|-------|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Token from Step 4 |
| `SNYK_TOKEN` | Token from Step 5 |
| `SONARQUBE_TOKEN` | Token from Step 6 |
| `SONARQUBE_URL` | SonarCloud or self-hosted URL |
| `MANIFEST_REPO_PAT` | PAT from Step 7 |
| `SMTP_USERNAME` | Gmail address (optional) |
| `SMTP_PASSWORD` | Gmail App Password (optional) |
| `ALERT_EMAIL` | Recipient email (optional) |

> **Tip:** Use Organization Secrets to set once for all repos:
> `GitHub → Org Settings → Secrets and variables → Actions`

### Step 9 — Configure Gmail for SMTP (optional)

1. Enable 2FA on your Gmail account
2. Go to [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
3. App: **Mail** | Device: **Other** → name it `medimesh-ci`
4. Copy the 16-character app password → use as `SMTP_PASSWORD`

### Step 10 — Create Production GitHub Environment

In **each microservice repo** (not template-ci):

1. `Settings → Environments → New environment`
2. Name: `production`
3. **Required reviewers:** Add yourself / teammates
4. **Deployment branches:** Restrict to `production` branch only
5. Click **Save protection rules**

### Step 11 — Install ArgoCD on Kubernetes

```bash
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods
kubectl wait --for=condition=available --timeout=300s \
  deployment/argocd-server -n argocd

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
```

Create the ArgoCD Application:
```bash
argocd login localhost:8080 --username admin --insecure

argocd app create medimesh \
  --repo https://github.com/medimesh-org/argocd-manifests \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace medimesh \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

---

## 6. User Guide — Day-to-Day Usage

### Triggering the Pipeline

The pipeline triggers **automatically** on:
- `git push` to `dev` or `production`
- Opening/updating a Pull Request targeting `dev` or `production`

```bash
# Start a feature on dev
git checkout dev
git pull origin dev
# ... make your changes ...
git add .
git commit -m "feat: add new endpoint"
git push origin dev
# ✅ Pipeline starts automatically
```

### What Happens on `dev` push

```
push → sonarqube → install → snyk → docker-build (tag: dev-abc1234)
     → trivy → docker-push → cd (updates values.yaml) → success email
```
No manual approval needed. ArgoCD syncs within 1-3 minutes.

### What Happens on `production` push

```
push → sonarqube → install → snyk → docker-build (tag: v1.0.x)
     → trivy → docker-push → ⏸ APPROVAL GATE → cd → success email
```
A reviewer must click **Approve** in GitHub before deployment proceeds.

### What Happens on Pull Request

```
PR → sonarqube → install → snyk → docker-build (tag: pr-abc1234)
   → trivy
   (docker-push, cd, approval are SKIPPED)
   (failure emails still send if any job fails)
```
PRs only validate code quality and security — no deployment.

### Approving a Production Deployment

1. Go to: `GitHub → <service-repo> → Actions`
2. Click the running workflow
3. Find the **"✅ Awaiting Production Approval"** job
4. Click **Review deployments** → check `production` → **Approve and deploy**
5. CD job runs and ArgoCD syncs

### Viewing Pipeline Results

- **GitHub Actions:** `https://github.com/medimesh-org/<service>/actions`
- **Step Summary:** Each run shows a job results table in the Summary tab
- **Security findings:** `Repo → Security → Code scanning`
- **ArgoCD UI:** `https://localhost:8080` (or your cluster URL)

### Checking Deployed Image Version

```bash
# In argocd-manifests repo
cat values.yaml | grep tag
# or
argocd app get medimesh
```

### Manually Re-running a Failed Job

1. Go to the failed run in GitHub Actions
2. Click **Re-run failed jobs** (top right)
3. Only the failed job and its dependents re-run — saves time

### Releasing to Production

```bash
# Merge dev → production
git checkout production
git merge dev
git push origin production
# Pipeline runs with semantic version tag (v1.0.x)
# Waits for your approval before deploying
```

---

## 7. Adding a New Microservice

1. Create the repo under `medimesh-org`
2. Add `Dockerfile` at repo root
3. Create `.github/workflows/<service>.yml`:

```yaml
name: <Service> CI/CD

on:
  push:
    branches: [dev, production]
  pull_request:
    branches: [dev, production]

jobs:
  pipeline:
    name: "<Service> — Full Pipeline"
    uses: medimesh-org/template-ci/.github/workflows/main-template.yml@main
    with:
      service_name:         "<service>"
      service_path:         "."
      docker_image_name:    "medimesh-<service>"
      sonar_project_key:    "medimesh-<service>"
      values_key:           "<service>.image.tag"
      node_version:         "20"
      manifest_repo:        "medimesh-org/argocd-manifests"
      manifest_values_path: "values.yaml"
    secrets:
      DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
      DOCKERHUB_TOKEN:    ${{ secrets.DOCKERHUB_TOKEN }}
      SNYK_TOKEN:         ${{ secrets.SNYK_TOKEN }}
      SONARQUBE_TOKEN:    ${{ secrets.SONARQUBE_TOKEN }}
      SONARQUBE_URL:      ${{ secrets.SONARQUBE_URL }}
      MANIFEST_REPO_PAT:  ${{ secrets.MANIFEST_REPO_PAT }}
      SMTP_USERNAME:      ${{ secrets.SMTP_USERNAME }}
      SMTP_PASSWORD:      ${{ secrets.SMTP_PASSWORD }}
      ALERT_EMAIL:        ${{ secrets.ALERT_EMAIL }}
```

4. Add to `argocd-manifests/values.yaml`:
```yaml
<service>:
  enabled: true
  image:
    repository: medimesh-<service>
    tag: dev-0000000
```

5. Add the 6 required secrets to the repo
6. Create `production` GitHub Environment with reviewers
7. Push to `dev` — done ✅

---

## 8. Secrets Reference

| Secret | Required | Description |
|--------|----------|-------------|
| `DOCKERHUB_USERNAME` | ✅ | Docker Hub login username |
| `DOCKERHUB_TOKEN` | ✅ | Docker Hub access token |
| `SNYK_TOKEN` | ✅ | Snyk API token |
| `SONARQUBE_TOKEN` | ✅ | SonarQube/SonarCloud user token |
| `SONARQUBE_URL` | ✅ | SonarQube server URL |
| `MANIFEST_REPO_PAT` | ✅ | GitHub PAT with Contents:Write on argocd-manifests |
| `SMTP_USERNAME` | ❌ | Gmail sender address |
| `SMTP_PASSWORD` | ❌ | Gmail App Password (16 chars) |
| `ALERT_EMAIL` | ❌ | Email recipient for notifications |

---

## 9. Inputs Reference

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `service_name` | ✅ | — | Short name: `auth`, `doctor`, etc. |
| `service_path` | ❌ | `.` | Path to service root in repo |
| `docker_image_name` | ✅ | — | Docker Hub image name |
| `sonar_project_key` | ✅ | — | SonarQube project key |
| `values_key` | ✅ | — | YAML key in values.yaml, e.g. `auth.image.tag` |
| `node_version` | ❌ | `20` | Node.js version |
| `manifest_repo` | ❌ | `medimesh-org/argocd-manifests` | Manifest repo |
| `manifest_values_path` | ❌ | `values.yaml` | Path to values file |

---

## 10. Troubleshooting

| Problem | Fix |
|---------|-----|
| `workflow_call` not found | Make `template-ci` public or internal in your org |
| SonarQube Quality Gate always RED | Token needs Execute Analysis permission; create project first |
| Snyk SARIF upload fails | Non-blocking — `continue-on-error: true` is already set |
| Docker push unauthorized | Regenerate Docker Hub token; ensure username is lowercase |
| Approval gate not showing | `production` Environment must be in the microservice repo, not template-ci |
| ArgoCD not syncing | Verify PAT has `Contents: Write` on `argocd-manifests` repo |
| Semantic version conflict | Delete old tag: `git push origin :refs/tags/v1.0.x` |
| Emails not sending | Verify Gmail App Password (not your login password); enable 2FA first |
| CD job skipped on `dev` | Normal — `approval` is skipped for dev; CD runs automatically |

---

*MediMesh / CareSync — Production CI/CD. Zero duplication. One template. All services.*
