# Module 6: Simple CI Pipeline 🔄

> **Learning Objective**: Build complete CI pipelines that integrate with Git workflows, handle build failures gracefully, and implement industry-standard CI practices.

## Table of Contents
1. [Understanding CI Pipelines](#understanding-ci)
2. [Git Integration Strategies](#git-integration)
3. [Building Your First Complete CI Pipeline](#first-complete-pipeline)
4. [Handling Build Failures](#handling-failures)
5. [Artifact Management](#artifact-management)
6. [Branch Protection & Quality Gates](#quality-gates)
7. [Real-World CI Patterns](#real-world-patterns)
8. [Troubleshooting & Debugging](#troubleshooting)
9. [Hands-On Project](#hands-on-project)
10. [Assessment](#assessment)

## Understanding CI Pipelines {#understanding-ci}

### What Makes a Complete CI Pipeline?

A complete CI pipeline is more than just running tests. It's a comprehensive system that:

1. **Validates every change** before it reaches the main branch
2. **Provides fast feedback** to developers
3. **Maintains code quality** through automated checks
4. **Creates deployable artifacts** consistently
5. **Prevents broken code** from reaching production

### Essential CI Pipeline Components

#### 1. Source Code Management Integration
```yaml
# Trigger on the right events
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
    types: [opened, synchronize, reopened]
```

#### 2. Code Quality Checks
- **Linting**: Code style and syntax checking
- **Security scanning**: Vulnerability detection
- **Code coverage**: Test coverage measurement
- **Static analysis**: Code complexity and maintainability

#### 3. Build Process
- **Dependency management**: Install and cache dependencies
- **Compilation**: Build source code into artifacts
- **Asset optimization**: Minification, bundling
- **Environment-specific builds**: Different configurations

#### 4. Testing Strategy
- **Unit tests**: Individual component testing
- **Integration tests**: Component interaction testing
- **End-to-end tests**: Full application flow testing
- **Performance tests**: Load and stress testing

#### 5. Artifact Creation
- **Build artifacts**: Compiled code, binaries
- **Container images**: Docker images
- **Documentation**: API docs, user guides
- **Reports**: Test results, coverage reports

### CI Pipeline Lifecycle

```mermaid
graph LR
    A[Code Push] --> B[Trigger CI]
    B --> C[Checkout Code]
    C --> D[Install Dependencies]
    D --> E[Lint & Format]
    E --> F[Run Tests]
    F --> G[Build Application]
    G --> H[Security Scan]
    H --> I[Create Artifacts]
    I --> J[Deploy to Staging]
    J --> K[Integration Tests]
    K --> L[Quality Gates]
    L --> M{Pass?}
    M -->|Yes| N[Merge/Deploy]
    M -->|No| O[Notify Developer]
    O --> P[Fix Issues]
    P --> A
```

## Git Integration Strategies {#git-integration}

### Branching Strategies for CI

#### 1. GitHub Flow (Simple)
```yaml
name: GitHub Flow CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  ci:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      # Simple CI for feature branches
      - name: Install and Test
        run: |
          npm ci
          npm test
      
      # Deploy only from main
      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: npm run deploy
```

#### 2. Git Flow (Complex)
```yaml
name: Git Flow CI

on:
  push:
    branches: [ main, develop, 'release/*', 'hotfix/*' ]
  pull_request:
    branches: [ main, develop ]

jobs:
  # Fast feedback for all branches
  quick-checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Quick Lint and Unit Tests
        run: |
          npm ci
          npm run lint
          npm run test:unit

  # Comprehensive testing for important branches
  full-ci:
    if: contains(fromJSON('["main", "develop"]'), github.ref_name) || startsWith(github.ref, 'refs/heads/release/')
    runs-on: ubuntu-latest
    needs: quick-checks
    
    steps:
      - uses: actions/checkout@v4
      - name: Full Test Suite
        run: |
          npm ci
          npm run test:integration
          npm run test:e2e
          npm run build
          npm run security-scan

  # Deploy to different environments
  deploy:
    needs: full-ci
    runs-on: ubuntu-latest
    
    steps:
      - name: Deploy to Development
        if: github.ref == 'refs/heads/develop'
        run: echo "Deploy to dev environment"
      
      - name: Deploy to Staging
        if: startsWith(github.ref, 'refs/heads/release/')
        run: echo "Deploy to staging environment"
      
      - name: Deploy to Production
        if: github.ref == 'refs/heads/main'
        run: echo "Deploy to production environment"
```

#### 3. Feature Branch Strategy
```yaml
name: Feature Branch CI

on:
  pull_request:
    branches: [ main ]
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  # Skip draft PRs unless explicitly requested
  check-if-ready:
    runs-on: ubuntu-latest
    outputs:
      should-run: ${{ steps.check.outputs.result }}
    
    steps:
      - id: check
        run: |
          if [[ "${{ github.event.pull_request.draft }}" == "true" && "${{ github.event.action }}" != "ready_for_review" ]]; then
            echo "result=false" >> $GITHUB_OUTPUT
          else
            echo "result=true" >> $GITHUB_OUTPUT
          fi

  ci:
    needs: check-if-ready
    if: needs.check-if-ready.outputs.should-run == 'true'
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          # Fetch more history for better diff analysis
          fetch-depth: 0
      
      # Analyze what changed
      - name: Get changed files
        id: changed-files
        uses: tj-actions/changed-files@v40
        with:
          files_yaml: |
            src:
              - 'src/**'
            tests:
              - 'tests/**'
            docs:
              - 'docs/**'
              - '*.md'
            config:
              - 'package.json'
              - 'package-lock.json'
              - '.github/**'
      
      # Run appropriate checks based on changes
      - name: Install dependencies
        if: steps.changed-files.outputs.src_any_changed == 'true' || steps.changed-files.outputs.config_any_changed == 'true'
        run: npm ci
      
      - name: Run tests
        if: steps.changed-files.outputs.src_any_changed == 'true' || steps.changed-files.outputs.tests_any_changed == 'true'
        run: npm test
      
      - name: Build application
        if: steps.changed-files.outputs.src_any_changed == 'true'
        run: npm run build
      
      - name: Check documentation
        if: steps.changed-files.outputs.docs_any_changed == 'true'
        run: |
          npm run docs:build
          npm run docs:lint
```

### Advanced Git Integration

#### 1. Commit Status Integration
```yaml
name: Detailed Status Reporting

on:
  pull_request:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set pending status
        uses: myrotvorets/set-commit-status-action@master
        with:
          sha: ${{ github.event.pull_request.head.sha }}
          token: ${{ secrets.GITHUB_TOKEN }}
          status: pending
          context: 'lint'
          description: 'Running linting checks...'
      
      - name: Run Lint
        id: lint
        run: npm run lint
      
      - name: Set success status
        if: success()
        uses: myrotvorets/set-commit-status-action@master
        with:
          sha: ${{ github.event.pull_request.head.sha }}
          token: ${{ secrets.GITHUB_TOKEN }}
          status: success
          context: 'lint'
          description: 'Linting passed!'
      
      - name: Set failure status
        if: failure()
        uses: myrotvorets/set-commit-status-action@master
        with:
          sha: ${{ github.event.pull_request.head.sha }}
          token: ${{ secrets.GITHUB_TOKEN }}
          status: failure
          context: 'lint'
          description: 'Linting failed!'

  # Similar jobs for test, build, security, etc.
```

#### 2. PR Comments with Results
```yaml
name: PR Feedback

on:
  pull_request:
    branches: [ main ]

jobs:
  test-with-feedback:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run tests with coverage
        run: |
          npm ci
          npm run test:coverage
        continue-on-error: true
      
      - name: Comment PR with results
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            
            // Read test results
            let testResults = '';
            let coverage = '';
            
            try {
              testResults = fs.readFileSync('test-results.txt', 'utf8');
              coverage = fs.readFileSync('coverage/coverage-summary.txt', 'utf8');
            } catch (error) {
              console.log('Could not read test results:', error.message);
            }
            
            const body = `## 🧪 Test Results
            
            ### Coverage Report
            \`\`\`
            ${coverage}
            \`\`\`
            
            ### Detailed Results
            \`\`\`
            ${testResults}
            \`\`\`
            
            <details>
            <summary>📊 View detailed coverage report</summary>
            
            [Full Coverage Report](https://your-coverage-site.com)
            
            </details>
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: body
            });
```

## Building Your First Complete CI Pipeline {#first-complete-pipeline}

Let's build a comprehensive CI pipeline for a Node.js web application.

### Project Structure
```
my-web-app/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
├── src/
│   ├── index.js
│   ├── utils/
│   └── components/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docs/
├── package.json
├── .eslintrc.js
├── jest.config.js
└── Dockerfile
```

### Complete CI Pipeline Implementation

```yaml
name: Complete CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  NODE_VERSION: '18'
  CACHE_VERSION: 1

jobs:
  # Job 1: Code Quality & Security
  quality-checks:
    name: 🔍 Quality & Security
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for better analysis
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Cache ESLint
        uses: actions/cache@v3
        with:
          path: .eslintcache
          key: eslint-${{ env.CACHE_VERSION }}-${{ runner.os }}-${{ hashFiles('**/.eslintrc.*', '**/package-lock.json') }}
      
      - name: Lint Code
        run: |
          npm run lint -- --cache --format=json --output-file=lint-results.json
        continue-on-error: true
      
      - name: Format Check
        run: npm run format:check
      
      - name: Security Audit
        run: |
          npm audit --audit-level moderate
          npm run security:scan
      
      - name: Upload Lint Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: lint-results
          path: lint-results.json

  # Job 2: Unit Testing
  unit-tests:
    name: 🧪 Unit Tests
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16, 18, 20]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Unit Tests
        run: |
          npm run test:unit -- \
            --coverage \
            --testResultsProcessor=jest-sonar-reporter \
            --coverageReporters=lcov,json,text-summary
        env:
          CI: true
      
      - name: Upload Coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
          flags: unit-tests
          name: unit-tests-node${{ matrix.node-version }}
      
      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: unit-test-results-node${{ matrix.node-version }}
          path: |
            coverage/
            test-report.xml

  # Job 3: Integration Testing
  integration-tests:
    name: 🔗 Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Wait for Services
        run: |
          timeout 60 bash -c 'until nc -z localhost 5432; do sleep 1; done'
          timeout 60 bash -c 'until nc -z localhost 6379; do sleep 1; done'
      
      - name: Setup Test Database
        run: |
          npm run db:migrate
          npm run db:seed
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
      
      - name: Run Integration Tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          NODE_ENV: test

  # Job 4: Build Application
  build:
    name: 🏗️ Build Application
    runs-on: ubuntu-latest
    needs: [quality-checks, unit-tests]
    
    strategy:
      matrix:
        environment: [development, staging, production]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Build Application
        run: |
          npm run build:${{ matrix.environment }}
        env:
          NODE_ENV: ${{ matrix.environment }}
      
      - name: Generate Build Info
        run: |
          echo "Build Information" > build-info.txt
          echo "Environment: ${{ matrix.environment }}" >> build-info.txt
          echo "Commit: ${{ github.sha }}" >> build-info.txt
          echo "Branch: ${{ github.ref_name }}" >> build-info.txt
          echo "Build Time: $(date -u)" >> build-info.txt
          echo "Node Version: $(node --version)" >> build-info.txt
      
      - name: Upload Build Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.environment }}
          path: |
            dist/
            build-info.txt
          retention-days: 30

  # Job 5: End-to-End Testing
  e2e-tests:
    name: 🎭 E2E Tests
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Download Build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-development
          path: ./dist
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Start Application
        run: |
          npm start &
          sleep 10  # Wait for app to start
        env:
          NODE_ENV: test
          PORT: 3000
      
      - name: Run E2E Tests
        run: npm run test:e2e
        env:
          BASE_URL: http://localhost:3000
      
      - name: Upload E2E Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: e2e-results
          path: |
            playwright-report/
            test-results/

  # Job 6: Container Build
  container-build:
    name: 🐳 Container Build
    runs-on: ubuntu-latest
    needs: [build, integration-tests]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Download Build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-production
          path: ./dist
      
      - name: Build Container Image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: false
          tags: |
            my-app:latest
            my-app:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          outputs: type=docker,dest=/tmp/my-app.tar
      
      - name: Upload Container Image
        uses: actions/upload-artifact@v4
        with:
          name: container-image
          path: /tmp/my-app.tar

  # Job 7: Security Scanning
  security-scan:
    name: 🛡️ Security Scan
    runs-on: ubuntu-latest
    needs: container-build
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download Container Image
        uses: actions/download-artifact@v4
        with:
          name: container-image
          path: /tmp
      
      - name: Load Container Image
        run: docker load --input /tmp/my-app.tar
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'my-app:latest'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Run Container Security Scan
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            -v /tmp:/tmp anchore/grype my-app:latest

  # Job 8: Performance Testing
  performance-tests:
    name: ⚡ Performance Tests
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Download Build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-production
          path: ./dist
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Start Application
        run: |
          npm start &
          sleep 15
        env:
          NODE_ENV: production
          PORT: 3000
      
      - name: Install Artillery
        run: npm install -g artillery
      
      - name: Run Load Tests
        run: |
          artillery run tests/performance/load-test.yml \
            --output performance-results.json
      
      - name: Generate Performance Report
        run: |
          artillery report performance-results.json \
            --output performance-report.html
      
      - name: Upload Performance Results
        uses: actions/upload-artifact@v4
        with:
          name: performance-results
          path: |
            performance-results.json
            performance-report.html

  # Job 9: Deployment Readiness
  deployment-readiness:
    name: 🚀 Deployment Readiness
    runs-on: ubuntu-latest
    needs: [e2e-tests, security-scan, performance-tests]
    if: always()
    
    steps:
      - name: Check All Jobs Status
        run: |
          echo "E2E Tests: ${{ needs.e2e-tests.result }}"
          echo "Security Scan: ${{ needs.security-scan.result }}"
          echo "Performance Tests: ${{ needs.performance-tests.result }}"
          
          if [[ "${{ needs.e2e-tests.result }}" == "success" && 
                "${{ needs.security-scan.result }}" == "success" && 
                ("${{ needs.performance-tests.result }}" == "success" || "${{ needs.performance-tests.result }}" == "skipped") ]]; then
            echo "✅ Ready for deployment!"
            echo "deployment-ready=true" >> $GITHUB_OUTPUT
          else
            echo "❌ Not ready for deployment"
            echo "deployment-ready=false" >> $GITHUB_OUTPUT
            exit 1
          fi
        id: readiness-check
      
      - name: Create Deployment Badge
        if: steps.readiness-check.outputs.deployment-ready == 'true'
        run: |
          echo "Creating deployment readiness badge..."
          # This could integrate with your deployment system
```

## Handling Build Failures {#handling-failures}

### Failure Detection and Classification

#### 1. Categorizing Failures
```yaml
name: Smart Failure Handling

on: [push, pull_request]

jobs:
  test-with-failure-analysis:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup environment
        id: setup
        run: |
          npm ci
        continue-on-error: true
      
      - name: Analyze setup failure
        if: steps.setup.outcome == 'failure'
        run: |
          echo "Setup failed - likely dependency issue"
          echo "failure-type=dependency" >> $GITHUB_OUTPUT
          # Could automatically create issue or notify team
        id: failure-analysis
      
      - name: Run tests
        if: steps.setup.outcome == 'success'
        id: tests
        run: npm test
        continue-on-error: true
      
      - name: Analyze test failures
        if: steps.tests.outcome == 'failure'
        run: |
          # Parse test results to categorize failures
          if grep -q "timeout" test-results.xml; then
            echo "failure-type=timeout" >> $GITHUB_OUTPUT
          elif grep -q "assertion" test-results.xml; then
            echo "failure-type=logic" >> $GITHUB_OUTPUT
          else
            echo "failure-type=unknown" >> $GITHUB_OUTPUT
          fi
        id: test-analysis
      
      - name: Create failure report
        if: failure()
        uses: actions/github-script@v7
        with:
          script: |
            const failures = [];
            
            if ('${{ steps.setup.outcome }}' === 'failure') {
              failures.push({
                type: 'dependency',
                message: 'Setup failed - check dependencies',
                action: 'Review package.json and package-lock.json'
              });
            }
            
            if ('${{ steps.tests.outcome }}' === 'failure') {
              const testFailureType = '${{ steps.test-analysis.outputs.failure-type }}';
              failures.push({
                type: testFailureType,
                message: `Test failure: ${testFailureType}`,
                action: testFailureType === 'timeout' ? 'Increase test timeouts' : 'Review test logic'
              });
            }
            
            // Create detailed failure report
            const report = failures.map(f => 
              `## ${f.type.toUpperCase()} Failure\n\n**Message:** ${f.message}\n\n**Suggested Action:** ${f.action}`
            ).join('\n\n---\n\n');
            
            console.log('Failure Report:', report);
```

#### 2. Automatic Retry Logic
```yaml
name: Retry on Failure

jobs:
  test-with-retry:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Flaky test with retry
        uses: nick-fields/retry@v2
        with:
          timeout_minutes: 10
          max_attempts: 3
          retry_wait_seconds: 30
          command: |
            npm ci
            npm test
          on_retry_command: |
            echo "Test failed, cleaning up and retrying..."
            rm -rf node_modules
            npm cache clean --force
      
      - name: Network-dependent step with retry
        uses: nick-fields/retry@v2
        with:
          timeout_minutes: 5
          max_attempts: 5
          retry_wait_seconds: 60
          command: npm audit
          shell: bash
```

## Artifact Management {#artifact-management}

### Comprehensive Artifact Strategy

#### 1. Structured Artifact Organization
```yaml
name: Advanced Artifact Management

jobs:
  build-with-artifacts:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Application
        run: |
          npm ci
          npm run build
          npm run test:coverage
      
      # Separate artifacts by type and purpose
      - name: Upload Application Build
        uses: actions/upload-artifact@v4
        with:
          name: app-build-${{ github.sha }}
          path: |
            dist/
            !dist/**/*.map
          retention-days: 30
      
      - name: Upload Source Maps (Secure)
        uses: actions/upload-artifact@v4
        with:
          name: source-maps-${{ github.sha }}
          path: dist/**/*.map
          retention-days: 7  # Shorter retention for security
      
      - name: Upload Test Coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report-${{ github.sha }}
          path: |
            coverage/
          retention-days: 14
      
      - name: Upload Documentation
        uses: actions/upload-artifact@v4
        with:
          name: docs-${{ github.sha }}
          path: |
            docs/generated/
          retention-days: 60  # Keep docs longer
      
      # Create manifest of all artifacts
      - name: Create Artifact Manifest
        run: |
          cat << EOF > artifact-manifest.json
          {
            "build_id": "${{ github.run_id }}",
            "commit_sha": "${{ github.sha }}",
            "branch": "${{ github.ref_name }}",
            "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
            "artifacts": {
              "application": "app-build-${{ github.sha }}",
              "source_maps": "source-maps-${{ github.sha }}",
              "coverage": "coverage-report-${{ github.sha }}",
              "documentation": "docs-${{ github.sha }}"
            },
            "build_info": {
              "node_version": "$(node --version)",
              "npm_version": "$(npm --version)",
              "os": "${{ runner.os }}"
            }
          }
          EOF
      
      - name: Upload Manifest
        uses: actions/upload-artifact@v4
        with:
          name: manifest-${{ github.sha }}
          path: artifact-manifest.json
          retention-days: 90
```

## Branch Protection & Quality Gates {#quality-gates}

### Setting Up Branch Protection

#### 1. Basic Branch Protection Configuration
```yaml
# This goes in your repository settings, but here's the equivalent workflow checks
name: Quality Gates

on:
  pull_request:
    branches: [ main ]

jobs:
  # Required check: Code quality
  quality-gate:
    name: Quality Gate
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Lint check (Required)
        run: npm run lint
      
      - name: Type check (Required)
        run: npm run type-check
      
      - name: Unit tests (Required)
        run: npm run test:unit
      
      - name: Coverage threshold (Required)
        run: |
          npm run test:coverage
          # Check if coverage meets minimum threshold
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below minimum 80%"
            exit 1
          fi
          echo "Coverage: $COVERAGE% ✅"

  # Required check: Security
  security-gate:
    name: Security Gate
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Security audit
        run: |
          npm audit --audit-level moderate
      
      - name: License check
        run: |
          npx license-checker --onlyAllow 'MIT;Apache-2.0;BSD-3-Clause;BSD-2-Clause;ISC'
      
      - name: Secret scanning
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: main
          head: HEAD

  # Required check: Build verification
  build-gate:
    name: Build Gate
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build application
        run: npm run build
      
      - name: Verify build output
        run: |
          # Ensure critical files exist
          test -f dist/index.html || exit 1
          test -f dist/app.js || exit 1
          
          # Check bundle size
          BUNDLE_SIZE=$(stat -c%s dist/app.js)
          MAX_SIZE=1000000  # 1MB limit
          
          if [ $BUNDLE_SIZE -gt $MAX_SIZE ]; then
            echo "Bundle size $BUNDLE_SIZE exceeds limit $MAX_SIZE"
            exit 1
          fi
          
          echo "Build verification passed ✅"
```

## Real-World CI Patterns {#real-world-patterns}

### 1. Monorepo CI Strategy
```yaml
name: Monorepo CI

on: [push, pull_request]

jobs:
  # Detect which packages changed
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      packages: ${{ steps.changes.outputs.packages }}
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Detect changed packages
        id: changes
        run: |
          # Get list of changed files
          CHANGED_FILES=$(git diff --name-only ${{ github.event.before }} ${{ github.sha }})
          
          # Detect affected packages
          PACKAGES=$(echo "$CHANGED_FILES" | grep '^packages/' | cut -d'/' -f2 | sort -u | jq -R -s -c 'split("\n")[:-1]')
          
          echo "packages=$PACKAGES" >> $GITHUB_OUTPUT
          echo "Changed packages: $PACKAGES"

  # Build and test affected packages
  build-packages:
    needs: detect-changes
    if: ${{ needs.detect-changes.outputs.packages != '[]' }}
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        package: ${{ fromJson(needs.detect-changes.outputs.packages) }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build package
        run: |
          cd packages/${{ matrix.package }}
          npm run build
      
      - name: Test package
        run: |
          cd packages/${{ matrix.package }}
          npm test
      
      - name: Upload package artifacts
        uses: actions/upload-artifact@v4
        with:
          name: package-${{ matrix.package }}
          path: packages/${{ matrix.package }}/dist/
```

## Troubleshooting & Debugging {#troubleshooting}

### Common CI Issues and Solutions

#### 1. Debugging Workflow Issues
```yaml
name: Debug-Friendly CI

on: [push, pull_request]

jobs:
  debug-info:
    runs-on: ubuntu-latest
    
    steps:
      - name: Dump GitHub context
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
        run: echo "$GITHUB_CONTEXT"
      
      - name: Dump runner context
        env:
          RUNNER_CONTEXT: ${{ toJson(runner) }}
        run: echo "$RUNNER_CONTEXT"
      
      - name: Environment information
        run: |
          echo "=== System Info ==="
          uname -a
          echo "=== Environment Variables ==="
          env | sort
          echo "=== Disk Space ==="
          df -h
          echo "=== Memory ==="
          free -h

  test-with-debugging:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup with debugging
        run: |
          set -x  # Enable command echoing
          npm ci
        env:
          DEBUG: '*'  # Enable debug logging
      
      - name: Test with detailed output
        run: |
          # Enable verbose test output
          npm test -- --verbose --detectOpenHandles
        continue-on-error: true
      
      - name: Capture logs on failure
        if: failure()
        run: |
          echo "=== NPM Debug Log ==="
          cat ~/.npm/_logs/*.log || echo "No npm logs found"
          
          echo "=== Process List ==="
          ps aux
      
      - name: Upload debug artifacts
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: debug-logs-${{ github.run_id }}
          path: |
            ~/.npm/_logs/
          retention-days: 7
```

## Hands-On Project {#hands-on-project}

### Project: Complete CI Pipeline for a Real Application

Your task is to create a complete CI pipeline for a Node.js web application with the following requirements:

#### Requirements:
1. **Multi-stage pipeline** with quality gates
2. **Matrix testing** across Node.js versions
3. **Artifact management** with proper retention
4. **Failure handling** with retry logic
5. **Security scanning** integration
6. **Performance testing** on main branch
7. **Branch protection** setup

#### Deliverables:
1. Complete GitHub Actions workflow
2. Sample Node.js application
3. Test suites (unit, integration, e2e)
4. Documentation of CI strategy
5. Troubleshooting guide

## Assessment {#assessment}

### Practical Evaluation

Complete these tasks to demonstrate your CI pipeline mastery:

1. **Pipeline Creation** (40 points)
   - Create a complete CI pipeline for the provided application
   - Include all quality gates and testing stages
   - Implement proper error handling and notifications

2. **Git Integration** (20 points)
   - Configure branch protection rules
   - Implement different strategies for different branches
   - Set up proper PR workflows

3. **Performance Optimization** (20 points)
   - Optimize pipeline execution time
   - Implement proper caching strategies
   - Use parallel execution where appropriate

4. **Troubleshooting** (20 points)
   - Deliberately introduce and fix pipeline issues
   - Document debugging strategies
   - Implement monitoring and alerting

### Knowledge Assessment

1. **Multiple Choice**: Which job dependency strategy is most efficient for independent operations?
   a) Sequential execution
   b) Parallel execution with needs
   c) Matrix strategy
   d) Conditional execution

2. **Short Answer**: Explain the difference between `continue-on-error: true` and `if: failure()`.

3. **Code Review**: Identify issues in this workflow snippet:
   ```yaml
   jobs:
     test:
       runs-on: ubuntu-latest
       steps:
         - run: npm test
         - uses: actions/checkout@v4
         - run: npm ci
   ```

4. **Architecture Design**: Design a CI pipeline for a microservices application with 5 services.

### Success Criteria
- [ ] Successfully implement a complete CI pipeline
- [ ] Understand Git workflow integration
- [ ] Master artifact management and caching
- [ ] Implement effective error handling
- [ ] Optimize pipeline performance
- [ ] Demonstrate troubleshooting skills

---

**Next Module**: You're now ready for **Module 7: Automated Testing** where you'll dive deep into comprehensive testing strategies, test automation frameworks, and quality reporting.

**Estimated Completion Time**: 12-16 hours
**Key Skills Gained**: Complete CI pipeline design, Git integration, failure handling, performance optimization
