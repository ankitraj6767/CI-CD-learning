# Module 14: GitOps & ArgoCD

## Learning Objectives

By the end of this module, you will be able to:

- Understand GitOps principles and methodology
- Install and configure ArgoCD
- Implement GitOps workflows for application deployment
- Manage multi-environment deployments with ArgoCD
- Configure automated sync policies and rollback strategies
- Implement progressive delivery with ArgoCD
- Monitor and troubleshoot GitOps deployments
- Integrate ArgoCD with CI/CD pipelines

---

## 1. GitOps Fundamentals {#fundamentals}

### What is GitOps?

GitOps is a declarative approach to continuous deployment that uses Git as the single source of truth for infrastructure and application configuration.

#### Core Principles

1. **Declarative**: The entire system is described declaratively
2. **Versioned and Immutable**: The canonical desired system state is versioned in Git
3. **Pulled Automatically**: Software agents automatically pull the desired state declarations from the source
4. **Continuously Reconciled**: Software agents continuously observe actual system state and attempt to apply the desired state

```mermaid
graph LR
    DEV[Developer] --> GIT[Git Repository]
    GIT --> ARGOCD[ArgoCD]
    ARGOCD --> K8S[Kubernetes Cluster]
    K8S --> MONITOR[Monitoring]
    MONITOR --> ALERT[Alerts]
    ALERT --> DEV
    
    subgraph "GitOps Flow"
        GIT
        ARGOCD
        K8S
    end
```

### GitOps vs Traditional CI/CD

| Traditional CI/CD | GitOps |
|-------------------|--------|
| Push-based deployment | Pull-based deployment |
| CI system has cluster access | No cluster access from CI |
| Imperative commands | Declarative configuration |
| Limited audit trail | Complete Git history |
| Manual rollback | Automatic rollback |

---

## 2. ArgoCD Installation and Setup {#argocd-setup}

### Installing ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### ArgoCD Configuration

```yaml
# argocd/argocd-server-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-server-config
  namespace: argocd
data:
  # Enable insecure mode for development
  server.insecure: "true"
  
  # Configure OIDC
  oidc.config: |
    name: GitHub
    issuer: https://github.com
    clientId: $oidc.github.clientId
    clientSecret: $oidc.github.clientSecret
    requestedScopes: ["user:email"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
  
  # Configure repositories
  repositories: |
    - type: git
      url: https://github.com/myorg/k8s-manifests
    - type: git
      url: https://github.com/myorg/helm-charts
      name: helm-charts
  
  # Configure resource customizations
  resource.customizations: |
    networking.k8s.io/Ingress:
      health.lua: |
        hs = {}
        hs.status = "Healthy"
        return hs

---
# argocd/argocd-rbac-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    # Admin role
    p, role:admin, applications, *, */*, allow
    p, role:admin, clusters, *, *, allow
    p, role:admin, repositories, *, *, allow
    
    # Developer role
    p, role:developer, applications, get, */*, allow
    p, role:developer, applications, sync, */*, allow
    p, role:developer, applications, action/*, */*, allow
    
    # Read-only role
    p, role:readonly, applications, get, */*, allow
    p, role:readonly, clusters, get, *, allow
    
    # Group mappings
    g, myorg:platform-team, role:admin
    g, myorg:developers, role:developer
    g, myorg:viewers, role:readonly
```

### ArgoCD CLI Setup

```bash
# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

# Login to ArgoCD
argocd login localhost:8080 --username admin --password <initial-password>

# Change admin password
argocd account update-password

# List applications
argocd app list

# Get application details
argocd app get myapp

# Sync application
argocd app sync myapp
```

---

## 3. Application Deployment with ArgoCD {#application-deployment}

### Basic Application Configuration

```yaml
# apps/web-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: HEAD
    path: apps/web-app/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
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
  
  revisionHistoryLimit: 10
```

### Kustomize-based Application

```yaml
# Repository structure:
# k8s-manifests/
# ├── apps/
# │   └── web-app/
# │       ├── base/
# │       │   ├── deployment.yaml
# │       │   ├── service.yaml
# │       │   ├── configmap.yaml
# │       │   └── kustomization.yaml
# │       └── overlays/
# │           ├── staging/
# │           │   ├── kustomization.yaml
# │           │   └── patches/
# │           └── production/
# │               ├── kustomization.yaml
# │               └── patches/

# apps/web-app/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- configmap.yaml

commonLabels:
  app: web-app
  version: v1.0.0

images:
- name: web-app
  newTag: latest

# apps/web-app/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../../base

patchesStrategicMerge:
- patches/deployment-patch.yaml
- patches/service-patch.yaml

replicas:
- name: web-app-deployment
  count: 5

images:
- name: web-app
  newTag: v1.2.3
```

