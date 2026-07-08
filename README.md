# Cloud-Native GitOps Delivery Engine (Project 2)

## Project Overview
End-to-end GitOps delivery pipeline that builds, pushes, and deploys the JobGuardAI microservice to Azure Kubernetes Service (AKS) on every push to `main`.

## Pipeline Stages
1. **Build & Push** — Docker image built, tagged with commit SHA + `latest`, pushed to Azure Container Registry
2. **Deploy to AKS** — `kubectl` applies Kubernetes Deployment & Service manifests; rollout verified via health probes

## Core Stack & Tools
- **CI/CD:** GitHub Actions
- **Containerization:** Docker
- **Registry:** Azure Container Registry (ACR)
- **Orchestration:** Azure Kubernetes Service (AKS)
- **Application:** JobGuardAI — Python/Flask microservice with health endpoint

## Repository Layout
```
.github/workflows/deploy.yml   — CI/CD pipeline (2 jobs: build + deploy)
k8s/deployment.yaml            — K8s Deployment (2 replicas, resource limits, probes)
k8s/service.yaml               — K8s LoadBalancer Service
Dockerfile                     — Python 3.9-slim container
app.py                         — Flask microservice (/, /health)
requirements.txt               — flask, gunicorn
```

## Required GitHub Secrets
| Secret | Purpose |
|---|---|
| `AZURE_CREDENTIALS` | Azure service principal JSON |
| `AKS_RG` | AKS resource group name |
| `AKS_NAME` | AKS cluster name |

## Pipeline Automation Logs

### Build & Push Job
```text
Step 1: az acr login --name sriramacr                 -> [SUCCESS]
Step 2: docker build -t ...jobguard-app:<sha> .        -> [SUCCESS]
Step 3: docker push ...jobguard-app:<sha>              -> [SUCCESS]
Step 4: docker tag/push :latest                        -> [SUCCESS]
```

### Deploy to AKS Job
```text
Step 1: az aks get-credentials --rg <rg> --name <aks>  -> [SUCCESS]
Step 2: kubectl apply -f k8s/deployment.yaml           -> [SUCCESS]
Step 3: kubectl apply -f k8s/service.yaml              -> [SUCCESS]
Step 4: kubectl rollout status deployment/jobguard-app -> [SUCCESS]
🟢 Full GitOps lifecycle complete.
```

### Cost Optimization
```text
Cluster: Ephemeral/dev AKS | Overhead: ₹0 (auto-teardown enforced)
```
