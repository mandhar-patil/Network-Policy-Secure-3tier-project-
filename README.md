# 🔐 Kubernetes Security — Network Policies & Secret Management

> A complete guide to securing Pod communication and managing secrets the **right way** using **ESO + HashiCorp Vault**

---

## 📑 Table of Contents

- [Network Policies](#-network-policies)
  - [What are Network Policies?](#what-are-network-policies)
  - [Example Scenario](#example-scenario)
  - [How Network Policies Work](#how-network-policies-work)
- [Git-Safe Secret Management](#-git-safe-secret-management)
  - [The Real Problem](#the-real-problem--why-not-store-secrets-in-git)
  - [What is ESO?](#what-is-external-secrets-operator-eso)
  - [How ESO Works](#high-level-flow--how-eso-works)

---

## 🌐 Network Policies

### What are Network Policies?

By default, **every Pod in a Kubernetes cluster can communicate freely with every other Pod** — regardless of namespace.

> ⚠️ This open-by-default model is a **serious security risk** in multi-tenant or microservice deployments.

Network Policies let you declare **exactly**:
- Which Pods may **send or receive** traffic
- On which **ports**
- From which **namespaces**

---

### Example Scenario

| Source | Destination | Status |
|--------|------------|--------|
| `frontend` (payments ns) | `backend` (payments ns) | ✅ Allowed |
| `backend` (payments ns) | `database` (db ns) | ✅ Allowed |
| `frontend` (payments ns) | `database` (db ns) | ❌ Must be blocked |

---

### How Network Policies Work

```
Internet
   │
   ▼
Frontend (port 80)
   │  ✅ allowed
   ▼
Backend (port 8080)
   │  ✅ allowed
   ▼
MongoDB (port 27017)

Frontend ──✖──▶ MongoDB   ← ❌ Blocked by Network Policy
Outside  ──✖──▶ Backend   ← ❌ Blocked by Network Policy
Outside  ──✖──▶ MongoDB   ← ❌ Blocked by Network Policy
```

#### Example — Allow only Backend to access MongoDB:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-mongo
  namespace: three-tier
spec:
  podSelector:           # ← Protect THIS pod
    matchLabels:
      app: mongodb
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:   # ← Only allow FROM backend
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 27017    # ← MongoDB port
```

> 💡 **Remember:** `ports` = the port the **protected/destination pod** is listening on — NOT the source pod's port.

---

## 🔑 Git-Safe Secret Management

### The Real Problem — Why Not Store Secrets in Git?

Kubernetes Secrets **look** secure but they are just **Base64 encoded — not encrypted**.

```bash
# Anyone can decode a K8s secret with one command
echo "c3VwZXJzZWNyZXQ=" | base64 --decode
# Output: supersecret
```

If you push a Secret YAML to Git → **anyone with repo access can instantly decode it.**

---

### ❌ Wrong Way vs ✅ Right Way

| ❌ Wrong Way | ✅ Right Way (ESO) |
|-------------|-------------------|
| Store Secret YAML in Git | Store only a **reference** in Git |
| Anyone can decode base64 | Actual secret lives in **Vault** |
| Repo compromise = secret leaked | Repo compromise = **nothing exposed** |
| Manual updates = manual redeploy | ESO **auto-syncs** updated secrets |

---

### What is External Secrets Operator (ESO)?

ESO is a Kubernetes operator that acts as a **bridge** between Kubernetes and external secret stores like:

- 🏦 HashiCorp Vault
- ☁️ AWS Secrets Manager
- 🔷 Azure Key Vault

Instead of putting real secrets in your cluster YAML, you put only a **reference** — ESO fetches the actual value and injects it as a native Kubernetes Secret automatically.

| Component | What it does |
|-----------|-------------|
| `SecretStore` | Tells ESO **where** to find secrets (Vault address + auth credentials) |
| `ExternalSecret` | Tells ESO **which** secret to fetch — this is what goes in Git |
| `Kubernetes Secret` | Auto-created by ESO from fetched data — pods consume it normally |

---

### High-Level Flow — How ESO Works

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│   1️⃣  Store secret in Vault                         │
│       secret/three-tier/mongo                       │
│       ├── MONGO_INITDB_ROOT_USERNAME = admin        │
│       ├── MONGO_INITDB_ROOT_PASSWORD = secret       │
│       └── MONGO_CONN_STR = mongodb://...            │
│                                                     │
│   2️⃣  Git contains only a reference                 │
│       ExternalSecret YAML → "fetch from Vault"      │
│                                                     │
│   3️⃣  ESO syncs automatically (every 1h)            │
│       Reads Vault → Creates K8s Secret              │
│                                                     │
│   4️⃣  Pod consumes it normally                      │
│       secretKeyRef → env var injected at runtime    │
│                                                     │
└─────────────────────────────────────────────────────┘
```

#### SecretStore — Connect ESO to Vault

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: three-tier
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-token
          key: token
```

#### ExternalSecret — Fetch Secrets from Vault *(this goes in Git ✅)*

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: mongo-secret
  namespace: three-tier
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: mongo-secret
    creationPolicy: Owner
  data:
    - secretKey: MONGO_INITDB_ROOT_USERNAME
      remoteRef:
        key: three-tier/mongo
        property: MONGO_INITDB_ROOT_USERNAME
    - secretKey: MONGO_INITDB_ROOT_PASSWORD
      remoteRef:
        key: three-tier/mongo
        property: MONGO_INITDB_ROOT_PASSWORD
    - secretKey: MONGO_CONN_STR
      remoteRef:
        key: three-tier/mongo
        property: MONGO_CONN_STR
```

#### Pod Consumes Secret Normally

```yaml
env:
  - name: MONGO_INITDB_ROOT_USERNAME
    valueFrom:
      secretKeyRef:
        name: mongo-secret      # ← K8s Secret auto-created by ESO
        key: MONGO_INITDB_ROOT_USERNAME
  - name: MONGO_INITDB_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongo-secret
        key: MONGO_INITDB_ROOT_PASSWORD
```

---

> ✅ **Git stays clean. Secrets stay safe.**
> If someone clones your repo they only see the **path reference** — never the actual value.

---

## 📌 Quick Reference

```bash
# Store secret in Vault
vault kv put secret/three-tier/mongo \
  MONGO_INITDB_ROOT_USERNAME="admin" \
  MONGO_INITDB_ROOT_PASSWORD="supersecretpassword" \
  MONGO_CONN_STR="mongodb://admin:supersecretpassword@mongodb-service:27017/taskdb?authSource=admin"

# Verify ESO synced the secret
kubectl get externalsecret -n three-tier
kubectl get secret mongo-secret -n three-tier

# Verify Network Policy
kubectl get networkpolicy -n three-tier

# Test unauthorized pod is BLOCKED
kubectl run hacker-pod --image=busybox --rm -it --restart=Never -n three-tier -- \
  nc -zv mongodb-service 27017
# Expected: Connection timed out ✅
```
