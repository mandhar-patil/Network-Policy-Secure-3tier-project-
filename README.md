# Network-Policy-Secure-3tier-project-



🔐 Kubernetes Security — Network Policies & Secret Management

📡 1. Network Policies

By default, every Pod in a Kubernetes cluster can communicate freely with every other Pod — regardless of namespace. This open-by-default model is a serious security risk in multi-tenant or microservice deployments.

🔥 The Problem — Open by Default
SourceDestinationStatusfrontend (payments ns)backend (payments ns)✅ Allowedbackend (payments ns)database (db ns)✅ Allowedfrontend (payments ns)database (db ns)❌ Must be blocked
Network Policies let you declare exactly:

Which Pods may send or receive traffic
On which ports
From which namespaces

✅ Example — Allow Only Backend to Access MongoDB
yamlapiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-mongo
  namespace: three-tier
spec:
  podSelector:          # Protect THIS pod
    matchLabels:
      app: mongodb
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:  # Only allow FROM backend
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 27017   # MongoDB port
🔒 Traffic Flow
Internet
   ↓
Frontend  (port 80)
   ↓  ✅ allowed
Backend   (port 8080)
   ↓  ✅ allowed
MongoDB   (port 27017)

Frontend → MongoDB   ❌ blocked
Outside  → Backend   ❌ blocked
Outside  → MongoDB   ❌ blocked

🔑 2. Git-Safe Secret Management

ESO (External Secrets Operator) · HashiCorp Vault · GitOps-Friendly

❌ The Real Problem — Why Not Store Secrets in Git?
Kubernetes Secrets look secure but they are just Base64 encoded — not encrypted.
If you push a Secret YAML to Git, anyone with repo access can instantly decode it:
bashecho "c3VwZXJzZWNyZXQ=" | base64 --decode
# Output: supersecret
❌ Wrong Way✅ Right Way (ESO)Store Secret YAML in Git (anyone can decode base64)Store only a reference in Git (actual secret lives in Vault)Secret leaks if repo is compromisedRepo compromise doesn't expose secretsManual updates = manual redeployESO auto-syncs updated secrets into K8s

🌉 3. What is External Secrets Operator (ESO)?
ESO is a Kubernetes operator that acts as a bridge between Kubernetes and external secret stores like:

🏦 HashiCorp Vault
☁️ AWS Secrets Manager
🔵 Azure Key Vault

Instead of putting real secrets in your cluster YAML, you put only a reference — ESO fetches the actual value and injects it as a native Kubernetes Secret automatically.
ComponentWhat it doesSecretStoreTells ESO where to find secrets (e.g. Vault address + auth credentials)ExternalSecretTells ESO which secret to fetch and what to name it in Kubernetes (this goes in Git)Kubernetes SecretAuto-created by ESO from fetched data — pods consume it normally

🔄 4. High-Level Flow — How ESO Works
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   1. Store secret in Vault                              │
│      e.g. username & password at secret/three-tier/mongo│
│                                                         │
│   2. Git contains only a reference                      │
│      ExternalSecret YAML → "fetch from Vault at path"   │
│                                                         │
│   3. ESO syncs automatically                            │
│      Reads Vault → Creates K8s Secret in namespace      │
│                                                         │
│   4. Pod consumes it normally                           │
│      Uses secretKeyRef just like any K8s Secret         │
│                                                         │
└─────────────────────────────────────────────────────────┘
Vault (secret/three-tier/mongo)
          │
          │  ESO syncs every 1h
          ▼
  K8s Secret (mongo-secret)
    MONGO_INITDB_ROOT_USERNAME = admin
    MONGO_INITDB_ROOT_PASSWORD = supersecretpassword
    MONGO_CONN_STR             = mongodb://...
          │
          │  secretKeyRef
          ▼
  MongoDB Pod (env vars injected at runtime)

✅ Git stays clean. Secrets stay safe.
If someone clones your repo they only see the path reference — never the actual value.


🛠️ 5. Full Implementation
Step 1 — Store Secret in Vault
bashkubectl exec -it -n vault deploy/vault -- sh -c '
  export VAULT_ADDR=http://127.0.0.1:8200
  export VAULT_TOKEN=root
  vault kv put secret/three-tier/mongo \
    MONGO_INITDB_ROOT_USERNAME="admin" \
    MONGO_INITDB_ROOT_PASSWORD="supersecretpassword" \
    MONGO_CONN_STR="mongodb://admin:supersecretpassword@mongodb-service:27017/taskdb?authSource=admin"
  vault kv get secret/three-tier/mongo
'
Step 2 — Create SecretStore
yamlapiVersion: external-secrets.io/v1beta1
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
Step 3 — Create ExternalSecret (this goes in Git)
yamlapiVersion: external-secrets.io/v1beta1
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
Step 4 — Consume Secret in MongoDB StatefulSet
yamlenv:
  - name: MONGO_INITDB_ROOT_USERNAME
    valueFrom:
      secretKeyRef:
        name: mongo-secret
        key: MONGO_INITDB_ROOT_USERNAME
  - name: MONGO_INITDB_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mongo-secret
        key: MONGO_INITDB_ROOT_PASSWORD

✅ Verify Everything is Working
bash# 1. Check ESO synced the secret
kubectl get externalsecret -n three-tier
kubectl get secret mongo-secret -n three-tier

# 2. Check backend connected to DB
kubectl logs -n three-tier deploy/backend | grep -i connected

# 3. Prove network policy is BLOCKING others
kubectl run hacker-pod --image=busybox --rm -it --restart=Never -n three-tier -- \
  nc -zv mongodb-service 27017
# Expected: Connection timed out ✅

# 4. Prove allowed pod CAN connect
kubectl run allowed-pod --image=busybox --rm -it --restart=Never \
  --labels="app=backend" -n three-tier -- \
  nc -zv mongodb-service 27017
# Expected: mongodb-service:27017 open ✅