### Helm-based Application

```yaml
# apps/helm-web-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: helm-web-app
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/helm-charts
    targetRevision: HEAD
    path: web-app
    helm:
      valueFiles:
      - values-production.yaml
      parameters:
      - name: image.tag
        value: v1.2.3
      - name: replicaCount
        value: "5"
      - name: ingress.enabled
        value: "true"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Multi-Source Application

```yaml
# apps/multi-source-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: multi-source-app
  namespace: argocd
spec:
  project: default
  
  sources:
  # Helm chart source
  - repoURL: https://charts.bitnami.com/bitnami
    chart: postgresql
    targetRevision: 12.1.2
    helm:
      valueFiles:
      - $values/postgresql/values-production.yaml
  
  # Values repository
  - repoURL: https://github.com/myorg/helm-values
    targetRevision: HEAD
    ref: values
  
  # Application manifests
  - repoURL: https://github.com/myorg/k8s-manifests
    targetRevision: HEAD
    path: apps/web-app/overlays/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 4. Multi-Environment Management {#multi-environment}

### App of Apps Pattern

```yaml
# apps/app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  
  source:
    repoURL: https://github.com/myorg/argocd-apps
    targetRevision: HEAD
    path: environments/production
  
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# environments/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- web-app.yaml
- api-service.yaml
- database.yaml
- monitoring.yaml

commonLabels:
  environment: production
  managed-by: argocd
```

### ApplicationSet for Multi-Environment

```yaml
# appsets/web-app-environments.yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: web-app-environments
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - cluster: in-cluster
        url: https://kubernetes.default.svc
        environment: staging
        namespace: staging
        replicaCount: "2"
        imageTag: "staging-latest"
      - cluster: in-cluster
        url: https://kubernetes.default.svc
        environment: production
        namespace: production
        replicaCount: "5"
        imageTag: "v1.2.3"
  
  template:
    metadata:
      name: 'web-app-{{environment}}'
    spec:
      project: default
      
      source:
        repoURL: https://github.com/myorg/k8s-manifests
        targetRevision: HEAD
        path: 'apps/web-app/overlays/{{environment}}'
      
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

### Git Repository Structure

```
k8s-manifests/
├── apps/
│   ├── web-app/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── configmap.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── development/
│   │       │   ├── kustomization.yaml
│   │       │   └── patches/
│   │       ├── staging/
│   │       │   ├── kustomization.yaml
│   │       │   └── patches/
│   │       └── production/
│   │           ├── kustomization.yaml
│   │           └── patches/
│   └── api-service/
│       ├── base/
│       └── overlays/
├── infrastructure/
│   ├── monitoring/
│   ├── ingress/
│   └── security/
└── environments/
    ├── development/
    ├── staging/
    └── production/
```

---

## 5. Progressive Delivery {#progressive-delivery}

### Canary Deployment with Argo Rollouts

```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install Rollouts CLI
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x ./kubectl-argo-rollouts-linux-amd64
sudo mv ./kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

```yaml
# rollouts/canary-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web-app-rollout
  namespace: production
spec:
  replicas: 10
  strategy:
    canary:
      # Analysis run to perform in background
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 2
        args:
        - name: service-name
          value: web-app-service
      
      # Steps define sequence of rollout
      steps:
      - setWeight: 10
      - pause: {duration: 1m}
      - setWeight: 20
      - pause: {duration: 1m}
      - setWeight: 40
      - pause: {duration: 2m}
      - setWeight: 60
      - pause: {duration: 2m}
      - setWeight: 80
      - pause: {duration: 2m}
      
      # Traffic routing
      trafficRouting:
        istio:
          virtualService:
            name: web-app-vs
            routes:
            - primary
          destinationRule:
            name: web-app-dr
            canarySubsetName: canary
            stableSubsetName: stable
      
      # Automatic rollback
      abortScaleDownDelaySeconds: 30
      scaleDownDelaySeconds: 30
      scaleDownDelayRevisionLimit: 2
  
  selector:
    matchLabels:
      app: web-app
  
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myregistry/web-app:v1.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
# rollouts/analysis-template.yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: production
spec:
  args:
  - name: service-name
  
  metrics:
  - name: success-rate
    interval: 1m
    count: 5
    successCondition: result[0] >= 0.95
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status!~"5.."}[2m])) /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[2m]))
  
  - name: avg-response-time
    interval: 1m
    count: 5
    successCondition: result[0] <= 500
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus.monitoring.svc.cluster.local:9090
        query: |
          histogram_quantile(0.95,
            sum(rate(http_request_duration_seconds_bucket{service="{{args.service-name}}"}[2m])) by (le)
          ) * 1000
```

### Blue-Green Deployment

```yaml
# rollouts/blue-green-rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: web-app-bg-rollout
  namespace: production
spec:
  replicas: 5
  strategy:
    blueGreen:
      # Service that the rollout modifies as the active service
      activeService: web-app-active
      # Service that the rollout modifies as the preview service
      previewService: web-app-preview
      # Automatically promote the new version after 30 seconds
      autoPromotionEnabled: false
      # Time to wait before scaling down the old version
      scaleDownDelaySeconds: 30
      # Time to wait before scaling down the old version after promotion
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: web-app-preview
      # Analysis to run after promotion
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: web-app-active
  
  selector:
    matchLabels:
      app: web-app
  
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myregistry/web-app:v1.0.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

---

## 6. CI/CD Integration {#cicd-integration}

### GitHub Actions with ArgoCD

```yaml
# .github/workflows/gitops-deployment.yml
name: GitOps Deployment

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run tests
      run: npm test
    
    - name: Build application
      run: npm run build
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

  update-manifests:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout manifests repository
      uses: actions/checkout@v4
      with:
        repository: myorg/k8s-manifests
        token: ${{ secrets.GITOPS_TOKEN }}
        path: manifests
    
    - name: Update image tag in staging
      run: |
        cd manifests
        
        # Update staging overlay
        sed -i 's|newTag: .*|newTag: ${{ github.sha }}|' apps/web-app/overlays/staging/kustomization.yaml
        
        # Commit changes
        git config user.name "GitHub Actions"
        git config user.email "actions@github.com"
        git add .
        git commit -m "Update staging image to ${{ github.sha }}"
        git push
    
    - name: Wait for staging deployment
      run: |
        # Install ArgoCD CLI
        curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
        chmod +x argocd
        
        # Login to ArgoCD
        ./argocd login ${{ secrets.ARGOCD_SERVER }} --username ${{ secrets.ARGOCD_USERNAME }} --password ${{ secrets.ARGOCD_PASSWORD }} --insecure
        
        # Sync and wait for staging app
        ./argocd app sync web-app-staging
        ./argocd app wait web-app-staging --timeout 300
    
    - name: Run staging tests
      run: |
        # Run smoke tests against staging
        curl -f https://staging.myapp.com/health
        
        # Run integration tests
        npm run test:integration -- --env=staging
    
    - name: Promote to production
      if: success()
      run: |
        cd manifests
        
        # Update production overlay
        sed -i 's|newTag: .*|newTag: ${{ github.sha }}|' apps/web-app/overlays/production/kustomization.yaml
        
        # Commit changes
        git add .
        git commit -m "Promote ${{ github.sha }} to production"
        git push
    
    - name: Notify deployment
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        text: |
          Deployment Status: ${{ job.status }}
          Image: ${{ needs.build-and-test.outputs.image-tag }}
          Commit: ${{ github.sha }}
          Author: ${{ github.actor }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### ArgoCD Image Updater

```yaml
# Install ArgoCD Image Updater
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Configure Image Updater
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-image-updater-config
  namespace: argocd
data:
  registries.conf: |
    registries:
    - name: GitHub Container Registry
      api_url: https://ghcr.io
      prefix: ghcr.io
      credentials: secret:argocd/ghcr-creds#token
      default: true
  
  git.commit-message-template: |
    build: automatic update of {{ .AppName }}
    
    {{ range .AppChanges -}}
    updates image {{ .Image }} tag '{{ .OldTag }}' to '{{ .NewTag }}'
    {{ end -}}

---
# Annotate application for automatic updates
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: web-app=ghcr.io/myorg/web-app
    argocd-image-updater.argoproj.io/web-app.update-strategy: latest
    argocd-image-updater.argoproj.io/web-app.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ... rest of application spec
```

---

## 7. Monitoring and Observability {#monitoring}

### ArgoCD Metrics and Monitoring

```yaml
# monitoring/argocd-metrics.yaml
apiVersion: v1
kind: Service
metadata:
  name: argocd-metrics
  namespace: argocd
  labels:
    app.kubernetes.io/component: metrics
    app.kubernetes.io/name: argocd-metrics
    app.kubernetes.io/part-of: argocd
spec:
  ports:
  - name: metrics
    port: 8082
    protocol: TCP
    targetPort: 8082
  selector:
    app.kubernetes.io/name: argocd-application-controller

---
apiVersion: v1
kind: Service
metadata:
  name: argocd-server-metrics
  namespace: argocd
  labels:
    app.kubernetes.io/component: server
    app.kubernetes.io/name: argocd-server-metrics
    app.kubernetes.io/part-of: argocd
spec:
  ports:
  - name: metrics
    port: 8083
    protocol: TCP
    targetPort: 8083
  selector:
    app.kubernetes.io/name: argocd-server

---
apiVersion: v1
kind: Service
metadata:
  name: argocd-repo-server-metrics
  namespace: argocd
  labels:
    app.kubernetes.io/component: repo-server
    app.kubernetes.io/name: argocd-repo-server
    app.kubernetes.io/part-of: argocd
spec:
  ports:
  - name: metrics
    port: 8084
    protocol: TCP
    targetPort: 8084
  selector:
    app.kubernetes.io/name: argocd-repo-server
```

### Prometheus ServiceMonitor

```yaml
# monitoring/argocd-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-metrics
  namespace: argocd
  labels:
    app.kubernetes.io/component: metrics
    app.kubernetes.io/name: argocd-metrics
    app.kubernetes.io/part-of: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-server-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server-metrics
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics

---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-repo-server-metrics
  namespace: argocd
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "id": null,
    "title": "ArgoCD Dashboard",
    "tags": ["argocd", "gitops"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Application Sync Status",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(argocd_app_info{sync_status=\"Synced\"}) by (sync_status)",
            "legendFormat": "Synced"
          },
          {
            "expr": "sum(argocd_app_info{sync_status=\"OutOfSync\"}) by (sync_status)",
            "legendFormat": "Out of Sync"
          }
        ]
      },
      {
        "id": 2,
        "title": "Application Health Status",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(argocd_app_info{health_status=\"Healthy\"}) by (health_status)",
            "legendFormat": "Healthy"
          },
          {
            "expr": "sum(argocd_app_info{health_status=\"Degraded\"}) by (health_status)",
            "legendFormat": "Degraded"
          }
        ]
      },
      {
        "id": 3,
        "title": "Sync Operations",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(argocd_app_sync_total[5m])",
            "legendFormat": "{{name}} - {{phase}}"
          }
        ]
      },
      {
        "id": 4,
        "title": "Repository Sync Performance",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(argocd_git_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, rate(argocd_git_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

### ArgoCD Notifications

```yaml
# notifications/argocd-notifications-cm.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  
  service.email: |
    host: smtp.gmail.com
    port: 587
    from: $email-username
    username: $email-username
    password: $email-password
  
  template.app-deployed: |
    email:
      subject: New version of an application {{.app.metadata.name}} is up and running.
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} is now running new version.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          {{range $index, $c := .app.status.conditions}}
          ,{
            "title": "{{$c.type}}",
            "value": "{{$c.message}}",
            "short": true
          }
          {{end}}
          ]
        }]
  
  template.app-health-degraded: |
    email:
      subject: Application {{.app.metadata.name}} has degraded.
    message: |
      {{if eq .serviceType "slack"}}:exclamation:{{end}} Application {{.app.metadata.name}} has degraded.
      Application details: {{.context.argocdUrl}}/applications/{{.app.metadata.name}}.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link": "{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#f4c430",
          "fields": [
          {
            "title": "Health Status",
            "value": "{{.app.status.health.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          }
          {{range $index, $c := .app.status.conditions}}
          ,{
            "title": "{{$c.type}}",
            "value": "{{$c.message}}",
            "short": true
          }
          {{end}}
          ]
        }]
  
  trigger.on-deployed: |
    - description: Application is synced and healthy
      send:
      - app-deployed
      when: app.status.operationState.phase in ['Succeeded'] and app.status.health.status == 'Healthy'
  
  trigger.on-health-degraded: |
    - description: Application has degraded
      send:
      - app-health-degraded
      when: app.status.health.status == 'Degraded'
  
  subscriptions: |
    - recipients:
      - slack:deployments
      - email:team@company.com
      triggers:
      - on-deployed
      - on-health-degraded

