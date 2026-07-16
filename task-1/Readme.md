# Task 1 – Deploy & Harden the Workload

## Overview

This task focuses on deploying and securing the **ledger-api** application in Kubernetes using production-oriented security practices. The objective was to transform the provided insecure workload into a hardened deployment by applying least privilege, container security, configuration management, health monitoring, and Kubernetes admission controls.

The application was deployed on an **AWS EC2-based Kubernetes cluster**, validated successfully, and secured using Kubernetes native security features.

---

# Architecture

```text
                           Internet
                               │
                               │
                           Ingress
                               │
                               ▼
                     +------------------+
                     | ClusterIP Service|
                     +------------------+
                               │
                               ▼
                    +--------------------+
                    |  Ledger API Pods   |
                    |     (3 Replicas)   |
                    +--------------------+
                               │
        ┌──────────────────────┼─────────────────────┐
        │                      │                     │
        ▼                      ▼                     ▼
   ConfigMap               Secret             ServiceAccount
        │                      │                     │
        └───────────────RBAC Permissions────────────┘
                               │
                               ▼
                      Kyverno Admission Policies
```

---

# Project Structure

```text
Deploy-Harden-the-Workload/

├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── sealed-secret.yaml
│   ├── serviceaccount.yaml
│   ├── role.yaml
│   ├── rolebinding.yaml
│   ├── reporting.yaml
│   ├── kyverno-disallow-root.yaml
│   └── kyverno-disallow-latest.yaml
│
├── screenshots/
│
└── README.md
```

---

# Environment

| Component         | Details              |
| ----------------- | -------------------- |
| Kubernetes        | kubeadm Cluster      |
| Hosting           | AWS EC2              |
| Container Runtime | containerd           |
| Operating System  | Ubuntu 24.04         |
| kubectl           | Latest Stable        |
| Kyverno           | Admission Controller |

---

# Implementation

## 1. Namespace Isolation

A dedicated namespace named **payments** was created to isolate the application from other workloads running inside the Kubernetes cluster.

Benefits:

* Resource isolation
* Easier RBAC management
* Better organization
* Reduced blast radius

---

## 2. Application Deployment

The application was deployed using a Kubernetes Deployment with three replicas.

Deployment Features

* High Availability
* Replica Management
* Rolling Updates
* Self Healing
* Automatic Restart

The application listens on **port 8080**.

---

## 3. Kubernetes Service

A ClusterIP Service exposes the application internally within the cluster.

Traffic Flow

```text
Client
   │
ClusterIP Service
   │
Ledger API Pods
```

---

## 4. Externalized Configuration

Application configuration was separated from the container image by using ConfigMaps.

Stored values include:

* APP_ENV
* APP_PORT

Keeping configuration outside the image allows updates without rebuilding the application.

---

## 5. Secret Management

Sensitive information was removed from application code and stored separately.

Examples:

* STRIPE_API_KEY
* DB_PASSWORD

For production environments, a **Sealed Secret** manifest has been included to demonstrate encrypted secret management.

During local testing, a standard Kubernetes Secret was used because the Sealed Secrets controller was not installed in the cluster.

---

## 6. Service Account

A dedicated ServiceAccount (**ledger-api-sa**) was created instead of using the default Kubernetes ServiceAccount.

This follows the Principle of Least Privilege and avoids granting unnecessary permissions to the application.

---

## 7. RBAC

Role-Based Access Control was implemented to restrict what the application can access inside the cluster.

Only minimal read permissions were granted for Kubernetes resources required by the application.

This prevents unnecessary access to cluster resources.

---

## 8. Container Hardening

Each container was configured using a hardened SecurityContext.

Security controls implemented:

* Non-root execution
* User ID 1000
* Privilege escalation disabled
* Read-only root filesystem
* All Linux capabilities removed
* RuntimeDefault Seccomp Profile

These controls significantly reduce the attack surface of the running container.

---

## 9. Resource Management

CPU and Memory requests/limits were configured to improve scheduling and prevent resource exhaustion.

| Resource | Request | Limit |
| -------- | ------- | ----- |
| CPU      | 100m    | 500m  |
| Memory   | 128Mi   | 512Mi |

---

## 10. Health Monitoring

Initially, the deployment used **/** as both the Readiness and Liveness probe endpoint.

The application responded with **HTTP 404**, causing Kubernetes to continuously restart the Pods.

After reviewing the application source code, the correct endpoint **/health** was identified and configured for both probes.

The application now responds successfully:

```json
{
  "status": "ok"
}
```

All Pods transitioned to a healthy **Running (1/1 Ready)** state after updating the probe configuration.

---

## 11. Kyverno Policies

Two admission policies were implemented.

### Disallow Latest Image Tag

Rejects any Deployment using the **latest** image tag.

Reason:

Using immutable image tags improves deployment consistency and traceability.

---

### Disallow Root Containers

Rejects Pods attempting to run as the root user.

Reason:

Running containers as non-root significantly reduces the impact of container compromise.

---

## 12. Neighbour Service

As required by the assignment, an additional workload named **reporting** was deployed.

The reporting service uses:

* Dedicated ServiceAccount
* Hardened SecurityContext
* Resource Requests & Limits

This workload will be used in later tasks involving networking and service mesh.

---

# Challenges Encountered

### ImagePullBackOff

Initially, the Deployment referenced a local image.

```
ledger-api:v1.0.0
```

Since the image was not available in Docker Hub, Kubernetes returned an ImagePullBackOff error.

The Deployment was updated to reference the published Docker Hub image, resolving the issue.

---

### Health Probe Failure

The initial Liveness and Readiness probes targeted:

```
/
```

The application returned HTTP 404.

After reviewing the Flask application routes, both probes were updated to:

```
/health
```

The application then passed all health checks successfully.

---

### Sealed Secrets

The cluster used for testing did not have the Bitnami Sealed Secrets controller installed.

For this reason, a Kubernetes Secret was temporarily used for runtime validation, while the Sealed Secret manifest remains part of the repository to demonstrate the intended production implementation.

---

# Validation

The deployment was successfully validated on an **AWS EC2-based Kubernetes cluster**.

The following verifications were completed:

* All application Pods reached the **Running** state.
* Readiness and Liveness probes returned successful responses.
* ClusterIP Service routed traffic correctly to application Pods.
* Kyverno admission policies were applied successfully.
* The reporting workload was deployed alongside the ledger-api application.

### Deployment Evidence

**Screenshot 1**

`/screenshots/pods-running.png`

Shows:

* Three healthy ledger-api Pods
* Reporting Pod
* All Pods in Running state

---

**Screenshot 2**

`/screenshots/health-check.png`

Shows successful response from the application health endpoint.

```json
{
    "status": "ok"
}
```

---

## Security Controls Implemented

* Dedicated Namespace
* ConfigMaps
* Secrets
* Sealed Secret Manifest
* Dedicated ServiceAccount
* RBAC
* Non-root Containers
* Read-only Root Filesystem
* Dropped Linux Capabilities
* RuntimeDefault Seccomp
* Resource Requests & Limits
* Readiness Probe
* Liveness Probe
* Kyverno Admission Policies
* Three Replica Deployment

---

# Conclusion

The ledger-api application was successfully deployed and hardened on an AWS-hosted Kubernetes cluster by applying production security best practices. The workload now follows the principle of least privilege, externalizes configuration and secrets, runs as a non-root user, enforces admission policies, and includes health monitoring for reliable operations. These controls provide a strong security baseline and prepare the application for the remaining tasks in the assessment.
