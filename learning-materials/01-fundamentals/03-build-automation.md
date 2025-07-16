# Module 3: Basic Build Automation 🏗️

## Learning Objectives
- Understand what build automation is and why it's essential
- Learn different types of build systems and tools
- Master creating build scripts for various technologies
- Understand build artifacts and dependency management
- Implement build automation best practices

## What is Build Automation? 🤖

**Build automation** is the process of automatically compiling source code, running tests, and creating deployable artifacts without manual intervention.

### Why Automate Builds?
- **Consistency**: Same process every time
- **Speed**: Faster than manual builds
- **Reliability**: Eliminates human error
- **Reproducibility**: Anyone can build the project
- **CI/CD Foundation**: Essential for automated pipelines

### Traditional vs Automated Build Process

**Manual Build Process** ❌
```
1. Developer opens IDE
2. Manually compiles code
3. Manually runs tests
4. Manually packages application
5. Manually uploads to server
6. Hope everything works!
```

**Automated Build Process** ✅
```
1. Code commit triggers build
2. Automated compilation
3. Automated testing
4. Automated packaging
5. Automated artifact storage
6. Automated notifications
```

## Build System Components 🧩

### 1. Source Code Compilation
Transform source code into executable format:

```bash
# Java
javac -cp lib/*.jar src/**/*.java -d build/classes

# C++
g++ -std=c++17 src/*.cpp -o build/app

# TypeScript
tsc src/index.ts --outDir dist

# Go
go build -o build/app ./cmd/main.go
```

### 2. Dependency Management
Download and manage external libraries:

```bash
# Node.js
npm install

# Python
pip install -r requirements.txt

# Java (Maven)
mvn dependency:resolve

# Go
go mod download
```

### 3. Asset Processing
Optimize and bundle static assets:

```bash
# Minify CSS
cssmin styles.css > styles.min.css

# Bundle JavaScript
webpack --mode production

# Optimize images
imagemin src/images/* --out-dir=dist/images
```

### 4. Testing
Execute automated tests:

```bash
# Unit tests
npm test

# Integration tests
npm run test:integration

# Coverage
npm run test:coverage
```

### 5. Packaging
Create deployable artifacts:

```bash
# Create JAR file
jar -czf app.jar -C build/classes .

# Create Docker image
docker build -t myapp:latest .

# Create ZIP package
zip -r app.zip dist/
```

## Build Tools by Technology 🛠️

### JavaScript/Node.js

#### 1. npm Scripts
```json
{
  "name": "my-app",
  "scripts": {
    "build": "webpack --mode production",
    "test": "jest",
    "lint": "eslint src/",
    "start": "node dist/index.js",
    "dev": "webpack-dev-server --mode development",
    "clean": "rm -rf dist/",
    "prebuild": "npm run clean && npm run lint",
    "postbuild": "npm run test"
  }
}
```

#### 2. Webpack Configuration
```javascript
// webpack.config.js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env']
          }
        }
      },
      {
        test: /\.css$/,
        use: ['style-loader', 'css-loader']
      }
    ]
  },
  plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin({
      template: './src/index.html'
    })
  ]
};
```

#### 3. Complete Build Script
```bash
#!/bin/bash
# build.sh

set -e  # Exit on error

echo "🏗️  Starting build process..."

# Clean previous build
echo "🧹 Cleaning previous build..."
rm -rf dist/

# Install dependencies
echo "📦 Installing dependencies..."
npm ci

# Run linting
echo "🔍 Running linter..."
npm run lint

# Run tests
echo "🧪 Running tests..."
npm test

# Build application
echo "🏗️  Building application..."
npm run build

# Generate documentation
echo "📖 Generating documentation..."
npm run docs

echo "✅ Build completed successfully!"
echo "📁 Artifacts available in dist/"
```

### Java (Maven)

