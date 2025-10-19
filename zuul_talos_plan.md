# Zuul CI Homelab Deployment - GitOps with FluxCD on Talos

## Overview

A lightweight, single-node Zuul CI deployment on Talos Kubernetes using FluxCD for GitOps management. Optimized for personal homelab use with minimal resource overhead while maintaining declarative configuration management.

## Architecture

**Single Node Configuration:**
- 1x Scheduler
- 1x Executor  
- 1x Web
- 1x Merger
- 1x Launcher (Nodepool)
- 1x ZooKeeper (standalone)
- 1x PostgreSQL (single instance)

**Storage:** Local Path Provisioner (RBD-backed on your Talos node)

## Repository Structure

```
zuul-homelab/
├── README.md
├── docs/
│   └── setup-guide.md
├── clusters/
│   └── homelab/
│       ├── flux-system/          # FluxCD bootstrap
│       ├── infrastructure.yaml   # Infrastructure apps
│       └── zuul.yaml            # Zuul application
├── infrastructure/
│   ├── namespaces/
│   ├── storage/
│   ├── cert-manager/
│   ├── zookeeper/
│   └── database/
├── apps/
│   └── zuul/
│       ├── configs/
│       ├── secrets/
│       └── zuul-cr.yaml
└── scripts/
    ├── bootstrap.sh
    └── create-secrets.sh
```

## Phase 1: Repository Setup

### 1.1 Initialize Repository

```bash
# Create repository
mkdir zuul-homelab && cd zuul-homelab
git init

# Create directory structure
mkdir -p clusters/homelab/flux-system
mkdir -p infrastructure/{namespaces,storage,cert-manager,zookeeper,database}
mkdir -p apps/zuul/{configs,secrets}
mkdir -p scripts docs

# Create .gitignore
cat > .gitignore <<EOF
# Unencrypted secrets
secrets/*.yaml
!secrets/*.enc.yaml
!secrets/kustomization.yaml

# Local files
.env
*.local
EOF

git add .
git commit -m "Initial repository structure"
```

### 1.2 Create Namespace Manifests

**File: `infrastructure/namespaces/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - zuul-system.yaml
  - cert-manager.yaml
  - zookeeper-system.yaml
  - postgres-system.yaml
```

**File: `infrastructure/namespaces/zuul-system.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: zuul-system
  labels:
    app.kubernetes.io/name: zuul
    # Talos: Allow privileged pods
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

**File: `infrastructure/namespaces/postgres-system.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: postgres-system
  labels:
    app.kubernetes.io/name: postgres
    pod-security.kubernetes.io/enforce: baseline
```

**File: `infrastructure/namespaces/zookeeper-system.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: zookeeper-system
  labels:
    app.kubernetes.io/name: zookeeper
    pod-security.kubernetes.io/enforce: baseline
```

## Phase 2: Storage Configuration

### 2.1 Local Path Provisioner

**File: `infrastructure/storage/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.26/deploy/local-path-storage.yaml
  - storageclass.yaml

patches:
  - patch: |-
      apiVersion: v1
      kind: Namespace
      metadata:
        name: local-path-storage
        labels:
          pod-security.kubernetes.io/enforce: privileged
          pod-security.kubernetes.io/audit: privileged
          pod-security.kubernetes.io/warn: privileged
    target:
      kind: Namespace
      name: local-path-storage
  
  # Configure for Talos
  - patch: |-
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: local-path-config
        namespace: local-path-storage
      data:
        config.json: |-
          {
            "nodePathMap":[
              {
                "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                "paths":["/var/lib/local-path-provisioner"]
              }
            ]
          }
    target:
      kind: ConfigMap
      name: local-path-config
```

**File: `infrastructure/storage/storageclass.yaml`**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

## Phase 3: Dependencies

### 3.1 Cert-Manager

**File: `infrastructure/cert-manager/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
  - cluster-issuer.yaml
```

**File: `infrastructure/cert-manager/cluster-issuer.yaml`**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com  # Change this
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

### 3.2 PostgreSQL (Lightweight Single Instance)

**File: `infrastructure/database/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: postgres-system

resources:
  - statefulset.yaml
  - service.yaml
  - pvc.yaml

secretGenerator:
  - name: postgres-credentials
    literals:
      - POSTGRES_DB=zuul
      - POSTGRES_USER=zuul
      - POSTGRES_PASSWORD=CHANGEME_postgres_password
    options:
      disableNameSuffixHash: true
