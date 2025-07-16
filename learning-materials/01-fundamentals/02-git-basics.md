# Module 2: Version Control with Git 🌿

## Learning Objectives
- Master Git fundamentals essential for CI/CD
- Understand branching strategies for CI/CD workflows
- Learn Git hooks and their role in automation
- Understand how Git integrates with CI/CD pipelines

## Why Git is Essential for CI/CD 🤝

Git is the foundation of modern CI/CD pipelines because:
- **Triggers**: Git events trigger CI/CD pipelines
- **History**: Complete audit trail of changes
- **Branching**: Enables parallel development
- **Collaboration**: Multiple developers can work simultaneously
- **Integration**: Works with all major CI/CD platforms

## Git Fundamentals Review 📚

### Basic Git Workflow
```bash
# Clone repository
git clone https://github.com/username/repo.git

# Make changes
echo "Hello CI/CD" > file.txt

# Stage changes
git add file.txt

# Commit changes
git commit -m "feat: add greeting message"

# Push to remote
git push origin main
```

### Essential Git Commands for CI/CD
```bash
# Check status
git status

# View commit history
git log --oneline

# Create and switch to branch
git checkout -b feature/new-feature

# Merge branch
git merge feature/new-feature

# Tag a release
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# View differences
git diff HEAD~1 HEAD
```

## Branching Strategies for CI/CD 🌳

### 1. Git Flow
Traditional branching model with specific branch purposes:

```
main (production)
├── develop (integration)
│   ├── feature/user-auth
│   ├── feature/payment-system
│   └── feature/dashboard
├── release/v1.2.0
└── hotfix/critical-bug
```

**Branches**:
- `main`: Production-ready code
- `develop`: Integration branch for features
- `feature/*`: Individual features
- `release/*`: Prepare releases
- `hotfix/*`: Critical production fixes

**CI/CD Integration**:
```yaml
# Example workflow triggers
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:
    branches: [release/*]
```

### 2. GitHub Flow (Recommended for CI/CD)
Simplified workflow perfect for continuous deployment:

```
main (always deployable)
├── feature/add-login
├── feature/improve-ui
└── hotfix/fix-login-bug
```

**Workflow**:
1. Create feature branch from `main`
2. Make changes and commit
3. Open Pull Request
4. Code review and discussion
5. Merge to `main`
6. Deploy automatically

**Benefits for CI/CD**:
- Simple and fast
- Always deployable main branch
- Perfect for continuous deployment
- Minimal merge conflicts

### 3. GitLab Flow
Combines simplicity with environment-specific branches:

```
main (development)
├── pre-production
├── production
└── feature/new-feature
```

**Environments**:
- `main` → Development environment
- `pre-production` → Staging environment  
- `production` → Production environment

## Git Hooks for CI/CD Automation ⚡

Git hooks are scripts that run automatically on Git events. They're crucial for CI/CD automation.

### Client-Side Hooks

#### 1. Pre-commit Hook
Runs before commit is created:

```bash
#!/bin/sh
# .git/hooks/pre-commit

echo "Running pre-commit checks..."

# Run linting
npm run lint
if [ $? -ne 0 ]; then
    echo "❌ Linting failed!"
    exit 1
fi

# Run tests
npm test
if [ $? -ne 0 ]; then
    echo "❌ Tests failed!"
    exit 1
fi

echo "✅ Pre-commit checks passed!"
```

#### 2. Commit-msg Hook
Validates commit messages:

```bash
#!/bin/sh
# .git/hooks/commit-msg

commit_regex='^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .{1,50}'

if ! grep -qE "$commit_regex" "$1"; then
    echo "❌ Invalid commit message format!"
    echo "Format: type(scope): description"
    echo "Example: feat(auth): add login functionality"
    exit 1
fi

echo "✅ Commit message format is valid!"
```

### Server-Side Hooks

#### 1. Pre-receive Hook
Runs on server before accepting push:

```bash
#!/bin/sh
# hooks/pre-receive

while read oldrev newrev refname; do
    # Prevent force push to main
    if [ "$refname" = "refs/heads/main" ]; then
        if git rev-list --count "$oldrev..$newrev" -eq 0; then
            echo "❌ Force push to main branch is not allowed!"
            exit 1
        fi
    fi
done

echo "✅ Push validation passed!"
```

#### 2. Post-receive Hook
Triggers after successful push (CI/CD trigger):

```bash
#!/bin/sh
# hooks/post-receive

# Trigger CI/CD pipeline
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/owner/repo/dispatches \
  -d '{"event_type":"deploy"}'

echo "✅ CI/CD pipeline triggered!"
```

## Commit Message Conventions 📝

Standardized commit messages enable automation and better tracking:

### Conventional Commits Format
```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### Types:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks
- `ci`: CI/CD related changes

### Examples:
```bash
# Feature
git commit -m "feat(auth): add OAuth2 integration"

# Bug fix
git commit -m "fix(api): resolve null pointer exception in user service"

# Breaking change
git commit -m "feat!: upgrade to Node.js 18" -m "BREAKING CHANGE: Node.js 16 no longer supported"

# Chore
git commit -m "chore(deps): update dependencies to latest versions"
```

## Git Configuration for CI/CD 🔧

### Global Configuration
```bash
# Set user information
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# Set default branch name
git config --global init.defaultBranch main

# Enable auto-setup of remote tracking
git config --global push.autoSetupRemote true

# Set pull strategy
git config --global pull.rebase false
```

### Repository-Specific Settings
```bash
# Set up commit template
git config commit.template .gitmessage

