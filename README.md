# Micro-Frontend CI/CD Pipeline Documentation

**Applications:** Shell (host), ABC (remote, Next.js 13 App Router), XYZ (remote)
**Toolchain:** GitLab, Jenkins, Docker, Kubernetes, ArgoCD

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Tools and Their Roles](#3-tools-and-their-roles)
4. [Repository and Branching Strategy](#4-repository-and-branching-strategy)
5. [Continuous Integration](#5-continuous-integration)
6. [Continuous Deployment](#6-continuous-deployment)
7. [Environments and Configuration](#7-environments-and-configuration)
8. [Micro-Frontend Integration Considerations](#8-micro-frontend-integration-considerations)
9. [Versioning, Rollback and Release Strategy](#9-versioning-rollback-and-release-strategy)
10. [Security](#10-security)
11. [Monitoring and Observability](#11-monitoring-and-observability)
12. [Troubleshooting](#12-troubleshooting)
13. [Appendix: Sample Files](#13-appendix-sample-files)

---

## 1. Overview

The application follows a **Micro-Frontend architecture** made up of three independently built and deployed units:

| App | Role | Status | Tech |
|-----|------|--------|------|
| **Shell** | Host / container. Owns layout, routing, auth and loads remotes at runtime | To be built | TBD |
| **ABC** | Remote app | Already built | Next.js 13 (App Router) |
| **XYZ** | Remote app | To be built | TBD |

All three apps share **one common pipeline design**. Each app has its own repository (or folder) and runs the same stages, so a change to one app never forces a rebuild of the others.

### High-level flow

```
Developer push / MR
        |
        v
   GitLab (SCM) ---- webhook ----> Jenkins (CI)
                                       |
                 install -> lint -> test -> build -> scan
                                       |
                                  Docker image
                                       |
                                       v
                              Container Registry
                                       |
                    Jenkins updates image tag in GitOps repo
                                       |
                                       v
                       ArgoCD (CD) detects change in Git
                                       |
                                       v
                      Kubernetes (dev -> staging -> prod)
```

### Goals

- Independent build, test and deploy per app
- Repeatable, containerized builds
- Git as the single source of truth (GitOps)
- Fast, safe rollbacks
- Same process for Shell, ABC and XYZ

---

## 2. Prerequisites

### Accounts and access
- GitLab group/project access (Maintainer role for pipeline setup)
- Jenkins access with permission to create pipeline jobs and credentials
- Container registry access (GitLab Container Registry, or Harbor/ECR/ACR)
- Kubernetes cluster access (kubectl context for dev, staging, prod)
- ArgoCD access (UI/CLI, and permission to create Applications)

### Software versions (recommended)

| Tool | Minimum version |
|------|-----------------|
| Node.js | 18.17+ (required by Next.js 13) |
| npm / pnpm / yarn | Match lockfile in each repo |
| Docker | 24+ |
| kubectl | Matches cluster version (+/- 1 minor) |
| Helm or Kustomize | Helm 3.x / Kustomize 5.x |
| ArgoCD CLI | Matches server version |

### Infrastructure
- A Kubernetes cluster (or one per environment) with an ingress controller (NGINX/Traefik)
- Jenkins agents able to run Docker builds (Docker-in-Docker, Kaniko or Buildah)
- A separate **GitOps repository** for Kubernetes manifests
- DNS and TLS certificates (cert-manager recommended)

### Secrets and credentials (stored in Jenkins/GitLab/K8s, never in code)
- Registry credentials
- GitLab access token (for GitOps repo updates)
- Kubernetes image pull secret
- Any runtime environment variables / API keys

---

## 3. Tools and Their Roles

| Tool | Purpose in the pipeline |
|------|--------------------------|
| **GitLab** | Source control for all apps and the GitOps repo. Handles merge requests, code review, protected branches, webhooks, and (optionally) the container registry. GitLab CI can run lightweight checks (lint, unit tests) on every MR. |
| **Jenkins** | The main CI orchestrator. Pulls code, installs dependencies, runs lint/tests/build, builds and pushes Docker images, then updates the image tag in the GitOps repo. Defined as code in a `Jenkinsfile`. |
| **Docker** | Packages each app (and its runtime) into an immutable, portable image. Multi-stage builds keep images small and consistent across environments. |
| **Container Registry** | Stores versioned images (`shell`, `abc`, `xyz`) tagged by commit SHA or semantic version. |
| **Kubernetes** | Runs the containers. Provides scaling, self-healing, rolling updates, service discovery and ingress routing for Shell, ABC and XYZ. |
| **ArgoCD** | GitOps continuous delivery. Watches the GitOps repo and syncs the cluster to match it. Gives drift detection, visual status, and one-click rollback. |
| **Helm / Kustomize** | Templates Kubernetes manifests per environment (dev, staging, prod). |
| **Quality/Security scanners** (e.g. SonarQube, Trivy, npm audit) | Check code quality, dependency vulnerabilities and image vulnerabilities before release. |

### Division of responsibility

- **GitLab** = where code lives and is reviewed
- **Jenkins** = build, test, package (CI)
- **ArgoCD** = deploy and keep the cluster in sync (CD)
- **Kubernetes** = where it runs

> Note: if the team prefers a single CI tool, GitLab CI can replace Jenkins. This document keeps both: GitLab CI for fast MR checks, Jenkins for the full build and release pipeline.

---

## 4. Repository and Branching Strategy

### Repositories

```
gitlab.com/<group>/
  |-- shell-app          # Host app
  |-- abc-app            # Remote (Next.js 13)
  |-- xyz-app            # Remote
  |-- gitops-manifests   # K8s manifests / Helm values for all apps
  |-- pipeline-library   # (optional) Jenkins shared library
```

### GitOps repo layout

```
gitops-manifests/
  |-- base/
  |     |-- shell/
  |     |-- abc/
  |     |-- xyz/
  |-- overlays/
        |-- dev/
        |-- staging/
        |-- prod/
```

### Branching
- `main`: production-ready, protected
- `develop`: integration branch (deploys to dev)
- `feature/*`, `bugfix/*`: short-lived, merged via MR
- `release/*` or Git tags (`v1.2.0`): trigger staging and prod promotion

---

## 5. Continuous Integration

CI runs on **every app** using the same stage structure.

### Triggers
- **Merge Request:** GitLab CI runs lint and unit tests (fast feedback)
- **Push to `develop` / `main` / tag:** GitLab webhook triggers the Jenkins pipeline

### Stages

| # | Stage | What happens | Tool |
|---|-------|--------------|------|
| 1 | Checkout | Clone repo at the triggering commit | Jenkins + GitLab |
| 2 | Install | `npm ci` using the lockfile (cached) | Node.js |
| 3 | Lint and Type-check | ESLint, Prettier check, `tsc --noEmit` | Node.js |
| 4 | Unit Tests | Jest/Vitest with coverage report | Node.js |
| 5 | Static Analysis | Code quality gate | SonarQube |
| 6 | Dependency Audit | `npm audit` / license check | npm / Snyk |
| 7 | Build | `next build` (ABC) / bundler build (Shell, XYZ) | Node.js |
| 8 | Docker Build | Multi-stage image build | Docker/Kaniko |
| 9 | Image Scan | Scan for CVEs; fail on critical | Trivy |
| 10 | Push | Push image tagged `<short-sha>` (and `latest` for dev only) | Registry |
| 11 | Update GitOps | Commit new image tag to `gitops-manifests` | Jenkins + Git |

### Change detection (optional, for monorepos)
If apps live in one repo, use path filters (`changes:` in GitLab, `when { changeset }` in Jenkins) so only the affected app is built.

### Image tagging convention

```
<registry>/<group>/<app>:<short-sha>        # every build
<registry>/<group>/<app>:v1.4.2             # release tags
```

Avoid deploying `latest` to staging or prod.

### CI exit criteria
A build is promotable only if: tests pass, quality gate passes, no critical vulnerabilities, and the image is pushed successfully.

---

## 6. Continuous Deployment

Deployment follows **GitOps** with ArgoCD.

### Flow

1. Jenkins pushes the image, then commits the new tag to `gitops-manifests` (e.g. `overlays/dev/abc/kustomization.yaml`).
2. ArgoCD detects the Git change (webhook or polling).
3. ArgoCD renders the manifests and applies them to the target cluster/namespace.
4. Kubernetes performs a **rolling update**: new pods start, readiness probes pass, old pods terminate.
5. ArgoCD reports the app as `Healthy` and `Synced`.

### Environment promotion

| Environment | Trigger | Sync policy | Approval |
|-------------|---------|-------------|----------|
| **dev** | Merge to `develop` | Automated | None |
| **staging** | Release branch / tag | Automated | QA sign-off |
| **prod** | Git tag on `main` | **Manual sync** | Release manager approval |

### ArgoCD structure
Use an **App of Apps** (or ApplicationSet) pattern so one parent app manages `shell`, `abc` and `xyz` per environment.

### Kubernetes resources per app
- `Deployment` (replicas, resources, probes)
- `Service` (ClusterIP)
- `Ingress` (routing and TLS)
- `ConfigMap` / `Secret`
- `HorizontalPodAutoscaler`

### Ingress routing example

| Path | Service |
|------|---------|
| `/` | shell |
| `/abc/*` or `abc.example.com` | abc |
| `/xyz/*` or `xyz.example.com` | xyz |

### Deployment order
Remotes (ABC, XYZ) should be deployed **before** a Shell release that depends on them. Remotes must stay backward compatible so the Shell can be released independently.

---

## 7. Environments and Configuration

| Setting | Dev | Staging | Prod |
|---------|-----|---------|------|
| Replicas | 1 | 2 | 3+ (HPA) |
| Domain | dev.example.com | stg.example.com | app.example.com |
| Remote URLs | dev remotes | staging remotes | prod remotes |
| Log level | debug | info | warn |

Configuration rules:
- Build once, deploy many: the same image is promoted across environments.
- Environment-specific values come from ConfigMaps/Secrets at **runtime**, not build time.
- For Next.js, public values (`NEXT_PUBLIC_*`) are inlined at build time. Use runtime config or a `/config.json` endpoint so one image works everywhere.

---

## 8. Micro-Frontend Integration Considerations

- **Remote loading:** The Shell loads ABC and XYZ at runtime (Module Federation, or another runtime composition approach). The remote URLs should come from configuration, not hard-coded.
- **Next.js 13 App Router caveat:** Module Federation support for the App Router is limited (the `nextjs-mf` plugin focuses on the Pages Router). Confirm the integration approach for ABC early. Common alternatives: expose ABC via iframe, web components, or route-level composition behind the ingress/reverse proxy.
- **Shared dependencies:** Pin and share React/React-DOM versions across apps as singletons to avoid duplicate runtimes.
- **Contract versioning:** Treat each remote's exposed modules as a public API. Avoid breaking changes without a version bump.
- **Fallbacks:** The Shell should handle a remote being unavailable (error boundary and fallback UI).
- **Cache control:** Use content-hashed filenames for assets. Serve remote entry files (e.g. `remoteEntry.js`) with `no-cache` so new releases are picked up immediately.
- **CORS:** Allow the Shell origin on remote hosts.

---

## 9. Versioning, Rollback and Release Strategy

- **Versioning:** Semantic versioning per app, using Git tags.
- **Rollback (fast):** In ArgoCD, roll back to a previous synced revision, or `git revert` the image-tag commit in the GitOps repo.
- **Rollback (Kubernetes):** `kubectl rollout undo deployment/<app>` (emergency only; ArgoCD will show drift).
- **Strategies:** Rolling update by default. Blue/green or canary (via Argo Rollouts) for high-risk releases.
- **Health gates:** Readiness and liveness probes must pass before traffic shifts.

---

## 10. Security

- Store secrets in Jenkins credentials, GitLab CI variables, or an external manager (Vault, Sealed Secrets, External Secrets).
- Run containers as **non-root**, with a read-only root filesystem where possible.
- Scan images (Trivy) and dependencies on every build.
- Protect `main` and `develop`; require MR approvals.
- Apply RBAC: Jenkins can push images and update GitOps only; ArgoCD alone has deploy rights to the cluster.
- Use NetworkPolicies and TLS everywhere.
- Sign images (cosign) if compliance requires it.

---

## 11. Monitoring and Observability

- **Metrics:** Prometheus and Grafana (pod CPU/memory, request rates, errors)
- **Logs:** Loki or ELK, aggregated per app
- **Frontend monitoring:** Sentry or similar for runtime JS errors per remote
- **Alerts:** Failed pipelines (Jenkins/GitLab notifications), ArgoCD `OutOfSync` or `Degraded` (Slack/email)
- **Pipeline metrics:** Build duration, success rate, deployment frequency, lead time

---

## 12. Troubleshooting

| Symptom | Likely cause | Action |
|---------|--------------|--------|
| Jenkins build not triggered | Webhook misconfigured | Check GitLab webhook delivery log |
| Docker build fails on install | Lockfile/Node mismatch | Align Node version in Dockerfile and CI |
| Image push denied | Expired registry credentials | Rotate credentials in Jenkins |
| ArgoCD `OutOfSync` | Manual cluster change or bad manifest | Review diff; re-sync or fix Git |
| Pods `ImagePullBackOff` | Missing pull secret or wrong tag | Verify secret and tag in GitOps repo |
| Pods `CrashLoopBackOff` | Bad config or missing env vars | `kubectl logs` / `describe pod` |
| Shell cannot load a remote | Wrong remote URL, CORS, or cached entry file | Check config, CORS headers, cache headers |

---

## 13. Appendix: Sample Files

### 13.1 `.gitlab-ci.yml` (MR checks)

```yaml
stages: [verify]

verify:
  stage: verify
  image: node:18-alpine
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths: [node_modules/]
  script:
    - npm ci
    - npm run lint
    - npm test -- --ci
```

### 13.2 `Dockerfile` (Next.js app such as ABC)

```dockerfile
# ---- Build stage ----
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- Runtime stage ----
FROM node:18-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup -S app && adduser -S app -G app
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
USER app
EXPOSE 3000
CMD ["node", "server.js"]
```

> Requires `output: 'standalone'` in `next.config.js`. For a static Shell/XYZ build, use an `nginx:alpine` runtime stage to serve the `dist/` folder.

### 13.3 `Jenkinsfile`

```groovy
pipeline {
  agent any
  environment {
    REGISTRY = 'registry.example.com/mygroup'
    APP_NAME = 'abc'              // shell | abc | xyz
    IMAGE_TAG = "${env.GIT_COMMIT.take(8)}"
  }
  stages {
    stage('Install')  { steps { sh 'npm ci' } }
    stage('Lint')     { steps { sh 'npm run lint' } }
    stage('Test')     { steps { sh 'npm test -- --ci --coverage' } }
    stage('Build')    { steps { sh 'npm run build' } }
    stage('Docker Build') {
      steps { sh "docker build -t $REGISTRY/$APP_NAME:$IMAGE_TAG ." }
    }
    stage('Scan') {
      steps { sh "trivy image --exit-code 1 --severity CRITICAL $REGISTRY/$APP_NAME:$IMAGE_TAG" }
    }
    stage('Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'registry-creds',
                         usernameVariable: 'U', passwordVariable: 'P')]) {
          sh '''
            echo $P | docker login registry.example.com -u $U --password-stdin
            docker push $REGISTRY/$APP_NAME:$IMAGE_TAG
          '''
        }
      }
    }
    stage('Update GitOps Repo') {
      steps {
        withCredentials([string(credentialsId: 'gitlab-token', variable: 'TOKEN')]) {
          sh '''
            git clone https://oauth2:$TOKEN@gitlab.com/mygroup/gitops-manifests.git
            cd gitops-manifests/overlays/dev/$APP_NAME
            kustomize edit set image $REGISTRY/$APP_NAME=$REGISTRY/$APP_NAME:$IMAGE_TAG
            git commit -am "chore($APP_NAME): deploy $IMAGE_TAG to dev"
            git push origin main
          '''
        }
      }
    }
  }
  post {
    failure { echo 'Build failed - notify team' }
  }
}
```

### 13.4 Kubernetes `Deployment` and `Service`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: abc
spec:
  replicas: 2
  selector:
    matchLabels: { app: abc }
  template:
    metadata:
      labels: { app: abc }
    spec:
      containers:
        - name: abc
          image: registry.example.com/mygroup/abc:REPLACED_BY_PIPELINE
          ports: [{ containerPort: 3000 }]
          readinessProbe:
            httpGet: { path: /, port: 3000 }
          resources:
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 512Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: abc
spec:
  selector: { app: abc }
  ports: [{ port: 80, targetPort: 3000 }]
```

### 13.5 ArgoCD `Application`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: abc-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/mygroup/gitops-manifests.git
    targetRevision: main
    path: overlays/dev/abc
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Implementation Checklist

- [ ] Create repos for Shell and XYZ; create `gitops-manifests` repo
- [ ] Add `Dockerfile` and `Jenkinsfile` to each app
- [ ] Configure GitLab webhooks to Jenkins
- [ ] Add credentials to Jenkins (registry, GitLab token)
- [ ] Create namespaces, ingress and pull secrets in Kubernetes
- [ ] Register ArgoCD Applications (dev, staging, prod)
- [ ] Decide and validate the remote-loading approach for ABC (Next.js 13 App Router)
- [ ] Set up monitoring and alerts
- [ ] Run end-to-end test: commit, build, push, sync, verify in dev