```

**File: `infrastructure/database/pvc.yaml`**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: postgres-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi
```

**File: `infrastructure/database/statefulset.yaml`**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres-system
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-credentials
                  key: POSTGRES_PASSWORD
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-data
```

**File: `infrastructure/database/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres-system
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
  clusterIP: None
```

### 3.3 ZooKeeper (Standalone)

**File: `infrastructure/zookeeper/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: zookeeper-system

resources:
  - statefulset.yaml
  - service.yaml
  - pvc.yaml
```

**File: `infrastructure/zookeeper/pvc.yaml`**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: zookeeper-data
  namespace: zookeeper-system
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
```

**File: `infrastructure/zookeeper/statefulset.yaml`**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zookeeper
  namespace: zookeeper-system
spec:
  serviceName: zookeeper
  replicas: 1
  selector:
    matchLabels:
      app: zookeeper
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      containers:
        - name: zookeeper
          image: zookeeper:3.9.1
          ports:
            - containerPort: 2181
              name: client
            - containerPort: 2888
              name: follower
            - containerPort: 3888
              name: election
          env:
            - name: ZOO_STANDALONE_ENABLED
              value: "true"
            - name: ZOO_AUTOPURGE_PURGEINTERVAL
              value: "1"
            - name: ZOO_AUTOPURGE_SNAPRETAINCOUNT
              value: "3"
          volumeMounts:
            - name: data
              mountPath: /data
            - name: datalog
              mountPath: /datalog
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: zookeeper-data
        - name: datalog
          emptyDir: {}
```

**File: `infrastructure/zookeeper/service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: zookeeper-system
spec:
  selector:
    app: zookeeper
  ports:
    - port: 2181
      targetPort: 2181
      name: client
  clusterIP: None
```

## Phase 4: Zuul Application

### 4.1 Zuul Configuration Secrets

**File: `apps/zuul/secrets/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: zuul-system

secretGenerator:
  # Database connection
  - name: zuul-db-uri
    literals:
      - dburi=postgresql://zuul:CHANGEME_postgres_password@postgres.postgres-system.svc.cluster.local:5432/zuul
    options:
      disableNameSuffixHash: true

# Note: For actual secrets, use SOPS encryption (see secrets management section)
```

**File: `apps/zuul/secrets/.sops.yaml`**
```yaml
creation_rules:
  - path_regex: .*secrets.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: >-
      age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

**File: `apps/zuul/secrets/github-connection.enc.yaml`** (template - encrypt with SOPS)
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: github-connection
  namespace: zuul-system
type: Opaque
stringData:
  app_id: "YOUR_GITHUB_APP_ID"
  webhook_token: "YOUR_WEBHOOK_SECRET"
  app_key: |
    -----BEGIN RSA PRIVATE KEY-----
    YOUR_GITHUB_APP_PRIVATE_KEY
    -----END RSA PRIVATE KEY-----
```

### 4.2 Zuul Configuration Files

**File: `apps/zuul/configs/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: zuul-system

secretGenerator:
  - name: zuul-tenant-config
    files:
      - main.yaml=tenant-config.yaml
    options:
      disableNameSuffixHash: true
  
  - name: nodepool-config
    files:
      - nodepool.yaml=nodepool-config.yaml
    options:
      disableNameSuffixHash: true
```

**File: `apps/zuul/configs/tenant-config.yaml`**
```yaml
- tenant:
    name: main
    max-nodes-per-job: 3
    max-job-timeout: 7200  # 2 hours
    
    source:
      github:
        config-projects:
          - YOUR_GITHUB_USER/zuul-jobs:
              load-branch: main
        
        untrusted-projects:
          - YOUR_GITHUB_USER/your-project
```

**File: `apps/zuul/configs/nodepool-config.yaml`**
```yaml
zookeeper-servers:
  - host: zookeeper.zookeeper-system.svc.cluster.local
    port: 2181

labels:
  - name: ubuntu-jammy
    min-ready: 0  # On-demand for homelab

providers:
  - name: kubernetes-pods
    driver: kubernetes
    context: zuul-context
    max-servers: 5
    
    pools:
      - name: main
        labels:
          - name: ubuntu-jammy
            type: pod
            image: ubuntu:22.04
            python-path: /usr/bin/python3
            cpu: 2
            memory: 2048
```

### 4.3 Zuul Custom Resource

**File: `apps/zuul/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: zuul-system

