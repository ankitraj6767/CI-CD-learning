# Module 4: Testing Fundamentals 🧪

## Learning Objectives
- Understand the importance of testing in CI/CD pipelines
- Learn different types of testing and when to use them
- Master the testing pyramid concept
- Implement automated testing strategies
- Integrate testing into CI/CD workflows

## Why Testing is Critical in CI/CD 🎯

Testing is the safety net that makes CI/CD possible. Without comprehensive testing, frequent deployments become dangerous rather than beneficial.

### The Cost of Bugs
```
Cost to fix a bug:
- During development: $1
- During testing phase: $10  
- In production: $100
- After customer finds it: $1000+
```

### CI/CD Testing Goals
- **Fast feedback** - Know immediately if changes break anything
- **Confidence** - Deploy with certainty that code works
- **Quality gates** - Prevent bad code from reaching production
- **Regression prevention** - Ensure new changes don't break existing features

## The Testing Pyramid 🔺

The testing pyramid shows the ideal distribution of different types of tests:

```
           /\
          /  \
         / E2E \      ← Few, expensive, slow
        /______\
       /        \
      / Integration \   ← Some, moderate cost
     /______________\
    /                \
   /   Unit Tests     \  ← Many, cheap, fast
  /____________________\
```

### 1. Unit Tests (Base of Pyramid)
**70% of your tests should be unit tests**

```javascript
// Example: Testing a simple function
function calculateTotal(price, tax) {
    if (price < 0) throw new Error('Price cannot be negative');
    return price + (price * tax);
}

// Unit test
describe('calculateTotal', () => {
    test('should calculate total with tax', () => {
        expect(calculateTotal(100, 0.1)).toBe(110);
    });
    
    test('should throw error for negative price', () => {
        expect(() => calculateTotal(-10, 0.1))
            .toThrow('Price cannot be negative');
    });
});
```

**Characteristics:**
- Test individual functions/methods in isolation
- Very fast (milliseconds)
- No external dependencies
- Easy to write and maintain
- Pinpoint exact location of failures

### 2. Integration Tests (Middle)
**20% of your tests should be integration tests**

```javascript
// Example: Testing API endpoints
describe('User API', () => {
    test('should create user and return ID', async () => {
        const userData = { name: 'John', email: 'john@example.com' };
        
        const response = await request(app)
            .post('/api/users')
            .send(userData)
            .expect(201);
            
        expect(response.body.id).toBeDefined();
        expect(response.body.name).toBe('John');
        
        // Verify user was actually saved to database
        const user = await User.findById(response.body.id);
        expect(user.email).toBe('john@example.com');
    });
});
```

**Characteristics:**
- Test interaction between components
- Moderate speed (seconds)
- May use real databases/services
- Test data flow between modules
- Catch interface/contract issues

### 3. End-to-End Tests (Top)
**10% of your tests should be E2E tests**

```javascript
// Example: Browser automation test
describe('User Registration Flow', () => {
    test('should allow user to register and login', async () => {
        // Navigate to registration page
        await page.goto('http://localhost:3000/register');
        
        // Fill registration form
        await page.fill('#name', 'John Doe');
        await page.fill('#email', 'john@example.com');
        await page.fill('#password', 'password123');
        
        // Submit form
        await page.click('#register-button');
        
        // Verify redirect to dashboard
        await page.waitForURL('**/dashboard');
        
        // Verify user is logged in
        const welcomeText = await page.textContent('#welcome-message');
        expect(welcomeText).toContain('Welcome, John Doe');
    });
});
```

**Characteristics:**
- Test complete user workflows
- Slow (minutes)
- Use real browsers/environments
- Test everything working together
- Catch user experience issues

## Types of Testing in CI/CD 🔍

### 1. Functional Testing

#### Unit Testing
```bash
# JavaScript (Jest)
npm test

# Python (pytest)
pytest tests/unit/

# Java (JUnit)
mvn test

# Go
go test ./...
```

#### API Testing
```javascript
// Using Supertest for Node.js
const request = require('supertest');
const app = require('../app');

describe('POST /api/users', () => {
    test('should validate required fields', async () => {
        const response = await request(app)
            .post('/api/users')
            .send({}) // Empty body
            .expect(400);
            
        expect(response.body.errors).toContain('Name is required');
    });
});
```

