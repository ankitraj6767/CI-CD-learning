# Module 15: Multi-service Pipelines

## Learning Objectives

By the end of this module, you will be able to:

- Design CI/CD pipelines for microservices architectures
- Implement service dependency management and orchestration
- Configure parallel and sequential pipeline execution
- Implement cross-service testing strategies
- Manage service versioning and compatibility
- Implement distributed tracing and monitoring
- Handle service deployment coordination
- Optimize pipeline performance for multiple services

---

## 1. Microservices CI/CD Architecture {#architecture}

### Microservices Pipeline Challenges

```mermaid
graph TB
    subgraph "Monolith Pipeline"
        M1[Build] --> M2[Test] --> M3[Deploy]
    end
    
    subgraph "Microservices Pipelines"
        subgraph "Service A"
            A1[Build A] --> A2[Test A] --> A3[Deploy A]
        end
        
        subgraph "Service B"
            B1[Build B] --> B2[Test B] --> B3[Deploy B]
        end
        
        subgraph "Service C"
            C1[Build C] --> C2[Test C] --> C3[Deploy C]
        end
        
        subgraph "Integration"
            I1[Integration Tests]
            I2[E2E Tests]
            I3[Performance Tests]
        end
        
        A3 --> I1
        B3 --> I1
        C3 --> I1
        I1 --> I2
        I2 --> I3
    end
```

### Pipeline Orchestration Strategies

#### 1. Independent Pipelines

```yaml
# .github/workflows/service-a.yml
name: Service A Pipeline

on:
  push:
    paths:
    - 'services/service-a/**'
    - '.github/workflows/service-a.yml'
  pull_request:
    paths:
    - 'services/service-a/**'

env:
  SERVICE_NAME: service-a
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}/service-a

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      service-changed: ${{ steps.changes.outputs.service-a }}
    steps:
    - uses: actions/checkout@v4
    - uses: dorny/paths-filter@v2
      id: changes
      with:
        filters: |
          service-a:
            - 'services/service-a/**'

  build-and-test:
    needs: changes
    if: needs.changes.outputs.service-changed == 'true'
    runs-on: ubuntu-latest
    
    defaults:
      run:
        working-directory: services/service-a
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: services/service-a/package-lock.json
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm test
    
    - name: Run integration tests
      run: npm run test:integration
    
    - name: Build application
      run: npm run build
    
    - name: Build Docker image
      run: |
        docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .
        docker build -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest .
    
    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Push Docker image
      run: |
        docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
        docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
    
    outputs:
      image-tag: ${{ github.sha }}

  deploy-staging:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Deploy to staging
      run: |
        # Update Kubernetes manifests
        sed -i 's|image: .*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build-and-test.outputs.image-tag }}|' \
          k8s/staging/service-a-deployment.yaml
        
        # Apply to cluster
        kubectl apply -f k8s/staging/service-a-deployment.yaml
        kubectl rollout status deployment/service-a -n staging --timeout=300s
    
    - name: Run smoke tests
      run: |
        # Wait for service to be ready
        kubectl wait --for=condition=ready pod -l app=service-a -n staging --timeout=300s
        
        # Get service URL
        SERVICE_URL=$(kubectl get service service-a -n staging -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
        
        # Run smoke tests
        curl -f http://$SERVICE_URL/health
        curl -f http://$SERVICE_URL/api/version
    
    outputs:
      deployment-status: success

  trigger-integration-tests:
    needs: deploy-staging
    runs-on: ubuntu-latest
    
    steps:
    - name: Trigger integration test workflow
      uses: peter-evans/repository-dispatch@v2
      with:
        token: ${{ secrets.GITHUB_TOKEN }}
        event-type: service-deployed
        client-payload: |
          {
            "service": "service-a",
            "version": "${{ needs.build-and-test.outputs.image-tag }}",
            "environment": "staging"
          }
```

#### 2. Coordinated Pipeline

