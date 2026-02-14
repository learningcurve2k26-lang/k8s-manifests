# HashiCorp Vault Deployment for ArgoCD Vault Plugin (AVP)

This directory contains Kubernetes manifests for deploying HashiCorp Vault using Kustomize with Helm, configured for ArgoCD Vault Plugin integration.

## Directory Structure

```
vault/
├── base/
│   ├── kustomization.yaml           # Base kustomization with Helm chart
│   ├── namespace.yaml                # Vault namespace
│   ├── values.yaml                   # Base Helm values
│   ├── vault-auth-serviceaccount.yaml # SA for AVP authentication
│   └── vault-policy-configmap.yaml   # Vault policies for AVP
└── overlays/
    └── staging/
        ├── kustomization.yaml        # Staging overlay
        ├── values-staging.yaml       # Staging-specific values
        └── ingressroute.yaml         # Traefik ingress for UI
```

## Key Parameters

### Storage Options
| Parameter | Description | Staging | Production |
|-----------|-------------|---------|------------|
| `server.standalone.enabled` | Single node mode | `true` | `false` |
| `server.ha.enabled` | High availability mode | `false` | `true` |
| `server.ha.replicas` | Number of HA replicas | - | `3` or `5` |

### Security Parameters
| Parameter | Description | Staging | Production |
|-----------|-------------|---------|------------|
| `global.tlsDisable` | Disable TLS | `true` | `false` |
| `server.dev.enabled` | Dev mode (auto-unseal) | `false` | `false` |

### Resources
| Component | Staging | Production |
|-----------|---------|------------|
| Server Memory | 128Mi-256Mi | 512Mi-1Gi |
| Server CPU | 100m-250m | 500m-1000m |
| Injector Memory | 64Mi-128Mi | 128Mi-256Mi |

## Post-Deployment Configuration

After Vault is deployed, you need to initialize and configure it:

### 1. Initialize Vault

```bash
# Port forward to Vault
kubectl port-forward svc/vault -n vault 8200:8200

# Initialize Vault (save the unseal keys and root token!)
vault operator init

# Unseal Vault (repeat 3 times with different keys)
vault operator unseal <unseal-key>
```

### 2. Enable Kubernetes Auth Method

```bash
# Login with root token
vault login <root-token>

# Enable Kubernetes auth
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc:443" \
    token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### 3. Create AVP Policy and Role

```bash
# Create the argocd policy
vault policy write argocd - <<EOF
path "secret/data/*" {
  capabilities = ["read", "list"]
}
EOF

# Create a role for ArgoCD
vault write auth/kubernetes/role/argocd \
    bound_service_account_names=argocd-vault-plugin \
    bound_service_account_namespaces=argocd \
    policies=argocd \
    ttl=1h
```

### 4. Enable KV Secrets Engine

```bash
# Enable KV v2 secrets engine
vault secrets enable -path=secret kv-v2

# Add a test secret
vault kv put secret/myapp/config \
    username="admin" \
    password="supersecret"
```

## ArgoCD Vault Plugin Configuration

### AVP Environment Variables

Add these to your ArgoCD repo-server deployment:

```yaml
env:
  - name: AVP_TYPE
    value: vault
  - name: AVP_AUTH_TYPE
    value: k8s
  - name: AVP_K8S_ROLE
    value: argocd
  - name: VAULT_ADDR
    value: http://vault.vault.svc.cluster.local:8200
```

### Secret Annotation in Manifests

Use AVP placeholders in your manifests:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  annotations:
    avp.kubernetes.io/path: "secret/data/myapp/config"
stringData:
  username: <username>
  password: <password>
```

## Accessing Vault UI

- **URL**: https://vault.kitetsu.xyz (via Traefik)
- **NodePort**: https://<node-ip>:30821

## Troubleshooting

### Check Vault Status
```bash
kubectl exec -it vault-0 -n vault -- vault status
```

### View Vault Logs
```bash
kubectl logs -f vault-0 -n vault
```

### Check Injector Logs
```bash
kubectl logs -f -l app.kubernetes.io/name=vault-agent-injector -n vault
```
