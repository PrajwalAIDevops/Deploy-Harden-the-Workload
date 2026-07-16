# 🚀 Ledger API -   Kubernetes Deployment with ArgoCD, Sealed Secrets & GitOps

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.33-blue)
![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-orange)
![Docker](https://img.shields.io/badge/Docker-Containerization-blue)
![Sealed Secrets](https://img.shields.io/badge/Secrets-SealedSecrets-green)
![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-black)

---

# 📌 Project Overview

This project demonstrates a -style Kubernetes deployment workflow using modern DevOps practices.

The application is deployed using a **GitOps approach** where Kubernetes manifests are maintained in GitHub and automatically synchronized into the Kubernetes cluster using **ArgoCD**.

The project includes:

- Flask API application containerization
- Docker image build and deployment
- Kubernetes Deployment and Service
- Namespace isolation
- ConfigMap management
- Bitnami Sealed Secrets
- Kubernetes RBAC security
- ServiceAccount configuration
- Liveness and Readiness probes
- GitHub Actions CI pipeline
- ArgoCD GitOps continuous deployment
- Kubernetes troubleshooting and  hardening


---

# 🏗️ Architecture


```
Developer
    |
    |
    v
GitHub Repository
    |
    |
    +----------------+
    |                |
    v                v

GitHub Actions     ArgoCD

(CI Pipeline)      (GitOps CD)

    |                |
    |                |
    +-------+--------+
            |
            v

    Kubernetes Cluster

            |
            |
   +----------------+
   |                |
   v                v

Ledger API       Reporting

Deployment       Deployment

            |
            v

 Kubernetes Service

            |
            v

        Users
```

---

# 🛠️ Technology Stack


| Component | Technology |
|---|---|
| Application | Python Flask |
| Container | Docker |
| CI | GitHub Actions |
| CD | ArgoCD |
| Platform | Kubernetes |
| Secrets | Bitnami Sealed Secrets |
| Security | RBAC |
| Configuration | ConfigMap |
| Cloud | AWS EC2 Kubernetes Cluster |


---

# 📂 Repository Structure


```
Deploy-Harden-the-Workload/

│
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── Dockerfile
│
├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── sealed-secret.yaml
│   ├── serviceaccount.yaml
│   ├── role.yaml
│   └── rolebinding.yaml
│
└── README.md

```

---

# 🐳 Docker Image Build


Build Docker image:


```bash
docker build -t ledger-api:v1 .
```


Tag image:


```bash
docker tag ledger-api:v1 prajwaldevops1027/ledger-api:v1
```


Push image:


```bash
docker push prajwaldevops1027/ledger-api:v1
```


---

# ☸️ Kubernetes Deployment


Create namespace:


```bash
kubectl apply -f namespace.yaml
```


Deploy application:


```bash
kubectl apply -f deploy/
```


Verify:


```bash
kubectl get all -n payments
```


---

# 🔐 Secret Management Using Sealed Secrets


Normal Kubernetes secrets are only base64 encoded.


Sealed Secrets provides encrypted secrets that can safely be stored in Git.


Flow:


```
Secret Data

      |
      v

kubeseal

      |
      v

Encrypted Secret

      |
      v

GitHub Repository

      |
      v

Sealed Secret Controller

      |
      v

Kubernetes Secret

```


Generate secret:


```bash
kubectl create secret generic ledger-api-secret \
-n payments \
--from-literal=DB_PASSWORD=password \
--from-literal=STRIPE_API_KEY=testkey \
--dry-run=client -o yaml | \
kubeseal \
--controller-name=sealed-secrets-controller \
--controller-namespace=kube-system \
--format yaml > sealed-secret.yaml
```


---

# ❤️ Kubernetes Health Checks


The application exposes:


```
/health
```


Kubernetes uses:

## Liveness Probe

Checks whether application is alive.

Failure:

```
Container Restart
```


## Readiness Probe

Checks whether application can receive traffic.

Failure:

```
Traffic removed from Service
```


Example:


```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080


readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

---

# 🔐 RBAC Security


RBAC controls application permissions.


Flow:


```
ServiceAccount

      |
      v

RoleBinding

      |
      v

Role

      |
      v

Permissions
```


Example:


```yaml
resources:
- configmaps
- secrets

verbs:
- get
- list
```

---

# 🔄 GitHub Actions CI Pipeline


GitHub Actions automates:


- Code checkout
- Docker build
- Application validation
- Docker image push


Pipeline:


```
Developer Push

      |
      v

GitHub Actions

      |
      v

Docker Build

      |
      v

Docker Hub Push

```


Screenshot:


![GitHub Actions](screenshots/github-actions.png)


---

# 🚢 ArgoCD GitOps Deployment


ArgoCD continuously monitors GitHub repository and synchronizes Kubernetes manifests.


Flow:


```
GitHub Repository

        |

        v

ArgoCD Controller

        |

        v

Kubernetes Cluster

        |

        v

Running Application

```


ArgoCD Status:


```
Application:

ledger-api


Sync Status:

Synced ✅


Health:

Healthy ✅


Auto Sync:

Enabled ✅

```


Screenshot:


![ArgoCD Dashboard](screenshots/argocd-dashboard.png)


---

# ☸️ Kubernetes Running Pods


Check pods:


```bash
kubectl get pods -n payments
```


Expected:


```
NAME                         READY   STATUS

ledger-api                   1/1     Running

ledger-api                   1/1     Running

ledger-api                   1/1     Running

reporting                    1/1     Running

```


Screenshot:


![Kubernetes Pods](screenshots/kubernetes-pods.png)


---

# 🌐 Application Serving Successfully


Health check:


```bash
curl http://localhost:8080/health
```


Response:


```json
{
 "status": "healthy"
}
```


Application running:


![Application Running](screenshots/application-running.png)


---

# 🔍 Troubleshooting Commands


Check pods:


```bash
kubectl get pods -n payments
```


Logs:


```bash
kubectl logs <pod-name> -n payments
```


Describe:


```bash
kubectl describe pod <pod-name> -n payments
```


Events:


```bash
kubectl get events -n payments --sort-by=.lastTimestamp
```


ArgoCD:


```bash
argocd app get ledger-api
```


---

# 🧪  Issues Handled


## ImagePullBackOff

Cause:

- Wrong image name
- Image unavailable


Solution:

```bash
kubectl describe pod <pod-name>
```


---

## SealedSecret Sync Failure

Cause:

- Missing CRD
- Invalid encrypted values


Solution:

Generate secrets using kubeseal.


---

## Pod Running but Not Ready


Cause:

Health probe failure.


Example:


```
HTTP probe failed with status code 404
```


Solution:

Added `/health` endpoint and corrected probes.


---

# 📊 Deployment Validation


| Component | Status |
|-|-|
| Docker Build | ✅ |
| GitHub Actions CI | ✅ |
| Docker Hub Push | ✅ |
| Kubernetes Deployment | ✅ |
| ConfigMap | ✅ |
| Sealed Secrets | ✅ |
| RBAC | ✅ |
| ArgoCD Sync | ✅ |
| Application Health | ✅ |


---

# 📈 Future Improvements


- Prometheus Monitoring
- Grafana Dashboard
- Loki Logging
- Jenkins Pipeline
- Trivy Security Scan
- Terraform Infrastructure
- Kubernetes HPA
- Ingress TLS
- External Secrets Operator


---

# 👨‍💻 Author


**Prajwal B**

DevOps Engineer | Kubernetes | Docker | GitOps | Cloud Automation


---

# ⭐ Project Highlights


✅  Kubernetes Deployment  
✅ GitOps with ArgoCD  
✅ CI/CD Automation  
✅ Secure Secret Management  
✅ Kubernetes Security with RBAC  
✅ Application Reliability  
✅ Real  Troubleshooting Experience  
