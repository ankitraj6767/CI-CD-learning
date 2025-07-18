# Module 5: GitHub Actions Basics 🚀

> **Learning Objective**: Master GitHub Actions fundamentals, understand YAML syntax, and create your first automated workflows.

## Table of Contents
1. [Introduction to GitHub Actions](#introduction)
2. [YAML Fundamentals for CI/CD](#yaml-fundamentals)
3. [Creating Your First Workflow](#first-workflow)
4. [Understanding Actions Marketplace](#actions-marketplace)
5. [Common Workflow Patterns](#workflow-patterns)
6. [Hands-On Exercises](#exercises)
7. [Best Practices](#best-practices)
8. [Assessment](#assessment)

## Introduction to GitHub Actions {#introduction}

### What is GitHub Actions?
GitHub Actions is a powerful CI/CD platform that allows you to automate your software development workflows directly in your GitHub repository. It enables you to:

- **Automate builds** when code is pushed
- **Run tests** on multiple environments
- **Deploy applications** to various platforms
- **Respond to repository events** (issues, PRs, releases)
- **Schedule recurring tasks**

### Key Concepts

#### 1. Workflows
- **Definition**: A configurable automated process made up of jobs
- **Trigger**: Events that cause workflows to run
- **Location**: `.github/workflows/` directory in your repository

#### 2. Events
Events that can trigger workflows:
```yaml
# Push to any branch
on: push

# Pull request events
on: pull_request

# Multiple events
on: [push, pull_request]

# Scheduled events
on:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight

# Manual trigger
on: workflow_dispatch
```

#### 3. Jobs
- Independent units of work that run in parallel by default
- Each job runs on a fresh virtual machine (runner)
- Jobs can depend on other jobs

#### 4. Steps
- Individual tasks within a job
- Can run shell commands or use actions
- Execute sequentially within a job

#### 5. Actions
- Reusable units of code
- Can be from GitHub marketplace or custom-built
- Encapsulate complex functionality

#### 6. Runners
- Servers that execute workflows
- GitHub-hosted or self-hosted
- Support multiple operating systems

## YAML Fundamentals for CI/CD {#yaml-fundamentals}

### YAML Basics
YAML (YAML Ain't Markup Language) is a human-readable data serialization standard used for configuration files.

#### Key YAML Rules:
```yaml
# Comments start with #
key: value                    # Key-value pairs
list:                        # Lists/Arrays
  - item1
  - item2
  - item3

nested:                      # Nested objects
  level2:
    key: value

multiline_string: |          # Literal block scalar
  This is a multiline string
  that preserves line breaks

folded_string: >             # Folded block scalar
  This is a long string
  that will be folded into
  a single line

boolean_values:
  - true
  - false
  - yes
  - no

numbers:
  integer: 42
  float: 3.14

# Indentation matters! Use 2 spaces (not tabs)
```

### GitHub Actions YAML Structure
```yaml
name: Workflow Name           # Optional workflow name

on:                          # Events that trigger the workflow
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:                         # Environment variables for entire workflow
  NODE_VERSION: '18'

jobs:                        # Define jobs
  job-name:                  # Job identifier
    runs-on: ubuntu-latest   # Runner environment
    
    env:                     # Job-level environment variables
      JOB_VAR: value
    
    steps:                   # Steps within the job
      - name: Step name      # Optional step name
        uses: actions/checkout@v4    # Use an action
        
      - name: Another step
        run: echo "Hello World"      # Run shell command
        
      - name: Step with environment
        run: echo $MY_VAR
        env:                 # Step-level environment variables
          MY_VAR: "Hello"
```

## Creating Your First Workflow {#first-workflow}

Let's create a simple "Hello World" workflow step by step.

### Step 1: Create Workflow Directory
```bash
# In your repository root
mkdir -p .github/workflows
```

### Step 2: Create Your First Workflow
Create `.github/workflows/hello-world.yml`:

```yaml
name: Hello World Workflow

# Trigger on push to main branch
on:
  push:
    branches: [ main ]
  # Also allow manual triggering
  workflow_dispatch:

# Define jobs
jobs:
  # Job 1: Greeting
  say-hello:
    runs-on: ubuntu-latest
    
    steps:
      # Step 1: Print hello message
      - name: Say Hello
        run: echo "Hello, GitHub Actions World! 🌟"
      
      # Step 2: Show system information
      - name: Show System Info
        run: |
          echo "Runner OS: $RUNNER_OS"
          echo "Runner Architecture: $RUNNER_ARCH"
          echo "Current Directory: $(pwd)"
          echo "Current User: $(whoami)"
          ls -la
      
      # Step 3: Create a file
      - name: Create Test File
        run: |
          echo "This file was created by GitHub Actions" > test-file.txt
          echo "Timestamp: $(date)" >> test-file.txt
          cat test-file.txt

  # Job 2: Multiple languages
  multi-language:
    runs-on: ubuntu-latest
    
    steps:
      - name: Python Hello
        run: python3 -c "print('Hello from Python! 🐍')"
      
      - name: Node.js Hello
        run: node -e "console.log('Hello from Node.js! 🟢')"
      
      - name: Bash Hello
        run: echo "Hello from Bash! 🐚"

  # Job 3: Dependent job (runs after say-hello completes)
  farewell:
    runs-on: ubuntu-latest
    needs: say-hello  # This job depends on say-hello
    
    steps:
      - name: Say Goodbye
        run: echo "Goodbye from the dependent job! 👋"
```

### Step 3: Advanced First Workflow
Create `.github/workflows/comprehensive-example.yml`:

```yaml
name: Comprehensive Workflow Example

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    # Run every day at 9 AM UTC
    - cron: '0 9 * * *'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
        - staging
        - production
      debug:
        description: 'Enable debug mode'
        required: false
        default: false
        type: boolean

env:
  GLOBAL_VAR: "I'm available to all jobs"

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Set up dynamic matrix
        id: set-matrix
        run: |
          # Create a dynamic matrix based on files in repository
          if [ -f "package.json" ]; then
            echo "matrix=[\"node\", \"lint\", \"test\"]" >> $GITHUB_OUTPUT
          else
            echo "matrix=[\"build\", \"test\"]" >> $GITHUB_OUTPUT
          fi

  build:
    runs-on: ${{ matrix.os }}
    needs: setup
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]
        # Exclude specific combinations
        exclude:
          - os: windows-latest
            node-version: 16
    
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4
      
      - name: Setup Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      
      - name: Display Environment Info
        run: |
          echo "OS: ${{ runner.os }}"
          echo "Node Version: ${{ matrix.node-version }}"
          echo "Event: ${{ github.event_name }}"
          echo "Branch: ${{ github.ref_name }}"
          echo "Commit SHA: ${{ github.sha }}"
      
      - name: Conditional Step (Only on main branch)
        if: github.ref == 'refs/heads/main'
        run: echo "This only runs on main branch pushes"
      
      - name: Install Dependencies
        run: |
          if [ -f "package.json" ]; then
            npm ci
          else
            echo "No package.json found, skipping npm install"
          fi
      
      - name: Build Application
        run: |
          if [ -f "package.json" ]; then
            npm run build --if-present
          else
            echo "No build script found, creating mock build"
            mkdir -p dist
            echo "Mock build output" > dist/app.js
          fi
      
      - name: Upload Build Artifacts
        uses: actions/upload-artifact@v4
        with:
          name: build-artifacts-${{ matrix.os }}-node${{ matrix.node-version }}
          path: |
            dist/
            build/
          retention-days: 30

  test:
    runs-on: ubuntu-latest
    needs: build
    
    services:
      # Example: Redis service for testing
      redis:
        image: redis:6-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Download Build Artifacts
        uses: actions/download-artifact@v4
        with:
          name: build-artifacts-ubuntu-latest-node18
          path: ./artifacts
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Dependencies
        run: npm ci
      
      - name: Run Tests
        run: |
          npm test --if-present
        env:
          REDIS_URL: redis://localhost:6379
      
      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()  # Upload even if tests fail
        with:
          name: test-results
          path: |
            coverage/
            test-results.xml

  security-scan:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

  notify:
    runs-on: ubuntu-latest
    needs: [build, test, security-scan]
    if: always()  # Run even if previous jobs fail
    
    steps:
      - name: Determine Status
        id: status
        run: |
          if [[ "${{ needs.build.result }}" == "success" && 
                "${{ needs.test.result }}" == "success" && 
                "${{ needs.security-scan.result }}" == "success" ]]; then
            echo "status=✅ All checks passed!" >> $GITHUB_OUTPUT
            echo "color=good" >> $GITHUB_OUTPUT
          else
            echo "status=❌ Some checks failed!" >> $GITHUB_OUTPUT
            echo "color=danger" >> $GITHUB_OUTPUT
          fi
      
      - name: Send notification
        run: |
          echo "Workflow completed with status: ${{ steps.status.outputs.status }}"
          echo "Build: ${{ needs.build.result }}"
          echo "Test: ${{ needs.test.result }}"
          echo "Security: ${{ needs.security-scan.result }}"
```

## Understanding Actions Marketplace {#actions-marketplace}

The GitHub Actions Marketplace contains thousands of pre-built actions that you can use in your workflows.

### Popular Actions Categories

#### 1. Code & Repository Actions
```yaml
# Checkout repository code
- uses: actions/checkout@v4
  with:
    # Checkout specific branch
    ref: main
    # Fetch full history
    fetch-depth: 0
    # Checkout multiple repositories
    path: sub-directory

# Setup programming languages
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'

- uses: actions/setup-python@v4
  with:
    python-version: '3.11'
    cache: 'pip'

- uses: actions/setup-java@v4
  with:
    distribution: 'temurin'
    java-version: '17'
```

#### 2. Build & Test Actions
```yaml
# Cache dependencies
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Upload/Download artifacts
- uses: actions/upload-artifact@v4
  with:
    name: my-artifact
    path: path/to/artifact/

- uses: actions/download-artifact@v4
  with:
    name: my-artifact
    path: ./downloads
```

#### 3. Security Actions
```yaml
# Security scanning
- uses: github/codeql-action/init@v2
  with:
    languages: javascript, python

- uses: github/codeql-action/analyze@v2

# Dependency scanning
- uses: github/dependency-review-action@v3
```

#### 4. Deployment Actions
```yaml
# Deploy to AWS
- uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

# Deploy to Azure
- uses: azure/login@v1
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

# Deploy to Heroku
- uses: akhileshns/heroku-deploy@v3.12.14
  with:
    heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
    heroku_app_name: "your-app-name"
    heroku_email: "your-email@example.com"
```

### Creating Custom Actions

#### 1. JavaScript Action
Create `action.yml`:
```yaml
name: 'Hello World Action'
description: 'Greet someone and record the time'
inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time:
    description: 'The time we greeted you'
runs:
  using: 'node20'
  main: 'index.js'
```

Create `index.js`:
```javascript
const core = require('@actions/core');
const github = require('@actions/github');

try {
  const nameToGreet = core.getInput('who-to-greet');
  console.log(`Hello ${nameToGreet}!`);
  
  const time = (new Date()).toTimeString();
  core.setOutput("time", time);
  
  // Get the JSON webhook payload for the event that triggered the workflow
  const payload = JSON.stringify(github.context.payload, undefined, 2)
  console.log(`The event payload: ${payload}`);
} catch (error) {
  core.setFailed(error.message);
}
```

#### 2. Docker Action
Create `action.yml`:
```yaml
name: 'Hello World Docker Action'
description: 'Greet someone and record the time'
inputs:
  who-to-greet:
    description: 'Who to greet'
    required: true
    default: 'World'
outputs:
  time:
    description: 'The time we greeted you'
runs:
  using: 'docker'
  image: 'Dockerfile'
  args:
    - ${{ inputs.who-to-greet }}
```

Create `Dockerfile`:
```dockerfile
FROM alpine:3.10

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

ENTRYPOINT ["/entrypoint.sh"]
```

Create `entrypoint.sh`:
```bash
#!/bin/sh -l

echo "Hello $1"
time=$(date)
echo "time=$time" >> $GITHUB_OUTPUT
```

## Common Workflow Patterns {#workflow-patterns}

### 1. Multi-Environment Deployment
```yaml
name: Multi-Environment Deploy

on:
  push:
    branches: [ main, develop ]

jobs:
  deploy-staging:
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Staging
        run: echo "Deploying to staging..."

  deploy-production:
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Production
        run: echo "Deploying to production..."
```

### 2. Matrix Strategy for Multi-Platform Testing
```yaml
name: Cross Platform Testing

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.8', '3.9', '3.10', '3.11']
        exclude:
          - os: windows-latest
            python-version: '3.8'
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      - run: python -m pytest
```

### 3. Conditional Workflows
```yaml
name: Conditional Logic

on: [push, pull_request]

jobs:
  changes:
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.changes.outputs.backend }}
      frontend: ${{ steps.changes.outputs.frontend }}
    
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: |
            backend:
              - 'api/**'
              - 'server/**'
            frontend:
              - 'web/**'
              - 'client/**'

  backend-test:
    needs: changes
    if: ${{ needs.changes.outputs.backend == 'true' }}
    runs-on: ubuntu-latest
    
    steps:
      - name: Test Backend
        run: echo "Testing backend changes"

  frontend-test:
    needs: changes
    if: ${{ needs.changes.outputs.frontend == 'true' }}
    runs-on: ubuntu-latest
    
    steps:
      - name: Test Frontend
        run: echo "Testing frontend changes"
```

## Hands-On Exercises {#exercises}

### Exercise 1: Basic Workflow Creation
**Objective**: Create a workflow that runs on multiple triggers

**Tasks**:
1. Create a workflow that triggers on:
   - Push to main branch
   - Pull requests to main
   - Manual dispatch
   - Schedule (daily at 6 AM UTC)

2. The workflow should:
   - Print system information
   - Create a timestamp file
   - Upload the file as an artifact

**Solution Template**:
```yaml
name: Exercise 1 - Basic Workflow

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:
  schedule:
    - cron: '0 6 * * *'

jobs:
  basic-job:
    runs-on: ubuntu-latest
    
    steps:
      # Your implementation here
```

### Exercise 2: Matrix Strategy
**Objective**: Test across multiple environments

**Tasks**:
1. Create a matrix strategy testing:
   - Node.js versions: 16, 18, 20
   - Operating systems: Ubuntu, Windows, macOS
   - Exclude Node 16 on Windows

2. Each job should:
   - Setup the Node.js version
   - Install dependencies
   - Run tests
   - Upload test results

### Exercise 3: Custom Action Usage
**Objective**: Use marketplace actions effectively

**Tasks**:
1. Create a workflow that:
   - Uses `actions/checkout@v4`
   - Uses `actions/setup-node@v4`
   - Uses `actions/cache@v3` for npm dependencies
   - Uses `actions/upload-artifact@v4`

2. Implement proper caching strategy for npm

### Exercise 4: Conditional Logic
**Objective**: Implement smart conditional workflows

**Tasks**:
1. Create jobs that only run when:
   - Specific files are changed
   - On specific branches
   - On specific events
   - Based on previous job results

### Exercise 5: Environment Variables & Secrets
**Objective**: Manage configuration securely

**Tasks**:
1. Create a workflow using:
   - Global environment variables
   - Job-level environment variables
   - Step-level environment variables
   - Repository secrets (simulated)

## Best Practices {#best-practices}

### 1. Security Best Practices
```yaml
# Use specific action versions
- uses: actions/checkout@v4  # Good: Specific version
- uses: actions/checkout@main  # Avoid: Moving target

# Handle secrets properly
- name: Deploy
  env:
    API_KEY: ${{ secrets.API_KEY }}  # Good: Use secrets
  run: deploy.sh
  # Never: run: deploy.sh ${{ secrets.API_KEY }}  # Exposes secret in logs

# Use least privilege
permissions:
  contents: read  # Only read repository contents
  issues: write   # Only write to issues
```

### 2. Performance Best Practices
```yaml
# Use caching effectively
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-

# Skip unnecessary steps
- name: Skip if no changes
  if: steps.changes.outputs.src == 'false'
  run: echo "Skipping because no source changes"

# Use appropriate runners
runs-on: ubuntu-latest  # Fastest and cheapest for most tasks
# runs-on: windows-latest  # Only when Windows-specific testing needed
```

### 3. Maintainability Best Practices
```yaml
# Use descriptive names
name: "Build and Test Application"

# Document complex logic
- name: Calculate deployment strategy
  # This step determines whether to deploy based on branch and file changes
  run: |
    if [[ "${{ github.ref }}" == "refs/heads/main" ]] && [[ "${{ steps.changes.outputs.app }}" == "true" ]]; then
      echo "deploy=true" >> $GITHUB_OUTPUT
    fi

# Use reusable workflows for common patterns
uses: ./.github/workflows/reusable-test.yml
with:
  node-version: '18'
```

### 4. Error Handling Best Practices
```yaml
# Continue on error when appropriate
- name: Optional linting
  run: npm run lint
  continue-on-error: true

# Use conditional execution based on previous steps
- name: Deploy only if tests pass
  if: success()
  run: deploy.sh

# Always upload artifacts for debugging
- name: Upload logs
  uses: actions/upload-artifact@v4
  if: always()  # Upload even if job fails
  with:
    name: debug-logs
    path: logs/
```

## Assessment {#assessment}

### Knowledge Check Questions

1. **YAML Syntax** (Multiple Choice)
   What's wrong with this YAML?
   ```yaml
   jobs:
   build:
     runs-on: ubuntu-latest
     steps:
     - name: Test
       run: echo "test"
   ```
   a) Missing indentation for `build`
   b) Missing indentation for `steps`
   c) Both a and b
   d) Nothing is wrong

2. **Event Triggers** (Short Answer)
   Write the YAML configuration to trigger a workflow on:
   - Pushes to main and develop branches
   - Pull requests to main branch only
   - Manual trigger with an input parameter for environment

3. **Matrix Strategy** (Code Completion)
   Complete this matrix strategy to test Node.js versions 16, 18, 20 on Ubuntu and Windows, but exclude Node 16 on Windows:
   ```yaml
   strategy:
     matrix:
       # Your code here
   ```

4. **Action Usage** (Practical)
   Write the steps to:
   - Checkout code
   - Setup Node.js version 18 with npm caching
   - Install dependencies
   - Run tests
   - Upload test coverage as artifacts

5. **Conditional Logic** (Problem Solving)
   How would you create a job that only runs:
   - On pushes to main branch
   - When files in the `src/` directory are changed
   - And the previous job succeeded

### Practical Assignments

#### Assignment 1: Personal Website CI
Create a GitHub Actions workflow for a personal website that:
- Triggers on push to main
- Builds the site (can be mock)
- Runs basic validation
- Uploads build artifacts
- Simulates deployment

#### Assignment 2: Multi-Environment Testing
Create a workflow that:
- Tests on multiple Node.js versions
- Tests on multiple operating systems
- Uses matrix exclusions appropriately
- Caches dependencies efficiently
- Uploads test results

#### Assignment 3: Smart CI Pipeline
Create an intelligent CI pipeline that:
- Only runs relevant jobs based on file changes
- Uses conditional logic for deployment
- Implements proper error handling
- Uses secrets and environment variables
- Follows security best practices

### Success Criteria
- [ ] Successfully create and run basic workflows
- [ ] Understand and implement YAML syntax correctly
- [ ] Use marketplace actions effectively
- [ ] Implement matrix strategies for multi-environment testing
- [ ] Apply conditional logic in workflows
- [ ] Follow security and performance best practices
- [ ] Create reusable and maintainable workflow patterns

---

## Next Steps
Once you've mastered GitHub Actions basics, you'll move on to **Module 6: Simple CI Pipeline** where you'll learn to build complete, production-ready CI pipelines that integrate with Git workflows and handle real-world scenarios like build failures and artifact management.

**Estimated Time**: 8-12 hours
**Prerequisites**: Completed Phase 1 (Fundamentals)
**Tools Needed**: GitHub account, Git, text editor