# Enable Git hooks
git config core.hooksPath .githooks
chmod +x .githooks/*
```

### .gitignore for CI/CD
```gitignore
# Dependencies
node_modules/
vendor/
.venv/

# Build outputs
dist/
build/
target/
*.war
*.jar

# Environment files
.env
.env.local
.env.production

# CI/CD artifacts
coverage/
test-results/
.nyc_output/

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/
```

## GitHub Integration for CI/CD 🐙

### 1. Repository Settings
```bash
# Enable branch protection rules
curl -X PUT \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/owner/repo/branches/main/protection \
  -d '{
    "required_status_checks": {
      "strict": true,
      "contexts": ["ci/tests", "ci/build"]
    },
    "enforce_admins": true,
    "required_pull_request_reviews": {
      "required_approving_review_count": 1
    },
    "restrictions": null
  }'
```

### 2. Webhooks for External CI/CD
```javascript
// Express.js webhook handler
app.post('/webhook', (req, res) => {
    const event = req.headers['x-github-event'];
    const payload = req.body;

    if (event === 'push') {
        // Trigger CI pipeline
        triggerBuild(payload.repository.name, payload.after);
    }

    if (event === 'pull_request' && payload.action === 'opened') {
        // Trigger PR validation
        triggerPRValidation(payload.pull_request.head.sha);
    }

    res.status(200).send('OK');
});
```

## Advanced Git Techniques for CI/CD 🚀

### 1. Atomic Commits
Create focused, single-purpose commits:

```bash
# Bad: Multiple unrelated changes
git add .
git commit -m "fix bugs and add feature"

# Good: Separate commits
git add src/auth.js
git commit -m "fix(auth): resolve token expiration issue"

git add src/dashboard.js
git commit -m "feat(dashboard): add user statistics widget"
```

### 2. Interactive Rebase for Clean History
```bash
# Clean up commits before merging
git rebase -i HEAD~3

# Squash related commits
pick abc1234 feat(auth): add login form
squash def5678 fix(auth): fix form validation
squash ghi9012 refactor(auth): improve error handling
```

### 3. Git Bisect for Bug Hunting
```bash
# Find the commit that introduced a bug
git bisect start
git bisect bad HEAD          # Current commit is bad
git bisect good v1.0.0       # Version 1.0.0 was good

# Git will checkout commits for testing
# After each test:
git bisect good   # if test passes
git bisect bad    # if test fails

# When done
git bisect reset
```

## Hands-on Exercise: Setting Up Git for CI/CD 💻

### Step 1: Create a Practice Repository
```bash
# Create new directory
mkdir cicd-practice
cd cicd-practice

# Initialize Git repository
git init
git branch -M main

# Create initial files
echo "# CI/CD Practice Repository" > README.md
echo "node_modules/" > .gitignore
echo "Hello World" > app.js

# First commit
git add .
git commit -m "feat: initial project setup"
```

### Step 2: Set Up Branch Protection
Create a simple Node.js project and set up branch protection:

```bash
# Create package.json
cat > package.json << EOF
{
  "name": "cicd-practice",
  "version": "1.0.0",
  "scripts": {
    "test": "echo 'Running tests...' && exit 0",
    "lint": "echo 'Running linter...' && exit 0"
  }
}
EOF

git add package.json
git commit -m "chore: add package.json with test scripts"
```

### Step 3: Create Pre-commit Hook
```bash
# Create hooks directory
mkdir .githooks

# Create pre-commit hook
cat > .githooks/pre-commit << 'EOF'
#!/bin/sh
echo "🔍 Running pre-commit checks..."

# Run tests
npm test
if [ $? -ne 0 ]; then
    echo "❌ Tests failed!"
    exit 1
fi

# Run linting
npm run lint
if [ $? -ne 0 ]; then
    echo "❌ Linting failed!"
    exit 1
fi

echo "✅ All checks passed!"
EOF

# Make executable
chmod +x .githooks/pre-commit

# Configure Git to use custom hooks directory
git config core.hooksPath .githooks

git add .githooks/
git commit -m "ci: add pre-commit hooks for quality checks"
```

### Step 4: Practice Branching Workflow
```bash
# Create feature branch
git checkout -b feature/add-functionality

# Make changes
echo "console.log('Hello CI/CD!');" >> app.js
git add app.js
git commit -m "feat: add greeting functionality"

# Switch back to main
git checkout main

# Merge feature branch
git merge feature/add-functionality

# Clean up
git branch -d feature/add-functionality

# Tag the release
git tag -a v1.0.0 -m "Release version 1.0.0"
```

## Troubleshooting Common Git Issues 🔧

### 1. Merge Conflicts
```bash
# When merge conflicts occur
git status  # See conflicted files

# Edit files to resolve conflicts
# Look for conflict markers: <<<<<<<, =======, >>>>>>>

# After resolving
git add resolved-file.js
git commit -m "resolve: merge conflict in resolved-file.js"
```

### 2. Accidentally Committed to Wrong Branch
```bash
# Reset the commit
git reset --soft HEAD~1

# Switch to correct branch
git checkout correct-branch

# Commit again
git commit -m "feat: add feature to correct branch"
```

### 3. Revert a Commit
```bash
# Revert a specific commit
git revert abc1234

# Revert merge commit
git revert -m 1 abc1234
```

## Assessment Questions 📋

1. **What branching strategy would you recommend for a team practicing continuous deployment?**
   - [ ] Git Flow
   - [ ] GitHub Flow
   - [ ] GitLab Flow
   - [ ] Custom strategy

2. **Which Git hook would you use to prevent commits with poor code quality?**
   - [ ] pre-commit
   - [ ] post-commit
   - [ ] pre-receive
   - [ ] post-receive

3. **What's the benefit of conventional commit messages in CI/CD?**
   - [ ] Better documentation
   - [ ] Automated changelog generation
   - [ ] Semantic versioning automation
   - [ ] All of the above

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

Great job mastering Git for CI/CD! Next, we'll explore [Module 3: Basic Build Automation](./03-build-automation.md) where you'll learn how to automate the build process that forms the foundation of any CI pipeline.

## Additional Resources 📚

- [Pro Git Book](https://git-scm.com/book) - Free online Git book
- [Git Flow vs GitHub Flow](https://lucamezzalira.com/2014/03/10/git-flow-vs-github-flow/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [GitHub Flow](https://guides.github.com/introduction/flow/)
- [Git Hooks Documentation](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)
