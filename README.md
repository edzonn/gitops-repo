# GitOps Process and Implementation Guide

## What is GitOps?

GitOps is a operational framework that applies DevOps best practices (version control, collaboration, compliance, CI/CD) to infrastructure automation. Git becomes the single source of truth for declarative infrastructure and applications.

## Core Principles

1. **Declarative Configuration**: All infrastructure and application configurations are declared in Git
2. **Version Controlled**: Git as the single source of truth
3. **Automated Delivery**: Changes are automatically applied when committed
4. **Continuous Reconciliation**: Agents ensure actual state matches desired state

## Repository Structure

### Option 1: Monorepo Structure
```
gitops-repo/
├── apps/
│   ├── app1/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   ├── overlays/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── production/
│   ├── app2/
│   └── ...
├── infrastructure/
│   ├── namespaces/
│   ├── ingress/
│   ├── monitoring/
│   └── storage/
├── clusters/
│   ├── dev/
│   ├── staging/
│   └── production/
└── README.md
```

### Option 2: Multi-Repo Structure
```
app-repo/                    config-repo/
├── src/                     ├── apps/
├── Dockerfile              │   └── app1/
├── .gitlab-ci.yml          │       ├── dev/
└── README.md               │       ├── staging/
                            │       └── production/
                            ├── infrastructure/
                            └── README.md
```

## Implementation Steps

### Phase 1: Setup Foundation

#### 1. Choose Your GitOps Tool

**ArgoCD (Recommended for Beginners)**
- User-friendly UI
- Multi-tenancy support
- Built-in SSO
- Better for application deployment

**Flux (Recommended for Advanced)**
- Lightweight
- Native Kubernetes integration
- Better for infrastructure automation
- GitOps Toolkit approach

#### 2. Install GitOps Operator

**ArgoCD Installation:**
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Access the UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get initial password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Flux Installation:**
```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux
flux bootstrap github \
  --owner=<your-username> \
  --repository=<repo-name> \
  --branch=main \
  --path=clusters/production \
  --personal
```

### Phase 2: Repository Setup

#### 1. Create Git Repository Structure
```bash
mkdir gitops-infrastructure
cd gitops-infrastructure

# Create directory structure
mkdir -p {apps,infrastructure,clusters}/{dev,staging,production}
mkdir -p apps/sample-app/{base,overlays/{dev,staging,production}}
```

#### 2. Create Base Manifests

**apps/sample-app/base/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: app
        image: nginx:latest
        ports:
        - containerPort: 80
```

**apps/sample-app/base/service.yaml:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

**apps/sample-app/base/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
```

#### 3. Create Environment Overlays

**apps/sample-app/overlays/production/kustomization.yaml:**
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
nameSuffix: -prod
replicas:
  - name: sample-app
    count: 5
images:
  - name: nginx
    newTag: "1.21"
```

### Phase 3: Configure GitOps Application

#### ArgoCD Application Manifest
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/gitops-repo
    targetRevision: HEAD
    path: apps/sample-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
    - Validate=true
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
```

#### Flux Kustomization
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: sample-app-production
  namespace: flux-system
spec:
  interval: 5m
  path: ./apps/sample-app/overlays/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  validation: client
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: sample-app
      namespace: production
```

### Phase 4: CI/CD Integration

#### GitLab CI Example
```yaml
stages:
  - build
  - update-manifest

build:
  stage: build
  script:
    - docker build -t myapp:${CI_COMMIT_SHA} .
    - docker push myapp:${CI_COMMIT_SHA}

update-manifest:
  stage: update-manifest
  script:
    - git clone https://github.com/your-org/gitops-repo
    - cd gitops-repo
    - |
      sed -i "s|newTag:.*|newTag: ${CI_COMMIT_SHA}|g" \
        apps/sample-app/overlays/production/kustomization.yaml
    - git add .
    - git commit -m "Update image to ${CI_COMMIT_SHA}"
    - git push
```

#### GitHub Actions Example
```yaml
name: Update GitOps Repo
on:
  push:
    branches: [main]

jobs:
  update-gitops:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
      
      - name: Build and Push
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker push myapp:${{ github.sha }}
      
      - name: Update Manifest
        run: |
          git clone https://github.com/your-org/gitops-repo
          cd gitops-repo
          sed -i "s|newTag:.*|newTag: ${{ github.sha }}|g" \
            apps/sample-app/overlays/production/kustomization.yaml
          git add .
          git commit -m "Update image to ${{ github.sha }}"
          git push
```

## Best Practices

### 1. Repository Organization
- **Separate config from code**: Keep application code and deployment configs in different repos
- **Environment branching**: Use different branches or directories for each environment
- **Clear naming**: Use descriptive names for manifests and directories

### 2. Security
- **Secret management**: Use Sealed Secrets, External Secrets Operator, or Vault
- **RBAC**: Implement proper role-based access control
- **Git access**: Use SSH keys or deploy keys, never personal access tokens in production
- **Audit logging**: Enable audit logs for all changes

### 3. Deployment Strategy
- **Progressive delivery**: Implement canary or blue-green deployments
- **Health checks**: Define proper liveness and readiness probes
- **Resource limits**: Always set CPU and memory limits
- **Auto-sync carefully**: Consider manual approval for production

### 4. Monitoring & Observability
- **Sync status monitoring**: Track sync failures and delays
- **Application metrics**: Monitor application health through GitOps tools
- **Notification**: Set up Slack/email notifications for sync events
- **Git commit tracking**: Correlate deployments with git commits

### 5. Disaster Recovery
- **Backup**: Regularly backup your Git repository
- **Rollback plan**: Document rollback procedures
- **Multi-cluster**: Consider disaster recovery across clusters
- **Documentation**: Keep runbooks updated

## Common Pitfalls to Avoid

1. **Don't commit secrets directly** - Use secret management solutions
2. **Avoid manual kubectl apply** - All changes should go through Git
3. **Don't skip testing** - Test manifests before merging to main
4. **Avoid tight coupling** - Keep applications loosely coupled
5. **Don't ignore drift** - Configure auto-sync or regular drift detection
6. **Avoid over-automation** - Some changes need manual approval
7. **Don't skip documentation** - Document your processes and decisions

## Troubleshooting

### Sync Issues
```bash
# ArgoCD - Check application status
argocd app get <app-name>
argocd app sync <app-name> --force

# Flux - Check Kustomization status
flux get kustomizations
flux reconcile kustomization <name> --with-source
```

### Drift Detection
```bash
# ArgoCD - Detect drift
argocd app diff <app-name>

# Flux - Check for drift
flux diff kustomization <name>
```

## Success Metrics

Track these metrics to measure GitOps success:

1. **Deployment Frequency**: How often you deploy to production
2. **Lead Time**: Time from commit to production
3. **Mean Time to Recovery (MTTR)**: Time to recover from failures
4. **Change Failure Rate**: Percentage of deployments causing issues
5. **Sync Success Rate**: Percentage of successful syncs
6. **Drift Detection Time**: Time to detect configuration drift

## Next Steps

1. Start with a non-critical application
2. Implement for one environment first (dev)
3. Add monitoring and alerts
4. Expand to other environments
5. Add progressive delivery features
6. Implement multi-cluster management
7. Add policy enforcement (OPA/Kyverno)

## Additional Resources

- ArgoCD Documentation: https://argo-cd.readthedocs.io/
- Flux Documentation: https://fluxcd.io/docs/
- Kustomize: https://kustomize.io/
- GitOps Principles: https://opengitops.dev/