```yaml
# .github/workflows/multi-service-pipeline.yml
name: Multi-Service Pipeline

on:
  push:
    branches: [ main ]
  workflow_dispatch:
    inputs:
      services:
        description: 'Services to deploy (comma-separated)'
        required: false
        default: 'all'

env:
  REGISTRY: ghcr.io

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      changed-services: ${{ steps.changes.outputs.changes }}
      build-matrix: ${{ steps.matrix.outputs.matrix }}
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - uses: dorny/paths-filter@v2
      id: changes
      with:
        list-files: json
        filters: |
          service-a:
            - 'services/service-a/**'
          service-b:
            - 'services/service-b/**'
          service-c:
            - 'services/service-c/**'
          shared:
            - 'shared/**'
    
    - name: Create build matrix
      id: matrix
      run: |
        services=[]
        if [[ "${{ steps.changes.outputs.service-a }}" == "true" ]]; then
          services+='{"name":"service-a","path":"services/service-a"}'
        fi
        if [[ "${{ steps.changes.outputs.service-b }}" == "true" ]]; then
          services+='{"name":"service-b","path":"services/service-b"}'
        fi
        if [[ "${{ steps.changes.outputs.service-c }}" == "true" ]]; then
          services+='{"name":"service-c","path":"services/service-c"}'
        fi
        
        matrix=$(printf '%s\n' "${services[@]}" | jq -s .)
        echo "matrix=$matrix" >> $GITHUB_OUTPUT

  build-test-parallel:
    needs: detect-changes
    if: needs.detect-changes.outputs.build-matrix != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.build-matrix) }}
      fail-fast: false
      max-parallel: 3
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: ${{ matrix.service.path }}/package-lock.json
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ${{ matrix.service.path }}/node_modules
          ~/.npm
        key: ${{ runner.os }}-node-${{ hashFiles(format('{0}/package-lock.json', matrix.service.path)) }}
    
    - name: Install dependencies
      working-directory: ${{ matrix.service.path }}
      run: npm ci
    
    - name: Run tests in parallel
      working-directory: ${{ matrix.service.path }}
      run: |
        npm run test:unit &
        npm run test:integration &
        npm run lint &
        wait
    
    - name: Build Docker image
      run: |
        docker build -t ${{ env.REGISTRY }}/${{ github.repository }}/${{ matrix.service.name }}:${{ github.sha }} ${{ matrix.service.path }}
        docker push ${{ env.REGISTRY }}/${{ github.repository }}/${{ matrix.service.name }}:${{ github.sha }}

  integration-tests:
    needs: build-test-parallel
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Run integration tests
      run: |
        export IMAGE_TAG=${{ github.sha }}
        docker-compose -f docker-compose.test.yml up --abort-on-container-exit
        docker-compose -f docker-compose.test.yml down -v
```

---

## 2. Service Dependency Management {#dependency-management}

### Dependency Graph Definition

```yaml
# .github/dependency-graph.yml
services:
  service-a:
    dependencies: []
    consumers: [service-b, service-c]
    deployment_order: 1
    
  service-b:
    dependencies: [service-a]
    consumers: [service-c]
    deployment_order: 2
    
  service-c:
    dependencies: [service-a, service-b]
    consumers: []
    deployment_order: 3
    
  database:
    dependencies: []
    consumers: [service-a, service-b, service-c]
    deployment_order: 0
    
  message-queue:
    dependencies: []
    consumers: [service-b, service-c]
    deployment_order: 0
```

### Dependency-Aware Deployment

```python
# scripts/dependency-manager.py
import yaml
import json
import sys
from typing import Dict, List, Set

class DependencyManager:
    def __init__(self, config_file: str):
        with open(config_file, 'r') as f:
            self.config = yaml.safe_load(f)
        self.services = self.config['services']
    
    def get_deployment_order(self, changed_services: List[str]) -> List[List[str]]:
        """Get deployment order based on dependencies"""
        # Find all services that need to be deployed
        services_to_deploy = set(changed_services)
        
        # Add consumers of changed services
        for service in changed_services:
            services_to_deploy.update(self.services[service].get('consumers', []))
        
        # Sort by deployment order
        ordered_services = sorted(
            services_to_deploy,
            key=lambda s: self.services[s].get('deployment_order', 999)
        )
        
        # Group by deployment order
        deployment_batches = []
        current_order = None
        current_batch = []
        
        for service in ordered_services:
            order = self.services[service].get('deployment_order', 999)
            if current_order is None or order != current_order:
                if current_batch:
                    deployment_batches.append(current_batch)
                current_batch = [service]
                current_order = order
            else:
                current_batch.append(service)
        
        if current_batch:
            deployment_batches.append(current_batch)
        
        return deployment_batches
    
    def validate_dependencies(self, service: str, available_services: Set[str]) -> bool:
        """Check if all dependencies are available"""
        dependencies = self.services[service].get('dependencies', [])
        return all(dep in available_services for dep in dependencies)
    
    def get_health_check_order(self, services: List[str]) -> List[str]:
        """Get order for health checks based on dependencies"""
        # Dependencies should be checked first
        return sorted(services, key=lambda s: self.services[s].get('deployment_order', 999))

if __name__ == '__main__':
    if len(sys.argv) != 2:
        print("Usage: python dependency-manager.py '<changed_services_json>'")
        sys.exit(1)
    
    changed_services = json.loads(sys.argv[1])
    manager = DependencyManager('.github/dependency-graph.yml')
    
    deployment_batches = manager.get_deployment_order(changed_services)
    
    # Output for GitHub Actions
    print(f"::set-output name=deployment-batches::{json.dumps(deployment_batches)}")
    
    for i, batch in enumerate(deployment_batches):
        print(f"Batch {i + 1}: {', '.join(batch)}")
```