#### Database Testing
```javascript
// Testing database operations
describe('User Repository', () => {
    beforeEach(async () => {
        // Clean database before each test
        await User.deleteMany({});
    });
    
    test('should save user to database', async () => {
        const userData = { name: 'John', email: 'john@test.com' };
        const user = await userRepository.create(userData);
        
        expect(user.id).toBeDefined();
        
        // Verify in database
        const saved = await User.findById(user.id);
        expect(saved.name).toBe('John');
    });
});
```

### 2. Non-Functional Testing

#### Performance Testing
```javascript
// Load testing with artillery
// artillery.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
scenarios:
  - name: "API Load Test"
    requests:
      - get:
          url: "/api/users"
```

#### Security Testing
```bash
# OWASP ZAP security scan
docker run -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-stable \
  zap-baseline.py -t http://localhost:3000 -g gen.conf -r testreport.html
```

#### Accessibility Testing
```javascript
// Using axe-core for accessibility testing
const { AxePuppeteer } = require('@axe-core/puppeteer');

test('should be accessible', async () => {
    await page.goto('http://localhost:3000');
    
    const results = await new AxePuppeteer(page).analyze();
    
    expect(results.violations).toHaveLength(0);
});
```

## Test-Driven Development (TDD) 🔄

TDD follows the Red-Green-Refactor cycle:

### 1. Red: Write a Failing Test
```javascript
// Write test first (it will fail)
test('should calculate compound interest', () => {
    const result = calculateCompoundInterest(1000, 0.05, 2);
    expect(result).toBe(1102.50);
});
```

### 2. Green: Make the Test Pass
```javascript
// Write minimal code to make test pass
function calculateCompoundInterest(principal, rate, years) {
    return principal * Math.pow(1 + rate, years);
}
```

### 3. Refactor: Improve the Code
```javascript
// Refactor for better code quality
function calculateCompoundInterest(principal, rate, years) {
    if (principal <= 0) throw new Error('Principal must be positive');
    if (rate < 0) throw new Error('Rate cannot be negative');
    if (years < 0) throw new Error('Years cannot be negative');
    
    return Math.round(principal * Math.pow(1 + rate, years) * 100) / 100;
}
```

## Testing Strategies for Different Technologies 🛠️

### JavaScript/Node.js Testing Stack

#### Package.json Configuration
```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:unit": "jest tests/unit",
    "test:integration": "jest tests/integration",
    "test:e2e": "playwright test"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.0",
    "@playwright/test": "^1.40.0",
    "jest-environment-node": "^29.0.0"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": [
      "src/**/*.js",
      "!src/index.js"
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

#### Jest Configuration
```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  setupFilesAfterEnv: ['<rootDir>/tests/setup.js'],
  testMatch: [
    '**/tests/**/*.test.js',
    '**/src/**/__tests__/*.js'
  ],
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/index.js',
    '!**/node_modules/**'
  ],
  coverageReporters: ['text', 'lcov', 'html'],
  testTimeout: 10000
};
```

### Python Testing Stack

#### pytest Configuration
```python
# conftest.py
import pytest
from app import create_app, db

@pytest.fixture
def app():
    app = create_app(testing=True)
    with app.app_context():
        db.create_all()
        yield app
        db.drop_all()

@pytest.fixture
def client(app):
    return app.test_client()
```

#### Sample Test
```python
# test_user_api.py
def test_create_user(client):
    """Test user creation endpoint"""
    user_data = {
        'name': 'John Doe',
        'email': 'john@example.com',
        'password': 'password123'
    }
    
    response = client.post('/api/users', json=user_data)
    
    assert response.status_code == 201
    assert response.json['name'] == 'John Doe'
    assert 'password' not in response.json
```

### Java Testing Stack

#### Maven Configuration
```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-engine</artifactId>
        <version>5.9.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.1.1</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

#### JUnit 5 Test
```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void shouldCreateUser() {
        // Given
        User user = new User("John", "john@example.com");
        when(userRepository.save(any(User.class))).thenReturn(user);
        
        // When
        User result = userService.createUser("John", "john@example.com");
        
        // Then
        assertThat(result.getName()).isEqualTo("John");
        verify(userRepository).save(any(User.class));
    }
}
```

## Test Data Management 📊

