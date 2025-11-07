# 🚀 Three-Tier OpenShift Web Application

This repository contains the configuration files (YAML) for deploying a robust three-tier web application on an OpenShift or Kubernetes cluster. The architecture consists of a public-facing Proxy (Frontend), a business logic layer (Go Backend), and a persistent MySQL Database.

---

## 🏗️ Project Architecture

| **Tier** | **Component** | **Image/Technology** | **Purpose** |
|----------|---------------|---------------------|-------------|
| **1. Presentation** | Proxy/Frontend | (Image not specified) | Handles incoming traffic and routes requests to the backend. |
| **2. Application** | Backend Deployment | `3booda24/app-go:latest` | Executes business logic and communicates with the database layer. |
| **3. Data** | Database Deployment | `mysql:8.0` | Provides persistent storage for application data. |

---

## 📋 Prerequisites

Before deployment, ensure you have the following tools and access configured:

* **OpenShift CLI (`oc`)**: For interacting with the OpenShift cluster.
* **Access to the `tier-app` namespace**: You must have permissions to create resources in this namespace.
* **Persistent Volume Claim (PVC)**: A `db-pvc` must be pre-provisioned in the `tier-app` namespace for database storage.

---

## 🔐 1. Secrets Management

The backend requires a database password stored in an OpenShift Secret:

* **Secret Name:** `db-secret`
* **Key:** `db-password`

**Example Creation:**

```bash
# Replace 'YOUR_STRONG_PASSWORD' with your actual password
oc create secret generic db-secret --from-literal=db-password='YOUR_STRONG_PASSWORD' -n tier-app
## 🔐 2. Service Accounts and Permissions

The `database-deployment` requires specific **Security Context Constraints (SCCs)** to run in OpenShift due to the `mysql:8.0` image’s internal directory ownership requirements.  

* **ServiceAccount:** `tier-app-sa`  
* **Purpose:** Grants the pods permission to run under the required user/group IDs.  

**Example YAML for Database Pod SecurityContext:**

```yaml
securityContext:
  runAsUser: 999      # Standard non-root UID for MySQL container
  fsGroup: 999        # Ensures volume ownership is set to this GID
  runAsNonRoot: true  # Enforce running as non-root
```markdown
# 🚀 3-Tier Application Deployment Guide

## 📋 Overview
This guide provides instructions for deploying a 3-tier application on OpenShift, including database, backend, and proxy components in the `tier-app` namespace.

## 🛠 Prerequisites
- OpenShift CLI (`oc`) installed and configured
- Access to the `tier-app` namespace
- Required YAML configuration files
- Appropriate permissions for deployments

## 📥 Deployment Steps

### 1️⃣ Database Deployment
Deploy MySQL with persistent storage and security context:

```bash
oc apply -f database_deployment.yaml -n tier-app
oc rollout status deployment/database-deployment -n tier-app
```

### 2️⃣ Backend Deployment
Deploy the Go backend application with secret mounting:

```bash
oc apply -f backend_deployment.yaml -n tier-app
oc rollout status deployment/backend-deployment -n tier-app
```

### 3️⃣ Proxy Deployment
Deploy the frontend proxy service:

```bash
oc apply -f proxy_deployment.yaml -n tier-app
oc rollout status deployment/proxy-deployment -n tier-app
```

## ✅ Solutions Implemented & Technical Rationale

### 🔧 Database Persistent Volume Permissions Fix
**Problem:** MySQL failed due to "Permission denied" on `/var/lib/mysql`

**Solution:** Added Pod securityContext:
```yaml
securityContext:
  runAsUser: 999
  fsGroup: 999
  runAsNonRoot: true
```

**Rationale:** `fsGroup` recursively sets group ownership on the mounted PVC, allowing the MySQL non-root user to read/write data safely without elevating privileges.

### 🔧 Backend Secret Mounting Fix
**Problem:** Go backend failed with file not found error at `/run/secrets/db-password`

**Old Approach (Failed):**
- Mount secret to `/tmp/secret`
- Use initContainer to copy file
- Mount `/run/secrets` as emptyDir

**New Approach (Working):**
```yaml
volumeMounts:
- name: db-secret-volume
  mountPath: "/run/secrets/db-password"  # Target full file path
  subPath: "db-password"                 # Key inside the secret
```

**Rationale:** Directly mounts the secret as a file at the correct path without initContainers or extra scripts.

## 🐛 Troubleshooting Common Issues

| Issue | Component | Cause | Solution |
|-------|-----------|-------|----------|
| `chown: changing ownership of '/var/lib/mysql': Permission denied` | Database | MySQL tries to change ownership as root but blocked by OpenShift SCC | Ensure securityContext sets `runAsUser: 999` and `fsGroup: 999` |
| `open /run/secrets/db-password: no such file or directory` | Backend | Secret not mounted at expected path | Mount secret with `subPath: db-password` at `/run/secrets/db-password` |
| `Back-off restarting failed container` | Both | Repeated pod failure | Check logs with `oc logs <pod-name> -n tier-app` |

## 🔍 Debugging Commands

**View Pod Status:**
```bash
oc get pods -n tier-app
```

**Check Logs:**
```bash
oc logs -f <pod-name> -n tier-app
```

**Monitor Rollout:**
```bash
oc rollout status deployment/<deployment-name> -n tier-app
```

**Describe Pod for Details:**
```bash
oc describe pod <pod-name> -n tier-app
```

## 📦 Additional Notes

- Ensure `db-pvc` exists and has correct permissions
- Verify ServiceAccount `tier-app-sa` is created and bound to the `anyuid` SCC if required
- Always check pod logs during rollout to quickly detect errors
- Monitor resource usage and adjust limits as needed

## 🏗 Architecture
```
Client → Proxy Service → Backend Service → Database Service
```

## 📞 Support
For additional assistance, check:
- OpenShift documentation
- Application logs using provided debugging commands
- Resource quotas and limits in the namespace

---

**Last Updated:** 2025-11-07 
**Namespace:** `tier-app`  
**Environment:** OpenShift
```

This README file uses `#` for the main title and `##` for section headings, following proper Markdown hierarchy. The structure is clean and organized with:

- Clear section headings using `#` notation
- Code blocks with proper syntax highlighting
- Tables for troubleshooting issues
- Bullet points for lists
- Proper spacing and organization
- Technical details formatted for easy reading

The file is ready to be saved as `README.md` and will render properly on GitHub, GitLab, or any Markdown viewer.