---

## 3. Cross-Service Testing {#cross-service-testing}

### Integration Test Environment

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  service-a:
    build:
      context: ./services/service-a
      dockerfile: Dockerfile.test
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://test:test@postgres:5432/testdb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis
    ports:
      - "3001:3000"
    networks:
      - test-network

  service-b:
    build:
      context: ./services/service-b
      dockerfile: Dockerfile.test
    environment:
      - NODE_ENV=test
      - SERVICE_A_URL=http://service-a:3000
      - DATABASE_URL=postgresql://test:test@postgres:5432/testdb
    depends_on:
      - service-a
      - postgres
    ports:
      - "3002:3000"
    networks:
      - test-network

  service-c:
    build:
      context: ./services/service-c
      dockerfile: Dockerfile.test
    environment:
      - NODE_ENV=test
      - SERVICE_A_URL=http://service-a:3000
      - SERVICE_B_URL=http://service-b:3000
      - MESSAGE_QUEUE_URL=amqp://guest:guest@rabbitmq:5672
    depends_on:
      - service-a
      - service-b
      - rabbitmq
    ports:
      - "3003:3000"
    networks:
      - test-network

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=testdb
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
    ports:
      - "5432:5432"
    networks:
      - test-network

  redis:
    image: redis:6
    ports:
      - "6379:6379"
    networks:
      - test-network

  rabbitmq:
    image: rabbitmq:3-management
    environment:
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
    ports:
      - "5672:5672"
      - "15672:15672"
    networks:
      - test-network

networks:
  test-network:
    driver: bridge
```

---

## 4. Service Versioning and Compatibility {#versioning}

### Semantic Versioning Strategy

```yaml
# .github/workflows/version-management.yml
name: Version Management

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  version-check:
    runs-on: ubuntu-latest
    outputs:
      version-bump: ${{ steps.version.outputs.bump-type }}
      new-version: ${{ steps.version.outputs.new-version }}
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Analyze commits for version bump
      id: version
      run: |
        # Get commit messages since last tag
        LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "v0.0.0")
        COMMITS=$(git log $LAST_TAG..HEAD --oneline)
        
        # Determine version bump type
        if echo "$COMMITS" | grep -q "BREAKING CHANGE\|!:"; then
          BUMP_TYPE="major"
        elif echo "$COMMITS" | grep -q "^feat"; then
          BUMP_TYPE="minor"
        elif echo "$COMMITS" | grep -q "^fix\|^perf"; then
          BUMP_TYPE="patch"
        else
          BUMP_TYPE="none"
        fi
        
        echo "bump-type=$BUMP_TYPE" >> $GITHUB_OUTPUT
        
        # Calculate new version
        if [ "$BUMP_TYPE" != "none" ]; then
          CURRENT_VERSION=$(echo $LAST_TAG | sed 's/v//')
          NEW_VERSION=$(npx semver -i $BUMP_TYPE $CURRENT_VERSION)
          echo "new-version=v$NEW_VERSION" >> $GITHUB_OUTPUT
        fi
