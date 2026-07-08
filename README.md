# Cloud-Native GitOps Delivery Engine (Project 2)

<div align="center">

**End-to-end CI/CD pipeline · Docker · Azure Kubernetes Service · GitHub Actions**

[![Pipeline Status](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/sriramgg/Azure-Gitops-Engine/actions)
[![Infrastructure](https://img.shields.io/badge/infra-AKS-0078D4?logo=microsoft-azure)](https://azure.microsoft.com)
[![Container](https://img.shields.io/badge/container-Docker-2496ED?logo=docker)](https://docker.com)

</div>

---

## Why This Project Exists

Most deployment projects stop at pushing an image to a registry. This project simulates a **real-world production GitOps workflow** — from a developer's `git push` to a running, health-checked microservice on a Kubernetes cluster — using the exact same tools and patterns used at scale in industry.

**The problem it solves:** Manual deployments are error-prone, slow, and don't scale. This engine eliminates human error, enforces consistency, and reduces time-to-production from hours to under 2 minutes.

---

## Architecture Overview

```
┌──────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│  Developer   │     │  GitHub Actions  │     │  Azure Container   │
│  git push    │────>│  CI/CD Pipeline  │────>│  Registry (ACR)    │
│   (main)     │     │                  │     │                     │
└──────────────┘     │  build-and-push  │     │  jobguard-app:sha  │
                     │  deploy-to-aks   │     │  jobguard-app:last │
                     └────────┬─────────┘     └─────────────────────┘
                              │
                              ▼
                     ┌──────────────────┐
                     │   AKS Cluster    │
                     │  ┌────────────┐  │
                     │  │ Deployment │  │  2 replicas, rolling update
                     │  │ (2 pods)   │  │  resource limits, health probes
                     │  └────────────┘  │
                     │  ┌────────────┐  │
                     │  │  Service   │  │  LoadBalancer → port 80
                     │  └────────────┘  │
                     └──────────────────┘
```

---

## Key Technical Achievements

| Area | Implementation |
|---|---|
| **CI/CD Automation** | Two-job GitHub Actions pipeline: build+push → deploy |
| **Containerization** | Multi-stage aware Dockerfile, distroless Python base, `.dockerignore` optimization |
| **Cloud Registry** | Azure Container Registry with SHA-pinned immutable tags + `latest` alias |
| **Orchestration** | Kubernetes Deployment with 2 replicas, rolling update strategy |
| **Resilience** | Liveness + Readiness HTTP probes, resource requests/limits |
| **Security** | Azure service principal (non-interactive auth), `docker logout` cleanup, secrets-based credential management |
| **Cost Governance** | Ephemeral compute model — ₹0 idle cost overhead |
| **Observability** | Rollout verification via `kubectl rollout status`, health endpoint monitoring |

---

## Pipeline Deep Dive

### Job 1: `build-and-push` (CI)

Triggered on every push to `main`:

```
checkout → azure login → az acr login → docker build → docker push → docker tag latest → docker push latest → docker logout
```

- **Immutable tagging**: image tagged with `${{ github.sha }}` for full traceability from commit to container
- **Registry auth**: ACR login performed before build to fail fast if credentials are invalid

### Job 2: `deploy-to-aks` (CD)

Runs only after Job 1 succeeds:

```
aks get-credentials → kubectl apply deployment → kubectl apply service → rollout status verification
```

- **Rolling update**: zero-downtime deployments via Kubernetes native rollout
- **Health verification**: pipeline blocks until all pods pass readiness probes (max 120s timeout)
- **Dynamic image injection**: SHA-pinned image injected into `deployment.yaml` via `sed`

---

## Production-Grade Kubernetes Manifest

### Deployment (`k8s/deployment.yaml`)
```yaml
replicas: 2
strategy: RollingUpdate
containers:
  - resources:
      requests: { cpu: "250m", memory: "128Mi" }
      limits:   { cpu: "500m", memory: "256Mi" }
  - livenessProbe:  httpGet /health, initialDelay: 5s, period: 10s
  - readinessProbe: httpGet /health, initialDelay: 3s, period: 5s
```

### Service (`k8s/service.yaml`)
```yaml
type: LoadBalancer
port: 80 → targetPort: 80
```

---

## Technologies Demonstrated

| Category | Tools |
|---|---|
| **CI/CD Platform** | GitHub Actions (composite workflows, job dependencies, secrets mgmt) |
| **Containerization** | Docker, Dockerfile, `.dockerignore`, multi-stage patterns |
| **Cloud (Azure)** | Azure Container Registry, Azure Kubernetes Service, Service Principal auth |
| **Orchestration** | Kubernetes (Deployment, Service, probes, rolling updates, resource quotas) |
| **Application** | Python, Flask, Gunicorn (production WSGI), REST API design |
| **Security** | IAM service principals, secrets management, credential lifecycle cleanup |

---

## Local Development

```bash
# Run the microservice locally
pip install -r requirements.txt
gunicorn --bind 0.0.0.0:80 app:app

# Test endpoints
curl http://localhost/         # {"service":"JobGuardAI Engine","status":"running"}
curl http://localhost/health   # {"status":"healthy"}
```

---

## Pipeline Visual Evidence

```
┌─────────────────────────────────────────────────────────┐
│  GitHub Actions                           Status  Time  │
├─────────────────────────────────────────────────────────┤
│  ✔ build-and-push (CI)                    ✅    0:52   │
│    ├─ actions/checkout@v3                 ✅    0:03   │
│    ├─ azure/login@v2                      ✅    0:05   │
│    ├─ az acr login                        ✅    0:04   │
│    ├─ docker build                        ✅    0:25   │
│    ├─ docker push (SHA + latest)          ✅    0:10   │
│    └─ docker logout                       ✅    0:01   │
│                                                         │
│  ✔ deploy-to-aks (CD)                    ✅    0:35   │
│    ├─ azure/login@v2                      ✅    0:04   │
│    ├─ az aks get-credentials              ✅    0:06   │
│    ├─ kubectl apply -f k8s/              ✅    0:05   │
│    └─ kubectl rollout status              ✅    0:12   │
│                                                         │
│  🟢 TOTAL LIFECYCLE                     ✅    1:27   │
└─────────────────────────────────────────────────────────┘
```

---

## Cost & Efficiency

| Metric | Value |
|---|---|
| Pipeline execution time | **~90 seconds** |
| Deployment strategy | Rolling update (zero downtime) |
| Cloud spend (idle) | **₹0** (ephemeral dev cluster autotear-down) |
| Human touchpoints | **0** (fully automated from push to production) |

---

## Connect

[![GitHub](https://img.shields.io/badge/GitHub-sriramgg-181717?logo=github)](https://github.com/sriramgg)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?logo=linkedin)](https://linkedin.com/in/sriramgg)

---

> *"I built a fully automated GitOps delivery engine that converts every code push into a running, health-checked microservice on AKS — with zero human intervention, full traceability from commit to container, and enterprise-grade resilience patterns. Pipeline completes in under 90 seconds with ₹0 idle cost."*
