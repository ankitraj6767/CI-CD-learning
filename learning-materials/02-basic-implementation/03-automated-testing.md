# Module 7: Automated Testing in CI/CD 🧪

## Learning Objectives 🎯

By the end of this module, you will:
- Understand the importance of automated testing in CI/CD pipelines
- Learn different types of testing (unit, integration, end-to-end)
- Implement test automation in GitHub Actions
- Configure test reporting and quality gates
- Handle test failures and debugging
- Set up code coverage reporting
- Implement parallel testing strategies

## Table of Contents

1. [Introduction to Automated Testing](#introduction)
2. [Types of Testing](#types-of-testing)
3. [Testing Pyramid](#testing-pyramid)
4. [Setting Up Test Automation](#setup)
5. [GitHub Actions for Testing](#github-actions)
6. [Test Reporting](#test-reporting)
7. [Quality Gates](#quality-gates)
8. [Code Coverage](#code-coverage)
9. [Parallel Testing](#parallel-testing)
10. [Best Practices](#best-practices)
11. [Hands-on Exercises](#exercises)
12. [Troubleshooting](#troubleshooting)

---

## 1. Introduction to Automated Testing {#introduction}

### What is Automated Testing?

Automated testing is the practice of running tests automatically as part of your CI/CD pipeline, without manual intervention. It ensures that:

- **Code Quality**: Maintains high standards
- **Regression Prevention**: Catches bugs before they reach production
- **Faster Feedback**: Immediate results on code changes
- **Confidence**: Safe deployments with verified functionality
- **Documentation**: Tests serve as living documentation

### Why Automate Testing?

```mermaid
graph LR
    A[Code Commit] --> B[Automated Tests]
    B --> C{Tests Pass?}
    C -->|Yes| D[Deploy to Staging]
    C -->|No| E[Block Deployment]
    E --> F[Fix Issues]
    F --> A
```

**Benefits:**
- ⚡ **Speed**: Tests run in minutes, not hours
- 🔄 **Consistency**: Same tests, same environment, every time
- 📊 **Coverage**: Comprehensive testing across all scenarios
- 💰 **Cost-Effective**: Catch bugs early when they're cheaper to fix
- 🚀 **Confidence**: Deploy with assurance

---

## 2. Types of Testing {#types-of-testing}

### Unit Tests
**Purpose**: Test individual components in isolation

```javascript
// Example: JavaScript Unit Test
function add(a, b) {
    return a + b;
}

// Test
test('adds 1 + 2 to equal 3', () => {
    expect(add(1, 2)).toBe(3);
});
```

**Characteristics:**
- ✅ Fast execution (milliseconds)
- ✅ Isolated and independent
- ✅ Easy to debug
- ✅ High code coverage

### Integration Tests
**Purpose**: Test how components work together

```python
# Example: Python Integration Test
def test_user_registration_flow():
    # Test database + API integration
    user_data = {"email": "test@example.com", "password": "secure123"}
    response = api_client.post("/register", json=user_data)
    
    assert response.status_code == 201
    assert User.objects.filter(email="test@example.com").exists()
```

**Characteristics:**
- 🔄 Tests component interactions
- 🗄️ May involve databases, APIs
- ⏱️ Slower than unit tests
- 🔍 Catches integration issues

### End-to-End (E2E) Tests
**Purpose**: Test complete user workflows

```javascript
// Example: Cypress E2E Test
describe('User Login Flow', () => {
    it('should allow user to login successfully', () => {
        cy.visit('/login');
        cy.get('[data-cy=email]').type('user@example.com');
        cy.get('[data-cy=password]').type('password123');
        cy.get('[data-cy=login-btn]').click();
        cy.url().should('include', '/dashboard');
        cy.contains('Welcome back!');
    });
});
```

**Characteristics:**
- 🌐 Tests entire application flow
- 🖥️ Simulates real user interactions
- 🐌 Slowest but most comprehensive
- 🎯 High confidence in functionality

---

## 3. Testing Pyramid {#testing-pyramid}

```
        /\     E2E Tests
       /  \    (Few, Slow, Expensive)
      /____\   
     /      \  Integration Tests
    /        \ (Some, Medium Speed)
   /__________\
  /            \ Unit Tests
 /              \ (Many, Fast, Cheap)
/________________\
```

### Pyramid Principles:

1. **70% Unit Tests**: Fast, reliable foundation
2. **20% Integration Tests**: Verify component interactions
3. **10% E2E Tests**: Validate critical user journeys

### Anti-Pattern: Ice Cream Cone 🍦
```
     /\     
    /  \    E2E Tests (Too Many!)
   /____\   
  /      \  Integration Tests
 /        \ 
/          \ Unit Tests (Too Few!)
```
**Problems:**
- Slow feedback
- Flaky tests
- Hard to debug
- Expensive maintenance

---

## 4. Setting Up Test Automation {#setup}

### Project Structure
```
my-app/
├── src/
│   ├── components/
│   ├── services/
│   └── utils/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── package.json
└── jest.config.js
```

### Test Configuration Files

#### Jest Configuration (JavaScript)
```javascript
// jest.config.js
module.exports = {
    testEnvironment: 'node',
    collectCoverage: true,
    coverageDirectory: 'coverage',
    coverageReporters: ['text', 'lcov', 'html'],
    testMatch: [
        '**/tests/**/*.test.js',
        '**/src/**/*.test.js'
    ],
    setupFilesAfterEnv: ['<rootDir>/tests/setup.js']
};
```

#### PyTest Configuration (Python)
```ini
# pytest.ini
[tool:pytest]
testpaths = tests
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*
addopts = 
    --verbose
    --cov=src
    --cov-report=html
    --cov-report=term-missing
```

#### Go Test Configuration
```go
// main_test.go
package main

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    assert.Equal(t, 5, result, "Add function should return 5")
}
```

---

## 5. GitHub Actions for Testing {#github-actions}

### Basic Test Workflow

```yaml
# .github/workflows/test.yml
name: Run Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16, 18, 20]
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linting
      run: npm run lint
    
    - name: Run unit tests
      run: npm run test:unit
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
    
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella
```

### Multi-Language Testing

```yaml
# .github/workflows/multi-lang-test.yml
name: Multi-Language Tests

on: [push, pull_request]

jobs:
  test-frontend:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
      with:
        node-version: '18'
    - run: |
        cd frontend
        npm ci
        npm run test
  
  test-backend:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    - run: |
        cd backend
        pip install -r requirements.txt
        pytest
  
  test-api:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-go@v4
      with:
        go-version: '1.19'
    - run: |
        cd api
        go test ./...
```

### Database Testing

```yaml
# Testing with PostgreSQL
jobs:
  test-with-db:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v4
    - name: Run tests
      run: |
        export DATABASE_URL=postgresql://postgres:postgres@localhost:5432/test_db
        npm run test
```

---

## 6. Test Reporting {#test-reporting}

### JUnit XML Reports

```yaml
# Generate and publish test reports
- name: Run tests with JUnit output
  run: npm run test -- --reporter=jest-junit
  env:
    JEST_JUNIT_OUTPUT_DIR: ./test-results
    JEST_JUNIT_OUTPUT_NAME: junit.xml

- name: Publish test results
  uses: dorny/test-reporter@v1
  if: always()
  with:
    name: Test Results
    path: test-results/junit.xml
    reporter: jest-junit
```

### Custom Test Summary

```yaml
- name: Generate test summary
  if: always()
  run: |
    echo "## Test Results 📊" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "| Test Type | Status | Count |" >> $GITHUB_STEP_SUMMARY
    echo "|-----------|--------|-------|" >> $GITHUB_STEP_SUMMARY
    echo "| Unit Tests | ✅ Passed | $(grep -o 'passed' test-results.txt | wc -l) |" >> $GITHUB_STEP_SUMMARY
    echo "| Integration | ✅ Passed | $(grep -o 'integration.*passed' test-results.txt | wc -l) |" >> $GITHUB_STEP_SUMMARY
```

### Slack Notifications

```yaml
- name: Notify Slack on failure
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: failure
    text: "🚨 Tests failed in ${{ github.repository }}"
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 7. Quality Gates {#quality-gates}

### Coverage Thresholds

```javascript
// jest.config.js
module.exports = {
    coverageThreshold: {
        global: {
            branches: 80,
            functions: 80,
            lines: 80,
            statements: 80
        },
        './src/critical/': {
            branches: 95,
            functions: 95,
            lines: 95,
            statements: 95
        }
    }
};
```

### GitHub Actions Quality Gates

```yaml
- name: Check test coverage
  run: |
    COVERAGE=$(npm run test:coverage | grep "All files" | awk '{print $10}' | sed 's/%//')
    echo "Current coverage: $COVERAGE%"
    if [ "$COVERAGE" -lt "80" ]; then
      echo "❌ Coverage $COVERAGE% is below threshold (80%)"
      exit 1
    else
      echo "✅ Coverage $COVERAGE% meets threshold"
    fi

- name: Require all tests to pass
  run: |
    if [ "${{ job.status }}" != "success" ]; then
      echo "❌ Tests must pass before merge"
      exit 1
    fi
```

### Branch Protection Rules

```yaml
# Configure in GitHub Settings > Branches
# Require status checks to pass:
- continuous-integration
- test-coverage
- security-scan

# Require branches to be up to date
# Require review from code owners
# Dismiss stale reviews
```

---

## 8. Code Coverage {#code-coverage}

### Setting Up Coverage

#### JavaScript (Jest)
```json
{
  "scripts": {
    "test": "jest",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch"
  }
}
```

#### Python (Coverage.py)
```bash
# Install coverage
pip install coverage

# Run with coverage
coverage run -m pytest
coverage report
coverage html
```

#### Go (Built-in)
```bash
# Run tests with coverage
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out -o coverage.html
```

### Coverage Reports in CI

```yaml
- name: Generate coverage report
  run: npm run test:coverage

- name: Upload to Codecov
  uses: codecov/codecov-action@v3
  with:
    file: ./coverage/lcov.info
    flags: unittests
    name: codecov-umbrella
    fail_ci_if_error: true

- name: Comment coverage on PR
  uses: romeovs/lcov-reporter-action@v0.3.1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    lcov-file: ./coverage/lcov.info
```

---

## 9. Parallel Testing {#parallel-testing}

### Matrix Strategy

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    node-version: [16, 18, 20]
    test-suite: [unit, integration, e2e]
  fail-fast: false

steps:
- name: Run ${{ matrix.test-suite }} tests
  run: npm run test:${{ matrix.test-suite }}
```

### Parallel Test Execution

```yaml
# Split tests across multiple jobs
jobs:
  test-unit:
    runs-on: ubuntu-latest
    steps:
    - name: Run unit tests
      run: npm run test:unit
  
  test-integration:
    runs-on: ubuntu-latest
    steps:
    - name: Run integration tests
      run: npm run test:integration
  
  test-e2e:
    runs-on: ubuntu-latest
    steps:
    - name: Run E2E tests
      run: npm run test:e2e
  
  # Require all tests to pass
  test-complete:
    needs: [test-unit, test-integration, test-e2e]
    runs-on: ubuntu-latest
    steps:
    - name: All tests passed
      run: echo "✅ All test suites completed successfully"
```

### Test Sharding

```yaml
# Split large test suites
strategy:
  matrix:
    shard: [1, 2, 3, 4]

steps:
- name: Run tests (shard ${{ matrix.shard }}/4)
  run: |
    npm run test -- --shard=${{ matrix.shard }}/4
```

---

## 10. Best Practices {#best-practices}

### ✅ Do's

1. **Write Tests First** (TDD)
   ```javascript
   // Write test first
   test('should calculate total price with tax', () => {
       expect(calculateTotal(100, 0.1)).toBe(110);
   });
   
   // Then implement function
   function calculateTotal(price, taxRate) {
       return price * (1 + taxRate);
   }
   ```

2. **Keep Tests Independent**
   ```javascript
   // Good: Each test is independent
   beforeEach(() => {
       database.clear();
       setupTestData();
   });
   ```

3. **Use Descriptive Names**
   ```javascript
   // Good
   test('should return 404 when user does not exist', () => {});
   
   // Bad
   test('user test', () => {});
   ```

4. **Test Edge Cases**
   ```javascript
   test.each([
       [null, 'should handle null input'],
       [undefined, 'should handle undefined input'],
       ['', 'should handle empty string'],
       [0, 'should handle zero value']
   ])('edge case: %s - %s', (input, description) => {
       expect(() => processInput(input)).not.toThrow();
   });
   ```

### ❌ Don'ts

1. **Don't Test Implementation Details**
   ```javascript
   // Bad: Testing internal state
   expect(component.state.isLoading).toBe(true);
   
   // Good: Testing behavior
   expect(screen.getByText('Loading...')).toBeInTheDocument();
   ```

2. **Don't Write Flaky Tests**
   ```javascript
   // Bad: Time-dependent test
   setTimeout(() => {
       expect(result).toBe(expected);
   }, 1000);
   
   // Good: Wait for specific condition
   await waitFor(() => {
       expect(screen.getByText('Success')).toBeInTheDocument();
   });
   ```

3. **Don't Ignore Test Failures**
   ```yaml
   # Bad: Allowing tests to fail
   - name: Run tests
     run: npm test || true
   
   # Good: Fail the build on test failure
   - name: Run tests
     run: npm test
   ```

---

## 11. Hands-on Exercises {#exercises}

### Exercise 1: Basic Unit Testing

**Task**: Create unit tests for a simple calculator

```javascript
// calculator.js
class Calculator {
    add(a, b) {
        return a + b;
    }
    
    subtract(a, b) {
        return a - b;
    }
    
    multiply(a, b) {
        return a * b;
    }
    
    divide(a, b) {
        if (b === 0) {
            throw new Error('Division by zero');
        }
        return a / b;
    }
}

module.exports = Calculator;
```

**Your Task**: Write comprehensive tests in `calculator.test.js`

<details>
<summary>Solution</summary>

```javascript
// calculator.test.js
const Calculator = require('./calculator');

describe('Calculator', () => {
    let calculator;
    
    beforeEach(() => {
        calculator = new Calculator();
    });
    
    describe('add', () => {
        test('should add two positive numbers', () => {
            expect(calculator.add(2, 3)).toBe(5);
        });
        
        test('should add negative numbers', () => {
            expect(calculator.add(-2, -3)).toBe(-5);
        });
        
        test('should add zero', () => {
            expect(calculator.add(5, 0)).toBe(5);
        });
    });
    
    describe('divide', () => {
        test('should divide two numbers', () => {
            expect(calculator.divide(10, 2)).toBe(5);
        });
        
        test('should throw error when dividing by zero', () => {
            expect(() => calculator.divide(10, 0)).toThrow('Division by zero');
        });
    });
});
```
</details>

### Exercise 2: GitHub Actions Test Workflow

**Task**: Create a complete test workflow for a Node.js project

**Requirements**:
- Run on push and PR
- Test on multiple Node.js versions
- Include linting
- Generate coverage report
- Upload coverage to Codecov

<details>
<summary>Solution</summary>

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16, 18, 20]
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Setup Node.js ${{ matrix.node-version }}
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run ESLint
      run: npm run lint
    
    - name: Run tests with coverage
      run: npm run test:coverage
    
    - name: Upload coverage to Codecov
      if: matrix.node-version == '18'
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
        flags: unittests
        name: codecov-umbrella
        fail_ci_if_error: true
```
</details>

### Exercise 3: Integration Testing

**Task**: Write integration tests for a REST API

```javascript
// api.js
const express = require('express');
const app = express();

app.use(express.json());

let users = [];
let nextId = 1;

app.get('/users', (req, res) => {
    res.json(users);
});

app.post('/users', (req, res) => {
    const user = {
        id: nextId++,
        name: req.body.name,
        email: req.body.email
    };
    users.push(user);
    res.status(201).json(user);
});

app.get('/users/:id', (req, res) => {
    const user = users.find(u => u.id === parseInt(req.params.id));
    if (!user) {
        return res.status(404).json({ error: 'User not found' });
    }
    res.json(user);
});

module.exports = app;
```

**Your Task**: Write integration tests using supertest

<details>
<summary>Solution</summary>

```javascript
// api.test.js
const request = require('supertest');
const app = require('./api');

describe('Users API', () => {
    beforeEach(() => {
        // Reset users array before each test
        app.locals.users = [];
        app.locals.nextId = 1;
    });
    
    describe('GET /users', () => {
        test('should return empty array initially', async () => {
            const response = await request(app)
                .get('/users')
                .expect(200);
            
            expect(response.body).toEqual([]);
        });
    });
    
    describe('POST /users', () => {
        test('should create a new user', async () => {
            const userData = {
                name: 'John Doe',
                email: 'john@example.com'
            };
            
            const response = await request(app)
                .post('/users')
                .send(userData)
                .expect(201);
            
            expect(response.body).toMatchObject({
                id: 1,
                name: 'John Doe',
                email: 'john@example.com'
            });
        });
    });
    
    describe('GET /users/:id', () => {
        test('should return 404 for non-existent user', async () => {
            await request(app)
                .get('/users/999')
                .expect(404);
        });
    });
});
```
</details>

---

## 12. Troubleshooting {#troubleshooting}

### Common Issues and Solutions

#### 1. Flaky Tests

**Problem**: Tests pass sometimes, fail other times

**Solutions**:
```javascript
// Bad: Race condition
test('async operation', () => {
    performAsyncOperation();
    expect(result).toBe(expected); // May fail
});

// Good: Proper async handling
test('async operation', async () => {
    await performAsyncOperation();
    expect(result).toBe(expected);
});

// Good: Wait for condition
test('UI update', async () => {
    await waitFor(() => {
        expect(screen.getByText('Updated')).toBeInTheDocument();
    });
});
```

#### 2. Slow Tests

**Problem**: Tests take too long to run

**Solutions**:
```javascript
// Use test doubles
const mockApiCall = jest.fn().mockResolvedValue({ data: 'mocked' });

// Parallel execution
// jest.config.js
module.exports = {
    maxWorkers: '50%', // Use half of available cores
    testTimeout: 10000 // 10 second timeout
};
```

#### 3. Environment Issues

**Problem**: Tests work locally but fail in CI

**Solutions**:
```yaml
# Ensure consistent environment
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '18.x'
    cache: 'npm'

# Use exact dependency versions
- name: Install dependencies
  run: npm ci # Instead of npm install

# Set environment variables
env:
  NODE_ENV: test
  CI: true
```

#### 4. Memory Issues

**Problem**: Tests consume too much memory

**Solutions**:
```javascript
// Clean up after tests
afterEach(() => {
    jest.clearAllMocks();
    cleanup(); // Clean up DOM, close connections
});

// Limit concurrent tests
// jest.config.js
module.exports = {
    maxWorkers: 2, // Limit workers
    workerIdleMemoryLimit: '512MB'
};
```

### Debugging Test Failures

```javascript
// Add debug information
test('should process data correctly', () => {
    const input = { /* test data */ };
    const result = processData(input);
    
    console.log('Input:', JSON.stringify(input, null, 2));
    console.log('Result:', JSON.stringify(result, null, 2));
    
    expect(result).toEqual(expected);
});

// Use debugger
test('debug test', () => {
    debugger; // Will pause execution in debug mode
    // Test code here
});
```

---

## Summary 📋

In this module, you learned:

✅ **Automated Testing Fundamentals**
- Importance of automated testing in CI/CD
- Different types of tests (unit, integration, E2E)
- Testing pyramid principles

✅ **Implementation Skills**
- Setting up test automation
- Configuring GitHub Actions for testing
- Creating comprehensive test workflows

✅ **Quality Assurance**
- Implementing quality gates
- Code coverage reporting
- Test result reporting and notifications

✅ **Advanced Techniques**
- Parallel testing strategies
- Matrix testing across environments
- Performance optimization

✅ **Best Practices**
- Writing maintainable tests
- Debugging test failures
- Avoiding common pitfalls

### Next Steps 🚀

1. **Practice**: Implement automated testing in your projects
2. **Experiment**: Try different testing frameworks and tools
3. **Optimize**: Focus on test performance and reliability
4. **Learn**: Explore advanced testing patterns
5. **Module 8**: Move on to Basic Deployment strategies

### Key Takeaways 💡

- **Test Early, Test Often**: Automated tests catch issues before they reach production
- **Follow the Pyramid**: More unit tests, fewer E2E tests
- **Quality Gates**: Use coverage and test results to enforce quality
- **Fast Feedback**: Optimize for quick test execution
- **Reliability**: Write stable, non-flaky tests

---

**Congratulations! 🎉** You've mastered automated testing in CI/CD pipelines. You're now ready to ensure code quality and catch bugs before they impact users!

**Next Module**: [Basic Deployment →](./04-basic-deployment.md)