resources:
  - https://opendev.org/zuul/zuul-operator/raw/branch/master/deploy/crds/zuul-ci_v1alpha2_zuul_crd.yaml
  - operator/
  - configs/
  - secrets/
  - zuul-cr.yaml
  - ingress.yaml
```

**File: `apps/zuul/operator/kustomization.yaml`**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: zuul-system

resources:
  - serviceaccount.yaml
  - rbac.yaml
  - deployment.yaml
```

**File: `apps/zuul/operator/serviceaccount.yaml`**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: zuul-operator
  namespace: zuul-system
```

**File: `apps/zuul/operator/rbac.yaml`**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: zuul-operator
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "persistentvolumeclaims", "events", "configmaps", "secrets"]
    verbs: ["*"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["*"]
  - apiGroups: ["operator.zuul-ci.org"]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: zuul-operator
subjects:
  - kind: ServiceAccount
    name: zuul-operator
    namespace: zuul-system
roleRef:
  kind: ClusterRole
  name: zuul-operator
  apiGroup: rbac.authorization.k8s.io
```

**File: `apps/zuul/operator/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zuul-operator
  namespace: zuul-system
spec:
  replicas: 1
  selector:
    matchLabels:
      name: zuul-operator
  template:
    metadata:
      labels:
        name: zuul-operator
    spec:
      serviceAccountName: zuul-operator
      containers:
        - name: zuul-operator
          image: quay.io/zuul-ci/zuul-operator:latest
          imagePullPolicy: Always
          env:
            - name: WATCH_NAMESPACE
              value: zuul-system
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: OPERATOR_NAME
              value: zuul-operator
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

**File: `apps/zuul/zuul-cr.yaml`**
```yaml
apiVersion: operator.zuul-ci.org/v1alpha2
kind: Zuul
metadata:
  name: zuul
  namespace: zuul-system
spec:
  # Image configuration
  imagePrefix: quay.io/zuul-ci/
  imageTag: latest
  
  # External database
  database:
    secretName: zuul-db-uri
    key: dburi
  
  # External ZooKeeper
  zookeeper:
    hosts: zookeeper.zookeeper-system.svc.cluster.local:2181
  
  # Scheduler - single replica for homelab
  scheduler:
    replicas: 1
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 1000m
        memory: 2Gi
  
  # Web UI - single replica
  web:
    replicas: 1
    resources:
      requests:
        cpu: 250m
        memory: 512Mi
      limits:
        cpu: 500m
        memory: 1Gi
    service:
      type: ClusterIP
      port: 9000
  
  # Executor - single replica
  executor:
    replicas: 1
    resources:
      requests:
        cpu: 1000m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 4Gi
    # Talos: Use allowed volume path
    volumeMounts:
      - name: workspace
        mountPath: /var/lib/zuul
    volumes:
      - name: workspace
        emptyDir:
          sizeLimit: 10Gi
  
  # Merger - single replica
  merger:
    replicas: 1
    resources:
      requests:
        cpu: 250m
        memory: 512Mi
      limits:
        cpu: 500m
        memory: 1Gi
  
  # Nodepool launcher
  launcher:
    config:
      secretName: nodepool-config
    resources:
      requests:
        cpu: 250m
        memory: 512Mi
      limits:
        cpu: 500m
        memory: 1Gi
  
  # Source code connections
  connections:
    - name: github
      driver: github
      secretName: github-connection
  
  # Tenant configuration
  tenants:
    secretName: zuul-tenant-config
```

**File: `apps/zuul/ingress.yaml`**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: zuul-web
  namespace: zuul-system
  annotations:
    cert-manager.io/cluster-issuer: selfsigned  # or letsencrypt-prod if you have external access
    nginx.ingress.kubernetes.io/ssl-redirect: "false"  # set to true for production
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - zuul.homelab.local  # Change to your domain
      secretName: zuul-web-tls
  rules:
    - host: zuul.homelab.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: zuul-web
                port:
                  number: 9000
```

## Phase 5: FluxCD Configuration

### 5.1 Bootstrap FluxCD

**Create GitHub Personal Access Token** with `repo` permissions, then:

```bash
# Export token
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=zuul-homelab \
  --branch=main \
  --path=clusters/homelab \
  --personal
```

This creates `clusters/homelab/flux-system/` with FluxCD manifests.

### 5.2 Create SOPS Age Key

