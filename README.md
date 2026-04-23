# Kubernetes Security Guide

> A practical guide to **Network Policies** and **Git-Safe Secret Management** in Kubernetes.

---

## Table of Contents

- [1. Network Policies](#1-network-policies)
- [16. The Real Problem — Why Not Store Secrets in Git?](#16-the-real-problem--why-not-store-secrets-in-git)
- [17. What is External Secrets Operator (ESO)?](#17-what-is-external-secrets-operator-eso)
- [18. High-Level Flow — How ESO Works](#18-high-level-flow--how-eso-works)

---

## 1. Network Policies

By default, every Pod in a Kubernetes cluster can communicate freely with every other Pod — regardless of namespace. This open-by-default model is a **serious security risk** in multi-tenant or microservice deployments.

### Example Scenario

| Source | Destination | Status |
|---|---|---|
| `frontend` (payments ns) | `backend` (payments ns) | ✅ Allowed |
| `backend` (payments ns) | `database` (db ns) | ✅ Allowed |
| `frontend` (payments ns) | `database` (db ns) | ❌ Must be blocked |

> **Network Policies** let you declare exactly which Pods may send or receive traffic, on which ports, and from which namespaces.

---

## Git-Safe Secret Management

> **External Secrets Operator (ESO) · HashiCorp Vault · GitOps-Friendly**

---

## 16. The Real Problem — Why Not Store Secrets in Git?

Kubernetes Secrets **look** secure but they are just **Base64 encoded — not encrypted**. If you push a Secret YAML to Git, anyone with repo access can instantly decode it with a single command.

| ❌ The Wrong Way | ✅ The Right Way (ESO) |
|---|---|
| Store Secret YAML in Git (anyone can decode base64) | Store only a **reference** in Git (actual secret lives in Vault) |
| Secret leaks if repo is compromised | Repo compromise doesn't expose secrets |
| Manual updates to secrets = manual redeploy | ESO auto-syncs updated secrets into K8s |

---

## 17. What is External Secrets Operator (ESO)?

ESO is a Kubernetes operator that acts as a **bridge** between Kubernetes and external secret stores like **HashiCorp Vault**, **AWS Secrets Manager**, or **Azure Key Vault**.

Instead of putting real secrets in your cluster YAML, you put only a **reference** — ESO fetches the actual value and injects it as a native Kubernetes Secret automatically.

| Component | What it does |
|---|---|
| `SecretStore` | Tells ESO **where** to find secrets (e.g. Vault address + credentials to authenticate) |
| `ExternalSecret` | Tells ESO **which** secret to fetch and what to name it in Kubernetes — this goes in Git |
| `Kubernetes Secret` | Auto-created by ESO from the fetched data — pods consume it normally |

---

## 18. High-Level Flow — How ESO Works

```
┌─────────────────────────────────────────────────────────────────┐
│                     ESO Sync Flow                               │
│                                                                 │
│  [Vault]  ──────►  [ESO Operator]  ──────►  [K8s Secret]       │
│  secret/          reads reference           auto-created        │
│  payments/db      from Git                  in namespace        │
│                        ▲                         │              │
│                        │                         ▼              │
│                      [Git]                    [Pod]             │
│                   ExternalSecret           secretKeyRef         │
│                   (path only)              (consumed normally)  │
└─────────────────────────────────────────────────────────────────┘
```

### Step-by-Step

| Step | Action |
|---|---|
| **1️⃣ Store secret in Vault** | e.g. `username` and `password` at path `secret/payments/db` |
| **2️⃣ Git contains only a reference** | An `ExternalSecret` YAML that says *"fetch from Vault at this path"* |
| **3️⃣ ESO syncs automatically** | Reads from Vault and creates a real Kubernetes Secret in your namespace |
| **4️⃣ Pod consumes it normally** | Uses `secretKeyRef` just like any other Kubernetes Secret |

### Key Takeaway

> 🔐 **Git stays clean. Secrets stay safe.**  
> If someone clones your repo they only see the **path reference** — never the actual value.

---

*Part of the Kubernetes Security series.*
