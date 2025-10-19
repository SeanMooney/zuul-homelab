# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a GitOps-based Zuul CI homelab deployment designed for Talos Kubernetes using FluxCD. The project provides a lightweight, single-node Zuul CI setup optimized for personal homelab use with minimal resource overhead.

## Repository Structure

```
zuul-homelab/
├── clusters/homelab/          # FluxCD configurations
│   ├── flux-system/          # FluxCD bootstrap (auto-generated)
│   ├── infrastructure.yaml   # Infrastructure deployment order
│   └── zuul.yaml            # Zuul application deployment
├── infrastructure/           # Kubernetes infrastructure components
│   ├── namespaces/          # Namespace definitions
│   ├── storage/             # Local Path Provisioner configuration
│   ├── cert-manager/        # Certificate management
│   ├── zookeeper/          # ZooKeeper single-node setup
│   └── database/           # PostgreSQL single instance
├── apps/zuul/              # Zuul application manifests
│   ├── configs/            # Zuul configuration files
│   ├── secrets/            # SOPS-encrypted secrets
│   ├── operator/           # Zuul operator deployment
│   ├── zuul-cr.yaml       # Zuul Custom Resource
│   └── ingress.yaml       # Web UI ingress
└── scripts/               # Helper scripts
    ├── bootstrap.sh       # Initial setup automation
    └── create-secrets.sh  # Secret creation helper
```

## Architecture

**Single-Node Zuul Components:**
- 1x Scheduler, Executor, Web, Merger, Launcher (Nodepool)
- 1x ZooKeeper (standalone)
- 1x PostgreSQL (single instance)
- Local Path Provisioner for storage (RBD-backed)

## Common Commands

### FluxCD Management
```bash
# Bootstrap FluxCD (initial setup only)
export GITHUB_TOKEN=<token>
flux bootstrap github --owner=<user> --repository=zuul-homelab --branch=main --path=clusters/homelab --personal

# Watch deployment status
flux get kustomizations --watch

# Force reconciliation
flux reconcile kustomization zuul --with-source

# Check all FluxCD resources
flux get all
```

### SOPS Secret Management
```bash
# Encrypt a secret file
sops -e -i apps/zuul/secrets/secret-file.yaml

# Edit encrypted file
sops apps/zuul/secrets/secret-file.enc.yaml

# Create age key (if needed)
age-keygen -o ~/.config/sops/age/keys.txt

# Create SOPS secret in cluster
cat ~/.config/sops/age/keys.txt | kubectl create secret generic sops-age \
  --namespace=flux-system --from-file=age.agekey=/dev/stdin
```

### Deployment and Updates
```bash
# Initial deployment
./scripts/bootstrap.sh
./scripts/create-secrets.sh

# Apply configuration changes (GitOps workflow)
git add . && git commit -m "Update configuration" && git push

# Monitor deployments
kubectl get pods -n zuul-system -w
kubectl get pods -n postgres-system -w
kubectl get pods -n zookeeper-system -w
```

### Debugging
```bash
# Check component logs
kubectl logs -n zuul-system deployment/zuul-scheduler -f
kubectl logs -n zuul-system deployment/zuul-web -f
kubectl logs -n zuul-system deployment/zuul-operator -f

# Access Zuul Web UI locally
kubectl port-forward -n zuul-system svc/zuul-web 9000:9000

# Test database connectivity
kubectl exec -it -n zuul-system deployment/zuul-scheduler -- \
  psql postgresql://zuul:password@postgres.postgres-system.svc.cluster.local:5432/zuul

# Test ZooKeeper connectivity
kubectl exec -it -n zuul-system deployment/zuul-scheduler -- \
  nc -zv zookeeper.zookeeper-system.svc.cluster.local 2181
```

## GitOps Workflow

1. **Make changes** to configurations in respective directories
2. **Encrypt secrets** with SOPS if needed: `sops -e -i path/to/secret.yaml`
3. **Commit and push** changes to trigger automatic deployment
4. **Monitor** with `flux get kustomizations --watch`

## Key Configuration Files

- **Database credentials**: `infrastructure/database/kustomization.yaml` (SOPS encrypted)
- **Zuul DB connection**: `apps/zuul/secrets/kustomization.yaml` (SOPS encrypted)
- **GitHub connection**: `apps/zuul/secrets/github-connection.enc.yaml` (SOPS encrypted)
- **Tenant config**: `apps/zuul/configs/tenant-config.yaml`
- **Nodepool config**: `apps/zuul/configs/nodepool-config.yaml`
- **Zuul CR**: `apps/zuul/zuul-cr.yaml`

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

## Important Notes

- All secrets must be SOPS-encrypted before committing
- FluxCD automatically deploys changes on git push
- Use privileged pod security for zuul-system namespace (required for executors)
- Local Path Provisioner uses `/var/lib/local-path-provisioner` on Talos nodes
- Single-node setup optimized for homelab use, not production clusters