### Test Fixtures
```javascript
// fixtures/users.js
module.exports = {
  validUser: {
    name: 'John Doe',
    email: 'john@example.com',
    password: 'password123'
  },
  
  adminUser: {
    name: 'Admin User',
    email: 'admin@example.com',
    password: 'admin123',
    role: 'admin'
  },
  
  invalidUser: {
    name: '',
    email: 'invalid-email',
    password: '123' // Too short
  }
};
```

### Database Seeding
```javascript
// tests/setup.js
const { User } = require('../src/models');
const fixtures = require('./fixtures/users');

beforeEach(async () => {
  // Clean database
  await User.deleteMany({});
  
  // Seed test data
  await User.create(fixtures.validUser);
  await User.create(fixtures.adminUser);
});
```

### Mock Data Generation
```javascript
// Using faker.js for dynamic test data
const faker = require('faker');

function generateUser() {
  return {
    name: faker.name.findName(),
    email: faker.internet.email(),
    password: faker.internet.password(8),
    age: faker.datatype.number({ min: 18, max: 80 })
  };
}

test('should handle multiple users', () => {
  const users = Array.from({ length: 10 }, generateUser);
  // Test with generated data
});
```

## Mocking and Stubbing 🎭

### External API Mocking
```javascript
// Mock external API calls
jest.mock('axios');
const mockedAxios = axios as jest.Mocked<typeof axios>;

test('should fetch user data from external API', async () => {
  // Setup mock response
  mockedAxios.get.mockResolvedValue({
    data: { id: 1, name: 'John' }
  });
  
  const user = await userService.fetchUserFromAPI(1);
  
  expect(user.name).toBe('John');
  expect(mockedAxios.get).toHaveBeenCalledWith('/api/users/1');
});
```

### Database Mocking
```javascript
// Mock database operations
jest.mock('../models/User');
const MockedUser = User as jest.MockedClass<typeof User>;

test('should save user to database', async () => {
  MockedUser.prototype.save.mockResolvedValue({
    id: 1,
    name: 'John'
  });
  
  const result = await userService.createUser('John', 'john@test.com');
  
  expect(result.id).toBe(1);
  expect(MockedUser.prototype.save).toHaveBeenCalled();
});
```

## Test Organization and Structure 📁

### Directory Structure
```
project/
├── src/
│   ├── controllers/
│   ├── services/
│   └── models/
├── tests/
│   ├── unit/
│   │   ├── controllers/
│   │   ├── services/
│   │   └── models/
│   ├── integration/
│   │   ├── api/
│   │   └── database/
│   ├── e2e/
│   │   ├── user-flows/
│   │   └── admin-flows/
│   ├── fixtures/
│   ├── helpers/
│   └── setup.js
└── package.json
```

### Test Naming Conventions
```javascript
// Good test names are descriptive
describe('UserController', () => {
  describe('POST /users', () => {
    test('should create user with valid data', () => {});
    test('should return 400 when name is missing', () => {});
    test('should return 409 when email already exists', () => {});
  });
});
```

## Code Coverage 📈

### Coverage Types
- **Line Coverage**: Percentage of code lines executed
- **Branch Coverage**: Percentage of decision branches taken
- **Function Coverage**: Percentage of functions called
- **Statement Coverage**: Percentage of statements executed

### Coverage Configuration
```javascript
// jest.config.js
module.exports = {
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './src/controllers/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    }
  }
};
```

### Reading Coverage Reports
```bash
# Generate coverage report
npm run test:coverage

# Coverage output
--------------------|---------|----------|---------|---------|
File                | % Stmts | % Branch | % Funcs | % Lines |
--------------------|---------|----------|---------|---------|
All files           |   85.71 |    83.33 |   88.89 |   85.71 |
 controllers        |   90.00 |    85.71 |   100.00|   90.00 |
  userController.js |   90.00 |    85.71 |   100.00|   90.00 |
 services           |   80.00 |    80.00 |   75.00 |   80.00 |
  userService.js    |   80.00 |    80.00 |   75.00 |   80.00 |
--------------------|---------|----------|---------|---------|
```

## Testing in CI/CD Pipelines 🔄

### GitHub Actions Testing Workflow
```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [16, 18, 20]
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
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
          DATABASE_URL: postgres://test:test@localhost:5432/testdb
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload coverage reports
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
```

