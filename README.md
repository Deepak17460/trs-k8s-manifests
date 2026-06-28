# Kubernetes GitOps Repository for TRS Application

This repository contains Kubernetes manifests and Helm charts for deploying the Task Resource Scheduler (TRS) application in a GitOps workflow using ArgoCD.

## Repository Structure

```
k8s-manifests/
├── manifests/
│   ├── poc/                    # TRS application manifests for POC environment
│   │   ├── trs-namespace.yaml     # Namespace definition
│   │   ├── trs-sa.yaml            # Service account
│   │   ├── trs-configmap.yaml     # Application configuration
│   │   ├── trs-secret-store.yaml  # External secrets store config
│   │   ├── trs-external-secret.yaml # AWS secrets integration
│   │   ├── trs-backend.yaml       # Backend deployment & service
│   │   ├── trs-frontend.yaml      # Frontend deployment & service
│   │   └── trs-ingress.yaml       # Ingress configuration
│   ├── monitoring/             # Monitoring stack configuration
│   │   └── kube-prometheus-stack-values.yaml
│   └── default/               # Default namespace resources
│       ├── nginx-configmap.yaml   # NGINX configuration
│       ├── nginx.yaml             # NGINX deployment & service
│       └── nginx-hpa.yaml         # Horizontal pod autoscaler
└── helm/
    └── trs-chart/             # Helm chart for TRS application
        ├── Chart.yaml             # Chart metadata
        ├── values.yaml            # Default configuration values
        ├── NOTES.txt              # Post-install notes
        └── templates/             # Helm templates
            ├── namespace.yaml
            ├── serviceaccount.yaml
            ├── configmap.yaml
            ├── secret-store.yaml
            ├── external-secret.yaml
            ├── backend-deployment.yaml
            ├── backend-service.yaml
            ├── frontend-deployment.yaml
            ├── frontend-service.yaml
            └── ingress.yaml
```

## Prerequisites

Before deploying, ensure you have the following tools installed:

- **kubectl**: Kubernetes command-line tool
- **helm**: Helm package manager for Kubernetes
- **argocd CLI**: ArgoCD command-line interface

### Installation Commands

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
```

## Manual Deployment with kubectl

Deploy the TRS application in the correct order to ensure dependencies are satisfied:

### Step 1: Deploy Core Resources
```bash
# Create namespace first
kubectl apply -f manifests/poc/trs-namespace.yaml

# Create service account
kubectl apply -f manifests/poc/trs-sa.yaml

# Create configuration
kubectl apply -f manifests/poc/trs-configmap.yaml
```

### Step 2: Setup External Secrets
```bash
# Create secret store (requires External Secrets Operator)
kubectl apply -f manifests/poc/trs-secret-store.yaml

# Create external secret
kubectl apply -f manifests/poc/trs-external-secret.yaml
```

### Step 3: Deploy Applications
```bash
# Deploy backend
kubectl apply -f manifests/poc/trs-backend.yaml

# Deploy frontend
kubectl apply -f manifests/poc/trs-frontend.yaml

# Setup ingress
kubectl apply -f manifests/poc/trs-ingress.yaml
```

### Step 4: Deploy Default Resources (Optional)
```bash
kubectl apply -f manifests/default/nginx-configmap.yaml
kubectl apply -f manifests/default/nginx.yaml
kubectl apply -f manifests/default/nginx-hpa.yaml
```

## Deploy with Helm

Deploy the TRS application using the Helm chart:

```bash
# Install the TRS application
helm install trs-app ./helm/trs-chart -n poc --create-namespace

# Upgrade existing deployment
helm upgrade trs-app ./helm/trs-chart -n poc

# Uninstall
helm uninstall trs-app -n poc
```

### Custom Values

Create a custom values file to override defaults:

```bash
# Create custom values
cat > custom-values.yaml << EOF
backend:
  replicas: 2
  resources:
    limits:
      cpu: 1000m
      memory: 512Mi

