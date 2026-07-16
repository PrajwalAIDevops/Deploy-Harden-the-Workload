# Task 1 – Deploy & Harden the Workload

## Overview

This task focuses on deploying the **ledger-api** application into Kubernetes and hardening it using production security best practices. The objective was not only to make the application run successfully, but also to reduce its attack surface and follow the principle of least privilege, similar to how workloads are deployed in PCI DSS or other regulated environments.

The application was deployed into a dedicated namespace with its own ServiceAccount, RBAC permissions, security contexts, resource limits, health probes, ConfigMaps, Secrets, and Kyverno admission policies.

---

# Architecture

```
                      +----------------------+
                      |      Ingress         |
                      +----------+-----------+
                                 |
                                 |
                     +-----------v------------+
                     |   Service (ClusterIP)  |
                     +-----------+------------+
                                 |
                     +-----------v------------+
                     |      Deployment        |
                     |    ledger-api (3 Pods) |
                     +-----------+------------+
                                 |
          +----------------------+----------------------+
          |                                             |
          |                                             |
+---------v----------+                     +-------------v-----------+
|     ConfigMap      |                     |        Secret           |
| Application Config |                     | Sensitive Credentials   |
+--------------------+                     +-------------------------+

                     +-----------------------------+
                     |      ServiceAccount         |
                     +-------------+---------------+
                                   |
                          +--------v--------+
                          |      RBAC       |
                          | Least Privilege |
                          +-----------------+

                     +-----------------------------+
                     |      Kyverno Policies       |
                     | No Root / No Latest Image   |
                     +-----------------------------+
```

---

# Project Structure

```
Deploy-Harden-the-Workload/

├── deploy/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── serviceaccount.yaml
│   ├── role.yaml
│   ├── rolebinding.yaml
│   ├── reporting.yaml
│   ├── kyverno-disallow-root.yaml
│   ├── kyverno-disallow-latest.yaml
│   └── ingress.yaml
│
├── screenshots/
│
└── README.md
```

---

# Environment

| Component         | Version         |
| ----------------- | --------------- |
| Ubuntu            | 24.04           |
| Kubernetes        | kubeadm Cluster |
| Container Runtime | containerd      |
| kubectl           | Latest Stable   |
| Kyverno           | Latest          |
| Docker            | Latest          |

---

# Step 1 - Namespace

A dedicated namespace was created to isolate the application from other workloads running inside the cluster.

```yaml
Namespace: payments
```

This provides better security boundaries and simplifies RBAC management.

---

# Step 2 - Deploy the Application

The application was deployed as a Kubernetes Deployment with **3 replicas**.

Features included:

* Rolling Updates
* Replica Management
* Self Healing
* Pod Restart
* High Availability

The Deployment exposes port **8080**.

---

# Step 3 - Service

A ClusterIP Service was created.

```
Client
   |
ClusterIP Service
   |
Deployment
   |
Pods
```

The service forwards traffic to the application running on port **8080**.

---

# Step 4 - Configuration Management

Application configuration was separated from the container image.

Configuration values such as

* APP_ENV
* APP_PORT

were stored inside a ConfigMap.

This allows changing runtime configuration without rebuilding the Docker image.

---

# Step 5 - Secret Management

Sensitive values were removed from application code.

Secrets such as

* STRIPE_API_KEY
* DB_PASSWORD

were stored as Kubernetes Secrets.

For production, Sealed Secrets was chosen because encrypted secrets can safely remain inside Git.

During local testing, a normal Kubernetes Secret was used because the Sealed Secrets controller was not installed in the cluster.

---

# Step 6 - Service Account

Instead of using the default Kubernetes ServiceAccount, a dedicated ServiceAccount named

```
ledger-api-sa
```

was created.

This follows the Principle of Least Privilege.

---

# Step 7 - RBAC

The application only requires read access to specific Kubernetes resources.

Role Permissions

```
ConfigMaps
Secrets
```

Allowed Verbs

```
get
list
```

No write permissions were granted.