```

---

## 5. Pipeline Optimization {#optimization}

### Parallel Execution Strategy

```yaml
# .github/workflows/optimized-pipeline.yml
name: Optimized Multi-Service Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  NODE_VERSION: '18'

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.changes.outputs.matrix }}
      has-changes: ${{ steps.changes.outputs.has-changes }}
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Detect service changes
      id: changes
      run: |
        # Get changed files
        if [ "${{ github.event_name }}" = "pull_request" ]; then
          CHANGED_FILES=$(git diff --name-only ${{ github.event.pull_request.base.sha }} ${{ github.sha }})
        else
          CHANGED_FILES=$(git diff --name-only HEAD~1 HEAD)
        fi
        
        # Detect which services changed
        SERVICES_CHANGED=$(echo "$CHANGED_FILES" | grep -E '^services/' | cut -d'/' -f2 | sort -u | jq -R -s -c 'split("\n")[:-1]')
        
        if [ "$SERVICES_CHANGED" = "[]" ]; then
          echo "has-changes=false" >> $GITHUB_OUTPUT
          echo "matrix=[]" >> $GITHUB_OUTPUT
        else
          echo "has-changes=true" >> $GITHUB_OUTPUT
          echo "matrix=$SERVICES_CHANGED" >> $GITHUB_OUTPUT
        fi
        
        echo "Changed services: $SERVICES_CHANGED"

  build-and-test:
    needs: detect-changes
    if: needs.detect-changes.outputs.has-changes == 'true'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: ${{ fromJson(needs.detect-changes.outputs.matrix) }}
      max-parallel: 3
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'
        cache-dependency-path: services/${{ matrix.service }}/package-lock.json
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          services/${{ matrix.service }}/node_modules
          ~/.npm
        key: ${{ runner.os }}-node-${{ matrix.service }}-${{ hashFiles('services/${{ matrix.service }}/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-node-${{ matrix.service }}-
          ${{ runner.os }}-node-
    
    - name: Install dependencies
      working-directory: services/${{ matrix.service }}
      run: npm ci --prefer-offline --no-audit
    
    - name: Run linting
      working-directory: services/${{ matrix.service }}
      run: npm run lint
    
    - name: Run unit tests
      working-directory: services/${{ matrix.service }}
      run: npm run test:unit -- --coverage
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: services/${{ matrix.service }}/coverage/lcov.info
        flags: ${{ matrix.service }}
    
    - name: Build application
      working-directory: services/${{ matrix.service }}
      run: npm run build
    
    - name: Build Docker image
      working-directory: services/${{ matrix.service }}
      run: |
        docker build -t ${{ env.REGISTRY }}/${{ github.repository }}/${{ matrix.service }}:${{ github.sha }} .
        docker build -t ${{ env.REGISTRY }}/${{ github.repository }}/${{ matrix.service }}:latest .
    
    - name: Push Docker image
      if: github.ref == 'refs/heads/main'
      run: |
        echo ${{ secrets.GITHUB_TOKEN }} | docker login ${{ env.REGISTRY }} -u ${{ github.actor }} --password-stdin
        docker push ${{ env.REGISTRY }}/${{ github.repository }}/${{ matrix.service }}:${{ github.sha }}
        docker push ${{ env.REGISTRY }}/${{ github.repository }}/${{ matrix.service }}:latest
```

### Cache Optimization

```yaml
# .github/workflows/cache-optimization.yml
name: Cache Optimization Example

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    # Multi-level caching strategy
    - name: Cache Docker layers
      uses: actions/cache@v3
      with:
        path: /tmp/.buildx-cache
        key: ${{ runner.os }}-buildx-${{ github.sha }}
        restore-keys: |
          ${{ runner.os }}-buildx-
    
    - name: Cache npm dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.npm
          node_modules
          */node_modules
        key: ${{ runner.os }}-npm-${{ hashFiles('**/package-lock.json') }}
        restore-keys: |
          ${{ runner.os }}-npm-
    
    - name: Cache test results
      uses: actions/cache@v3
      with:
        path: |
          coverage
          test-results
        key: ${{ runner.os }}-tests-${{ hashFiles('**/*.js', '**/*.json') }}
        restore-keys: |
          ${{ runner.os }}-tests-
    
    # Build with cache mount
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Build and push
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ env.REGISTRY }}/myapp:${{ github.sha }}
        cache-from: type=local,src=/tmp/.buildx-cache
        cache-to: type=local,dest=/tmp/.buildx-cache-new,mode=max
    
    - name: Move cache
      run: |
        rm -rf /tmp/.buildx-cache
        mv /tmp/.buildx-cache-new /tmp/.buildx-cache
```

---

## 6. Distributed Tracing and Monitoring {#monitoring}

### OpenTelemetry Integration

```javascript
// shared/tracing/tracer.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

const jaegerExporter = new JaegerExporter({
  endpoint: process.env.JAEGER_ENDPOINT || 'http://localhost:14268/api/traces',
});

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME || 'unknown-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.SERVICE_VERSION || '1.0.0',
  }),
  traceExporter: jaegerExporter,
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

