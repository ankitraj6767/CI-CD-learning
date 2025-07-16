# Simple Web App CI/CD Project 🌐

This is your first hands-on CI/CD project! You'll build a complete CI/CD pipeline for a simple Node.js web application.

## Project Overview

**What you'll build:**
- A simple Express.js web application
- Automated testing with Jest
- Code quality checks with ESLint
- Automated build process
- Basic GitHub Actions workflow

**What you'll learn:**
- Setting up a basic CI pipeline
- Automated testing integration
- Code quality enforcement
- Build artifact creation
- Deployment basics

## Getting Started 🚀

### Prerequisites
- Node.js (v16 or higher)
- Git
- GitHub account
- Code editor (VS Code recommended)

### Setup Instructions

1. **Initialize the project:**
```bash
# Clone or create the project directory
cd /Users/ankitraj/CI:CD/projects/01-simple-web-app

# Initialize npm project
npm init -y

# Install dependencies
npm install express
npm install --save-dev jest supertest eslint nodemon
```

2. **Follow the step-by-step guide in the modules**

3. **Complete all exercises before moving to the next project**

## Project Structure
```
01-simple-web-app/
├── src/
│   ├── app.js              # Main application
│   ├── routes/             # Route handlers
│   └── middleware/         # Custom middleware
├── tests/
│   ├── app.test.js         # Application tests
│   └── routes/             # Route tests
├── .github/
│   └── workflows/
│       └── ci.yml          # GitHub Actions workflow
├── package.json            # Dependencies and scripts
├── .eslintrc.js           # ESLint configuration
├── .gitignore             # Git ignore rules
├── Dockerfile             # Container configuration
└── README.md              # Project documentation
```

## Learning Objectives 🎯

By completing this project, you will:
- ✅ Understand basic CI/CD concepts
- ✅ Create automated build processes
- ✅ Implement automated testing
- ✅ Set up code quality checks
- ✅ Use GitHub Actions for CI/CD
- ✅ Containerize an application
- ✅ Deploy to a simple environment

## Exercises 📝

### Exercise 1: Basic Application Setup
Create a simple Express.js application with the following endpoints:
- `GET /` - Welcome message
- `GET /health` - Health check
- `GET /api/users` - Mock user data
- `POST /api/users` - Create user (mock)

### Exercise 2: Testing Setup
Implement comprehensive tests:
- Unit tests for all endpoints
- Integration tests
- Code coverage > 80%

### Exercise 3: CI Pipeline
Create GitHub Actions workflow:
- Trigger on push and pull requests
- Run tests and linting
- Build and package application
- Store build artifacts

### Exercise 4: Docker Integration
- Create Dockerfile
- Build Docker image in CI
- Push to container registry

### Exercise 5: Deployment
- Deploy to simple hosting platform
- Implement health checks
- Set up monitoring

## Success Criteria ✅

Your project is complete when:
- [ ] All tests pass automatically
- [ ] Code quality checks pass
- [ ] CI pipeline runs successfully
- [ ] Application builds without errors
- [ ] Docker image is created
- [ ] Application deploys successfully
- [ ] Health endpoints work
- [ ] Documentation is complete

## Next Steps 👉

After completing this project:
1. Review your implementation
2. Identify areas for improvement
3. Move to Project 2: Containerized Application
4. Share your learnings with the team

## Troubleshooting 🔧

**Common Issues:**
- Node.js version compatibility
- npm dependency conflicts
- GitHub Actions permissions
- Docker build failures

**Getting Help:**
- Check the troubleshooting guides in each module
- Review GitHub Actions logs
- Test locally before pushing
- Ask for help in discussions

Let's build something amazing! 🚀