#### 1. pom.xml Configuration
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>
    
    <properties>
        <maven.compiler.source>11</maven.compiler.source>
        <maven.compiler.target>11</maven.compiler.target>
        <junit.version>5.8.2</junit.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter-engine</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
            </plugin>
            
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.0.0-M5</version>
            </plugin>
            
            <plugin>
                <groupId>org.jacoco</groupId>
                <artifactId>jacoco-maven-plugin</artifactId>
                <version>0.8.7</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>prepare-agent</goal>
                        </goals>
                    </execution>
                    <execution>
                        <id>report</id>
                        <phase>test</phase>
                        <goals>
                            <goal>report</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

#### 2. Maven Build Commands
```bash
# Clean and compile
mvn clean compile

# Run tests
mvn test

# Package application
mvn package

# Install to local repository
mvn install

# Complete build lifecycle
mvn clean install

# Skip tests (not recommended for CI/CD)
mvn clean install -DskipTests

# Run with specific profile
mvn clean install -Pproduction
```

### Python

#### 1. setup.py
```python
from setuptools import setup, find_packages

setup(
    name="my-app",
    version="1.0.0",
    packages=find_packages(),
    install_requires=[
        "requests>=2.25.1",
        "flask>=2.0.1",
        "pytest>=6.2.4",
    ],
    extras_require={
        "dev": [
            "pytest-cov>=2.12.1",
            "black>=21.6b0",
            "flake8>=3.9.2",
            "mypy>=0.910",
        ]
    },
    entry_points={
        "console_scripts": [
            "my-app=myapp.main:main",
        ],
    },
)
```

#### 2. Build Script
```bash
#!/bin/bash
# build.sh

set -e

echo "🐍 Starting Python build process..."

# Create virtual environment
echo "🌐 Setting up virtual environment..."
python -m venv venv
source venv/bin/activate

# Install dependencies
echo "📦 Installing dependencies..."
pip install -e .[dev]

# Run code formatting
echo "🎨 Formatting code..."
black src/ tests/

# Run linting
echo "🔍 Running linter..."
flake8 src/ tests/

# Run type checking
echo "🔍 Running type checker..."
mypy src/

# Run tests
echo "🧪 Running tests..."
pytest tests/ --cov=src/ --cov-report=html

# Build package
echo "📦 Building package..."
python setup.py sdist bdist_wheel

echo "✅ Build completed successfully!"
```

#### 3. requirements.txt and setup.cfg
```bash
# requirements.txt
requests>=2.25.1
flask>=2.0.1

# requirements-dev.txt
-r requirements.txt
pytest>=6.2.4
pytest-cov>=2.12.1
black>=21.6b0
flake8>=3.9.2
mypy>=0.910
```

```ini
# setup.cfg
[flake8]
max-line-length = 88
exclude = venv/,build/,dist/

[mypy]
python_version = 3.9
strict = True

[tool:pytest]
testpaths = tests/
python_files = test_*.py
python_classes = Test*
python_functions = test_*
```

### Docker-based Builds

#### 1. Multi-stage Dockerfile
```dockerfile
# Build stage
FROM node:16-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY . .

# Build application
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy built assets
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

#### 2. Build Script
```bash
#!/bin/bash
# docker-build.sh

set -e

APP_NAME="my-app"
VERSION=${1:-latest}
REGISTRY="your-registry.com"

echo "🐳 Building Docker image..."

# Build image
docker build -t ${APP_NAME}:${VERSION} .

# Tag for registry
docker tag ${APP_NAME}:${VERSION} ${REGISTRY}/${APP_NAME}:${VERSION}
docker tag ${APP_NAME}:${VERSION} ${REGISTRY}/${APP_NAME}:latest

# Run tests on built image
echo "🧪 Testing Docker image..."
docker run --rm ${APP_NAME}:${VERSION} npm test

# Push to registry
echo "📤 Pushing to registry..."
docker push ${REGISTRY}/${APP_NAME}:${VERSION}
docker push ${REGISTRY}/${APP_NAME}:latest

echo "✅ Docker build completed!"
```

## Build Artifacts and Management 📦

### What are Build Artifacts?
Artifacts are the outputs of the build process:
- Compiled binaries
- JAR/WAR files
- Docker images
- ZIP/TAR packages
- Documentation
- Test reports

### Artifact Naming Conventions
```bash
# Semantic versioning
myapp-1.2.3.jar
myapp-1.2.3-SNAPSHOT.jar

