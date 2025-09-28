# GitOps Management Guide

This guide explains how to configure and manage your repository-of-repositories (RoR) structure, format deployment files, and organize application repositories for GitOps with ArgoCD on your Raspberry Pi Kubernetes cluster.

## Table of Contents

1. [Quick Overview](#quick-overview)
2. [Initial Setup and Configuration](#initial-setup-and-configuration)
3. [Repository-of-Repositories Configuration](#repository-of-repositories-configuration)
4. [Application Repository Structure](#application-repository-structure)
5. [ArgoCD Application Management](#argocd-application-management)
6. [Authentication and Access Control](#authentication-and-access-control)
7. [Sync Policies and Configuration](#sync-policies-and-configuration)
8. [Environment Management](#environment-management)
9. [Best Practices](#best-practices)
10. [Troubleshooting](#troubleshooting)

## Quick Overview

This GitOps setup uses:
- **Repository-of-Repositories (RoR)**: Central Git repository that references all application repositories
- **ArgoCD**: Manages deployments by watching the RoR and application repositories
- **App-of-Apps Pattern**: Single ArgoCD application that manages all other applications
- **Git**: Single source of truth for all cluster state

The ansible deployment automatically configures ArgoCD with an "app-of-apps" pattern, where a single application in your RoR manages all other applications in the cluster.

## Initial Setup and Configuration

### ArgoCD Access

After the ansible deployment completes, ArgoCD is accessible at:
- **URL**: `http://any-pi-ip:30080` (HTTP) or `http://any-pi-ip:30443` (HTTPS)
- **Username**: `admin`
- **Password**: Set in `secrets.yaml` as `argocd_admin_password` (default: "change-me")

**Note**: ArgoCD is automatically installed during the K3s server deployment via the `firstboot-ops.service`, not through a separate ArgoCD role. The `argocd_apps` role then configures the applications and access.

### Required Secrets Configuration

Before deploying applications, configure these secrets in your `secrets.yaml`:

```yaml
# ArgoCD Admin Access
argocd_admin_password: "your-secure-password"

# GitHub Access (for private repositories)
github_username: "your-github-username"
github_token: "your-github-personal-access-token"

# Repository-of-Repositories Configuration
argocd_apps_repo_url: "https://github.com/your-username/your-argocd-apps.git"
argocd_apps_repo_branch: "main"
argocd_apps_repo_path: "."

# Docker Registry Access (optional - for private images)
docker_registry_secret:
  url: "https://index.docker.io/v1/"
  username: "your-docker-username"
  password: "your-docker-password"
```

## Repository-of-Repositories Configuration

The Repository-of-Repositories (RoR) is your central control hub that ArgoCD monitors to discover and manage all applications in your cluster.

### App-of-Apps Pattern

Your RoR uses the "App-of-Apps" pattern configured in the ansible deployment:

```yaml
# From group_vars/all.yaml
argocd_applications:
  - name: app-of-apps
    repo_url: "{{ argocd_apps_repo_url }}"
    branch: "{{ argocd_apps_repo_branch }}"
    path: "{{ argocd_apps_repo_path }}"
    namespace: argocd
    auto_sync: true
    auto_prune: true
    directory_recurse: true
```

### RoR Repository Structure

Your Repository-of-Repositories should be organized as follows:

```
your-argocd-apps/
├── README.md
├── apps/
│   ├── production/
│   │   ├── web-app.yaml
│   │   ├── database.yaml
│   │   └── monitoring.yaml
│   ├── development/
│   │   ├── web-app.yaml
│   │   └── test-services.yaml
│   └── infrastructure/
│       ├── ingress-controller.yaml
│       ├── cert-manager.yaml
│       └── storage-classes.yaml
├── projects/
│   └── default.yaml
└── repositories/
    ├── github-repos.yaml
    └── helm-repos.yaml
```

### Creating Your App-of-Apps Repository

1. **Create a new GitHub repository** for your ArgoCD applications
2. **Initialize the basic structure**:

```bash
mkdir your-argocd-apps
cd your-argocd-apps
git init
mkdir -p apps/{production,development,infrastructure}
mkdir -p {projects,repositories}
```

3. **Create the root application manifest** (`apps/app-of-apps.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/your-argocd-apps.git
    targetRevision: HEAD
    path: apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Application Repository Structure

Individual application repositories should follow these patterns:

### Standard Kubernetes Application

```
my-web-app/
├── README.md
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── ingress.yaml
├── docker/
│   └── Dockerfile
└── .github/
    └── workflows/
        └── ci-cd.yaml
```

### Helm Chart Application

```
my-helm-app/
├── Chart.yaml
├── values.yaml
├── values-production.yaml
├── values-development.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   └── ingress.yaml
└── charts/
    └── (dependencies)
```

## ArgoCD Application Management

### Creating Applications via RoR

Add applications to your RoR by creating application manifests in the appropriate environment folder:

**Example: Web Application (`apps/production/web-app.yaml`)**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/web-app.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: web-app-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

**Example: Helm Application (`apps/production/helm-app.yaml`)**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-app-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/helm-app.git
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - values-production.yaml
      values: |
        image:
          tag: "v1.2.3"
        service:
          type: NodePort
          nodePort: 31000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  destination:
    server: https://kubernetes.default.svc
    namespace: helm-app-prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Authentication and Access Control

### Repository Access Configuration

The ansible deployment creates repository secrets for authentication:

**GitHub Repository Access** (automatically configured if `github_token` is provided):

```yaml
# Created by ansible from github-repo-secret.yaml.j2
apiVersion: v1
kind: Secret
metadata:
  name: github-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
type: Opaque
stringData:
  type: git
  url: https://github.com
  password: your-github-token
  username: your-github-username
```

**Docker Registry Access** (optional, for private images):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: docker-registry
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository  
type: Opaque
stringData:
  type: helm
  name: docker-registry
  url: https://index.docker.io/v1/
  username: your-docker-username
  password: your-docker-password
```

### Setting Up Private Repository Access

1. **Generate a GitHub Personal Access Token** with repo permissions
2. **Configure in your secrets.yaml**:

```yaml
github_username: "your-github-username"
github_token: "ghp_your-personal-access-token"
```

3. **Re-run the ansible playbook** to apply the new secrets

## Sync Policies and Configuration

### Automatic Sync Configuration

The ansible deployment template supports these sync policy options:

```yaml
syncPolicy:
  automated:
    prune: true          # Remove resources not in Git
    selfHeal: true       # Revert manual changes
    allowEmpty: false    # Don't sync empty repositories
  syncOptions:
    - Validate=false              # Skip kubectl validation
    - CreateNamespace=true        # Auto-create target namespaces
    - PrunePropagationPolicy=foreground  # How to handle resource deletion
    - PruneLast=true             # Delete resources after successful sync
  retry:
    limit: 5                     # Number of retry attempts
    backoff:
      duration: 5s               # Initial retry delay
      factor: 2                  # Backoff multiplier
      maxDuration: 3m            # Maximum retry delay
```

### Manual Sync Applications

For sensitive applications, disable automatic sync:

```yaml
spec:
  syncPolicy:
    # Remove automated section for manual sync
    syncOptions:
      - CreateNamespace=true
```

### Sync Waves and Dependencies

Use sync waves to control deployment order:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"  # Deploy in wave 1
```

**Example Order**:
- Wave 0: Namespaces, RBAC
- Wave 1: ConfigMaps, Secrets
- Wave 2: Services, PVCs
- Wave 3: Deployments, StatefulSets
- Wave 4: Ingress, NetworkPolicies

## Environment Management

### Multi-Environment Setup

Organize applications by environment in your RoR:

```
apps/
├── infrastructure/     # Shared infrastructure
│   ├── cert-manager.yaml
│   └── ingress-controller.yaml
├── production/         # Production applications
│   ├── web-app.yaml
│   └── database.yaml
├── staging/           # Staging applications
│   ├── web-app.yaml
│   └── database.yaml
└── development/       # Development applications
    ├── web-app.yaml
    └── test-tools.yaml
```

### Environment-Specific Values

**Using Helm with different values files**:

```yaml
# Production application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-username/helm-app.git
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-prod

---
# Development application  
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-development
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-username/helm-app.git
    targetRevision: main
    path: .
    helm:
      valueFiles:
        - values-development.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-dev
```

**Using different Git branches**:

```yaml
# Production application using main branch
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-production
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-username/my-app.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-prod

---
# Development application using develop branch
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-development
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/your-username/my-app.git
    targetRevision: develop
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app-dev
```

## Best Practices

### Repository Organization
1. **Separate application code from deployment manifests**
2. **Use semantic versioning for application releases**
3. **Keep environment-specific configurations in separate files**
4. **Use descriptive names for applications and namespaces**

### Security
1. **Use least-privilege RBAC policies**
2. **Store sensitive data in Kubernetes secrets, not in Git**
3. **Use private repositories for proprietary applications**
4. **Regularly rotate access tokens and passwords**

### Deployment Strategy
1. **Start with manual sync for new applications**
2. **Enable automatic sync after testing**
3. **Use sync waves for complex dependencies**
4. **Monitor application health and sync status**

### ARM64 Considerations
1. **Ensure container images support ARM64 architecture**
2. **Use multi-architecture images when possible**
3. **Consider resource constraints on Raspberry Pi nodes**
4. **Test applications on ARM64 before production deployment**

## Troubleshooting

### Common Issues

**Application Stuck in Sync State**:
```bash
# Check application status
kubectl get applications -n argocd

# Get detailed application information
kubectl describe application <app-name> -n argocd

# Check ArgoCD server logs
kubectl logs -n argocd deployment/argocd-server
```

**Repository Access Issues**:
```bash
# Check repository secrets
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository

# Test repository access
kubectl logs -n argocd deployment/argocd-repo-server
```

**Image Pull Errors on ARM64**:
1. Verify image supports ARM64 architecture
2. Check for multi-architecture manifest
3. Use ARM64-specific image tags if needed

### Useful ArgoCD CLI Commands

```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-arm64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-arm64
sudo install -m 555 argocd-linux-arm64 /usr/local/bin/argocd

# Login to ArgoCD
argocd login <pi-node-ip>:30080

# List applications
argocd app list

# Sync application manually
argocd app sync <app-name>

# Get application details
argocd app get <app-name>
```

### Log Locations

- **ArgoCD Server**: `kubectl logs -n argocd deployment/argocd-server`
- **ArgoCD Application Controller**: `kubectl logs -n argocd deployment/argocd-application-controller`
- **ArgoCD Repository Server**: `kubectl logs -n argocd deployment/argocd-repo-server`

This guide provides a complete framework for managing your Raspberry Pi Kubernetes cluster using GitOps principles with ArgoCD. The ansible deployment handles the initial setup, and this guide helps you configure and manage applications going forward.