```bash
# Generate age key for SOPS
age-keygen -o age.agekey

# Display public key (add to .sops.yaml)
age-keygen -y age.agekey

# Create Kubernetes secret for Flux
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin

# Store private key securely (NOT in git)
mv age.agekey ~/.config/sops/age/keys.txt
```

### 5.3 Infrastructure Deployment

**File: `clusters/homelab/infrastructure.yaml`**
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: namespaces
  namespace: flux-system
spec:
  interval: 10m
  retryInterval: 1m
  timeout: 2m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/namespaces
  prune: true
  wait: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: storage
  namespace: flux-system
spec:
  interval: 10m
  dependsOn:
    - name: namespaces
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/storage
  prune: true
  wait: true
  timeout: 5m
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cert-manager
  namespace: flux-system
spec:
  interval: 10m
  dependsOn:
    - name: namespaces
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/cert-manager
  prune: true
  wait: true
  timeout: 5m
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: cert-manager
      namespace: cert-manager
    - apiVersion: apps/v1
      kind: Deployment
      name: cert-manager-webhook
      namespace: cert-manager
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: zookeeper
  namespace: flux-system
spec:
  interval: 10m
  dependsOn:
    - name: namespaces
    - name: storage
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/zookeeper
  prune: true
  wait: true
  timeout: 10m
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: database
  namespace: flux-system
spec:
  interval: 10m
  dependsOn:
    - name: namespaces
    - name: storage
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/database
  prune: true
  wait: true
  timeout: 10m
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### 5.4 Zuul Application Deployment

**File: `clusters/homelab/zuul.yaml`**
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: zuul
  namespace: flux-system
spec:
  interval: 5m
  retryInterval: 1m
  timeout: 10m
  dependsOn:
    - name: cert-manager
    - name: zookeeper
    - name: database
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/zuul
  prune: true
  wait: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

## Phase 6: Secrets Management

### 6.1 Encrypt Secrets with SOPS

**Update `.sops.yaml` in repository root:**
```yaml
creation_rules:
  - path_regex: apps/zuul/secrets/.*\.enc\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p  # Your public key
  - path_regex: infrastructure/database/.*\.yaml$
    encrypted_regex: ^(data|stringData|literals)$
    age: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
```

**Encrypt GitHub connection secret:**
```bash
# Create secret file
cat > apps/zuul/secrets/github-connection.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: github-connection
  namespace: zuul-system
type: Opaque
stringData:
  app_id: "12345"
  webhook_token: "your-webhook-secret"
  app_key: |
    -----BEGIN RSA PRIVATE KEY-----
    your-private-key-here
    -----END RSA PRIVATE KEY-----
EOF

# Encrypt it
sops -e -i apps/zuul/secrets/github-connection.yaml

# Rename to .enc.yaml
mv apps/zuul/secrets/github-connection.yaml apps/zuul/secrets/github-connection.enc.yaml

# Add to secrets kustomization
cat >> apps/zuul/secrets/kustomization.yaml <<EOF

resources:
  - github-connection.enc.yaml
EOF
```

**Encrypt database password:**
```bash
# Edit the database kustomization
sops infrastructure/database/kustomization.yaml

# Update password, save (it will be encrypted)
```

## Helper Scripts

**File: `scripts/bootstrap.sh`**
```bash
#!/bin/bash
set -euo pipefail

echo "=== Zuul Homelab Bootstrap ==="

# Check prerequisites
command -v flux >/dev/null 2>&1 || { echo "flux CLI required"; exit 1; }
command -v sops >/dev/null 2>&1 || { echo "sops required"; exit 1; }
command -v age >/dev/null 2>&1 || { echo "age required"; exit 1; }
command -v kubectl >/dev/null 2>&1 || { echo "kubectl required"; exit 1; }

# Check if already bootstrapped
if kubectl get namespace flux-system &>/dev/null; then
    echo "FluxCD already installed"
else
    echo "Please run Flux bootstrap first:"
    echo "  flux bootstrap github --owner=YOUR_USER --repository=zuul-homelab --branch=main --path=clusters/homelab --personal"
    exit 1
fi

# Check for age key
if [ ! -f ~/.config/sops/age/keys.txt ]; then
    echo "Generating age key..."
    mkdir -p ~/.config/sops/age
    age-keygen -o ~/.config/sops/age/keys.txt
    echo "Age public key:"
    age-keygen -y ~/.config/sops/age/keys.txt
    echo ""
    echo "Add this public key to .sops.yaml in the repository"
fi

# Create SOPS secret in cluster
if kubectl get secret sops-age -n flux-system &>/dev/null; then
    echo "SOPS age secret already exists"
else
    echo "Creating SOPS age secret..."
    cat ~/.config/sops/age/keys.txt | kubectl create secret generic sops-age \
        --namespace=flux-system \
        --from-file=age.agekey=/dev/stdin
fi

echo ""
echo "=== Bootstrap Complete ==="
echo "Next steps:"
echo "1. Update secrets in apps/zuul/secrets/ with your actual values"
echo "2. Encrypt secrets: sops -e -i apps/zuul/secrets/*.yaml"
echo "3. Update .sops.yaml with your age public key"
echo "4. Commit and push to trigger deployment"
echo "5. Monitor: flux get kustomizations --watch"
```