# Build metadata
myapp-1.2.3+build.123.jar
myapp-1.2.3+git.abc1234.jar

# Environment specific
myapp-1.2.3-dev.jar
myapp-1.2.3-staging.jar
myapp-1.2.3-prod.jar
```

### Artifact Storage
```bash
# Local storage structure
artifacts/
├── builds/
│   ├── 2023-07-16/
│   │   ├── build-123/
│   │   │   ├── app.jar
│   │   │   ├── tests-report.xml
│   │   │   └── coverage-report.html
│   │   └── build-124/
│   └── 2023-07-17/
├── releases/
│   ├── v1.0.0/
│   ├── v1.1.0/
│   └── v1.2.0/
└── snapshots/
    ├── main/
    └── develop/
```

## Build Environment Setup 🌍

### 1. Environment Variables
```bash
# Build configuration
export BUILD_ENV=production
export NODE_ENV=production
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk
export MAVEN_OPTS="-Xmx1024m"

# Version information
export BUILD_NUMBER=123
export GIT_COMMIT=$(git rev-parse HEAD)
export GIT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
export BUILD_TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

### 2. Build Configuration Files
```yaml
# build.yml
build:
  environment: production
  
  steps:
    - name: "Setup"
      command: "npm ci"
      
    - name: "Lint"
      command: "npm run lint"
      
    - name: "Test"
      command: "npm test"
      
    - name: "Build"
      command: "npm run build"
      
  artifacts:
    - "dist/**/*"
    - "coverage/**/*"
    
  notifications:
    slack: "#builds"
    email: "team@company.com"
```

### 3. Cross-platform Build Scripts
```bash
#!/bin/bash
# build.sh - Cross-platform build script

# Detect OS
case "$(uname -s)" in
    Linux*)     MACHINE=Linux;;
    Darwin*)    MACHINE=Mac;;
    CYGWIN*)    MACHINE=Cygwin;;
    MINGW*)     MACHINE=MinGw;;
    *)          MACHINE="UNKNOWN:${unameOut}"
esac

echo "Building on ${MACHINE}..."

# Set OS-specific variables
if [ "$MACHINE" = "Mac" ]; then
    SED_CMD="sed -i ''"
    OPEN_CMD="open"
elif [ "$MACHINE" = "Linux" ]; then
    SED_CMD="sed -i"
    OPEN_CMD="xdg-open"
fi

# Build process
npm ci
npm run build

# Open results
if [ "$OPEN_COVERAGE" = "true" ]; then
    $OPEN_CMD coverage/index.html
fi
```

## Performance Optimization 🚀

### 1. Parallel Builds
```bash
# Maven parallel builds
mvn -T 4 clean install  # Use 4 threads

# npm parallel scripts
npm-run-all --parallel lint test build

# Make parallel builds
make -j4  # Use 4 jobs
```

### 2. Build Caching
```bash
# Docker layer caching
FROM node:16-alpine
WORKDIR /app

# Cache dependencies separately
COPY package*.json ./
RUN npm ci

# Copy source (this layer will change more often)
COPY . .
RUN npm run build
```

### 3. Incremental Builds
```javascript
// webpack.config.js
module.exports = {
  cache: {
    type: 'filesystem',
    buildDependencies: {
      config: [__filename],
    },
  },
  optimization: {
    splitChunks: {
      chunks: 'all',
    },
  },
};
```

## Build Monitoring and Notifications 📊

### 1. Build Status Tracking
```bash
#!/bin/bash
# build-with-monitoring.sh

BUILD_START=$(date +%s)
BUILD_ID=$(uuidgen)

echo "🏗️  Build ${BUILD_ID} started at $(date)"

# Function to send status
send_status() {
    local status=$1
    local message=$2
    curl -X POST "https://monitoring.company.com/builds" \
        -H "Content-Type: application/json" \
        -d "{
            \"id\": \"${BUILD_ID}\",
            \"status\": \"${status}\",
            \"message\": \"${message}\",
            \"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\"
        }"
}

# Build steps with monitoring
send_status "started" "Build process initiated"

if npm run build; then
    BUILD_END=$(date +%s)
    DURATION=$((BUILD_END - BUILD_START))
    send_status "success" "Build completed in ${DURATION}s"
else
    send_status "failed" "Build failed during compilation"
    exit 1
fi
```

