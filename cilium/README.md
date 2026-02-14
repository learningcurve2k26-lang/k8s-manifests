# Cilium CNI Deployment

This directory contains Kustomize manifests for deploying Cilium CNI using the official Helm chart.

## Important: Node Taint Handling

This configuration is designed for clusters where nodes are **NOT pre-tainted** with `node.cilium.io/agent-not-ready`.

When nodes don't have this taint pre-applied:
- Pods may be scheduled before Cilium is ready
- Those pods will fail networking until Cilium initializes
- This config enables `nodeInit` and proper tolerations to handle this scenario

### Recommended: Pre-taint Nodes

For production clusters, it's recommended to pre-taint nodes during cluster creation:

```yaml
# kubeadm example
nodeRegistration:
  taints:
    - key: node.cilium.io/agent-not-ready
      value: "true"
      effect: NoSchedule
```

## Structure

```
cilium/
├── kustomization.yaml  # Kustomize config with Helm chart reference
├── values.yaml         # Helm values for customization
└── README.md           # This file
```

## Prerequisites

- Kubernetes cluster (v1.26+)
- kubectl configured
- Kustomize v5.0+ (with Helm support)
- No existing CNI installed (or replace existing)

## Deploy

### Build and preview manifests


### Apply to cluster

```bash
kustomize build --enable-helm argocd/manifests/cilium | kubectl apply -f -
```

## Verify Installation

### Check Cilium status

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/part-of=cilium
```

### Using Cilium CLI

```bash
cilium status
cilium connectivity test
```

## Features Enabled

- **Hubble**: Network observability with UI and Relay
- **BPF Masquerading**: Efficient NAT using eBPF
- **Bandwidth Manager**: Network QoS
- **L7 Proxy**: Layer 7 policy enforcement
- **Kubernetes IPAM**: Pod IP management via Kubernetes

## Customization

Edit `values.yaml` for common changes:

| Setting | Description |
|---------|-------------|
| `hubble.enabled` | Enable/disable observability |
| `hubble.ui.enabled` | Enable Hubble UI |
| `ipam.mode` | IPAM mode (kubernetes, cluster-pool) |
| `bandwidthManager.enabled` | Network QoS |
| `prometheus.enabled` | Metrics export |

## Access Hubble UI

```bash
kubectl port-forward -n kube-system svc/hubble-ui 12000:80
```

Access at: http://localhost:12000

## Troubleshooting

### Pods stuck in ContainerCreating

If pods are stuck after Cilium deployment:

```bash
# Check Cilium agent logs
kubectl -n kube-system logs -l k8s-app=cilium

# Restart pods to pick up CNI
kubectl delete pods --all -n <affected-namespace>
```

### Node not ready

```bash
# Check if Cilium agent is running on the node
kubectl -n kube-system get pods -l k8s-app=cilium -o wide

# Check node taints
kubectl describe node <node-name> | grep Taints
```