---
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret
  namespace: argocd
type: Opaque
stringData:
  slack-token: xoxb-your-slack-bot-token
  email-username: your-email@gmail.com
  email-password: your-app-password
```

---

## 8. Troubleshooting {#troubleshooting}

### Common ArgoCD Issues

#### Application Sync Issues

```bash
# Check application status
argocd app get myapp

# Check sync status
argocd app sync myapp --dry-run

# Force sync
argocd app sync myapp --force

# Check application events
kubectl describe application myapp -n argocd

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-application-controller
kubectl logs -n argocd deployment/argocd-server
kubectl logs -n argocd deployment/argocd-repo-server
```

#### Repository Access Issues

```bash
# Test repository access
argocd repo get https://github.com/myorg/k8s-manifests

# Add repository with credentials
argocd repo add https://github.com/myorg/k8s-manifests --username myuser --password mytoken

# Check repository connection
kubectl get secret -n argocd -l argocd.argoproj.io/secret-type=repository
```

#### Performance Issues

```bash
# Check resource usage
kubectl top pods -n argocd

# Scale ArgoCD components
kubectl scale deployment argocd-application-controller --replicas=2 -n argocd
kubectl scale deployment argocd-repo-server --replicas=3 -n argocd

# Check metrics
curl http://argocd-metrics.argocd.svc.cluster.local:8082/metrics
```

### Debugging GitOps Workflows

```bash
# Check Git repository status
git status
git log --oneline -10