### 2. Slack Notifications
```bash
# slack-notify.sh
#!/bin/bash

SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
BUILD_STATUS=$1
BUILD_URL=$2

if [ "$BUILD_STATUS" = "success" ]; then
    COLOR="good"
    MESSAGE="✅ Build succeeded!"
else
    COLOR="danger"
    MESSAGE="❌ Build failed!"
fi

curl -X POST $SLACK_WEBHOOK \
    -H 'Content-type: application/json' \
    --data "{
        \"attachments\": [
            {
                \"color\": \"${COLOR}\",
                \"text\": \"${MESSAGE}\",
                \"fields\": [
                    {
                        \"title\": \"Project\",
                        \"value\": \"$(basename $(pwd))\",
                        \"short\": true
                    },
                    {
                        \"title\": \"Branch\",
                        \"value\": \"$(git rev-parse --abbrev-ref HEAD)\",
                        \"short\": true
                    }
                ],
                \"actions\": [
                    {
                        \"type\": \"button\",
                        \"text\": \"View Build\",
                        \"url\": \"${BUILD_URL}\"
                    }
                ]
            }
        ]
    }"
```

## Hands-on Exercise: Build Automation Setup 💻

Let's create a complete build automation setup for a sample Node.js application:

### Step 1: Create Sample Application
```bash
# Create project directory
mkdir build-automation-demo
cd build-automation-demo

# Initialize Node.js project
npm init -y

# Install dependencies
npm install express
npm install --save-dev jest eslint webpack webpack-cli
```

### Step 2: Create Application Code
```javascript
// src/app.js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({ message: 'Hello CI/CD!' });
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

if (require.main === module) {
  app.listen(port, () => {
    console.log(`Server running on port ${port}`);
  });
}

module.exports = app;
```

```javascript
// tests/app.test.js
const request = require('supertest');
const app = require('../src/app');

describe('App', () => {
  test('GET / should return hello message', async () => {
    const response = await request(app).get('/');
    expect(response.status).toBe(200);
    expect(response.body.message).toBe('Hello CI/CD!');
  });

  test('GET /health should return health status', async () => {
    const response = await request(app).get('/health');
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('healthy');
    expect(response.body.timestamp).toBeDefined();
  });
});
```

### Step 3: Configure Build Tools
```json
{
  "name": "build-automation-demo",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/app.js",
    "dev": "nodemon src/app.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/ tests/",
    "lint:fix": "eslint src/ tests/ --fix",
    "build": "webpack --mode production",
    "build:dev": "webpack --mode development",
    "clean": "rm -rf dist/ coverage/",
    "prebuild": "npm run clean && npm run lint",
    "postbuild": "npm run test",
    "package": "npm pack",
    "full-build": "npm run clean && npm run lint && npm run test:coverage && npm run build && npm run package"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": [
      "src/**/*.js"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 80,
        "functions": 80,
        "lines": 80,
        "statements": 80
      }
    }
  }
}
```