### Test Parallelization
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-group: [unit, integration, e2e]
    
    steps:
      - uses: actions/checkout@v3
      - name: Setup Node.js
        uses: actions/setup-node@v3
      
      - name: Run tests
        run: npm run test:${{ matrix.test-group }}
```

## Quality Gates 🚪

### Automated Quality Checks
```yaml
# quality-gates.yml
name: Quality Gates

on: [pull_request]

jobs:
  quality-check:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Run tests with coverage
        run: npm run test:coverage
      
      - name: Check coverage threshold
        run: |
          coverage=$(node -p "require('./coverage/coverage-summary.json').total.lines.pct")
          if (( $(echo "$coverage < 80" | bc -l) )); then
            echo "Coverage $coverage% is below threshold of 80%"
            exit 1
          fi
      
      - name: Check test results
        run: |
          if [ -f "test-results.xml" ]; then
            failures=$(xmllint --xpath "//testcase[@failure]" test-results.xml | wc -l)
            if [ $failures -gt 0 ]; then
              echo "Tests have failures"
              exit 1
            fi
          fi
```

## Test Performance Optimization ⚡

### Parallel Test Execution
```javascript
// jest.config.js
module.exports = {
  maxWorkers: '50%', // Use half of available CPU cores
  testPathIgnorePatterns: ['/node_modules/', '/build/'],
  setupFilesAfterEnv: ['<rootDir>/tests/setup.js']
};
```

### Test Categorization
```javascript
// Fast tests
describe('UserValidator (fast)', () => {
  test('should validate email format', () => {
    expect(validateEmail('test@example.com')).toBe(true);
  });
});

// Slow tests
describe('UserController (slow)', () => {
  test('should create user in database', async () => {
    // Database operation - slower
  });
});
```

### Selective Test Running
```bash
# Run only changed files
npm test -- --changedSince=main

# Run tests matching pattern
npm test -- --testNamePattern="user"

# Run specific test file
npm test -- tests/unit/userService.test.js
```

## Hands-on Exercise: Building a Test Suite 💻

Let's create a comprehensive test suite for a simple API:

### Step 1: Setup Test Environment
```bash
# Create test project
mkdir testing-demo
cd testing-demo
npm init -y

# Install dependencies
npm install express
npm install --save-dev jest supertest @types/jest
```

### Step 2: Create Application Code
```javascript
// src/app.js
const express = require('express');
const app = express();

app.use(express.json());

let users = [];
let nextId = 1;

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.get('/users', (req, res) => {
  res.json(users);
});