module.exports = sdk;
```

### Prometheus Metrics

```javascript
// shared/metrics/prometheus.js
const client = require('prom-client');

// Create a Registry
const register = new client.Registry();

// Add default metrics
client.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code', 'service'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
});

const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code', 'service'],
});

const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  labelNames: ['service'],
});

// Register metrics
register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestsTotal);
register.registerMetric(activeConnections);

module.exports = {
  register,
  httpRequestDuration,
  httpRequestsTotal,
  activeConnections,
};
```

### Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "id": null,
    "title": "Multi-Service Pipeline Monitoring",
    "tags": ["microservices", "pipeline"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate by Service",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{service}} - {{method}} {{route}}"
          }
        ],
        "yAxes": [
          {
            "label": "Requests/sec"
          }
        ]
      },
      {
        "id": 2,
        "title": "Response Time P95",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "{{service}} P95"
          }
        ]
      },
      {
        "id": 3,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total{status_code=~\"5..\"}[5m]) / rate(http_requests_total[5m])",
            "legendFormat": "{{service}} Error Rate"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
```

---

## 7. Best Practices {#best-practices}

### Pipeline Design Principles

1. **Fail Fast**: Detect issues early in the pipeline
2. **Parallel Execution**: Run independent tasks concurrently
3. **Dependency Management**: Respect service dependencies
4. **Resource Optimization**: Use caching and efficient resource allocation
5. **Monitoring**: Implement comprehensive observability

### Service Communication Patterns

```javascript
// shared/patterns/circuit-breaker.js
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.monitoringPeriod = options.monitoringPeriod || 10000;
    
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.lastFailureTime = null;
    this.successCount = 0;
  }
  
  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() - this.lastFailureTime > this.resetTimeout) {
        this.state = 'HALF_OPEN';
        this.successCount = 0;
      } else {
        throw new Error('Circuit breaker is OPEN');
      }
    }
    
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }
  
  onSuccess() {
    this.failureCount = 0;
    
    if (this.state === 'HALF_OPEN') {
      this.successCount++;
      if (this.successCount >= 3) {
        this.state = 'CLOSED';
      }
    }
  }
  
  onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'OPEN';
    }
  }
}

module.exports = CircuitBreaker;
```

---

## 8. Hands-on Exercises {#exercises}

### Exercise 1: Multi-Service Pipeline Setup

**Objective**: Create a coordinated pipeline for 3 microservices

**Tasks**:
1. Set up service change detection
2. Implement parallel build and test
3. Configure dependency-aware deployment
4. Add integration testing

### Exercise 2: Contract Testing Implementation

**Objective**: Implement contract testing between services

**Tasks**:
1. Set up Pact consumer tests
2. Configure provider verification
3. Integrate with CI/CD pipeline
4. Add contract compatibility checks

### Exercise 3: Performance Testing Pipeline

**Objective**: Add performance testing to multi-service pipeline

**Tasks**:
1. Create load testing scenarios
2. Set up performance benchmarks
3. Implement automated performance regression detection
4. Configure performance monitoring

---

## 9. Troubleshooting {#troubleshooting}

### Common Issues

#### Service Dependencies

**Problem**: Services fail to start due to dependency issues

**Solutions**:
- Implement health checks and readiness probes
- Use dependency management tools
- Add retry mechanisms with exponential backoff

#### Pipeline Timing

**Problem**: Race conditions in parallel deployments

**Solutions**:
- Use proper dependency ordering
- Implement deployment locks
- Add synchronization points

#### Resource Conflicts

**Problem**: Multiple pipelines competing for resources

**Solutions**:
- Implement resource quotas
- Use pipeline queuing
- Optimize resource allocation

### Debugging Tools

```bash
# Check service health
kubectl get pods -l app=service-a
kubectl describe pod <pod-name>
kubectl logs <pod-name> --follow

# Check pipeline status
gh workflow list
gh run list --workflow="multi-service-pipeline.yml"
gh run view <run-id>

# Monitor resource usage
kubectl top pods
kubectl top nodes

# Check service connectivity
kubectl exec -it <pod-name> -- curl http://service-b:3000/health
```

---