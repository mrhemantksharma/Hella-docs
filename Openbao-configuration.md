## 🔹 High-Level Design

### Add Cluster with OpenBao
- **Kubernetes Cluster Preparation**
  - Create ServiceAccount for OpenBao
  - Generate Reviewer JWT
  - Create ClusterRole and ClusterRoleBinding
  - Collect Cluster Details
    - Kubernetes API server
    - Kubernetes CA certificate

- **OpenBao Configuration (Add Cluster)**
  - Verify OpenBao Access
  - Enable Kubernetes Auth (Once)
  - Configure Cluster Trust

---

### OpenBao Access Control
- Create Policy (**WHAT** can be accessed)
- Create Role (**WHO** can authenticate)

---

### Enable OIDC Connection Setup for Users
- Login in KeyCloak
- Create group and add users in that

---

## 🔹 Secure Secret Management & Application Deployment (End-to-End Guide)

### 1. Purpose
This document explains:
- How a Kubernetes cluster is added (trusted) to OpenBao
- How users store secrets using OpenBao UI (SAML login)
- How applications fetch secrets securely during deployment via Argo CD
- How no user gets Kubernetes or OpenBao CLI access
- How to validate the setup using a test application

This is a **production-grade, zero-trust design**.

---

### 2. Target Architecture

```
User
 ├── OpenBao UI (SAML) → Store secrets
 └── Argo CD UI       → Deploy applications

Argo CD (ServiceAccount Identity)
        │
        ▼
OpenBao Cluster
  ├─ Kubernetes Auth (Trusts cluster)
  ├─ Roles (Who can authenticate)
  └─ Policies (Which secret paths allowed)
        │
        ▼
Kubernetes Cluster
```

**Key Principles**
- Users never access Kubernetes
- Users never see secrets in Argo CD
- Secrets are never stored in Git
- OpenBao is the single source of truth

---

### 3. Identity Model

| Purpose                        | ServiceAccount       | Namespace     | Used By      |
| ------------------------------ | -------------------- | ------------- | ------------ |
| Cluster trust                  | `openbao-auth`       | `kube-system` | OpenBao only |
| CD secret fetch                | `argocd-repo-server` | `argocd`      | Argo CD      |
| Application runtime (optional) | `app-sa`             | App namespace | App pods     |

⚠️ **These ServiceAccounts must NEVER be reused**

---

### 4. Phase 1 – Add Kubernetes Cluster to OpenBao

👤 **Owner:** DevOps / Platform Admin  
📍 **Where:** `kubectl` (Kubernetes cluster), `bao` (OpenBao admin machine)

#### 4.1 Kubernetes Cluster Preparation

##### Create ServiceAccount
```bash
kubectl create serviceaccount openbao-auth -n <namespace>
```

##### Generate Reviewer JWT (K8s ≥ 1.24)
```bash
kubectl -n <namespace> create token openbao-auth > hs-reviewer.jwt
```

✔ JWT is **used only by OpenBao**, never by apps.

##### Allow TokenReview API Access

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: openbao-auth-role
rules:
- apiGroups: [""]
  resources: ["serviceaccounts", "secrets", "pods"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: openbao-auth-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: openbao-auth-role
subjects:
- kind: ServiceAccount
  name: openbao-auth
  namespace: <your-app-namespace>  #CHANGE THIS ACCORDING TO THE NAMESPACE YOU ARE GOING TO USE
```


##### Collect Cluster Details
```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```

```bash
kubectl config view --raw -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 --decode > k8s-ca.crt
```

---

### 4.2 OpenBao Configuration (Add Cluster)

#### 4.2.1 Verify OpenBao Access

```bash
bao status
```

#### 4.2.2 Enable Kubernetes Auth (Once)

```bash
bao auth enable kubernetes
```

#### 4.2.3 Configure Cluster Trust

```bash
bao write auth/kubernetes/config \
  token_reviewer_jwt=@/path/to/hs-reviewer.jwt \
  kubernetes_host="https://<API_SERVER>:6443" \
  kubernetes_ca_cert=@/path/to/k8s-ca.crt
```

✅ At this step, the **cluster is officially added to OpenBao**.

---

### 5. Phase 2 – Secret Management (User Flow)

👤 **Owner:** Application Developer  
🔑 **Access:** OpenBao UI via SAML

Example secret path:
```
secret/data/team-a/db
```

Example values:
```
username = appuser
password = securepass
```

✔ Users can only write to their allowed paths  
✔ All actions are audited  

---

### 6. Phase 3 – OpenBao Access Control

#### Create Policy
```hcl
# argocd-team-a.hcl
path "secret/data/team-a/*" {
  capabilities = ["read"]
}
```

```bash
bao policy write argocd-team-a argocd-team-a.hcl
```

#### Create Role
```bash
bao write auth/kubernetes/role/argocd-team-a \
  bound_service_account_names=argocd-repo-server \
  bound_service_account_namespaces=argocd \
  policies=argocd-team-a \
  ttl=30m
```

---

### 7. Phase 4 – Argo CD Integration

Configure `argocd-repo-server` with Vault Plugin:
```yaml
env:
- name: VAULT_ADDR
  value: https://openbao.company.com
- name: AVP_AUTH_TYPE
  value: kubernetes
- name: AVP_K8S_ROLE
  value: argocd-team-a
```

---

### 8. Phase 5 – Test Application Deployment

#### Example Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: team-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: demo
        image: nginx
        env:
        - name: DB_USER
          value: "<vault:secret/data/team-a/db#username>"
        - name: DB_PASS
          value: "<vault:secret/data/team-a/db#password>"
```

---

### 9. Validation Checklist
- [ ] Cluster trust configured in OpenBao  
- [ ] User can store secrets via OpenBao UI  
- [ ] Argo CD sync succeeds  
- [ ] Secrets not visible in Git or Argo CD UI  
- [ ] Application pod starts successfully  

---

### 10. Security Guarantees

| Area                | Protection   |
| ------------------- | ------------ |
| Secrets in Git      | ❌ Never      |
| User cluster access | ❌ None       |
| Secret access       | Policy-based |
| Token lifetime      | Short-lived  |
| Audit logs          | Enabled      |

---

## 11. Final Takeaway

> **Clusters are trusted once**
