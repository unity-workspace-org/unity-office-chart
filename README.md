# Unity Office Helm Chart

Official production Helm chart for deploying the **Unity Office Document Suite** on Kubernetes, pre-configured with **UDrive (ownCloud Infinite Scale)** storage integration, **Collabora Online**, **WOPI Bridge**, and the **Next.js Web Portal**.

---

## 🏛️ Architecture

```text
                                [ Ingress Controller ]
                         (Traefik / NGINX / Cert-Manager)
                                  │             │
        https://office.domain.com │             │ https://collabora.domain.com
                                  ▼             ▼
                     ┌──────────────────┐   ┌───────────────────┐
                     │   uoffice-web    │   │      uoffice      │
                     │  (Next.js 16)    │   │ (Collabora Online)│
                     │   [Port 3000]    │   │    [Port 9980]    │
                     └────────┬─────────┘   └─────────┬─────────┘
                              │                       │
                       WebDAV │                       │ WOPI Protocol
                              ▼                       ▼
                     ┌──────────────────────────────────────────┐
                     │               uoffice-wopi               │
                     │           (WOPI Storage Bridge)          │
                     │                [Port 8055]               │
                     └────────────────────┬─────────────────────┘
                                          │
                                   WebDAV │ PROPFIND/PUT/GET
                                          ▼
                     ┌──────────────────────────────────────────┐
                     │          UDrive Storage Cluster          │
                     │        (ownCloud Infinite Scale)         │
                     │     https://drive.domain.com (:9200)     │
                     └──────────────────────────────────────────┘
```

---

## 🚀 Quickstart Installation

### 1. Add Repository or Clone

```bash
git clone https://github.com/unity-workspace-org/unity-office-chart.git
cd unity-office-chart
```

### 2. Install Chart with UDrive Connection

```bash
helm install unity-office . \
  --namespace unity-office \
  --create-namespace \
  --set udrive.url="http://udrive.udrive.svc.cluster.local:9200" \
  --set udrive.auth.username="admin" \
  --set udrive.auth.password="your-secure-password" \
  --set ingress.hosts.web.host="office.yourdomain.com" \
  --set ingress.hosts.collabora.host="collabora.yourdomain.com"
```

### 3. Production Deployment with `values-prod.yaml`

```bash
helm install unity-office . \
  --namespace unity-office \
  --create-namespace \
  -f values-prod.yaml \
  --set udrive.url="https://drive.yourdomain.com" \
  --set ingress.hosts.web.host="office.yourdomain.com" \
  --set ingress.hosts.collabora.host="collabora.yourdomain.com"
```

---

## ⚙️ Configuration Reference

### Global Settings

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `global.domain` | Base domain for the workspace | `workspace.com` |
| `global.imagePullSecrets` | Docker registry credentials | `[{name: ghcr-secret}]` |

### Web Portal (`web`)

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `web.enabled` | Enable Next.js portal | `true` |
| `web.replicaCount` | Number of replicas | `2` |
| `web.image.repository` | Docker image repository | `ghcr.io/unity-workspace-org/uoffice-web` |
| `web.image.tag` | Image tag | `v1.1.5` |
| `web.autoscaling.enabled`| Enable Horizontal Pod Autoscaler | `true` |
| `web.autoscaling.minReplicas` | Minimum pod count | `2` |
| `web.autoscaling.maxReplicas` | Maximum pod count | `10` |
| `web.podDisruptionBudget.enabled` | Enable PDB | `true` |
| `web.resources.limits` | CPU / Memory resource limits | `1000m / 1024Mi` |
| `web.resources.requests` | CPU / Memory resource requests | `100m / 256Mi` |

### Collabora Engine (`uoffice`)

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `uoffice.enabled` | Enable Collabora Online engine | `true` |
| `uoffice.replicaCount` | Replicas | `1` |
| `uoffice.image.repository` | Docker image repository | `ghcr.io/unity-workspace-org/uoffice` |
| `uoffice.image.tag` | Image tag | `latest` |
| `uoffice.securityContext` | Requires MKNOD / SYS_ADMIN caps | *(configured)* |
| `uoffice.resources.limits` | CPU / Memory limits | `2000m / 2048Mi` |

### WOPI Bridge (`wopi`)

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `wopi.enabled` | Enable WOPI Python bridge | `true` |
| `wopi.replicaCount` | Replicas | `1` |
| `wopi.image.repository` | Docker image repository | `ghcr.io/unity-workspace-org/uoffice-wopi` |
| `wopi.image.tag` | Image tag | `latest` |
| `wopi.persistence.enabled` | Persist drafts volume | `true` |
| `wopi.persistence.size` | PVC storage size | `10Gi` |

### UDrive Integration (`udrive`)

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `udrive.url` | UDrive / oCIS base URL | `https://drive.workspace.com` |
| `udrive.storageDir` | Storage folder in UDrive | `/` |
| `udrive.insecureSkipVerify` | Skip TLS verification | `false` |
| `udrive.webUrl` | App launcher 9-dot icon URL | `https://drive.workspace.com` |
| `udrive.auth.username` | Service account username | `admin` |
| `udrive.auth.password` | Service account password | `admin` |
| `udrive.auth.existingSecret` | Use existing Kubernetes Secret | `""` |

