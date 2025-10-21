# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitOps-based Zuul CI homelab deployment designed for Talos Kubernetes using FluxCD. The project provides a lightweight, single-node Zuul CI setup optimized for personal homelab use with minimal resource overhead.

## Repository Structure

```
zuul-homelab/
├── clusters/homelab/                      # FluxCD deployment definitions
│   ├── flux-system/                      # FluxCD bootstrap (auto-generated)
│   ├── infrastructure.yaml               # Layer 1-3: namespaces, core services, dependencies
│   └── zuul.yaml                        # Layer 4: Zuul application
├── infrastructure/                       # Infrastructure layer components
│   ├── namespaces/                      # Namespace definitions (layer 1)
│   ├── ingress-nginx/                   # Ingress controller HelmRelease (layer 2)
│   ├── cert-manager/                    # Cert-manager HelmRelease (layer 2)
│   ├── reflector/                       # Secret/ConfigMap replication (layer 2)
│   ├── cert-manager-config/             # ClusterIssuer and Cloudflare token (layer 3)
│   ├── certificates/                    # Wildcard and ZooKeeper certs (layer 3)
│   ├── zookeeper/                       # ZooKeeper StatefulSet with TLS (layer 3)
│   └── database/                        # PostgreSQL StatefulSet (layer 3)
├── apps/zuul/                           # Zuul application manifests
│   ├── configs/                         # Zuul and Nodepool configurations
│   ├── secrets/                         # SOPS-encrypted connection secrets
│   ├── operator/                        # Custom zuul-operator deployment
│   ├── zuul-cr.yaml                    # Zuul Custom Resource
│   ├── ingress.yaml                    # Web UI ingress
│   └── kustomization.yaml              # Kustomize configuration
└── .sops.yaml                           # SOPS encryption rules
```

## Architecture

**Single-Node Zuul Components:**
- 1x Scheduler, Executor, Web, Merger, Launcher (Nodepool)
- 1x ZooKeeper (standalone with TLS via self-signed CA)
- 1x PostgreSQL (single instance)
- Local Path Provisioner for storage (RBD-backed)
- Custom Zuul operator with executor securityContext fixes for bubblewrap

**FluxCD Deployment Layers:**
FluxCD deploys infrastructure in strict dependency order (see `clusters/homelab/infrastructure.yaml`):
1. **Layer 1**: Namespaces (foundation)
2. **Layer 2**: ingress-nginx, cert-manager, reflector (core services)
3. **Layer 3**: cert-manager-config, certificates, zookeeper, database (dependencies)
4. **Layer 4**: Zuul application via `clusters/homelab/zuul.yaml` (depends on all above)

Each layer must be healthy before the next deploys.

## Common Commands

### FluxCD Management
```bash
# Bootstrap FluxCD (initial setup only)
export GITHUB_TOKEN=<token>
flux bootstrap github --owner=<user> --repository=zuul-homelab --branch=main --path=clusters/homelab --personal

# Watch deployment status
flux get kustomizations --watch

# Check specific layer status
flux get kustomization namespaces
flux get kustomization cert-manager
flux get kustomization database
flux get kustomization zuul

# Force reconciliation (propagates to dependencies)
flux reconcile kustomization infrastructure --with-source
flux reconcile kustomization zuul --with-source

# Reconcile specific components
flux reconcile kustomization cert-manager
flux reconcile kustomization database

# Check all FluxCD resources
flux get all

# View HelmRelease status (for cert-manager, ingress-nginx, reflector)
kubectl get helmrelease -A -o wide
```

### SOPS Secret Management

**Encryption Rules** (defined in `.sops.yaml`):
- `apps/zuul/secrets/*.yaml` - Auto-encrypts data/stringData/literals fields
- `infrastructure/database/*.yaml` - Auto-encrypts database credentials
- `*.enc.yaml` - Any file with .enc.yaml extension

```bash
# Encrypt a secret file (auto-detects path rules)
sops -e -i apps/zuul/secrets/github-connection.yaml

# Edit encrypted file (decrypts temporarily in editor)
sops apps/zuul/secrets/github-connection.enc.yaml

# Decrypt to view (stdout only, doesn't modify file)
sops -d apps/zuul/secrets/db-uri-secret.enc.yaml

# Create age key (initial setup)
age-keygen -o ~/.config/sops/age/keys.txt

# Deploy SOPS age key to cluster (required for FluxCD decryption)
cat ~/.config/sops/age/keys.txt | kubectl create secret generic sops-age \
  --namespace=flux-system --from-file=age.agekey=/dev/stdin

# Verify SOPS secret exists
kubectl get secret -n flux-system sops-age
```

### Deployment and Updates
```bash
# Apply configuration changes (GitOps workflow)
# 1. Make changes to YAML files
# 2. Encrypt secrets if needed: sops -e -i path/to/secret.yaml
# 3. Commit and push
git add . && git commit -m "Update configuration" && git push

# Monitor deployments
kubectl get pods -n zuul-system -w
kubectl get pods -n postgres-system -w
kubectl get pods -n zookeeper-system -w

# Check deployment health across all namespaces
kubectl get pods -A | grep -E 'zuul|postgres|zookeeper|cert-manager|ingress'
```

### Debugging