### Step 4: Create Build Script
```bash
#!/bin/bash
# build.sh

set -e  # Exit on any error

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Build information
BUILD_ID=$(date +%Y%m%d-%H%M%S)
BUILD_DIR="builds/${BUILD_ID}"
START_TIME=$(date +%s)

echo -e "${YELLOW}🏗️  Starting build ${BUILD_ID}...${NC}"

# Create build directory
mkdir -p "$BUILD_DIR"

# Function to log with timestamp
log() {
    echo -e "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Function to handle errors
handle_error() {
    local exit_code=$1
    local command="$2"
    if [ $exit_code -ne 0 ]; then
        echo -e "${RED}❌ Build failed at: $command${NC}"
        exit $exit_code
    fi
}

# Clean previous builds
log "${YELLOW}🧹 Cleaning previous build artifacts...${NC}"
npm run clean
handle_error $? "Clean"

# Install dependencies
log "${YELLOW}📦 Installing dependencies...${NC}"
npm ci
handle_error $? "Dependency installation"

# Run linting
log "${YELLOW}🔍 Running linter...${NC}"
npm run lint
handle_error $? "Linting"

# Run tests with coverage
log "${YELLOW}🧪 Running tests with coverage...${NC}"
npm run test:coverage
handle_error $? "Testing"

# Build application
log "${YELLOW}🏗️  Building application...${NC}"
npm run build
handle_error $? "Build"

# Create package
log "${YELLOW}📦 Creating package...${NC}"
npm run package
handle_error $? "Packaging"

# Copy artifacts to build directory
log "${YELLOW}📁 Collecting build artifacts...${NC}"
cp -r dist/ "$BUILD_DIR/"
cp coverage/ "$BUILD_DIR/" -r
cp *.tgz "$BUILD_DIR/" 2>/dev/null || true

# Calculate build time
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

# Build summary
echo -e "${GREEN}✅ Build ${BUILD_ID} completed successfully!${NC}"
echo -e "⏱️  Duration: ${DURATION}s"
echo -e "📁 Artifacts: $(ls -la $BUILD_DIR)"
echo -e "📊 Coverage: $(grep -o 'All files.*%' coverage/lcov-report/index.html || echo 'Coverage report generated')"

log "${GREEN}🎉 Build process finished!${NC}"
```

### Step 5: Make It Executable and Test
```bash
# Make build script executable
chmod +x build.sh

# Run the build
./build.sh
```

## Build Automation Best Practices 📋

### 1. Keep Builds Fast
- **Parallel execution** where possible
- **Incremental builds** to avoid rebuilding everything
- **Caching** dependencies and intermediate results
- **Optimize test suites** with proper test organization

### 2. Make Builds Reproducible
- **Pin dependency versions** exactly
- **Use consistent environments** (Docker, VMs)
- **Document all requirements** and prerequisites
- **Version your build scripts**

### 3. Fail Fast
- **Stop on first error** (`set -e` in bash)
- **Run fastest checks first** (linting before tests)
- **Clear error messages** with actionable information
- **Immediate feedback** to developers

### 4. Comprehensive Logging
- **Timestamp all operations**
- **Log build environment** information
- **Capture and store** build outputs
- **Structure logs** for easy parsing

### 5. Artifact Management
- **Consistent naming** conventions
- **Proper versioning** of artifacts
- **Retention policies** for old builds
- **Secure storage** of artifacts

## Assessment Questions 📋

1. **What's the main benefit of build automation in CI/CD?**
   - [ ] Faster development
   - [ ] Consistent, reproducible builds
   - [ ] Less code to write
   - [ ] Better user interface

2. **Which practice helps make builds faster?**
   - [ ] Running all tests sequentially
   - [ ] Rebuilding everything from scratch
   - [ ] Parallel execution and caching
   - [ ] Manual intervention points

3. **What should happen when a build fails?**
   - [ ] Continue with deployment anyway
   - [ ] Skip tests and try again
   - [ ] Stop immediately and notify team
   - [ ] Automatically fix the issues

**Your Answers:**
```
1. Answer: 
   Reasoning: 

2. Answer: 
   Reasoning: 

3. Answer: 
   Reasoning: 
```

## Next Steps 👉

Excellent work on mastering build automation! You now understand how to create reliable, automated build processes. Next, we'll explore [Module 4: Testing Fundamentals](./04-testing-basics.md) where you'll learn how to implement comprehensive testing strategies that integrate seamlessly with your build automation.

## Additional Resources 📚

- [Build Automation with Maven](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Webpack Documentation](https://webpack.js.org/concepts/)
- [Docker Multi-stage Builds](https://docs.docker.com/develop/dev-best-practices/dockerfile_best-practices/#use-multi-stage-builds)
- [npm Scripts](https://docs.npmjs.com/cli/v7/using-npm/scripts)
- [Make Tutorial](https://makefiletutorial.com/)