frontend:
  replicas: 2
  
ingress:
  host: trs.production.local
EOF

# Deploy with custom values
helm install trs-app ./helm/trs-chart -n poc -f custom-values.yaml
```

## Deploy Monitoring

Install the Prometheus monitoring stack:

```bash
# Add Prometheus community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack with custom values
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f manifests/monitoring/kube-prometheus-stack-values.yaml

# Access Grafana (default admin/admin123)
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
```

## GitOps with ArgoCD

### ArgoCD Application Configuration

Create an ArgoCD Application to manage the TRS deployment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trs-poc
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/dpcode72/trs-k8s-manifests
    targetRevision: HEAD
    path: manifests/poc
  destination:
    server: https://kubernetes.default.svc
    namespace: poc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Helm-based ArgoCD Application

For Helm chart deployment:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trs-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/dpcode72/trs-k8s-manifests
    targetRevision: HEAD
    path: helm/trs-chart
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: poc
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Deploy ArgoCD Applications

```bash
# Apply the ArgoCD application
kubectl apply -f argocd-application.yaml

# Sync the application
argocd app sync trs-poc

# Check application status
argocd app get trs-poc
```

## Useful Commands

### Monitoring and Troubleshooting

```bash
# Get all resources in poc namespace
kubectl get all -n poc

# Check pod resource usage
kubectl top pods -n poc

# View backend logs
kubectl logs -l app=trs-backend -n poc -f

# View frontend logs  
kubectl logs -l app=trs-frontend -n poc -f

# Describe a problematic pod
kubectl describe pod <pod-name> -n poc

# Get events in namespace
kubectl get events -n poc --sort-by='.lastTimestamp'

# Port forward to frontend for local testing
kubectl port-forward svc/trs-frontend-service 3000:80 -n poc --address 0.0.0.0

# Port forward to backend for local testing
kubectl port-forward svc/trs-backend-service 5000:5000 -n poc --address 0.0.0.0
```

### Scaling Operations

```bash
# Scale backend deployment
kubectl scale deployment trs-backend -n poc --replicas=3

# Scale frontend deployment
kubectl scale deployment trs-frontend -n poc --replicas=2

# Check horizontal pod autoscaler (for nginx in default namespace)
kubectl get hpa -n default
```

### Secret Management

```bash
# Check external secret status
kubectl get externalsecret -n poc

# Check secret store status
kubectl get secretstore -n poc

# View created secrets (values will be base64 encoded)
kubectl get secret trs-secret -n poc -o yaml
```

### Health Checks

```bash
# Check ingress status
kubectl get ingress -n poc

# Test application endpoints (if ingress is configured)
curl -H "Host: trs.local" http://<ingress-controller-ip>/
curl -H "Host: trs.local" http://<ingress-controller-ip>/api/v1/tasks

# Check service endpoints
kubectl get endpoints -n poc
```

### ArgoCD Operations

```bash
# Login to ArgoCD
argocd login <argocd-server>

# List applications
argocd app list

# Sync application
argocd app sync trs-poc

# Get application details
argocd app get trs-poc

# View application logs
argocd app logs trs-poc

# Rollback application
argocd app rollback trs-poc
```

## Application URLs

Once deployed and configured:

- **Frontend**: http://trs.local
- **Backend API**: http://trs.local/api/v1
- **NGINX (default)**: http://nginx.local
- **Grafana**: http://localhost:3000 (via port-forward)

## Notes

1. **External Secrets**: Requires External Secrets Operator to be installed in the cluster
2. **Ingress**: Requires NGINX Ingress Controller to be deployed
3. **Monitoring**: Grafana default credentials are admin/admin123
4. **AWS Integration**: Service account needs proper IAM roles for AWS Secrets Manager access
5. **DNS**: Add entries to `/etc/hosts` for local testing:
   ```
   <ingress-ip> trs.local nginx.local
   ```