This significantly reduces the impact if the application is compromised.

---

# Step 8 - Pod Security Hardening

Every container was hardened using Kubernetes SecurityContext.

Implemented controls:

* runAsNonRoot
* runAsUser:1000
* allowPrivilegeEscalation:false
* readOnlyRootFilesystem:true
* Drop ALL Linux Capabilities
* RuntimeDefault Seccomp Profile

These controls prevent privilege escalation and reduce container escape risks.

---

# Step 9 - Resource Limits

CPU and Memory requests/limits were configured.

```
Requests

CPU    : 100m
Memory : 128Mi

Limits

CPU    : 500m
Memory : 512Mi
```

Benefits

* Prevent noisy neighbour issues
* Better scheduling
* Predictable resource usage
* Prevent memory exhaustion

---

# Step 10 - Health Checks

Initially the deployment used

```
/
```

for both Liveness and Readiness probes.

The application returned

```
404 Not Found
```

which caused Kubernetes to continuously mark the Pods as unhealthy.

After reviewing the application source code, it was found that the application exposes a dedicated health endpoint.

```
/health
```

The probes were updated accordingly.

```
Readiness Probe

GET /health

Liveness Probe

GET /health
```

After updating the probes, all Pods reached the **Running (1/1 Ready)** state successfully.

This issue was identified by reviewing Pod events, checking container logs, and inspecting the Flask application routes.

---

# Step 11 - Kyverno Admission Policies

Two admission policies were implemented.

## Policy 1

Reject containers using

```
:latest
```

image tags.

Reason

Image tags should always be immutable to ensure reproducible deployments.

---

## Policy 2

Reject containers running as

```
root
```

Reason

Running containers as root increases the risk of privilege escalation and container breakout.

---

# Additional Neighbour Service

As required by the assignment, a second workload named

```
reporting
```

was deployed.

This workload runs with its own ServiceAccount and hardened SecurityContext.

It serves as a neighbouring application for future networking and service mesh tasks.

---

# Validation

The deployment was validated using the following checks.

Pods

```
kubectl get pods -n payments
```

Services

```
kubectl get svc -n payments
```

Health Endpoint

```
curl http://localhost:8080/health

{
  "status":"ok"
}
```



Kyverno

Attempting to deploy a container using

```
nginx:latest
```

is rejected by the admission controller.

---

# Security Improvements Achieved

* Dedicated Namespace
* Least Privilege ServiceAccount
* RBAC
* ConfigMaps for configuration
* Secrets separated from application code
* Hardened SecurityContext
* Non-root container
* Read-only root filesystem
* Dropped Linux capabilities
* RuntimeDefault Seccomp
* CPU & Memory resource limits
* Liveness & Readiness probes
* Kyverno admission policies
* Three replica deployment for high availability

---

# Challenges Faced

### 1. ImagePullBackOff

Initially the Deployment referenced a local Docker image.

```
ledger-api:v1.0.0
```

Since the image was unavailable in Docker Hub, Kubernetes returned an ImagePullBackOff error.

The Deployment was updated to reference the correct published image hosted in Docker Hub.

---

### 2. Health Probe Failures

The initial health probes targeted

```
/
```

which returned HTTP 404.

After reviewing the Flask application, the probes were changed to

```
/health
```

allowing Kubernetes to correctly determine application health.

---

### 3. Sealed Secrets

The cluster did not have the Bitnami Sealed Secrets controller installed.

As a result, SealedSecret resources could not be created during local testing.

For demonstration purposes, a standard Kubernetes Secret was used while retaining the SealedSecret manifest in the repository to illustrate the intended production approach.

---

# Conclusion

This task demonstrates how an insecure Kubernetes workload can be transformed into a production-ready deployment using security best practices. The application is isolated within its own namespace, follows least privilege principles, avoids running as root, uses externalized configuration and secrets, enforces admission policies, and includes health monitoring for reliable operations. These controls align with common Kubernetes security recommendations and provide a solid foundation for the remaining tasks in the assessment.
