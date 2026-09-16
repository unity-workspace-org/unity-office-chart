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

---

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
