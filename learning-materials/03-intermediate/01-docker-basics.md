# Module 9: Docker Fundamentals for CI/CD 🐳

## Learning Objectives 🎯

By the end of this module, you will:
- Understand Docker concepts and containerization principles
- Learn how to containerize applications for CI/CD
- Master multi-stage builds for optimized images
- Work with container registries and image management
- Implement Docker best practices for production
- Integrate Docker into CI/CD pipelines
- Handle container security and optimization
- Debug and troubleshoot container issues

## Table of Contents

1. [Introduction to Containerization](#introduction)
2. [Docker Architecture](#docker-architecture)
3. [Getting Started with Docker](#getting-started)
4. [Writing Dockerfiles](#dockerfiles)
5. [Multi-Stage Builds](#multi-stage-builds)
6. [Container Registries](#container-registries)
7. [Docker in CI/CD](#docker-cicd)
8. [Best Practices](#best-practices)
9. [Security Considerations](#security)
10. [Performance Optimization](#performance)
11. [Hands-on Exercises](#exercises)
12. [Troubleshooting](#troubleshooting)

---

## 1. Introduction to Containerization {#introduction}

### What is Containerization?

Containerization is a lightweight form of virtualization that packages applications and their dependencies into portable, self-sufficient containers.

```mermaid
graph TB
    A[Traditional Deployment] --> B[Physical Servers]
    B --> C[OS + Runtime + App]
    
    D[Virtual Machines] --> E[Hypervisor]
    E --> F[Guest OS + Runtime + App]
    
    G[Containers] --> H[Container Engine]
    H --> I[Shared OS Kernel]
    I --> J[Runtime + App Only]
```

### Why Docker for CI/CD?

**Benefits:**
- 🔄 **Consistency**: "Works on my machine" → "Works everywhere"
- 📦 **Isolation**: Applications run in isolated environments
- 🚀 **Portability**: Run anywhere Docker is supported
- ⚡ **Speed**: Fast startup and lightweight footprint
- 📈 **Scalability**: Easy horizontal scaling
- 🔧 **DevOps Integration**: Perfect for CI/CD pipelines

### Docker vs Virtual Machines

| Feature | Docker Containers | Virtual Machines |
|---------|------------------|------------------|
| **Resource Usage** | Lightweight (MB) | Heavy (GB) |
| **Startup Time** | Seconds | Minutes |
| **OS Overhead** | Shared kernel | Full OS each |
| **Isolation** | Process-level | Hardware-level |
| **Portability** | Excellent | Good |
| **Use Case** | Microservices, CI/CD | Legacy apps, different OS |

---

## 2. Docker Architecture {#docker-architecture}

### Core Components

```mermaid
graph TB
    A[Docker Client] --> B[Docker Daemon]
    B --> C[Images]
    B --> D[Containers]
    B --> E[Networks]
    B --> F[Volumes]
    
    G[Registry] --> C
    C --> H[Dockerfile]
```

#### Docker Engine
- **Docker Daemon** (`dockerd`): Background service managing containers
- **Docker Client**: Command-line interface (CLI)
- **REST API**: Interface between client and daemon

#### Key Concepts
- **Image**: Read-only template for creating containers
- **Container**: Running instance of an image
- **Dockerfile**: Instructions for building images
- **Registry**: Storage for Docker images (Docker Hub, ECR, etc.)

### Docker Workflow

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Docker as Docker Engine
    participant Registry as Container Registry
    
    Dev->>Docker: docker build
    Docker->>Docker: Create Image
    Dev->>Docker: docker push
    Docker->>Registry: Upload Image
    
    Dev->>Docker: docker pull
    Registry->>Docker: Download Image
    Dev->>Docker: docker run
    Docker->>Docker: Create Container
```

---

## 3. Getting Started with Docker {#getting-started}

### Installation

#### macOS
```bash
# Install Docker Desktop
brew install --cask docker

# Or download from docker.com
# Start Docker Desktop application
```

#### Linux (Ubuntu)
```bash
# Update package index
sudo apt-get update

# Install dependencies
sudo apt-get install ca-certificates curl gnupg lsb-release

# Add Docker's GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER
```

#### Windows
```bash
# Install Docker Desktop for Windows
# Download from docker.com
# Requires WSL2 for best performance
```

### First Docker Commands

```bash
# Check Docker installation
docker --version
docker info

# Hello World
docker run hello-world

# List images
docker images

# List running containers
docker ps

# List all containers
docker ps -a

# Get help
docker --help
docker run --help
```

### Basic Container Operations

```bash
# Run a container
docker run nginx

# Run in background (detached)
docker run -d nginx

# Run with custom name
docker run -d --name my-nginx nginx

# Run with port mapping
docker run -d -p 8080:80 --name web-server nginx

# Run interactively
docker run -it ubuntu bash

# Execute command in running container
docker exec -it my-nginx bash

# Stop container
docker stop my-nginx

# Remove container
docker rm my-nginx

# Remove image
docker rmi nginx
```

---

## 4. Writing Dockerfiles {#dockerfiles}

### Dockerfile Basics

A Dockerfile is a text file containing instructions to build a Docker image.

```dockerfile
# Basic Node.js application
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Define startup command
CMD ["npm", "start"]
```

### Common Dockerfile Instructions

#### FROM
```dockerfile
# Specify base image
FROM node:18-alpine
FROM python:3.9-slim
FROM openjdk:11-jre-slim
FROM scratch  # Empty base image
```

#### WORKDIR
```dockerfile
# Set working directory
WORKDIR /app
WORKDIR /var/www/html
```

#### COPY vs ADD
```dockerfile
# COPY: Simple file copying (preferred)
COPY package.json ./
COPY src/ ./src/

# ADD: More features (URL download, auto-extraction)
ADD https://example.com/file.tar.gz /tmp/
ADD archive.tar.gz /app/  # Auto-extracts
```

#### RUN
```dockerfile
# Execute commands during build
RUN apt-get update && apt-get install -y curl
RUN npm install
RUN pip install -r requirements.txt

# Combine commands to reduce layers
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

#### ENV
```dockerfile
# Set environment variables
ENV NODE_ENV=production
ENV API_URL=https://api.example.com
ENV PATH="/app/bin:${PATH}"
```

#### EXPOSE
```dockerfile
# Document which ports are used
EXPOSE 3000
EXPOSE 8080 8443
```

#### CMD vs ENTRYPOINT
```dockerfile
# CMD: Default command (can be overridden)
CMD ["npm", "start"]
CMD ["python", "app.py"]

# ENTRYPOINT: Always executed (append arguments)
ENTRYPOINT ["python", "app.py"]

# Combination
ENTRYPOINT ["python", "app.py"]
CMD ["--help"]
```

### Language-Specific Examples

#### Node.js Application
```dockerfile
FROM node:18-alpine

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Bundle app source
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001
USER nextjs

EXPOSE 3000
CMD ["node", "server.js"]
```

#### Python Application
```dockerfile
FROM python:3.9-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Set work directory
WORKDIR /code

# Install system dependencies
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt /code/
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . /code/

# Create non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

EXPOSE 8000
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "myproject.wsgi"]
```

#### Java Application
```dockerfile
# Multi-stage build for Java
FROM maven:3.8-openjdk-11 AS build

WORKDIR /app
COPY pom.xml .
COPY src ./src

RUN mvn clean package -DskipTests

FROM openjdk:11-jre-slim

WORKDIR /app

# Copy jar from build stage
COPY --from=build /app/target/*.jar app.jar

# Create non-root user
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Go Application
```dockerfile
# Multi-stage build for Go
FROM golang:1.19-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/

COPY --from=builder /app/main .

EXPOSE 8080
CMD ["./main"]
```

---

## 5. Multi-Stage Builds {#multi-stage-builds}

### Why Multi-Stage Builds?

Multi-stage builds allow you to:
- **Reduce image size**: Keep only runtime dependencies
- **Improve security**: Remove build tools from final image
- **Optimize builds**: Cache build dependencies separately
- **Simplify pipelines**: Build and runtime in one Dockerfile

### Basic Multi-Stage Example

```dockerfile
# Stage 1: Build environment
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: Production environment
FROM nginx:alpine AS production

# Copy built assets from builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy custom nginx config
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Advanced Multi-Stage Patterns

#### Development vs Production
```dockerfile
# Base stage with common dependencies
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

# Development stage
FROM base AS development
RUN npm install
COPY . .
CMD ["npm", "run", "dev"]

# Build stage
FROM base AS builder
RUN npm ci --only=production
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

# Test stage
FROM base AS test
RUN npm install
COPY . .
RUN npm test
```

#### Complex Build Pipeline
```dockerfile
# Dependencies stage
FROM node:18-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --frozen-lockfile

# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Test stage
FROM builder AS test
RUN npm run test:coverage

# Security scan stage
FROM builder AS security
RUN npm audit --audit-level moderate

# Production stage
FROM node:18-alpine AS runner
WORKDIR /app

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built application
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

### Building Specific Stages

```bash
# Build development stage
docker build --target development -t myapp:dev .

# Build production stage
docker build --target production -t myapp:prod .

# Build and run tests
docker build --target test -t myapp:test .
docker run --rm myapp:test

# Build for different platforms
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .
```

---

## 6. Container Registries {#container-registries}

### Types of Registries

#### Public Registries
- **Docker Hub**: Default public registry
- **GitHub Container Registry (ghcr.io)**: Integrated with GitHub
- **Google Container Registry (gcr.io)**: Google Cloud
- **Amazon ECR Public**: AWS public registry

#### Private Registries
- **Amazon ECR**: AWS managed registry
- **Azure Container Registry**: Microsoft Azure
- **Google Container Registry**: Google Cloud
- **Harbor**: Open-source enterprise registry
- **Nexus**: Sonatype repository manager

### Working with Docker Hub

```bash
# Login to Docker Hub
docker login

# Tag image for Docker Hub
docker tag myapp:latest username/myapp:latest

# Push to Docker Hub
docker push username/myapp:latest

# Pull from Docker Hub
docker pull username/myapp:latest

# Search for images
docker search nginx
```

### GitHub Container Registry

```bash
# Login to GitHub Container Registry
echo $GITHUB_TOKEN | docker login ghcr.io -u USERNAME --password-stdin

# Tag for GitHub Container Registry
docker tag myapp:latest ghcr.io/username/myapp:latest

# Push to GitHub Container Registry
docker push ghcr.io/username/myapp:latest

# Pull from GitHub Container Registry
docker pull ghcr.io/username/myapp:latest
```

### AWS ECR Integration

```bash
# Install AWS CLI
aws configure

# Get login token
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Create repository
aws ecr create-repository --repository-name myapp

# Tag and push
docker tag myapp:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
```

### Registry Best Practices

#### Image Tagging Strategy
```bash
# Semantic versioning
docker tag myapp:latest myapp:1.2.3
docker tag myapp:latest myapp:1.2
docker tag myapp:latest myapp:1

# Git-based tagging
docker tag myapp:latest myapp:commit-abc123
docker tag myapp:latest myapp:main
docker tag myapp:latest myapp:pr-456

# Environment-based
docker tag myapp:latest myapp:dev
docker tag myapp:latest myapp:staging
docker tag myapp:latest myapp:prod
```

#### Image Scanning
```bash
# Docker Hub vulnerability scanning
docker scout cves myapp:latest

# Trivy scanning
trivy image myapp:latest

# Snyk scanning
snyk container test myapp:latest
```

---

## 7. Docker in CI/CD {#docker-cicd}

### GitHub Actions with Docker

#### Basic Docker Workflow
```yaml
# .github/workflows/docker.yml
name: Docker Build and Push

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
    - name: Checkout repository
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Container Registry
      if: github.event_name != 'pull_request'
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
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: ${{ github.event_name != 'pull_request' }}
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

#### Multi-Stage Build with Testing
```yaml
name: Docker Multi-Stage Build

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build test image
      run: docker build --target test -t myapp:test .
    
    - name: Run tests
      run: docker run --rm myapp:test

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build image
      run: docker build -t myapp:scan .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'myapp:scan'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to registry
      uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ghcr.io/${{ github.repository }}:latest
        platforms: linux/amd64,linux/arm64
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

### Docker Compose in CI/CD

#### Development Environment
```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      target: development
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    depends_on:
      - db
      - redis

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

#### Testing Environment
```yaml
# docker-compose.test.yml
version: '3.8'

services:
  app-test:
    build:
      context: .
      target: test
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://user:password@db-test:5432/test_db
    depends_on:
      - db-test

  db-test:
    image: postgres:13
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
```

#### CI/CD Integration
```yaml
# GitHub Actions with Docker Compose
- name: Run tests with Docker Compose
  run: |
    docker-compose -f docker-compose.test.yml up --build --abort-on-container-exit
    docker-compose -f docker-compose.test.yml down

- name: Run integration tests
  run: |
    docker-compose up -d db redis
    docker-compose run --rm app npm run test:integration
    docker-compose down
```

---

## 8. Best Practices {#best-practices}

### Dockerfile Best Practices

#### 1. Use Specific Base Images
```dockerfile
# Good: Specific version
FROM node:18.17.1-alpine

# Bad: Latest tag
FROM node:latest
```

#### 2. Minimize Layers
```dockerfile
# Good: Combine RUN commands
RUN apt-get update && \
    apt-get install -y curl git && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Bad: Multiple RUN commands
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y git
```

#### 3. Use .dockerignore
```dockerfile
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
coverage
.pytest_cache
__pycache__
*.pyc
.DS_Store
```

#### 4. Run as Non-Root User
```dockerfile
# Create user
RUN addgroup --system app && adduser --system app --ingroup app

# Set ownership
COPY --chown=app:app . /app

# Switch to user
USER app
```

#### 5. Use Multi-Stage Builds
```dockerfile
# Build stage
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM node:18-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
USER node
CMD ["npm", "start"]
```

### Image Optimization

#### Size Optimization
```dockerfile
# Use Alpine images
FROM node:18-alpine

# Clean package manager cache
RUN npm ci --only=production && npm cache clean --force

# Remove unnecessary files
RUN rm -rf /tmp/* /var/tmp/* /var/cache/apk/*

# Use specific COPY commands
COPY package*.json ./
COPY src/ ./src/
```

#### Build Performance
```dockerfile
# Order layers by change frequency
FROM node:18-alpine

WORKDIR /app

# Dependencies (changes less frequently)
COPY package*.json ./
RUN npm ci --only=production

# Source code (changes more frequently)
COPY . .

CMD ["npm", "start"]
```

### Security Best Practices

#### 1. Scan for Vulnerabilities
```bash
# Trivy scanning
trivy image myapp:latest

# Docker Scout
docker scout cves myapp:latest

# Snyk scanning
snyk container test myapp:latest
```

#### 2. Use Distroless Images
```dockerfile
FROM gcr.io/distroless/nodejs18-debian11
COPY --from=builder /app .
USER nonroot:nonroot
CMD ["index.js"]
```

#### 3. Sign Images
```bash
# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Push signed image
docker push myapp:latest
```

#### 4. Runtime Security
```dockerfile
# Read-only filesystem
FROM alpine
RUN mkdir /app && chown nobody:nobody /app
USER nobody
COPY --chown=nobody:nobody app /app/
WORKDIR /app
CMD ["./app"]
```

---

## 9. Security Considerations {#security}

### Container Security Fundamentals

#### Principle of Least Privilege
```dockerfile
# Create specific user with minimal permissions
FROM alpine:latest

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Set read-only filesystem
FROM alpine:latest
RUN adduser -D -s /bin/sh appuser
USER appuser
CMD ["./app"]
```

#### Secrets Management
```yaml
# GitHub Actions secrets
- name: Build with secrets
  run: |
    echo "${{ secrets.DATABASE_PASSWORD }}" | docker secret create db_password -
    docker service create --secret db_password myapp:latest

# Docker Compose secrets
version: '3.8'
services:
  app:
    image: myapp:latest
    secrets:
      - db_password
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    external: true
```

### Vulnerability Scanning

#### Integrated Scanning Pipeline
```yaml
name: Security Scan

on: [push, pull_request]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build image
      run: docker build -t ${{ github.repository }}:${{ github.sha }} .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: '${{ github.repository }}:${{ github.sha }}'
        format: 'table'
        exit-code: '1'
        ignore-unfixed: true
        severity: 'CRITICAL,HIGH'
    
    - name: Run Snyk to check Docker image
      uses: snyk/actions/docker@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        image: '${{ github.repository }}:${{ github.sha }}'
        args: --severity-threshold=high
```

### Runtime Security

#### Security Profiles
```yaml
# docker-compose.yml with security options
version: '3.8'
services:
  app:
    image: myapp:latest
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-profile
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
```

---

## 10. Performance Optimization {#performance}

### Build Performance

#### Layer Caching
```dockerfile
# Good: Leverage build cache
FROM node:18-alpine

WORKDIR /app

# Cache dependencies separately
COPY package*.json ./
RUN npm ci --only=production

# App code changes more frequently
COPY . .

CMD ["npm", "start"]
```

#### BuildKit Features
```dockerfile
# syntax=docker/dockerfile:1
FROM node:18-alpine

# Mount cache for npm
RUN --mount=type=cache,target=/root/.npm \
    npm install -g npm@latest

# Mount secrets
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm ci --only=production
```

#### Multi-Platform Builds
```bash
# Create buildx builder
docker buildx create --name mybuilder --use

# Build for multiple platforms
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t myapp:latest \
    --push .
```

### Runtime Performance

#### Resource Limits
```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

#### Health Checks
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY . .

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["npm", "start"]
```

---

## 11. Hands-on Exercises {#exercises}

### Exercise 1: Containerize a Web Application

**Task**: Containerize a simple Node.js application

**Setup**:
```javascript
// app.js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.json({ 
        message: 'Hello Docker!',
        timestamp: new Date().toISOString(),
        version: process.env.APP_VERSION || '1.0.0'
    });
});

app.get('/health', (req, res) => {
    res.json({ status: 'healthy' });
});

app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
```

```json
{
    "name": "docker-exercise",
    "version": "1.0.0",
    "main": "app.js",
    "scripts": {
        "start": "node app.js",
        "dev": "nodemon app.js"
    },
    "dependencies": {
        "express": "^4.18.2"
    },
    "devDependencies": {
        "nodemon": "^2.0.22"
    }
}
```

**Your Tasks**:
1. Write a Dockerfile
2. Build the image
3. Run the container
4. Test the application

<details>
<summary>Solution</summary>

```dockerfile
# Dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Copy application code
COPY --chown=nextjs:nodejs . .

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
```

```bash
# Build and run commands
docker build -t my-node-app:1.0.0 .
docker run -d -p 3000:3000 --name my-app my-node-app:1.0.0
curl http://localhost:3000
curl http://localhost:3000/health
docker logs my-app
docker stop my-app && docker rm my-app
```
</details>

### Exercise 2: Multi-Stage Build

**Task**: Create a multi-stage build for a React application

**Setup**:
```json
{
    "name": "react-docker-exercise",
    "version": "1.0.0",
    "private": true,
    "dependencies": {
        "react": "^18.2.0",
        "react-dom": "^18.2.0",
        "react-scripts": "5.0.1"
    },
    "scripts": {
        "start": "react-scripts start",
        "build": "react-scripts build",
        "test": "react-scripts test",
        "eject": "react-scripts eject"
    }
}
```

**Your Tasks**:
1. Create a multi-stage Dockerfile
2. Optimize for production
3. Add development target
4. Configure nginx

<details>
<summary>Solution</summary>

```dockerfile
# Multi-stage Dockerfile for React app
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./

# Development stage
FROM base AS development
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]

# Build stage
FROM base AS builder
RUN npm ci --only=production
COPY . .
RUN npm run build

# Test stage
FROM builder AS test
RUN npm install --only=dev
RUN npm test -- --coverage --watchAll=false

# Production stage
FROM nginx:alpine AS production

# Copy built assets
COPY --from=builder /app/build /usr/share/nginx/html

# Copy custom nginx config
COPY nginx.conf /etc/nginx/nginx.conf

# Add health check
RUN apk add --no-cache curl
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:80/ || exit 1

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
```

```bash
# Build commands
docker build --target development -t react-app:dev .
docker build --target production -t react-app:prod .
docker build --target test -t react-app:test .

# Run commands
docker run -d -p 3000:3000 react-app:dev
docker run -d -p 8080:80 react-app:prod
docker run --rm react-app:test
```
</details>

### Exercise 3: Docker Compose Development Environment

**Task**: Create a complete development environment with Docker Compose

**Requirements**:
- Node.js API server
- PostgreSQL database
- Redis cache
- Development volumes
- Environment variables

<details>
<summary>Solution</summary>

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      target: development
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://user:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:13
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: unless-stopped

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - db
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  default:
    name: myapp-network
```

```yaml
# docker-compose.override.yml (for local development)
version: '3.8'

services:
  api:
    environment:
      - DEBUG=myapp:*
      - LOG_LEVEL=debug
    volumes:
      - .:/app:cached
      - /app/node_modules

  db:
    environment:
      - POSTGRES_LOG_STATEMENT=all
    ports:
      - "5432:5432"
```

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  api-test:
    build:
      context: .
      target: test
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://user:password@db-test:5432/test_db
    depends_on:
      - db-test

  db-test:
    image: postgres:13
    environment:
      POSTGRES_DB: test_db
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    tmpfs:
      - /var/lib/postgresql/data
```

```bash
# Development commands
docker-compose up -d
docker-compose logs -f api
docker-compose exec api npm run migrate
docker-compose exec db psql -U user -d myapp

# Testing commands
docker-compose -f docker-compose.test.yml up --build --abort-on-container-exit
docker-compose -f docker-compose.test.yml down

# Production build
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
</details>

---

## 12. Troubleshooting {#troubleshooting}

### Common Issues and Solutions

#### 1. Image Build Failures

**Problem**: Build context too large
```bash
ERROR: failed to solve: failed to read dockerfile: open /var/lib/docker/tmp/buildkit-mount123456789/Dockerfile: no such file or directory
```

**Solution**: Use .dockerignore
```dockerfile
# .dockerignore
node_modules
.git
.docker
*.log
coverage
.pytest_cache
__pycache__
.DS_Store
Thumbs.db
```

#### 2. Container Won't Start

**Problem**: Container exits immediately
```bash
docker run myapp:latest
# Container exits with code 125
```

**Solution**: Debug with interactive mode
```bash
# Run interactively
docker run -it myapp:latest sh

# Check logs
docker logs container_name

# Override entrypoint
docker run -it --entrypoint sh myapp:latest
```

#### 3. Port Binding Issues

**Problem**: Port already in use
```bash
Error: bind: address already in use
```

**Solution**: Check and kill processes
```bash
# Find process using port
sudo lsof -i :3000
netstat -tulpn | grep :3000

# Kill process
kill -9 PID

# Use different port
docker run -p 3001:3000 myapp:latest
```

#### 4. Permission Issues

**Problem**: Permission denied in container
```bash
npm ERR! Error: EACCES: permission denied, mkdir '/app/node_modules'
```

**Solution**: Fix user permissions
```dockerfile
# Create user with specific UID/GID
RUN addgroup --gid 1001 nodejs && \
    adduser --system --uid 1001 nextjs --ingroup nodejs

# Set correct ownership
COPY --chown=nextjs:nodejs . .
USER nextjs
```

#### 5. Networking Issues

**Problem**: Cannot connect between containers
```bash
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**Solution**: Use service names
```yaml
# docker-compose.yml
services:
  api:
    environment:
      # Use service name, not localhost
      - DATABASE_URL=postgresql://user:password@db:5432/myapp
  db:
    image: postgres:13
```

### Debugging Techniques

#### Container Inspection
```bash
# Inspect container
docker inspect container_name

# Check resource usage
docker stats

# View processes
docker top container_name

# Execute commands
docker exec -it container_name bash
docker exec -it container_name ps aux
```

#### Log Analysis
```bash
# View logs
docker logs container_name
docker logs -f container_name
docker logs --tail 50 container_name

# Compose logs
docker-compose logs service_name
docker-compose logs -f
```

#### Image Analysis
```bash
# View image layers
docker history myapp:latest

# Analyze image size
docker images
dive myapp:latest  # Using dive tool

# Check image vulnerabilities
trivy image myapp:latest
```

---

## Summary 📋

In this comprehensive module, you learned:

✅ **Docker Fundamentals**
- Container concepts and architecture
- Docker engine components and workflow
- Basic container operations and management

✅ **Dockerfile Mastery**
- Writing efficient Dockerfiles
- Best practices for different languages
- Layer optimization and caching strategies

✅ **Multi-Stage Builds**
- Advanced build patterns for production
- Development vs production environments
- Build optimization techniques

✅ **Container Registries**
- Working with Docker Hub and GitHub Container Registry
- AWS ECR integration and best practices
- Image tagging and scanning strategies

✅ **CI/CD Integration**
- GitHub Actions with Docker workflows
- Automated testing and security scanning
- Multi-platform builds and deployments

✅ **Security and Performance**
- Container security best practices
- Vulnerability scanning and runtime security
- Performance optimization techniques

### Next Steps 🚀

1. **Practice**: Containerize your existing applications
2. **Experiment**: Try different base images and optimization techniques
3. **Secure**: Implement security scanning in your pipelines
4. **Optimize**: Focus on build performance and image size
5. **Module 10**: Move on to Container-based CI/CD pipelines

### Key Takeaways 💡

- **Consistency**: Containers ensure "works everywhere" reliability
- **Efficiency**: Multi-stage builds optimize for size and security
- **Security**: Always scan images and run as non-root users
- **Performance**: Leverage layer caching and optimize Dockerfiles
- **Integration**: Docker is essential for modern CI/CD pipelines

---

**Congratulations! 🎉** You've mastered Docker fundamentals for CI/CD. You're now ready to build robust, secure, and efficient containerized applications!

**Next Module**: [Container-based CI/CD →](./02-container-cicd.md)