**File: `scripts/create-secrets.sh`**
```bash
#!/bin/bash
set -euo pipefail

echo "=== Create Zuul Secrets ==="

# Check if SOPS is configured
if [ ! -f ~/.config/sops/age/keys.txt ]; then
    echo "Error: Age key not found. Run ./bootstrap.sh first"
    exit 1
fi

# Create secrets directory
mkdir -p apps/zuul/secrets

# GitHub connection
if [ ! -f apps/zuul/secrets/github-connection.enc.yaml ]; then
    echo "Creating GitHub connection secret..."
    read -p "GitHub App ID: " APP_ID
    read -p "GitHub Webhook Token: " WEBHOOK_TOKEN
    echo "Paste GitHub App Private Key (Ctrl-D when done):"
    APP_KEY=$(cat)

    cat > apps/zuul/secrets/github-connection.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: github-connection
  namespace: zuul-system
type: Opaque
stringData:
  app_id: "${APP_ID}"
  webhook_token: "${WEBHOOK_TOKEN}"
  app_key: |
$(echo "${APP_KEY}" | sed 's/^/    /')
EOF

    sops -e -i apps/zuul/secrets/github-connection.yaml
    mv apps/zuul/secrets/github-connection.yaml apps/zuul/secrets/github-connection.enc.yaml
    echo "✓ GitHub connection secret created"
fi

# Database password
echo "Updating database password..."
read -sp "PostgreSQL password for zuul user: " DB_PASSWORD
echo

sops infrastructure/database/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: postgres-system

resources:
  - statefulset.yaml
  - service.yaml
  - pvc.yaml

secretGenerator:
  - name: postgres-credentials
    literals:
      - POSTGRES_DB=zuul
      - POSTGRES_USER=zuul
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    options:
      disableNameSuffixHash: true
EOF

# Update DB URI secret
sops apps/zuul/secrets/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: zuul-system

secretGenerator:
  - name: zuul-db-uri
    literals:
      - dburi=postgresql://zuul:${DB_PASSWORD}@postgres.postgres-system.svc.cluster.local:5432/zuul
    options:
      disableNameSuffixHash: true

resources:
  - github-connection.enc.yaml
EOF

echo "✓ Secrets created successfully"
echo ""
echo "Next: Commit and push to deploy"
```

## Deployment Workflow

### Initial Setup

```bash
# 1. Clone your repository
git clone https://github.com/YOUR_USER/zuul-homelab
cd zuul-homelab

# 2. Bootstrap FluxCD
export GITHUB_TOKEN=<your-token>
flux bootstrap github \
  --owner=YOUR_USER \
  --repository=zuul-homelab \
  --branch=main \
  --path=clusters/homelab \
  --personal

# 3. Run bootstrap script
./scripts/bootstrap.sh

# 4. Create secrets
./scripts/create-secrets.sh

# 5. Update configurations
# Edit apps/zuul/configs/tenant-config.yaml
# Edit apps/zuul/configs/nodepool-config.yaml
# Update apps/zuul/ingress.yaml with your domain

# 6. Commit and push
git add .
git commit -m "Initial Zuul deployment"
git push

# 7. Watch deployment
flux get kustomizations --watch

# Or watch specific components
kubectl get pods -n zuul-system -w
```

### Making Updates

```bash
# 1. Edit configuration
vim apps/zuul/configs/tenant-config.yaml

# 2. Commit and push
git add apps/zuul/configs/tenant-config.yaml
git commit -m "Add new project to Zuul"
git push

# 3. FluxCD automatically applies changes
# Watch: flux get kustomizations --watch
```