```bash
# Check Zuul component logs
kubectl logs -n zuul-system deployment/zuul-scheduler -f
kubectl logs -n zuul-system deployment/zuul-web -f
kubectl logs -n zuul-system deployment/zuul-executor -f
kubectl logs -n zuul-system deployment/zuul-operator -f

# Check FluxCD reconciliation issues
flux logs --level=error
kubectl logs -n flux-system deployment/kustomize-controller -f
kubectl logs -n flux-system deployment/helm-controller -f

# Check SOPS decryption issues
kubectl describe kustomization -n flux-system zuul
kubectl describe kustomization -n flux-system database

# Check certificate issues
kubectl get certificate -A
kubectl describe certificate -n cert-manager teim-app-wildcard
kubectl logs -n cert-manager deployment/cert-manager-cert-manager --tail=50

# Access Zuul Web UI locally
kubectl port-forward -n zuul-system svc/zuul-web 9000:9000

# Test database connectivity (ensure you have the correct password)
kubectl exec -it -n postgres-system statefulset/postgres -- \
  psql -U zuul -d zuul

# Test ZooKeeper connectivity (TLS port 2281)
kubectl exec -it -n zuul-system deployment/zuul-scheduler -- \
  nc -zv zookeeper.zookeeper-system.svc.cluster.local 2281

# Check ZooKeeper TLS certificates
kubectl get secret -n zuul-system zookeeper-client-tls
kubectl describe certificate -n zookeeper-system zookeeper-client-cert

# Verify reflector is replicating certificates
kubectl logs -n cert-manager deployment/reflector --tail=20
```

## GitOps Workflow

1. **Make changes** to configurations in respective directories
2. **Encrypt secrets** with SOPS if needed: `sops -e -i path/to/secret.yaml`
3. **Commit and push** changes to trigger automatic deployment
4. **Monitor** with `flux get kustomizations --watch`

## Key Configuration Files

**Zuul Application:**
- **Zuul CR**: `apps/zuul/zuul-cr.yaml` - Main Zuul Custom Resource defining all components
- **Zuul operator**: `apps/zuul/operator/deployment.yaml` - Custom build with executor fixes
- **Tenant config**: `apps/zuul/configs/tenant-config.yaml` - Project and pipeline definitions
- **Nodepool config**: `apps/zuul/configs/nodepool-config.yaml` - Provider and label configuration

**Secrets (SOPS encrypted):**
- **Database credentials**: `infrastructure/database/secret.yaml` - PostgreSQL user/password
- **Zuul DB URI**: `apps/zuul/secrets/db-uri-secret.enc.yaml` - Database connection string
- **GitHub connection**: `apps/zuul/secrets/github-connection.enc.yaml` - GitHub app credentials
- **Gerrit connection**: `apps/zuul/secrets/gerrit-connection.enc.yaml` - OpenDev Gerrit credentials
- **Cloudflare token**: `infrastructure/cert-manager-config/cloudflare-api-token-secret.enc.yaml` - DNS API token

**Infrastructure:**
- **ZooKeeper TLS**: `infrastructure/certificates/zookeeper-*.yaml` - Self-signed CA and certificates
- **Wildcard cert**: `infrastructure/certificates/teim-app-wildcard.yaml` - Let's Encrypt via Cloudflare DNS
- **Cluster issuer**: `infrastructure/cert-manager-config/cluster-issuer.yaml` - Let's Encrypt production issuer

## Resource Requirements

- **Minimum**: 6 cores, 16GB RAM, 50GB storage
- **Recommended**: 8+ cores, 24GB+ RAM
- **Expected usage**: ~3-4 cores, ~8-10GB RAM under normal load

## Backup Commands

```bash
# Backup PostgreSQL
kubectl exec -n postgres-system statefulset/postgres -- \
  pg_dump -U zuul zuul > zuul-backup-$(date +%Y%m%d).sql

# Restore PostgreSQL
cat backup.sql | kubectl exec -i -n postgres-system \
  statefulset/postgres -- psql -U zuul zuul
```

## TLS Certificate Architecture

**ZooKeeper (Self-Signed CA):**
1. `zookeeper-ca-issuer` - Self-signed CA issuer in cert-manager
2. `zookeeper-ca-certificate` - Root CA certificate (10-year validity)
3. `zookeeper-server-cert` - Server certificate for ZooKeeper StatefulSet
4. `zookeeper-client-cert` - Client certificate for Zuul components
5. Reflector replicates `zookeeper-client-tls` secret to zuul-system namespace

ZooKeeper listens on port 2281 (TLS) instead of 2181 (plaintext).

**Wildcard Certificate (Let's Encrypt):**
- ClusterIssuer uses Cloudflare DNS-01 challenge for `*.teim.app`
- Certificate automatically renews via cert-manager
- Used by ingress resources for HTTPS

## Important Notes

- **All secrets must be SOPS-encrypted before committing** - Check with `git diff` before push
- **FluxCD automatically deploys changes on git push** - Monitor with `flux get kustomizations --watch`
- **Privileged pod security required** - `zuul-system` namespace needs privileged PSP for executor bubblewrap
- **Custom Zuul operator in use** - Using `ghcr.io/seanmooney/zuul-operator:master-c3fe5ae` with executor security fixes
- **Dependency ordering matters** - FluxCD enforces strict layer dependencies; failures cascade
- **Local Path Provisioner** - Uses `/var/lib/local-path-provisioner` on Talos nodes (RBD-backed)
- **Single-node optimized** - Not suitable for production multi-node clusters without modification