# Validate Kubernetes manifests
kubectl apply --dry-run=client -f manifests/

# Check Kustomize output
kustomize build apps/web-app/overlays/production

# Validate Helm charts
helm template web-app charts/web-app --values values-production.yaml
helm lint charts/web-app
```

---

## 9. Best Practices {#best-practices}

### Repository Structure

1. **Separate application code from configuration**
2. **Use environment-specific overlays**
3. **Implement proper branching strategy**
4. **Use semantic versioning for releases**
5. **Maintain clear directory structure**

### Security Best Practices

1. **Use RBAC for access control**
2. **Implement proper secret management**
3. **Enable audit logging**
4. **Use signed commits**
5. **Implement policy enforcement**

### Operational Best Practices

1. **Monitor application and ArgoCD health**
2. **Implement proper alerting**
3. **Use progressive delivery strategies**
4. **Maintain disaster recovery procedures**
5. **Regular backup of ArgoCD configuration**

---

## 10. Hands-on Exercises {#exercises}

### Exercise 1: Basic GitOps Setup

**Objective**: Set up ArgoCD and deploy a simple application using GitOps.

**Tasks**:
1. Install ArgoCD on a Kubernetes cluster
2. Create a Git repository with Kubernetes manifests
3. Configure ArgoCD application
4. Deploy and verify the application
5. Make changes and observe automatic sync

### Exercise 2: Multi-Environment Deployment

**Objective**: Implement multi-environment deployment using ApplicationSets.

**Tasks**:
1. Create environment-specific overlays
2. Set up ApplicationSet for multiple environments
3. Implement promotion workflow
4. Test deployment across environments
5. Configure environment-specific policies

### Exercise 3: Progressive Delivery

**Objective**: Implement canary deployment with Argo Rollouts.

**Tasks**:
1. Install Argo Rollouts
2. Create rollout configuration
3. Set up analysis templates
4. Deploy using canary strategy
5. Monitor and validate deployment

### Exercise 4: CI/CD Integration

**Objective**: Integrate GitOps with CI/CD pipeline.

**Tasks**:
1. Set up GitHub Actions workflow
2. Implement image building and pushing
3. Configure manifest updates
4. Set up ArgoCD notifications
5. Test end-to-end workflow

---

This completes Module 14: GitOps & ArgoCD. You now have comprehensive knowledge of GitOps principles, ArgoCD configuration, progressive delivery, and CI/CD integration for modern application deployment workflows.