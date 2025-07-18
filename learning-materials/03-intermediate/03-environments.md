# Module 11: Environment Management 🌍

## Learning Objectives 🎯

By the end of this module, you will:
- Design and implement multi-environment CI/CD strategies
- Manage environment-specific configurations and secrets
- Implement environment promotion workflows
- Set up environment isolation and security
- Configure environment-specific testing and validation
- Implement Infrastructure as Code for environments
- Manage environment dependencies and data
- Monitor and maintain environment health

## Table of Contents

1. [Introduction to Environment Management](#introduction)
2. [Environment Strategy & Design](#strategy)
3. [Environment Configuration Management](#configuration)
4. [Environment Promotion Workflows](#promotion)
5. [Infrastructure as Code for Environments](#infrastructure)
6. [Environment Security & Isolation](#security)
7. [Environment Testing & Validation](#testing)
8. [Data Management Across Environments](#data)
9. [Environment Monitoring & Observability](#monitoring)
10. [Best Practices](#best-practices)
11. [Hands-on Exercises](#exercises)
12. [Troubleshooting](#troubleshooting)

---

## 1. Introduction to Environment Management {#introduction}

### What is Environment Management?

Environment management is the practice of creating, maintaining, and orchestrating multiple isolated environments throughout the software development lifecycle.

### Why Environment Management Matters

- 🔒 **Risk Reduction**: Test changes before production
- 🚀 **Faster Delivery**: Parallel development and testing
- 🎯 **Quality Assurance**: Validate in production-like environments
- 🔄 **Rollback Safety**: Quick recovery from issues
- 👥 **Team Collaboration**: Isolated workspaces for teams

### Common Environment Types

```mermaid
graph LR
    A[Development] --> B[Feature]
    B --> C[Integration]
    C --> D[Staging]
    D --> E[Pre-Production]
    E --> F[Production]
    
    G[Hotfix] --> D
    H[Demo] --> D
```

### Environment Characteristics

| Environment | Purpose | Data | Traffic | Monitoring |
|-------------|---------|------|---------|------------|
| Development | Active development | Synthetic | Internal | Basic |
| Feature | Feature testing | Synthetic | Internal | Basic |
| Integration | Integration testing | Synthetic | Internal | Enhanced |
| Staging | Pre-production testing | Production-like | Internal | Full |
| Pre-Production | Final validation | Production copy | Limited | Full |
| Production | Live system | Real | Public | Comprehensive |

---

## 2. Environment Strategy & Design {#strategy}

### Environment Branching Strategy

```yaml
# .github/workflows/environment-strategy.yml
name: Environment Strategy

on:
  push:
    branches: [ main, develop, 'feature/*', 'hotfix/*' ]
  pull_request:
    branches: [ main, develop ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  determine-environment:
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.env.outputs.environment }}
      deploy: ${{ steps.env.outputs.deploy }}
    
    steps:
    - name: Determine environment
      id: env
      run: |
        if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
          echo "environment=production" >> $GITHUB_OUTPUT
          echo "deploy=true" >> $GITHUB_OUTPUT
        elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
          echo "environment=staging" >> $GITHUB_OUTPUT
          echo "deploy=true" >> $GITHUB_OUTPUT
        elif [[ "${{ github.ref }}" == refs/heads/feature/* ]]; then
          FEATURE_NAME=$(echo ${{ github.ref }} | sed 's/refs\/heads\/feature\///g' | sed 's/[^a-zA-Z0-9]/-/g')
          echo "environment=feature-$FEATURE_NAME" >> $GITHUB_OUTPUT
          echo "deploy=true" >> $GITHUB_OUTPUT
        elif [[ "${{ github.ref }}" == refs/heads/hotfix/* ]]; then
          echo "environment=hotfix" >> $GITHUB_OUTPUT
          echo "deploy=true" >> $GITHUB_OUTPUT
        else
          echo "environment=review" >> $GITHUB_OUTPUT
          echo "deploy=false" >> $GITHUB_OUTPUT
        fi
  
  build:
    needs: determine-environment
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Log in to registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.determine-environment.outputs.environment }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
  
  deploy:
    needs: [determine-environment, build]
    if: needs.determine-environment.outputs.deploy == 'true'
    runs-on: ubuntu-latest
    environment: ${{ needs.determine-environment.outputs.environment }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to ${{ needs.determine-environment.outputs.environment }}
      run: |
        echo "Deploying to ${{ needs.determine-environment.outputs.environment }}"
        # Environment-specific deployment logic here
```

### Dynamic Environment Creation

```yaml
# .github/workflows/dynamic-environments.yml
name: Dynamic Environment Creation

on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches: [ main ]

jobs:
  create-environment:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Generate environment name
      id: env-name
      run: |
        ENV_NAME="pr-${{ github.event.number }}-$(echo ${{ github.head_ref }} | sed 's/[^a-zA-Z0-9]/-/g' | cut -c1-20)"
        echo "name=$ENV_NAME" >> $GITHUB_OUTPUT
        echo "url=https://$ENV_NAME.preview.myapp.com" >> $GITHUB_OUTPUT
    
    - name: Create namespace
      run: |
        kubectl create namespace ${{ steps.env-name.outputs.name }} --dry-run=client -o yaml | kubectl apply -f -
        kubectl label namespace ${{ steps.env-name.outputs.name }} \
          environment=preview \
          pr-number=${{ github.event.number }} \
          created-by=github-actions
    
    - name: Deploy application
      run: |
        helm upgrade --install ${{ steps.env-name.outputs.name }} ./helm-chart \
          --namespace ${{ steps.env-name.outputs.name }} \
          --set image.tag=${{ github.sha }} \
          --set ingress.host=${{ steps.env-name.outputs.name }}.preview.myapp.com \
          --set environment=preview \
          --set resources.requests.memory=128Mi \
          --set resources.requests.cpu=100m \
          --set replicaCount=1
    
    - name: Wait for deployment
      run: |
        kubectl wait --for=condition=available --timeout=300s \
          deployment/${{ steps.env-name.outputs.name }} \
          -n ${{ steps.env-name.outputs.name }}
    
    - name: Comment PR with environment URL
      uses: actions/github-script@v6
      with:
        script: |
          const { data: comments } = await github.rest.issues.listComments({
            owner: context.repo.owner,
            repo: context.repo.repo,
            issue_number: context.issue.number,
          });
          
          const botComment = comments.find(comment => 
            comment.user.type === 'Bot' && comment.body.includes('Preview Environment')
          );
          
          const body = `## 🚀 Preview Environment
          
          Your preview environment is ready!
          
          **URL**: ${{ steps.env-name.outputs.url }}
          **Environment**: \`${{ steps.env-name.outputs.name }}\`
          **Commit**: ${{ github.sha }}
          
          This environment will be automatically cleaned up when the PR is closed.`;
          
          if (botComment) {
            await github.rest.issues.updateComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              comment_id: botComment.id,
              body: body
            });
          } else {
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: body
            });
          }
```

### Environment Cleanup

```yaml
# .github/workflows/cleanup-environments.yml
name: Cleanup Preview Environments

on:
  pull_request:
    types: [closed]
  schedule:
    - cron: '0 2 * * *'  # Daily cleanup

jobs:
  cleanup-pr-environment:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    
    steps:
    - name: Generate environment name
      id: env-name
      run: |
        ENV_NAME="pr-${{ github.event.number }}-$(echo ${{ github.head_ref }} | sed 's/[^a-zA-Z0-9]/-/g' | cut -c1-20)"
        echo "name=$ENV_NAME" >> $GITHUB_OUTPUT
    
    - name: Delete environment
      run: |
        helm uninstall ${{ steps.env-name.outputs.name }} -n ${{ steps.env-name.outputs.name }} || true
        kubectl delete namespace ${{ steps.env-name.outputs.name }} || true
    
    - name: Comment PR
      uses: actions/github-script@v6
      with:
        script: |
          await github.rest.issues.createComment({
            owner: context.repo.owner,
            repo: context.repo.repo,
            issue_number: context.issue.number,
            body: `## 🧹 Environment Cleanup\n\nPreview environment \`${{ steps.env-name.outputs.name }}\` has been cleaned up.`
          });
  
  cleanup-stale-environments:
    if: github.event_name == 'schedule'
    runs-on: ubuntu-latest
    
    steps:
    - name: Cleanup stale environments
      run: |
        # Find namespaces older than 7 days
        kubectl get namespaces -l environment=preview \
          -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}' | \
        while read namespace timestamp; do
          if [[ $(date -d "$timestamp" +%s) -lt $(date -d '7 days ago' +%s) ]]; then
            echo "Deleting stale environment: $namespace"
            helm uninstall $(echo $namespace | cut -d'-' -f1-2) -n $namespace || true
            kubectl delete namespace $namespace || true
          fi
        done
```

---

## 3. Environment Configuration Management {#configuration}

### Configuration Strategy

```yaml
# config/environments/base.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
  CACHE_TTL: "3600"
  MAX_CONNECTIONS: "100"
  FEATURE_FLAGS: |
    {
      "new_ui": false,
      "advanced_analytics": false,
      "beta_features": false
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://user:pass@localhost:5432/myapp"
  REDIS_URL: "redis://localhost:6379"
  JWT_SECRET: "your-jwt-secret-here"
```

```yaml
# config/environments/development.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "debug"
  CACHE_TTL: "60"
  MAX_CONNECTIONS: "10"
  DEBUG_MODE: "true"
  FEATURE_FLAGS: |
    {
      "new_ui": true,
      "advanced_analytics": true,
      "beta_features": true
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: "postgresql://dev:dev@dev-db:5432/myapp_dev"
  REDIS_URL: "redis://dev-redis:6379"
  JWT_SECRET: "dev-jwt-secret"
```

```yaml
# config/environments/production.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "warn"
  CACHE_TTL: "7200"
  MAX_CONNECTIONS: "1000"
  FEATURE_FLAGS: |
    {
      "new_ui": false,
      "advanced_analytics": true,
      "beta_features": false
    }
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DATABASE_URL: "${DATABASE_URL}"
  REDIS_URL: "${REDIS_URL}"
  JWT_SECRET: "${JWT_SECRET}"
```

### Configuration Management with Kustomize

```yaml
# config/base/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yml
- service.yml
- configmap.yml
- secret.yml

commonLabels:
  app: myapp
  version: v1.0.0

images:
- name: myapp
  newTag: latest
```

```yaml
# config/overlays/development/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

resources:
- ../../base

patchesStrategicMerge:
- configmap-patch.yml
- deployment-patch.yml

replicas:
- name: myapp
  count: 1

images:
- name: myapp
  newTag: dev
```

```yaml
# config/overlays/development/configmap-patch.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "debug"
  DEBUG_MODE: "true"
  ENVIRONMENT: "development"
```

```yaml
# config/overlays/production/kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
- ../../base
- hpa.yml
- pdb.yml
- network-policy.yml

patchesStrategicMerge:
- configmap-patch.yml
- deployment-patch.yml

replicas:
- name: myapp
  count: 3

images:
- name: myapp
  newTag: stable
```

### Environment-specific Secrets Management

```yaml
# .github/workflows/secrets-management.yml
name: Environment Secrets Management

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ vars.AWS_REGION }}
    
    - name: Retrieve secrets from AWS Secrets Manager
      run: |
        # Get environment-specific secrets
        DATABASE_URL=$(aws secretsmanager get-secret-value \
          --secret-id "${{ inputs.environment }}/myapp/database-url" \
          --query SecretString --output text)
        
        REDIS_URL=$(aws secretsmanager get-secret-value \
          --secret-id "${{ inputs.environment }}/myapp/redis-url" \
          --query SecretString --output text)
        
        JWT_SECRET=$(aws secretsmanager get-secret-value \
          --secret-id "${{ inputs.environment }}/myapp/jwt-secret" \
          --query SecretString --output text)
        
        # Mask secrets in logs
        echo "::add-mask::$DATABASE_URL"
        echo "::add-mask::$REDIS_URL"
        echo "::add-mask::$JWT_SECRET"
        
        # Set environment variables
        echo "DATABASE_URL=$DATABASE_URL" >> $GITHUB_ENV
        echo "REDIS_URL=$REDIS_URL" >> $GITHUB_ENV
        echo "JWT_SECRET=$JWT_SECRET" >> $GITHUB_ENV
    
    - name: Deploy with environment-specific configuration
      run: |
        # Replace placeholders in configuration
        envsubst < config/environments/${{ inputs.environment }}.yml > config-final.yml
        
        # Apply configuration
        kubectl apply -f config-final.yml -n ${{ inputs.environment }}
        
        # Deploy application
        kubectl set image deployment/myapp \
          myapp=ghcr.io/${{ github.repository }}:${{ github.sha }} \
          -n ${{ inputs.environment }}
```

### Configuration Validation

```javascript
// scripts/validate-config.js
const Joi = require('joi');
const fs = require('fs');
const yaml = require('js-yaml');

const environmentSchema = Joi.object({
  LOG_LEVEL: Joi.string().valid('debug', 'info', 'warn', 'error').required(),
  CACHE_TTL: Joi.number().positive().required(),
  MAX_CONNECTIONS: Joi.number().positive().required(),
  DATABASE_URL: Joi.string().uri().required(),
  REDIS_URL: Joi.string().uri().required(),
  JWT_SECRET: Joi.string().min(32).required(),
  FEATURE_FLAGS: Joi.object().required()
});

function validateEnvironmentConfig(environment) {
  try {
    const configPath = `config/environments/${environment}.yml`;
    const config = yaml.load(fs.readFileSync(configPath, 'utf8'));
    
    // Extract configuration from ConfigMap and Secret
    const configData = {};
    
    config.forEach(resource => {
      if (resource.kind === 'ConfigMap') {
        Object.assign(configData, resource.data);
      } else if (resource.kind === 'Secret') {
        Object.assign(configData, resource.stringData);
      }
    });
    
    // Parse JSON fields
    if (configData.FEATURE_FLAGS) {
      configData.FEATURE_FLAGS = JSON.parse(configData.FEATURE_FLAGS);
    }
    
    const { error, value } = environmentSchema.validate(configData);
    
    if (error) {
      console.error(`❌ Configuration validation failed for ${environment}:`);
      error.details.forEach(detail => {
        console.error(`  - ${detail.message}`);
      });
      return false;
    }
    
    console.log(`✅ Configuration validation passed for ${environment}`);
    return true;
  } catch (err) {
    console.error(`❌ Error validating ${environment} configuration:`, err.message);
    return false;
  }
}

// Validate all environments
const environments = ['development', 'staging', 'production'];
let allValid = true;

environments.forEach(env => {
  if (!validateEnvironmentConfig(env)) {
    allValid = false;
  }
});

if (!allValid) {
  process.exit(1);
}

console.log('🎉 All environment configurations are valid!');
```

---

## 4. Environment Promotion Workflows {#promotion}

### Automated Promotion Pipeline

```yaml
# .github/workflows/promotion-pipeline.yml
name: Environment Promotion Pipeline

on:
  workflow_dispatch:
    inputs:
      source_environment:
        description: 'Source environment'
        required: true
        type: choice
        options:
        - development
        - staging
        - production
      target_environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
        - development
        - staging
        - production
      version:
        description: 'Version to promote'
        required: true
        type: string

jobs:
  validate-promotion:
    runs-on: ubuntu-latest
    outputs:
      allowed: ${{ steps.validate.outputs.allowed }}
    
    steps:
    - name: Validate promotion path
      id: validate
      run: |
        SOURCE="${{ inputs.source_environment }}"
        TARGET="${{ inputs.target_environment }}"
        
        # Define allowed promotion paths
        if [[ "$SOURCE" == "development" && "$TARGET" == "staging" ]]; then
          echo "allowed=true" >> $GITHUB_OUTPUT
        elif [[ "$SOURCE" == "staging" && "$TARGET" == "production" ]]; then
          echo "allowed=true" >> $GITHUB_OUTPUT
        elif [[ "$SOURCE" == "production" && "$TARGET" == "staging" ]]; then
          # Allow rollback from production to staging
          echo "allowed=true" >> $GITHUB_OUTPUT
        else
          echo "allowed=false" >> $GITHUB_OUTPUT
          echo "❌ Invalid promotion path: $SOURCE -> $TARGET"
          exit 1
        fi
  
  pre-promotion-tests:
    needs: validate-promotion
    if: needs.validate-promotion.outputs.allowed == 'true'
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Get current version in source environment
      id: current-version
      run: |
        CURRENT_VERSION=$(kubectl get deployment myapp \
          -n ${{ inputs.source_environment }} \
          -o jsonpath='{.spec.template.spec.containers[0].image}' | \
          cut -d':' -f2)
        echo "version=$CURRENT_VERSION" >> $GITHUB_OUTPUT
    
    - name: Verify version exists
      run: |
        if ! docker manifest inspect ghcr.io/${{ github.repository }}:${{ inputs.version }}; then
          echo "❌ Version ${{ inputs.version }} does not exist"
          exit 1
        fi
    
    - name: Run pre-promotion tests
      run: |
        echo "🧪 Running pre-promotion tests for ${{ inputs.version }}"
        # Run smoke tests, security scans, etc.
        docker run --rm ghcr.io/${{ github.repository }}:${{ inputs.version }} npm run test:smoke
  
  promote:
    needs: [validate-promotion, pre-promotion-tests]
    runs-on: ubuntu-latest
    environment: ${{ inputs.target_environment }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Record current state
      id: current-state
      run: |
        CURRENT_VERSION=$(kubectl get deployment myapp \
          -n ${{ inputs.target_environment }} \
          -o jsonpath='{.spec.template.spec.containers[0].image}' | \
          cut -d':' -f2 || echo "none")
        echo "previous_version=$CURRENT_VERSION" >> $GITHUB_OUTPUT
    
    - name: Deploy to target environment
      run: |
        echo "🚀 Promoting ${{ inputs.version }} to ${{ inputs.target_environment }}"
        
        # Update deployment
        kubectl set image deployment/myapp \
          myapp=ghcr.io/${{ github.repository }}:${{ inputs.version }} \
          -n ${{ inputs.target_environment }}
        
        # Wait for rollout
        kubectl rollout status deployment/myapp \
          -n ${{ inputs.target_environment }} \
          --timeout=300s
    
    - name: Post-deployment validation
      id: validation
      run: |
        echo "🔍 Validating deployment in ${{ inputs.target_environment }}"
        
        # Wait for pods to be ready
        sleep 30
        
        # Health check
        if [[ "${{ inputs.target_environment }}" == "production" ]]; then
          HEALTH_URL="https://myapp.com/health"
        else
          HEALTH_URL="https://${{ inputs.target_environment }}.myapp.com/health"
        fi
        
        for i in {1..5}; do
          if curl -f "$HEALTH_URL"; then
            echo "healthy=true" >> $GITHUB_OUTPUT
            echo "✅ Health check passed"
            break
          fi
          echo "⏳ Waiting for health check... ($i/5)"
          sleep 30
        done
        
        if [[ "$(cat $GITHUB_OUTPUT | grep healthy=true)" == "" ]]; then
          echo "healthy=false" >> $GITHUB_OUTPUT
          echo "❌ Health check failed"
        fi
    
    - name: Rollback on failure
      if: steps.validation.outputs.healthy == 'false'
      run: |
        echo "🔄 Rolling back due to failed validation"
        
        if [[ "${{ steps.current-state.outputs.previous_version }}" != "none" ]]; then
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ steps.current-state.outputs.previous_version }} \
            -n ${{ inputs.target_environment }}
          
          kubectl rollout status deployment/myapp \
            -n ${{ inputs.target_environment }}
        fi
        
        exit 1
    
    - name: Update promotion record
      if: steps.validation.outputs.healthy == 'true'
      run: |
        # Record successful promotion
        kubectl annotate deployment myapp \
          promotion.timestamp="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
          promotion.source="${{ inputs.source_environment }}" \
          promotion.version="${{ inputs.version }}" \
          promotion.previous-version="${{ steps.current-state.outputs.previous_version }}" \
          -n ${{ inputs.target_environment }}
    
    - name: Notify success
      if: steps.validation.outputs.healthy == 'true'
      uses: 8398a7/action-slack@v3
      with:
        status: success
        text: |
          ✅ Promotion successful!
          
          **Environment**: ${{ inputs.target_environment }}
          **Version**: ${{ inputs.version }}
          **Previous**: ${{ steps.current-state.outputs.previous_version }}
          **Source**: ${{ inputs.source_environment }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
    
    - name: Notify failure
      if: failure()
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: |
          🚨 Promotion failed!
          
          **Environment**: ${{ inputs.target_environment }}
          **Version**: ${{ inputs.version }}
          **Source**: ${{ inputs.source_environment }}
          **Status**: Rolled back to ${{ steps.current-state.outputs.previous_version }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### GitOps-based Promotion

```yaml
# .github/workflows/gitops-promotion.yml
name: GitOps Environment Promotion

on:
  push:
    branches: [ main ]
    paths: [ 'environments/**' ]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      environments: ${{ steps.changes.outputs.environments }}
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 2
    
    - name: Detect environment changes
      id: changes
      run: |
        CHANGED_ENVS=$(git diff --name-only HEAD~1 HEAD | \
          grep '^environments/' | \
          cut -d'/' -f2 | \
          sort -u | \
          jq -R -s -c 'split("\n")[:-1]')
        echo "environments=$CHANGED_ENVS" >> $GITHUB_OUTPUT
  
  promote:
    needs: detect-changes
    if: needs.detect-changes.outputs.environments != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: ${{ fromJson(needs.detect-changes.outputs.environments) }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Apply environment configuration
      run: |
        echo "🔄 Applying changes to ${{ matrix.environment }}"
        
        # Apply Kustomize configuration
        kubectl apply -k environments/${{ matrix.environment }} \
          --dry-run=server
        
        kubectl apply -k environments/${{ matrix.environment }}
    
    - name: Verify deployment
      run: |
        # Wait for rollout if deployment changed
        if kubectl get deployment myapp -n ${{ matrix.environment }} &>/dev/null; then
          kubectl rollout status deployment/myapp -n ${{ matrix.environment }}
        fi
```

---

## 5. Infrastructure as Code for Environments {#infrastructure}

### Terraform Environment Management

```hcl
# terraform/environments/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
}

variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "database_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "enable_monitoring" {
  description = "Enable enhanced monitoring"
  type        = bool
  default     = false
}

variable "backup_retention_days" {
  description = "Database backup retention period"
  type        = number
  default     = 7
}
```

```hcl
# terraform/environments/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    # Backend configuration will be provided via backend config files
  }
}

provider "aws" {
  region = var.region
  
  default_tags {
    tags = {
      Environment = var.environment
      Project     = "myapp"
      ManagedBy   = "terraform"
    }
  }
}

# VPC
module "vpc" {
  source = "../modules/vpc"
  
  environment = var.environment
  cidr_block  = local.environment_config[var.environment].vpc_cidr
}

# EKS Cluster
module "eks" {
  source = "../modules/eks"
  
  environment     = var.environment
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
  node_groups     = local.environment_config[var.environment].node_groups
}

# RDS Database
module "database" {
  source = "../modules/rds"
  
  environment           = var.environment
  vpc_id               = module.vpc.vpc_id
  subnet_ids           = module.vpc.database_subnet_ids
  instance_class       = var.database_instance_class
  backup_retention_days = var.backup_retention_days
  multi_az             = local.environment_config[var.environment].database_multi_az
}

# Redis Cache
module "cache" {
  source = "../modules/elasticache"
  
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnet_ids
  node_type   = local.environment_config[var.environment].cache_node_type
}

# Application Load Balancer
module "alb" {
  source = "../modules/alb"
  
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.public_subnet_ids
  
  certificate_arn = local.environment_config[var.environment].certificate_arn
}

# Monitoring
module "monitoring" {
  source = "../modules/monitoring"
  
  environment      = var.environment
  enable_detailed  = var.enable_monitoring
  eks_cluster_name = module.eks.cluster_name
}

locals {
  environment_config = {
    development = {
      vpc_cidr           = "10.0.0.0/16"
      database_multi_az  = false
      cache_node_type    = "cache.t3.micro"
      certificate_arn    = "arn:aws:acm:us-west-2:123456789012:certificate/dev-cert"
      node_groups = {
        main = {
          instance_types = ["t3.medium"]
          min_size      = 1
          max_size      = 3
          desired_size  = 1
        }
      }
    }
    
    staging = {
      vpc_cidr           = "10.1.0.0/16"
      database_multi_az  = false
      cache_node_type    = "cache.t3.small"
      certificate_arn    = "arn:aws:acm:us-west-2:123456789012:certificate/staging-cert"
      node_groups = {
        main = {
          instance_types = ["t3.medium"]
          min_size      = 1
          max_size      = 5
          desired_size  = 2
        }
      }
    }
    
    production = {
      vpc_cidr           = "10.2.0.0/16"
      database_multi_az  = true
      cache_node_type    = "cache.r6g.large"
      certificate_arn    = "arn:aws:acm:us-west-2:123456789012:certificate/prod-cert"
      node_groups = {
        main = {
          instance_types = ["m5.large"]
          min_size      = 3
          max_size      = 10
          desired_size  = 3
        }
        spot = {
          instance_types = ["m5.large", "m5a.large", "m4.large"]
          capacity_type  = "SPOT"
          min_size      = 0
          max_size      = 5
          desired_size  = 2
        }
      }
    }
  }
}
```

```hcl
# terraform/environments/outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "eks_cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
  sensitive   = true
}

output "database_endpoint" {
  description = "RDS database endpoint"
  value       = module.database.endpoint
  sensitive   = true
}

output "cache_endpoint" {
  description = "ElastiCache endpoint"
  value       = module.cache.endpoint
  sensitive   = true
}

output "load_balancer_dns" {
  description = "Load balancer DNS name"
  value       = module.alb.dns_name
}
```

### Environment-specific Terraform Configurations

```hcl
# terraform/environments/development/backend.tf
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "environments/development/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

```hcl
# terraform/environments/development/terraform.tfvars
environment = "development"
region      = "us-west-2"

instance_count            = 1
instance_type            = "t3.small"
database_instance_class  = "db.t3.micro"
enable_monitoring        = false
backup_retention_days    = 1
```

```hcl
# terraform/environments/production/terraform.tfvars
environment = "production"
region      = "us-west-2"

instance_count            = 3
instance_type            = "m5.large"
database_instance_class  = "db.r5.xlarge"
enable_monitoring        = true
backup_retention_days    = 30
```

### Infrastructure Deployment Pipeline

```yaml
# .github/workflows/infrastructure.yml
name: Infrastructure Deployment

on:
  push:
    branches: [ main ]
    paths: [ 'terraform/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'terraform/**' ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
        - development
        - staging
        - production
      action:
        description: 'Terraform action'
        required: true
        type: choice
        options:
        - plan
        - apply
        - destroy

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: ${{ github.event_name == 'workflow_dispatch' && fromJson(format('["{}"', inputs.environment, '"]')) || fromJson('["development", "staging"]') }}
    
    environment: ${{ matrix.environment }}
    
    defaults:
      run:
        working-directory: terraform/environments
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: 1.6.0
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
    
    - name: Terraform Init
      run: |
        terraform init -backend-config="${{ matrix.environment }}/backend.tf"
    
    - name: Terraform Validate
      run: terraform validate
    
    - name: Terraform Plan
      id: plan
      run: |
        terraform plan \
          -var-file="${{ matrix.environment }}/terraform.tfvars" \
          -out="${{ matrix.environment }}.tfplan" \
          -no-color
    
    - name: Comment PR with plan
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          const fs = require('fs');
          const plan = fs.readFileSync('terraform/environments/${{ matrix.environment }}.tfplan.txt', 'utf8');
          
          const output = `## Terraform Plan - ${{ matrix.environment }}
          
          \`\`\`
          ${plan}
          \`\`\`
          
          *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
          
          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: output
          });
    
    - name: Terraform Apply
      if: |
        (github.ref == 'refs/heads/main' && github.event_name == 'push') ||
        (github.event_name == 'workflow_dispatch' && inputs.action == 'apply')
      run: |
        terraform apply "${{ matrix.environment }}.tfplan"
    
    - name: Terraform Destroy
      if: github.event_name == 'workflow_dispatch' && inputs.action == 'destroy'
      run: |
        terraform destroy \
          -var-file="${{ matrix.environment }}/terraform.tfvars" \
          -auto-approve
    
    - name: Output infrastructure info
      if: |
        (github.ref == 'refs/heads/main' && github.event_name == 'push') ||
        (github.event_name == 'workflow_dispatch' && inputs.action == 'apply')
      run: |
        echo "## Infrastructure Outputs - ${{ matrix.environment }}" >> $GITHUB_STEP_SUMMARY
        terraform output -json | jq -r 'to_entries[] | "- **\(.key)**: \(.value.value)"' >> $GITHUB_STEP_SUMMARY
```

---

## 6. Environment Security & Isolation {#security}

### Network Isolation

```yaml
# k8s/network-policies/default-deny.yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: production
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: production
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-traffic
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: myapp
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from: []
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to: []
    ports:
    - protocol: TCP
      port: 443
    - protocol: TCP
      port: 80
    - protocol: UDP
      port: 53
```

### RBAC Configuration

```yaml
# k8s/rbac/environment-rbac.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: myapp-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: myapp-rolebinding
  namespace: production
subjects:
- kind: ServiceAccount
  name: myapp-sa
  namespace: production
roleRef:
  kind: Role
  name: myapp-role
  apiGroup: rbac.authorization.k8s.io
---
# Separate RBAC for CI/CD
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: cicd-deployer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["*"]
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cicd-deployer-binding
  namespace: production
subjects:
- kind: User
  name: github-actions
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: cicd-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Security Policies

```yaml
# k8s/security/pod-security-policy.yml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
  readOnlyRootFilesystem: true
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    fsGroup: 1001
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  - to: []
    ports:
    - protocol: TCP
      port: 443
```

### Environment Access Control

```yaml
# .github/workflows/environment-access.yml
name: Environment Access Control

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to access'
        required: true
        type: choice
        options:
        - development
        - staging
        - production
      action:
        description: 'Action to perform'
        required: true
        type: choice
        options:
        - deploy
        - rollback
        - debug
        - maintenance

jobs:
  access-control:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
    - name: Validate access permissions
      run: |
        USER="${{ github.actor }}"
        ENVIRONMENT="${{ inputs.environment }}"
        ACTION="${{ inputs.action }}"
        
        # Define access matrix
        case "$ENVIRONMENT" in
          "development")
            # All team members can access development
            ALLOWED_USERS=".*"
            ;;
          "staging")
            # Only senior developers and above
            ALLOWED_USERS="(senior-dev|lead-dev|devops|admin)"
            ;;
          "production")
            # Only leads and admins
            ALLOWED_USERS="(lead-dev|devops|admin)"
            # Production requires additional approval for destructive actions
            if [[ "$ACTION" == "rollback" || "$ACTION" == "maintenance" ]]; then
              echo "⚠️ Production $ACTION requires manual approval"
              # This would trigger a manual approval process
            fi
            ;;
        esac
        
        # Check user permissions (simplified - in reality, use proper RBAC)
        if [[ ! "$USER" =~ $ALLOWED_USERS ]]; then
          echo "❌ Access denied: $USER cannot perform $ACTION on $ENVIRONMENT"
          exit 1
        fi
        
        echo "✅ Access granted: $USER can perform $ACTION on $ENVIRONMENT"
    
    - name: Log access attempt
      run: |
        # Log to audit system
        echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) - User: ${{ github.actor }}, Environment: ${{ inputs.environment }}, Action: ${{ inputs.action }}, Status: Granted" >> audit.log
        
        # Send to monitoring system
        curl -X POST "${{ secrets.AUDIT_WEBHOOK_URL }}" \
          -H "Content-Type: application/json" \
          -d '{
            "timestamp": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
            "user": "${{ github.actor }}",
            "environment": "${{ inputs.environment }}",
            "action": "${{ inputs.action }}",
            "status": "granted",
            "source": "github-actions"
          }'
    
    - name: Perform action
      run: |
        echo "🚀 Performing ${{ inputs.action }} on ${{ inputs.environment }}"
        # Actual action implementation would go here
```

---

## 7. Environment Testing & Validation {#testing}

### Environment Health Checks

```yaml
# .github/workflows/environment-health.yml
name: Environment Health Checks

on:
  schedule:
    - cron: '*/15 * * * *'  # Every 15 minutes
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to check'
        required: true
        type: choice
        options:
        - development
        - staging
        - production

jobs:
  health-check:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: ${{ github.event_name == 'workflow_dispatch' && fromJson(format('["{}"', inputs.environment, '"]')) || fromJson('["development", "staging", "production"]') }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set environment URL
      id: env-url
      run: |
        case "${{ matrix.environment }}" in
          "development")
            echo "url=https://dev.myapp.com" >> $GITHUB_OUTPUT
            ;;
          "staging")
            echo "url=https://staging.myapp.com" >> $GITHUB_OUTPUT
            ;;
          "production")
            echo "url=https://myapp.com" >> $GITHUB_OUTPUT
            ;;
        esac
    
    - name: Basic health check
      id: health
      run: |
        URL="${{ steps.env-url.outputs.url }}"
        
        # Health endpoint check
        if curl -f "$URL/health" -o health.json; then
          echo "health=healthy" >> $GITHUB_OUTPUT
          echo "✅ Health check passed for ${{ matrix.environment }}"
        else
          echo "health=unhealthy" >> $GITHUB_OUTPUT
          echo "❌ Health check failed for ${{ matrix.environment }}"
        fi
    
    - name: Performance check
      id: performance
      run: |
        URL="${{ steps.env-url.outputs.url }}"
        
        # Measure response time
        RESPONSE_TIME=$(curl -o /dev/null -s -w '%{time_total}' "$URL")
        echo "response_time=$RESPONSE_TIME" >> $GITHUB_OUTPUT
        
        # Check if response time is acceptable
        if (( $(echo "$RESPONSE_TIME < 2.0" | bc -l) )); then
          echo "performance=good" >> $GITHUB_OUTPUT
          echo "✅ Performance check passed: ${RESPONSE_TIME}s"
        else
          echo "performance=slow" >> $GITHUB_OUTPUT
          echo "⚠️ Performance check warning: ${RESPONSE_TIME}s"
        fi
    
    - name: Database connectivity check
      id: database
      run: |
        URL="${{ steps.env-url.outputs.url }}"
        
        # Check database connectivity
        if curl -f "$URL/health/database" -o db-health.json; then
          DB_STATUS=$(jq -r '.database.status' db-health.json)
          if [[ "$DB_STATUS" == "connected" ]]; then
            echo "database=connected" >> $GITHUB_OUTPUT
            echo "✅ Database connectivity check passed"
          else
            echo "database=disconnected" >> $GITHUB_OUTPUT
            echo "❌ Database connectivity check failed"
          fi
        else
          echo "database=error" >> $GITHUB_OUTPUT
          echo "❌ Database health endpoint error"
        fi
    
    - name: External dependencies check
      id: dependencies
      run: |
        URL="${{ steps.env-url.outputs.url }}"
        
        # Check external service dependencies
        if curl -f "$URL/health/dependencies" -o deps-health.json; then
          FAILED_DEPS=$(jq -r '.dependencies[] | select(.status != "healthy") | .name' deps-health.json)
          if [[ -z "$FAILED_DEPS" ]]; then
            echo "dependencies=healthy" >> $GITHUB_OUTPUT
            echo "✅ All dependencies are healthy"
          else
            echo "dependencies=unhealthy" >> $GITHUB_OUTPUT
            echo "❌ Unhealthy dependencies: $FAILED_DEPS"
          fi
        else
          echo "dependencies=error" >> $GITHUB_OUTPUT
          echo "❌ Dependencies health endpoint error"
        fi
    
    - name: Generate health report
      run: |
        cat > health-report-${{ matrix.environment }}.json << EOF
        {
          "environment": "${{ matrix.environment }}",
          "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
          "url": "${{ steps.env-url.outputs.url }}",
          "health": "${{ steps.health.outputs.health }}",
          "performance": {
            "status": "${{ steps.performance.outputs.performance }}",
            "response_time": "${{ steps.performance.outputs.response_time }}"
          },
          "database": "${{ steps.database.outputs.database }}",
          "dependencies": "${{ steps.dependencies.outputs.dependencies }}"
        }
        EOF
    
    - name: Upload health report
      uses: actions/upload-artifact@v3
      with:
        name: health-reports
        path: health-report-${{ matrix.environment }}.json
    
    - name: Alert on critical issues
      if: |
        steps.health.outputs.health == 'unhealthy' ||
        steps.database.outputs.database == 'disconnected' ||
        steps.dependencies.outputs.dependencies == 'unhealthy'
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: |
          🚨 Critical health check failure!
          
          **Environment**: ${{ matrix.environment }}
          **URL**: ${{ steps.env-url.outputs.url }}
          **Health**: ${{ steps.health.outputs.health }}
          **Database**: ${{ steps.database.outputs.database }}
          **Dependencies**: ${{ steps.dependencies.outputs.dependencies }}
          
          Please investigate immediately!
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Smoke Tests

```javascript
// tests/smoke/smoke-tests.js
const axios = require('axios');
const { expect } = require('chai');

describe('Environment Smoke Tests', () => {
  const baseURL = process.env.TEST_URL || 'http://localhost:3000';
  
  describe('Basic Functionality', () => {
    it('should respond to health check', async () => {
      const response = await axios.get(`${baseURL}/health`);
      expect(response.status).to.equal(200);
      expect(response.data).to.have.property('status', 'healthy');
    });
    
    it('should serve the main application', async () => {
      const response = await axios.get(baseURL);
      expect(response.status).to.equal(200);
      expect(response.headers['content-type']).to.include('text/html');
    });
    
    it('should handle API requests', async () => {
      const response = await axios.get(`${baseURL}/api/version`);
      expect(response.status).to.equal(200);
      expect(response.data).to.have.property('version');
    });
  });
  
  describe('Database Connectivity', () => {
    it('should connect to database', async () => {
      const response = await axios.get(`${baseURL}/health/database`);
      expect(response.status).to.equal(200);
      expect(response.data.database.status).to.equal('connected');
    });
    
    it('should perform basic database operations', async () => {
      const response = await axios.get(`${baseURL}/api/users/count`);
      expect(response.status).to.equal(200);
      expect(response.data).to.have.property('count');
      expect(response.data.count).to.be.a('number');
    });
  });
  
  describe('External Dependencies', () => {
    it('should connect to cache', async () => {
      const response = await axios.get(`${baseURL}/health/cache`);
      expect(response.status).to.equal(200);
      expect(response.data.cache.status).to.equal('connected');
    });
    
    it('should connect to external APIs', async () => {
      const response = await axios.get(`${baseURL}/health/dependencies`);
      expect(response.status).to.equal(200);
      
      const dependencies = response.data.dependencies;
      dependencies.forEach(dep => {
        expect(dep.status).to.equal('healthy');
      });
    });
  });
  
  describe('Security', () => {
    it('should require authentication for protected routes', async () => {
      try {
        await axios.get(`${baseURL}/api/admin/users`);
        expect.fail('Should have thrown an error');
      } catch (error) {
        expect(error.response.status).to.equal(401);
      }
    });
    
    it('should have security headers', async () => {
      const response = await axios.get(baseURL);
      expect(response.headers).to.have.property('x-frame-options');
      expect(response.headers).to.have.property('x-content-type-options');
      expect(response.headers).to.have.property('x-xss-protection');
    });
  });
});

### Contract Testing

```javascript
// tests/contract/api-contract.js
const { Pact } = require('@pact-foundation/pact');
const { like, eachLike } = require('@pact-foundation/pact').Matchers;
const axios = require('axios');

describe('API Contract Tests', () => {
  const provider = new Pact({
    consumer: 'frontend',
    provider: 'backend-api',
    port: 1234,
    log: './logs/pact.log',
    dir: './pacts',
    logLevel: 'INFO'
  });
  
  before(() => provider.setup());
  after(() => provider.finalize());
  afterEach(() => provider.verify());
  
  describe('User API', () => {
    it('should get user profile', async () => {
      await provider
        .given('user exists')
        .uponReceiving('a request for user profile')
        .withRequest({
          method: 'GET',
          path: '/api/users/123',
          headers: {
            'Authorization': like('Bearer token123')
          }
        })
        .willRespondWith({
          status: 200,
          headers: {
            'Content-Type': 'application/json'
          },
          body: {
            id: like(123),
            name: like('John Doe'),
            email: like('john@example.com'),
            created_at: like('2023-01-01T00:00:00Z')
          }
        });
      
      const response = await axios.get('http://localhost:1234/api/users/123', {
        headers: { 'Authorization': 'Bearer token123' }
      });
      
      expect(response.status).to.equal(200);
      expect(response.data).to.have.property('id');
      expect(response.data).to.have.property('name');
      expect(response.data).to.have.property('email');
    });
  });
});
```

### Environment Validation Pipeline

```yaml
# .github/workflows/environment-validation.yml
name: Environment Validation

on:
  deployment_status:
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to validate'
        required: true
        type: choice
        options:
        - development
        - staging
        - production

jobs:
  validate-environment:
    runs-on: ubuntu-latest
    if: github.event.deployment_status.state == 'success' || github.event_name != 'deployment_status'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Determine environment
      id: env
      run: |
        if [[ "${{ github.event_name }}" == "deployment_status" ]]; then
          ENV="${{ github.event.deployment.environment }}"
        elif [[ "${{ github.event_name }}" == "workflow_dispatch" ]]; then
          ENV="${{ inputs.environment }}"
        else
          ENV="staging"  # Default for scheduled runs
        fi
        echo "environment=$ENV" >> $GITHUB_OUTPUT
        
        case "$ENV" in
          "development")
            echo "url=https://dev.myapp.com" >> $GITHUB_OUTPUT
            ;;
          "staging")
            echo "url=https://staging.myapp.com" >> $GITHUB_OUTPUT
            ;;
          "production")
            echo "url=https://myapp.com" >> $GITHUB_OUTPUT
            ;;
        esac
    
    - name: Run smoke tests
      env:
        TEST_URL: ${{ steps.env.outputs.url }}
        ENVIRONMENT: ${{ steps.env.outputs.environment }}
      run: |
        npm run test:smoke
    
    - name: Run contract tests
      if: steps.env.outputs.environment != 'production'
      env:
        TEST_URL: ${{ steps.env.outputs.url }}
      run: |
        npm run test:contract
    
    - name: Run performance tests
      env:
        TEST_URL: ${{ steps.env.outputs.url }}
      run: |
        npm run test:performance
    
    - name: Security scan
      run: |
        # OWASP ZAP baseline scan
        docker run -v $(pwd):/zap/wrk/:rw \
          -t owasp/zap2docker-stable zap-baseline.py \
          -t ${{ steps.env.outputs.url }} \
          -J zap-report.json
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: validation-results-${{ steps.env.outputs.environment }}
        path: |
          test-results/
          zap-report.json
    
    - name: Notify validation results
      if: failure()
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        text: |
          🚨 Environment validation failed!
          
          **Environment**: ${{ steps.env.outputs.environment }}
          **URL**: ${{ steps.env.outputs.url }}
          **Workflow**: ${{ github.workflow }}
          
          Please check the validation results.
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 8. Data Management Across Environments {#data}

### Database Migration Strategy

```yaml
# .github/workflows/database-migration.yml
name: Database Migration

on:
  push:
    branches: [ main ]
    paths: [ 'migrations/**' ]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to migrate'
        required: true
        type: choice
        options:
        - development
        - staging
        - production
      migration_direction:
        description: 'Migration direction'
        required: true
        type: choice
        options:
        - up
        - down
      target_version:
        description: 'Target migration version (optional)'
        required: false
        type: string

jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment || 'development' }}
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Configure database connection
      env:
        DATABASE_URL: ${{ secrets.DATABASE_URL }}
      run: |
        echo "DATABASE_URL=$DATABASE_URL" > .env
    
    - name: Check migration status
      run: |
        npm run migrate:status
    
    - name: Backup database (production only)
      if: inputs.environment == 'production'
      run: |
        BACKUP_NAME="backup-$(date +%Y%m%d-%H%M%S)"
        npm run db:backup -- --name="$BACKUP_NAME"
        echo "backup_name=$BACKUP_NAME" >> $GITHUB_ENV
    
    - name: Run migrations
      run: |
        if [[ "${{ inputs.migration_direction }}" == "down" ]]; then
          if [[ -n "${{ inputs.target_version }}" ]]; then
            npm run migrate:down -- --to="${{ inputs.target_version }}"
          else
            npm run migrate:down -- --steps=1
          fi
        else
          if [[ -n "${{ inputs.target_version }}" ]]; then
            npm run migrate:up -- --to="${{ inputs.target_version }}"
          else
            npm run migrate:up
          fi
        fi
    
    - name: Verify migration
      run: |
        npm run migrate:status
        npm run db:validate
    
    - name: Rollback on failure
      if: failure() && inputs.environment == 'production' && env.backup_name
      run: |
        echo "🔄 Rolling back database due to migration failure"
        npm run db:restore -- --name="${{ env.backup_name }}"
    
    - name: Notify migration result
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: |
          ${{ job.status == 'success' && '✅' || '🚨' }} Database migration ${{ job.status }}!
          
          **Environment**: ${{ inputs.environment || 'development' }}
          **Direction**: ${{ inputs.migration_direction || 'up' }}
          **Target**: ${{ inputs.target_version || 'latest' }}
          ${{ job.status == 'failure' && inputs.environment == 'production' && format('**Backup**: {0}', env.backup_name) || '' }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### Data Seeding and Fixtures

```javascript
// scripts/seed-data.js
const { Pool } = require('pg');
const fs = require('fs');
const path = require('path');

class DataSeeder {
  constructor(environment) {
    this.environment = environment;
    this.pool = new Pool({
      connectionString: process.env.DATABASE_URL
    });
  }
  
  async seedEnvironment() {
    console.log(`🌱 Seeding data for ${this.environment} environment`);
    
    try {
      await this.pool.query('BEGIN');
      
      // Load environment-specific seed data
      const seedPath = path.join(__dirname, '..', 'seeds', this.environment);
      const seedFiles = fs.readdirSync(seedPath)
        .filter(file => file.endsWith('.sql'))
        .sort();
      
      for (const file of seedFiles) {
        console.log(`📄 Executing ${file}`);
        const sql = fs.readFileSync(path.join(seedPath, file), 'utf8');
        await this.pool.query(sql);
      }
      
      // Load JSON fixtures
      const fixturesPath = path.join(__dirname, '..', 'fixtures', this.environment);
      if (fs.existsSync(fixturesPath)) {
        const fixtureFiles = fs.readdirSync(fixturesPath)
          .filter(file => file.endsWith('.json'));
        
        for (const file of fixtureFiles) {
          console.log(`📋 Loading fixtures from ${file}`);
          const fixtures = JSON.parse(
            fs.readFileSync(path.join(fixturesPath, file), 'utf8')
          );
          
          await this.loadFixtures(fixtures);
        }
      }
      
      await this.pool.query('COMMIT');
      console.log('✅ Data seeding completed successfully');
      
    } catch (error) {
      await this.pool.query('ROLLBACK');
      console.error('❌ Data seeding failed:', error);
      throw error;
    } finally {
      await this.pool.end();
    }
  }
  
  async loadFixtures(fixtures) {
    for (const [table, records] of Object.entries(fixtures)) {
      if (Array.isArray(records) && records.length > 0) {
        const columns = Object.keys(records[0]);
        const values = records.map(record => 
          columns.map(col => record[col])
        );
        
        const placeholders = values.map((_, i) => 
          `(${columns.map((_, j) => `$${i * columns.length + j + 1}`).join(', ')})`
        ).join(', ');
        
        const query = `
          INSERT INTO ${table} (${columns.join(', ')})
          VALUES ${placeholders}
          ON CONFLICT DO NOTHING
        `;
        
        await this.pool.query(query, values.flat());
        console.log(`  ✓ Loaded ${records.length} records into ${table}`);
      }
    }
  }
}

// Usage
const environment = process.argv[2] || process.env.NODE_ENV || 'development';
const seeder = new DataSeeder(environment);

seeder.seedEnvironment()
  .then(() => process.exit(0))
  .catch(() => process.exit(1));
```

### Environment Data Sync

```yaml
# .github/workflows/data-sync.yml
name: Environment Data Sync

on:
  workflow_dispatch:
    inputs:
      source_environment:
        description: 'Source environment'
        required: true
        type: choice
        options:
        - production
        - staging
      target_environment:
        description: 'Target environment'
        required: true
        type: choice
        options:
        - development
        - staging
      sync_type:
        description: 'Type of sync'
        required: true
        type: choice
        options:
        - full
        - schema-only
        - sample-data
      anonymize_data:
        description: 'Anonymize sensitive data'
        required: true
        type: boolean
        default: true

jobs:
  data-sync:
    runs-on: ubuntu-latest
    environment: data-sync
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Validate sync request
      run: |
        SOURCE="${{ inputs.source_environment }}"
        TARGET="${{ inputs.target_environment }}"
        
        # Prevent dangerous syncs
        if [[ "$TARGET" == "production" ]]; then
          echo "❌ Cannot sync TO production environment"
          exit 1
        fi
        
        if [[ "$SOURCE" == "$TARGET" ]]; then
          echo "❌ Source and target environments cannot be the same"
          exit 1
        fi
        
        echo "✅ Sync validation passed: $SOURCE -> $TARGET"
    
    - name: Setup database tools
      run: |
        # Install PostgreSQL client
        sudo apt-get update
        sudo apt-get install -y postgresql-client
        
        # Install data anonymization tool
        pip install faker
    
    - name: Create database dump
      env:
        SOURCE_DB_URL: ${{ secrets[format('{0}_DATABASE_URL', inputs.source_environment)] }}
      run: |
        DUMP_FILE="dump-$(date +%Y%m%d-%H%M%S).sql"
        echo "dump_file=$DUMP_FILE" >> $GITHUB_ENV
        
        case "${{ inputs.sync_type }}" in
          "schema-only")
            pg_dump "$SOURCE_DB_URL" --schema-only > "$DUMP_FILE"
            ;;
          "sample-data")
            # Create a sample dump with limited data
            pg_dump "$SOURCE_DB_URL" \
              --data-only \
              --where="created_at > NOW() - INTERVAL '30 days'" \
              > "$DUMP_FILE"
            ;;
          "full")
            pg_dump "$SOURCE_DB_URL" > "$DUMP_FILE"
            ;;
        esac
    
    - name: Anonymize sensitive data
      if: inputs.anonymize_data == true && inputs.sync_type != 'schema-only'
      run: |
        python scripts/anonymize-data.py "${{ env.dump_file }}"
    
    - name: Backup target database
      env:
        TARGET_DB_URL: ${{ secrets[format('{0}_DATABASE_URL', inputs.target_environment)] }}
      run: |
        BACKUP_FILE="backup-${{ inputs.target_environment }}-$(date +%Y%m%d-%H%M%S).sql"
        echo "backup_file=$BACKUP_FILE" >> $GITHUB_ENV
        
        pg_dump "$TARGET_DB_URL" > "$BACKUP_FILE"
        echo "✅ Target database backed up to $BACKUP_FILE"
    
    - name: Restore data to target
      env:
        TARGET_DB_URL: ${{ secrets[format('{0}_DATABASE_URL', inputs.target_environment)] }}
      run: |
        # Drop and recreate database (be careful!)
        if [[ "${{ inputs.sync_type }}" == "full" ]]; then
          psql "$TARGET_DB_URL" -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
        fi
        
        # Restore from dump
        psql "$TARGET_DB_URL" < "${{ env.dump_file }}"
        
        echo "✅ Data sync completed successfully"
    
    - name: Verify data integrity
      env:
        TARGET_DB_URL: ${{ secrets[format('{0}_DATABASE_URL', inputs.target_environment)] }}
      run: |
        # Run basic integrity checks
        psql "$TARGET_DB_URL" -c "SELECT COUNT(*) FROM users;"
        psql "$TARGET_DB_URL" -c "SELECT COUNT(*) FROM orders;"
        
        # Run custom validation queries
        node scripts/validate-data-integrity.js
    
    - name: Cleanup
      if: always()
      run: |
        rm -f "${{ env.dump_file }}"
        # Keep backup file for safety
    
    - name: Notify sync completion
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        text: |
          ${{ job.status == 'success' && '✅' || '🚨' }} Data sync ${{ job.status }}!
          
          **Source**: ${{ inputs.source_environment }}
          **Target**: ${{ inputs.target_environment }}
          **Type**: ${{ inputs.sync_type }}
          **Anonymized**: ${{ inputs.anonymize_data }}
          ${{ job.status == 'failure' && format('**Backup**: {0}', env.backup_file) || '' }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```