### Adding New Secrets

```bash
# 1. Create secret file
cat > apps/zuul/secrets/new-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: new-secret
  namespace: zuul-system
stringData:
  key: value
EOF

# 2. Encrypt with SOPS
sops -e -i apps/zuul/secrets/new-secret.yaml
mv apps/zuul/secrets/new-secret.yaml apps/zuul/secrets/new-secret.enc.yaml

# 3. Add to kustomization
echo "  - new-secret.enc.yaml" >> apps/zuul/secrets/kustomization.yaml

# 4. Commit and push
git add apps/zuul/secrets/
git commit -m "Add new secret"
git push
```

## Monitoring

### Check Deployment Status

```bash
# FluxCD status
flux get all

# Specific kustomizations
flux get kustomizations

# Source reconciliation
flux get sources git

# Component status
kubectl get all -n zuul-system
kubectl get all -n postgres-system
kubectl get all -n zookeeper-system

# Logs
kubectl logs -n zuul-system deployment/zuul-operator -f
kubectl logs -n zuul-system deployment/zuul-scheduler -f
kubectl logs -n zuul-system deployment/zuul-web -f
```

### Access Zuul Web UI

```bash
# Port forward (for local access)
kubectl port-forward -n zuul-system svc/zuul-web 9000:9000

# Then open: http://localhost:9000

# Or via ingress (if configured)
# https://zuul.homelab.local
```

## Resource Requirements

**Minimum Node Requirements:**
- CPU: 6 cores (8+ recommended)
- RAM: 16GB (24GB+ recommended)
- Storage: 50GB+ free space for RBD backing

**Expected Resource Usage:**
- **Total CPU**: ~3-4 cores under normal load
- **Total Memory**: ~8-10GB
- **Storage**: ~40GB (database + ZooKeeper + workspace)

## Troubleshooting

### FluxCD not reconciling

```bash
# Force reconciliation
flux reconcile kustomization zuul --with-source

# Check for errors
flux logs --level=error
```

### SOPS decryption fails

```bash
# Verify age secret exists
kubectl get secret sops-age -n flux-system

# Check if secret has correct key
kubectl get secret sops-age -n flux-system -o yaml

# Recreate if needed
cat ~/.config/sops/age/keys.txt | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Database connection issues

```bash
# Check PostgreSQL is running
kubectl get pods -n postgres-system

# Test connection from zuul pod
kubectl exec -it -n zuul-system deployment/zuul-scheduler -- \
  psql postgresql://zuul:password@postgres.postgres-system.svc.cluster.local:5432/zuul

# Check logs
kubectl logs -n postgres-system statefulset/postgres
```

### ZooKeeper connection issues

```bash
# Check ZooKeeper
kubectl get pods -n zookeeper-system

# Test connectivity
kubectl exec -it -n zuul-system deployment/zuul-scheduler -- \
  nc -zv zookeeper.zookeeper-system.svc.cluster.local 2181
```

## Backup Strategy

Since this is a homelab setup, simple backup approach:

```bash
# Backup PostgreSQL
kubectl exec -n postgres-system statefulset/postgres -- \
  pg_dump -U zuul zuul > zuul-backup-$(date +%Y%m%d).sql

# Backup ZooKeeper data (optional)
kubectl exec -n zookeeper-system statefulset/zookeeper -- \
  tar czf /tmp/zk-backup.tar.gz /data
kubectl cp zookeeper-system/zookeeper-0:/tmp/zk-backup.tar.gz \
  ./zk-backup-$(date +%Y%m%d).tar.gz

# Restore PostgreSQL
cat zuul-backup-20240101.sql | kubectl exec -i -n postgres-system \
  statefulset/postgres -- psql -U zuul zuul
```

## Summary

This lightweight setup provides:

✅ **GitOps workflow** - All configs in git, auto-deploy on push  
✅ **SOPS encryption** - Secrets safely in git  
✅ **Single-node optimized** - 1 replica each, ~4 cores, ~10GB RAM  
✅ **Local storage** - Uses your RBD-backed local-path-provisioner  
✅ **Easy maintenance** - Edit config, commit, push, done  
✅ **Talos compatible** - Proper volume paths, pod security labels  
✅ **Minimal dependencies** - Simple PostgreSQL and ZooKeeper  

Perfect for a personal homelab CI system!