### Ingress & SSL

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `ingress.enabled` | Deploy Ingress resource | `true` |
| `ingress.className` | Ingress controller class | `traefik` |
| `ingress.hosts.web.host` | Hostname for Web Portal | `office.workspace.com` |
| `ingress.hosts.collabora.host`| Hostname for Collabora Online | `collabora.workspace.com` |
| `ingress.tls` | Cert-manager TLS secret list | *(configured)* |

### 🌐 Why are there 2 Ingress Hosts (`web.host` vs `collabora.host`)?

You will notice two separate hostnames in the ingress configuration:
- **`office.yourdomain.com`**: Routes to `uoffice-web` (Next.js Dashboard on port 3000)
- **`collabora.yourdomain.com`**: Routes to `uoffice` (Collabora Online engine on port 9980)

Here is why this dual-domain architecture is the industry standard (used by Nextcloud, ownCloud, and Collabora):

1. **Root Path Conflict Prevention**:
   Collabora Online listens on fixed root-level paths (`/browser/*` for static assets, `/cool/*` for WebSocket sockets, and `/hosting/discovery` for WOPI discovery). Placing Collabora on the same hostname as Next.js would cause path conflicts with Next.js internal routes and static bundles.
2. **Security & Iframe Sandboxing (Same-Origin Isolation)**:
   The Next.js portal loads the document editor inside a sandboxed `<iframe>`. Hosting Collabora on a distinct subdomain ensures cross-origin isolation: untrusted document macros or scripts running inside LibreOfficeKit can never access your Next.js session cookies, localStorage, or auth tokens.
3. **Specialized Ingress & WebSocket Profiles**:
   Collabora uses long-lived bidirectional WebSocket streams (`wss://collabora...`) that require extended proxy timeouts (`3600s`) and disabled response buffering. In contrast, `office.yourdomain.com` handles standard stateless HTTP/REST traffic. Having separate hosts allows Traefik/NGINX to apply different middleware cleanly.
4. **Shared Engine across Unity Workspace**:
   Having a dedicated `collabora.yourdomain.com` allows both **Unity Office** (`office.yourdomain.com`) AND **UDrive** (`drive.yourdomain.com`) to embed the exact same Collabora instance seamlessly.

---

## 🔒 Security Architecture & Hardening

This Helm chart implements enterprise defense-in-depth security standards:

### 1. Zero-Trust Network Isolation (NetworkPolicy)
- **Isolated Storage Bridge**: The `uoffice-wopi` pod is never exposed to the public Ingress. It strictly accepts connections originating only from `uoffice-web` and `uoffice` within the cluster.
- **Controlled Egress**: Pods can only communicate with required cluster services (DNS port 53, UDrive port 9200, and Collabora port 9980).

### 2. Pod Security Standards (PSS Restricted)
- **Non-Root Execution**: `uoffice-web` and `uoffice-wopi` run as unprivileged non-root users (`1001:1001` and `1000:1000`).
- **Privilege Escalation Blocked**: `allowPrivilegeEscalation: false`.
- **Linux Capabilities Dropped**: All capabilities dropped (`drop: ["ALL"]`) except strictly bounded capabilities required by sandboxes.
- **Seccomp Profile**: Enforces `RuntimeDefault` seccomp profiles across all pods.

### 3. Iframe & Clickjacking Defense (CORS / CSP)
- Collabora enforces strict origin validation using `aliasgroup1`. Only explicitly allowed domains (`office.domain.com` and `drive.domain.com`) can frame the document editor.
- Ingress applies HTTP security headers: `HSTS` (`max-age=31536000`), `X-Content-Type-Options: nosniff`, `X-XSS-Protection`, and `Referrer-Policy`.

### 4. Enterprise Secret Injection
- **External Secrets & HashiCorp Vault**: Store credentials outside Helm using `existingSecret` or inject dynamically via HashiCorp Vault Agent sidecar integration.


## 🔗 Co-existence with UDrive Helm Chart

When deploying in the same Kubernetes cluster as `udrive`:

```yaml
# In your custom my-values.yaml
udrive:
  url: "http://udrive.udrive.svc.cluster.local:9200"
  insecureSkipVerify: true
  auth:
    existingSecret: "udrive-credentials"
    usernameKey: "username"
    passwordKey: "password"
```

---

## 🔄 Upgrades & Maintenance

```bash
# Upgrade release
helm upgrade unity-office . -n unity-office -f my-values.yaml

# Rollback release
helm rollback unity-office 1 -n unity-office

# Uninstall
helm uninstall unity-office -n unity-office
```

---

## 📄 License

Apache 2.0 / MPL 2.0. Maintained by [Unity Workspace](https://github.com/unity-workspace-org).
