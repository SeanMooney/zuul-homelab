# Zuul Homelab

A GitOps-managed Kubernetes deployment of [Zuul CI](https://zuul-ci.org) using [FluxCD](https://fluxcd.io) for automated continuous deployment.

## Overview

This repository contains a complete Zuul CI/CD system deployment for a homelab Kubernetes cluster, managed declaratively through FluxCD. All infrastructure and applications are defined as code and automatically synchronized to the cluster.

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         FluxCD                              │
│  Watches Git Repository → Applies Changes to Cluster        │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
    ┌───▼────┐      ┌──────▼──────┐      ┌─────▼─────┐
    │ Infra  │      │ Applications │      │  Cluster  │
    │ Layer  │      │    Layer     │      │  Config   │
    └────────┘      └──────────────┘      └───────────┘
```

**Key Components:**
- **Zuul CI**: Distributed CI/CD system with GitHub integration
- **Nodepool**: Dynamic node provisioning for CI jobs using Kubernetes pods
- **Zookeeper**: Coordination service for Zuul and Nodepool
- **PostgreSQL**: Database backend for Zuul
- **MinIO**: S3-compatible object storage for build logs
- **Ingress-NGINX**: HTTP/HTTPS ingress controller
- **Cert-Manager**: Automated TLS certificate management
- **SOPS**: Secret encryption with Age

## Repository Structure

```
zuul-homelab/
├── clusters/homelab/           # Cluster-specific FluxCD configurations
│   ├── flux-system/            # FluxCD bootstrap configuration
│   ├── infrastructure.yaml     # Infrastructure components (layered)
│   ├── zuul.yaml              # Zuul application
│   └── external-services.yaml # External service definitions
│
├── infrastructure/             # Infrastructure components
│   ├── namespaces/            # Kubernetes namespaces
│   ├── ingress-nginx/         # Ingress controller (Helm)
│   ├── cert-manager/          # Certificate manager (Helm)
│   ├── cert-manager-config/   # ClusterIssuers, certificates
│   ├── certificates/          # TLS certificates (Zookeeper, wildcard)
│   ├── zookeeper/             # Zookeeper StatefulSet
│   ├── database/              # PostgreSQL StatefulSet
│   ├── minio/                 # MinIO object storage
│   ├── nodepool-rbac/         # RBAC for nodepool pod creation
│   ├── reflector/             # Secret/ConfigMap replication (Helm)
│   ├── external-secrets/      # External Secrets Operator (Helm)
│   └── storage/               # StorageClass definitions
│
├── apps/                      # Application deployments
│   ├── zuul/                  # Zuul CI application
│   │   ├── operator/          # Zuul operator deployment
│   │   ├── configs/           # Zuul and Nodepool configurations
│   │   ├── secrets/           # SOPS-encrypted secrets
│   │   ├── zuul-cr.yaml       # Zuul custom resource
│   │   └── ingress.yaml       # Zuul ingress
│   ├── external-services/     # External service proxies
│   └── test-nginx/            # Test application
│
├── scripts/                   # Utility scripts
├── docs/                      # Documentation
└── README.md                  # This file
```

## Getting Started

### Prerequisites

1. **Kubernetes Cluster**: A running Kubernetes cluster (homelab, cloud, or local)
2. **kubectl**: Configured to access your cluster
3. **FluxCD CLI**: Install from https://fluxcd.io/docs/installation/
4. **SOPS**: For managing encrypted secrets (https://github.com/getsops/sops)
5. **Age**: Encryption tool for SOPS (https://github.com/FiloSottile/age)

### Installation

#### 1. Install FluxCD CLI

```bash
# Linux/macOS
curl -s https://fluxcd.io/install.sh | sudo bash

# Or using package managers
# Homebrew: brew install fluxcd/tap/flux
# Arch: pacman -S flux-bin
```

#### 2. Bootstrap FluxCD on Your Cluster

```bash
# Export your GitHub token
export GITHUB_TOKEN=<your-github-token>

# Bootstrap FluxCD
flux bootstrap github \
  --owner=<your-github-username> \
  --repository=zuul-homelab \
  --branch=master \
  --path=clusters/homelab \
  --personal
```

This will:
- Install FluxCD controllers in the `flux-system` namespace
- Create a deploy key in your GitHub repository
- Configure FluxCD to watch the repository
- Apply all manifests from `clusters/homelab/`

#### 3. Set Up Secret Encryption

```bash
# Generate an Age key
age-keygen -o age.key

# Create a Kubernetes secret with the Age private key
cat age.key |
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin

# Save the public key for encrypting secrets
# Add to .sops.yaml in repository root
```

#### 4. Verify Installation

```bash
# Check FluxCD status
flux check

# Watch reconciliation
flux get kustomizations --watch

# Check all resources
flux get all
```

## FluxCD Concepts

### GitRepository

Defines the Git repository FluxCD watches for changes:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: flux-system
spec:
  interval: 1m        # Check for changes every minute
  ref:
    branch: master
  url: https://github.com/SeanMooney/zuul-homelab
```

### Kustomization

Defines what to deploy from the repository:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: zuul
spec:
  interval: 5m                    # Reconcile every 5 minutes
  path: ./apps/zuul              # Path in repository
  sourceRef:
    kind: GitRepository
    name: flux-system
  dependsOn:                      # Wait for dependencies
    - name: cert-manager
    - name: zookeeper
  prune: true                     # Delete removed resources
  wait: true                      # Wait for resources to be ready
```

### Reconciliation Flow

1. **GitRepository** polls the Git repository (every 1m)
2. **Kustomization** checks if local state matches Git (every 5-10m)
3. If changes detected → applies them to the cluster
4. Monitors health checks and dependencies

## Common Operations

### 1. Triggering a Manual Reconciliation

When you've pushed changes and want them applied immediately:

```bash
# Reconcile a specific kustomization
flux reconcile kustomization zuul -n flux-system

# Reconcile the source first (fetch latest from Git), then kustomization
flux reconcile source git flux-system -n flux-system
flux reconcile kustomization zuul -n flux-system

# Reconcile all infrastructure
flux reconcile kustomization infrastructure -n flux-system
```

**When to use:**
- After pushing configuration changes
- After updating secrets
- When debugging deployment issues
- To force immediate sync instead of waiting for interval

### 2. Checking Status

```bash
# Overview of all FluxCD resources
flux get all

# Check specific resource types
flux get kustomizations
flux get helmreleases
flux get sources all

# Detailed status of a kustomization
flux get kustomization zuul -n flux-system

# Watch for changes in real-time
flux get kustomizations --watch
```

### 3. Viewing Logs

```bash
# FluxCD controller logs
flux logs --all-namespaces --follow

# Specific controller logs
kubectl logs -n flux-system -l app=kustomize-controller -f
kubectl logs -n flux-system -l app=source-controller -f
kubectl logs -n flux-system -l app=helm-controller -f
```

### 4. Suspending/Resuming Reconciliation

Useful during maintenance or when testing changes:

```bash
# Suspend reconciliation (FluxCD stops syncing)
flux suspend kustomization zuul -n flux-system

# Resume reconciliation
flux resume kustomization zuul -n flux-system
```

### 5. Managing Secrets with SOPS

Encrypt secrets before committing:

```bash
# Encrypt a secret file
sops --encrypt --in-place apps/zuul/secrets/github-connection.enc.yaml

# Edit an encrypted secret
sops apps/zuul/secrets/github-connection.enc.yaml

# Decrypt temporarily (view only, don't save)
sops -d apps/zuul/secrets/github-connection.enc.yaml
```

**.sops.yaml configuration** (in repository root):
```yaml
creation_rules:
  - path_regex: \.enc\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

### 6. Updating Zuul/Nodepool Configuration

The most common operation - updating CI configuration:

```bash
# 1. Edit the configuration file
vim apps/zuul/configs/nodepool-config.yaml

# 2. Commit and push changes
git add apps/zuul/configs/nodepool-config.yaml
git commit -m "Update nodepool configuration"
git push origin master

# 3. Trigger immediate reconciliation (optional - otherwise wait 5m)
flux reconcile kustomization zuul -n flux-system

# 4. Verify the secret was updated
kubectl get secret nodepool-config -n zuul-system -o yaml

# 5. Check nodepool logs to confirm config reload
kubectl logs -n zuul-system -l app.kubernetes.io/component=nodepool-launcher -f
```

**Files you'll commonly edit:**
- `apps/zuul/configs/nodepool-config.yaml` - Node pool configuration (labels, providers)
- `apps/zuul/configs/tenant-config.yaml` - Zuul tenant configuration (projects, jobs)
- `apps/zuul/zuul-cr.yaml` - Zuul custom resource (replicas, versions)

### 7. Updating Helm Releases

FluxCD manages Helm charts through HelmRelease resources:

```bash
# Check helm releases
flux get helmreleases -A

# Trigger helm release reconciliation
flux reconcile helmrelease cert-manager -n flux-system

# Check helm release status
flux get helmrelease cert-manager -n flux-system

# View helm release values
kubectl get helmrelease cert-manager -n flux-system -o yaml
```

### 8. Troubleshooting Failed Reconciliation

```bash
# Check kustomization status with details
flux get kustomization zuul -n flux-system

# View events
flux events --for Kustomization/zuul -n flux-system

# Describe the resource
kubectl describe kustomization zuul -n flux-system

# Check for validation errors
flux check

# Validate manifests locally before pushing
kustomize build apps/zuul
kubectl apply --dry-run=server -k apps/zuul
```

### 9. Adding a New Application

1. Create application directory: `apps/myapp/`
2. Add kustomization.yaml and manifests
3. Create FluxCD Kustomization in `clusters/homelab/myapp.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/myapp
  sourceRef:
    kind: GitRepository
    name: flux-system
  prune: true
  wait: true
```

4. Commit and push - FluxCD will automatically deploy it

### 10. Disaster Recovery

If something goes wrong, you can always re-sync from Git:

```bash
# Reconcile everything
flux reconcile source git flux-system
flux reconcile kustomization flux-system

# Nuclear option: recreate all resources
flux reconcile kustomization --with-source

# Restore from Git (if cluster state is broken)
kubectl apply -k clusters/homelab/
```

## Deployment Layers

Infrastructure is deployed in layers with dependencies:

**Layer 1 (Foundation):**
- Namespaces

**Layer 2 (Core Infrastructure):**
- Ingress-NGINX
- Cert-Manager
- Reflector
- External Secrets Operator
- Nodepool RBAC

**Layer 3 (Configuration):**
- Cert-Manager Config (ClusterIssuers)
- Certificates
- External Secrets Config

**Layer 4 (Storage & Databases):**
- Zookeeper
- PostgreSQL
- MinIO

**Layer 5 (Applications):**
- Zuul Operator
- Zuul (depends on all above)

This layering ensures proper startup order and dependency resolution.

## Important Notes

### Automatic vs Manual Reconciliation

- **Automatic**: FluxCD checks Git every 1 minute, reconciles resources every 5-10 minutes
- **Manual**: Use `flux reconcile` for immediate updates
- **Best Practice**: Use automatic for production, manual for testing/debugging

### Configuration Changes

Changes to these directories require reconciliation:
- `apps/zuul/configs/` - Zuul/Nodepool configuration (generates Secrets)
- `apps/zuul/zuul-cr.yaml` - Zuul operator settings
- `infrastructure/*/` - Infrastructure component updates

### Secret Management

- All secrets use `.enc.yaml` extension
- Encrypted with SOPS using Age
- FluxCD automatically decrypts during deployment
- **Never commit unencrypted secrets!**

### Resource Pruning

`prune: true` means FluxCD will delete resources removed from Git. Be careful when removing resources!

## Useful Links

- [FluxCD Documentation](https://fluxcd.io/docs/)
- [Zuul Documentation](https://zuul-ci.org/docs/)
- [Zuul Operator](https://opendev.org/zuul/zuul-operator)
- [SOPS Documentation](https://github.com/getsops/sops)
- [Kustomize Documentation](https://kustomize.io/)

## Quick Reference

| Task | Command |
|------|---------|
| Check FluxCD status | `flux check` |
| View all resources | `flux get all` |
| Reconcile Zuul | `flux reconcile kustomization zuul -n flux-system` |
| Reconcile infrastructure | `flux reconcile kustomization infrastructure -n flux-system` |
| Sync from Git | `flux reconcile source git flux-system` |
| Watch reconciliation | `flux get kustomizations --watch` |
| View logs | `flux logs --follow` |
| Suspend sync | `flux suspend kustomization <name>` |
| Resume sync | `flux resume kustomization <name>` |
| Edit encrypted secret | `sops <path-to-secret.enc.yaml>` |
| Export kustomization | `flux export kustomization <name>` |

## Getting Help

```bash
# FluxCD help
flux --help
flux reconcile --help

# Check documentation
flux docgen

# Community support
# - FluxCD Slack: https://slack.cncf.io/ (#flux)
# - Zuul Matrix: https://matrix.to/#/#zuul:opendev.org
```

---

**Note**: This is a homelab setup. For production deployments, consider additional monitoring, backup strategies, and high-availability configurations.