app.post('/users', (req, res) => {
  const { name, email } = req.body;
  
  if (!name || !email) {
    return res.status(400).json({ error: 'Name and email are required' });
  }
  
  if (users.find(u => u.email === email)) {
    return res.status(409).json({ error: 'Email already exists' });
  }
  
  const user = { id: nextId++, name, email };
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

### Step 3: Create Unit Tests
```javascript
// tests/unit/userValidation.test.js
const { validateUser } = require('../../src/validators/userValidator');

describe('User Validation', () => {
  test('should validate correct user data', () => {
    const userData = { name: 'John', email: 'john@example.com' };
    const result = validateUser(userData);
    expect(result.isValid).toBe(true);
  });
  
  test('should reject user without name', () => {
    const userData = { email: 'john@example.com' };
    const result = validateUser(userData);
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('Name is required');
  });
  
  test('should reject invalid email format', () => {
    const userData = { name: 'John', email: 'invalid-email' };
    const result = validateUser(userData);
    expect(result.isValid).toBe(false);
    expect(result.errors).toContain('Invalid email format');
  });
});
```

### Step 4: Create Integration Tests
```javascript
// tests/integration/userApi.test.js
const request = require('supertest');
const app = require('../../src/app');

describe('User API Integration', () => {
  beforeEach(() => {
    // Reset users array before each test
    app.locals.users = [];
    app.locals.nextId = 1;
  });
  
  describe('POST /users', () => {
    test('should create user with valid data', async () => {
      const userData = { name: 'John', email: 'john@example.com' };
      
      const response = await request(app)
        .post('/users')
        .send(userData)
        .expect(201);
      
      expect(response.body.id).toBeDefined();
      expect(response.body.name).toBe('John');
      expect(response.body.email).toBe('john@example.com');
    });
    
    test('should return 400 for missing name', async () => {
      const userData = { email: 'john@example.com' };
      
      const response = await request(app)
        .post('/users')
        .send(userData)
        .expect(400);
      
      expect(response.body.error).toBe('Name and email are required');
    });
    
    test('should return 409 for duplicate email', async () => {
      const userData = { name: 'John', email: 'john@example.com' };
      
      // Create first user
      await request(app).post('/users').send(userData);
      
      // Try to create duplicate
      const response = await request(app)
        .post('/users')
        .send(userData)
        .expect(409);
      
      expect(response.body.error).toBe('Email already exists');
    });
  });
  
  describe('GET /users/:id', () => {
    test('should return user by ID', async () => {
      // Create user first
      const createResponse = await request(app)
        .post('/users')
        .send({ name: 'John', email: 'john@example.com' });
      
      const userId = createResponse.body.id;
      
      // Get user by ID
      const response = await request(app)
        .get(`/users/${userId}`)
        .expect(200);
      
      expect(response.body.name).toBe('John');
    });
    
    test('should return 404 for non-existent user', async () => {
      const response = await request(app)
        .get('/users/999')
        .expect(404);
      
      expect(response.body.error).toBe('User not found');
    });
  });
});
```

### Step 5: Configure Package.json
```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:unit": "jest tests/unit",
    "test:integration": "jest tests/integration"
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

## Best Practices for Testing 📋

### 1. Test Organization
- **Arrange-Act-Assert**: Structure tests clearly
- **One assertion per test**: Focus on single behavior
- **Descriptive names**: Make test purpose obvious
- **Independent tests**: No dependencies between tests

### 2. Test Data Management
- **Use fixtures**: Consistent test data
- **Clean state**: Reset between tests
- **Realistic data**: Mirror production scenarios
- **Edge cases**: Test boundary conditions

### 3. Mock Strategy
- **Mock external dependencies**: Keep tests isolated
- **Don't mock what you own**: Test real integration
- **Verify interactions**: Check mock calls
- **Reset mocks**: Clean state between tests

### 4. Performance
- **Fast feedback**: Unit tests should be milliseconds
- **Parallel execution**: Run tests concurrently
- **Selective running**: Only test changed code
- **Optimize setup**: Minimize test initialization

### 5. Maintenance
- **Regular updates**: Keep tests current with code
- **Remove obsolete tests**: Clean up unused tests
- **Refactor together**: Update tests with code changes
- **Documentation**: Comment complex test scenarios

## Assessment Questions 📋

1. **According to the testing pyramid, what percentage of tests should be unit tests?**
   - [ ] 50%
   - [ ] 70%
   - [ ] 80%
   - [ ] 90%

2. **What's the main benefit of TDD (Test-Driven Development)?**
   - [ ] Faster development
   - [ ] Better code coverage
   - [ ] Improved design and fewer bugs
   - [ ] Easier debugging

3. **Which type of test would you write to verify a complete user registration flow?**
   - [ ] Unit test
   - [ ] Integration test
   - [ ] End-to-end test
   - [ ] Performance test

4. **What's a good coverage threshold for production applications?**
   - [ ] 60%
   - [ ] 70%
   - [ ] 80%
   - [ ] 100%

**Your Answers:**
```
1. Answer: 
   Reasoning: 

2. Answer: 
   Reasoning: 

3. Answer: 
   Reasoning: 

4. Answer: 
   Reasoning: 
```

## Next Steps 👉

Congratulations! You've completed the fundamentals phase. You now understand:
- ✅ CI/CD concepts and principles
- ✅ Git workflows for CI/CD
- ✅ Build automation strategies
- ✅ Comprehensive testing approaches

**Ready for Phase 2?** Let's move to [Module 5: GitHub Actions Basics](../02-basic-implementation/01-github-actions.md) where you'll start implementing your first CI/CD pipelines!

## Additional Resources 📚

- [Testing JavaScript Applications](https://testingjavascript.com/) - Kent C. Dodds
- [Jest Documentation](https://jestjs.io/docs/getting-started)
- [Playwright Documentation](https://playwright.dev/)
- [Testing Best Practices](https://github.com/goldbergyoni/javascript-testing-best-practices)
- [The Art of Unit Testing](https://www.manning.com/books/the-art-of-unit-testing-second-edition) - Book
