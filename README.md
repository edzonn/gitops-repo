#!/bin/bash

# GitOps Quick Start Setup Script
# This script sets up ArgoCD and creates a sample GitOps repository structure

set -e

echo "╔════════════════════════════════════════════╗"
echo "║   GitOps Quick Start Setup Script         ║"
echo "╚════════════════════════════════════════════╝"
echo ""

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Function to print colored output
print_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

print_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

print_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Check prerequisites
check_prerequisites() {
    print_info "Checking prerequisites..."
    
    if ! command -v kubectl &> /dev/null; then
        print_error "kubectl not found. Please install kubectl first."
        exit 1
    fi
    
    if ! command -v git &> /dev/null; then
        print_error "git not found. Please install git first."
        exit 1
    fi
    
    # Check if kubectl can connect to cluster
    if ! kubectl cluster-info &> /dev/null; then
        print_error "Cannot connect to Kubernetes cluster. Please configure kubectl."
        exit 1
    fi
    
    print_info "All prerequisites met!"
}

# Install ArgoCD
install_argocd() {
    print_info "Installing ArgoCD..."
    
    # Create namespace
    kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
    
    # Install ArgoCD
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    
    print_info "Waiting for ArgoCD to be ready..."
    kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
    
    print_info "ArgoCD installed successfully!"
}

# Get ArgoCD password
get_argocd_password() {
    print_info "Retrieving ArgoCD admin password..."
    
    PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
    
    echo ""
    echo "╔════════════════════════════════════════════╗"
    echo "║       ArgoCD Credentials                   ║"
    echo "╚════════════════════════════════════════════╝"
    echo "Username: admin"
    echo "Password: $PASSWORD"
    echo ""
    echo "Access ArgoCD UI:"
    echo "  kubectl port-forward svc/argocd-server -n argocd 8080:443"
    echo "  Then visit: https://localhost:8080"
    echo ""
}

# Create GitOps repository structure
create_repo_structure() {
    print_info "Creating GitOps repository structure..."
    
    REPO_DIR="gitops-repo"
    
    if [ -d "$REPO_DIR" ]; then
        print_warn "Directory $REPO_DIR already exists. Skipping creation."
        return
    fi
    
    mkdir -p "$REPO_DIR"
    cd "$REPO_DIR"
    
    # Initialize git
    git init
    
    # Create directory structure
    mkdir -p apps/sample-app/{base,overlays/{dev,staging,production}}
    mkdir -p infrastructure/{namespaces,monitoring,ingress}
    mkdir -p clusters/{dev,staging,production}
    
    # Create base deployment
    cat > apps/sample-app/base/deployment.yaml << 'EOF'
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
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
EOF

    # Create base service
    cat > apps/sample-app/base/service.yaml << 'EOF'
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
  type: ClusterIP
EOF

    # Create base kustomization
    cat > apps/sample-app/base/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
EOF

    # Create production overlay
    cat > apps/sample-app/overlays/production/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
nameSuffix: -prod
namespace: production
replicas:
  - name: sample-app
    count: 3
EOF

    # Create dev overlay
    cat > apps/sample-app/overlays/dev/kustomization.yaml << 'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
bases:
  - ../../base
nameSuffix: -dev
namespace: dev
replicas:
  - name: sample-app
    count: 1
EOF

    # Create namespace manifests
    cat > infrastructure/namespaces/namespaces.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: dev
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
EOF

    # Create ArgoCD application for production
    cat > clusters/production/sample-app.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sample-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/gitops-repo
    targetRevision: HEAD
    path: apps/sample-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF

    # Create README
    cat > README.md << 'EOF'
# GitOps Repository

This repository contains Kubernetes manifests managed via GitOps principles.

## Structure

- `apps/` - Application manifests
- `infrastructure/` - Infrastructure components
- `clusters/` - Cluster-specific configurations

## Usage

1. Make changes to manifests
2. Commit and push to Git
3. ArgoCD will automatically sync changes to the cluster

## Applications

- **sample-app**: Demo nginx application

EOF

    # Initial commit
    git add .
    git commit -m "Initial GitOps repository structure"
    
    cd ..
    
    print_info "GitOps repository structure created in $REPO_DIR"
    echo ""
    print_warn "Next steps:"
    echo "  1. Create a GitHub repository"
    echo "  2. Update the repoURL in clusters/production/sample-app.yaml"
    echo "  3. Push this repository to GitHub"
    echo "  4. Apply the ArgoCD application manifest"
    echo ""
}

# Create namespaces
create_namespaces() {
    print_info "Creating namespaces..."
    
    kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -
    kubectl create namespace staging --dry-run=client -o yaml | kubectl apply -f -
    kubectl create namespace production --dry-run=client -o yaml | kubectl apply -f -
    
    print_info "Namespaces created!"
}

# Install ArgoCD CLI (optional)
install_argocd_cli() {
    print_info "Would you like to install ArgoCD CLI? (y/n)"
    read -r INSTALL_CLI
    
    if [ "$INSTALL_CLI" = "y" ]; then
        if [[ "$OSTYPE" == "linux-gnu"* ]]; then
            curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
            sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
            rm argocd-linux-amd64
            print_info "ArgoCD CLI installed!"
        elif [[ "$OSTYPE" == "darwin"* ]]; then
            brew install argocd
            print_info "ArgoCD CLI installed!"
        else
            print_warn "Automatic installation not supported for your OS. Please install manually."
        fi
    fi
}

# Main execution
main() {
    echo ""
    print_info "Starting GitOps setup..."
    echo ""
    
    check_prerequisites
    install_argocd
    create_namespaces
    get_argocd_password
    create_repo_structure
    install_argocd_cli
    
    echo ""
    echo "╔════════════════════════════════════════════╗"
    echo "║      Setup Complete!                       ║"
    echo "╚════════════════════════════════════════════╝"
    echo ""
    print_info "GitOps environment is ready!"
    echo ""
    echo "Quick Start:"
    echo "  1. Access ArgoCD UI: kubectl port-forward svc/argocd-server -n argocd 8080:443"
    echo "  2. Visit: https://localhost:8080"
    echo "  3. Login with the credentials shown above"
    echo "  4. Push your gitops-repo to GitHub"
    echo "  5. Create an application in ArgoCD pointing to your repo"
    echo ""
}

# Run main function
main
