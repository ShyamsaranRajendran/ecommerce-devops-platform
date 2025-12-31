# Git Branch Strategy

## Overview

This document defines the Git branching model and workflow for the e-commerce platform. We follow a modified GitFlow strategy optimized for CI/CD.

## Branch Structure

```
main (production)
  ├── release/v1.0.0 (pre-production)
  │     └── develop (integration)
  │           ├── feature/user-authentication
  │           ├── feature/product-catalog
  │           └── feature/payment-gateway
  └── hotfix/critical-bug-fix
```

## Branch Types and Purposes

### Main Branches

#### `main` (Production Branch)

- **Purpose**: Production-ready code
- **Deployment Target**: Production environment
- **Protection Rules**:
  - ✅ No direct push allowed
  - ✅ Pull request required
  - ✅ Minimum 2 approvals required
  - ✅ All CI checks must pass
  - ✅ Code review mandatory
- **Naming**: `main`
- **Lifetime**: Permanent
- **Merge From**: `release/*` or `hotfix/*` branches only

#### `develop` (Integration Branch)

- **Purpose**: Integration branch for ongoing development
- **Deployment Target**: Dev environment (auto-deploy)
- **Protection Rules**:
  - ✅ Pull request required
  - ✅ At least 1 approval
  - ✅ CI checks must pass
- **Naming**: `develop`
- **Lifetime**: Permanent
- **Merge From**: `feature/*` branches

### Supporting Branches

#### `feature/*` (Feature Branches)

- **Purpose**: Develop new features or enhancements
- **Branched From**: `develop`
- **Merge Back To**: `develop`
- **Naming Convention**: `feature/<ticket-id>-<short-description>`
  - Examples:
    - `feature/ECOM-123-user-authentication`
    - `feature/ECOM-456-product-search`
    - `feature/ECOM-789-payment-integration`
- **Deployment**: CI only (no auto-deploy)
- **Lifetime**: Short-lived (delete after merge)
- **Rules**:
  - Keep branch up-to-date with `develop`
  - Squash commits before merging (optional)
  - Delete after successful merge

#### `release/*` (Release Branches)

- **Purpose**: Prepare for production release, final testing
- **Branched From**: `develop`
- **Merge Back To**: `main` AND `develop`
- **Deployment Target**: QA environment (auto-deploy)
- **Naming Convention**: `release/v<version>`
  - Examples:
    - `release/v1.0.0`
    - `release/v1.1.0`
    - `release/v2.0.0`
- **Allowed Changes**: Bug fixes, documentation, version bumps
- **NOT Allowed**: New features
- **Lifetime**: Until released to production
- **Process**:
  1. Create from `develop` when ready for release
  2. Auto-deploy to QA
  3. QA team performs testing
  4. Fix bugs in release branch
  5. Merge to `main` (production)
  6. Tag with version number
  7. Merge back to `develop`
  8. Delete release branch

#### `hotfix/*` (Hotfix Branches)

- **Purpose**: Emergency fixes for production issues
- **Branched From**: `main`
- **Merge Back To**: `main` AND `develop`
- **Deployment Target**: QA first, then Prod with approval
- **Naming Convention**: `hotfix/<ticket-id>-<short-description>`
  - Examples:
    - `hotfix/ECOM-999-payment-failure`
    - `hotfix/ECOM-888-security-vulnerability`
- **Priority**: Highest
- **Lifetime**: Very short (hours, not days)
- **Process**:
  1. Create from `main`
  2. Fix the issue
  3. Deploy to QA for validation
  4. Merge to `main` with fast-track approval
  5. Deploy to production
  6. Merge back to `develop`
  7. Delete hotfix branch

## Branching Workflow Diagrams

### Feature Development Workflow

```
develop
  │
  ├─→ feature/ECOM-123-user-auth
  │     │
  │     ├─ commit: Add login endpoint
  │     ├─ commit: Add JWT validation
  │     └─ commit: Add unit tests
  │     │
  │     └─→ Pull Request → develop
  │
develop (merged)
```

### Release Workflow

```
develop
  │
  └─→ release/v1.0.0
        │
        ├─ commit: Update version to 1.0.0
        ├─ commit: Fix QA bug #1
        └─ commit: Fix QA bug #2
        │
        ├─→ Pull Request → main (Production)
        │     │
        │     └─→ Tag: v1.0.0
        │
        └─→ Pull Request → develop (Backport fixes)
```

### Hotfix Workflow

```
main (v1.0.0)
  │
  └─→ hotfix/ECOM-999-payment-failure
        │
        ├─ commit: Fix payment gateway issue
        │
        ├─→ Deploy to QA → Test
        │
        ├─→ Pull Request → main
        │     │
        │     └─→ Deploy to Production
        │     │
        │     └─→ Tag: v1.0.1
        │
        └─→ Pull Request → develop
```

## Pull Request (PR) Rules

### PR Requirements

#### Feature → Develop

- ✅ At least 1 approval required
- ✅ All CI checks pass (build, test, SonarQube)
- ✅ Code review completed
- ✅ Branch up-to-date with `develop`
- ✅ Meaningful commit messages
- ⏱️ Auto-merge disabled

#### Release → Main (Production)

- ✅ Minimum 2 approvals required (DevOps Lead + Release Manager)
- ✅ All CI/CD checks pass
- ✅ QA sign-off
- ✅ Security scans passed
- ✅ Release notes attached
- ✅ Rollback plan documented
- ⏱️ Manual merge only

#### Hotfix → Main

- ✅ Fast-track approval (1 senior engineer)
- ✅ CI checks pass
- ✅ QA validation completed
- ✅ Incident documented
- ⏱️ Emergency process

### PR Template

```markdown
## Description

Brief description of changes

## Type of Change

- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Related Ticket

ECOM-XXX

## Testing

- [ ] Unit tests added/updated
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist

- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] Documentation updated
- [ ] No new warnings generated

## Screenshots (if applicable)
```

## Commit Message Convention

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style changes (formatting)
- **refactor**: Code refactoring
- **test**: Adding/updating tests
- **chore**: Maintenance tasks

### Examples

```
feat(auth): add JWT token validation

Implement JWT token validation middleware for all protected routes.
Added unit tests and integration tests.

Closes ECOM-123
```

```
fix(payment): resolve payment gateway timeout issue

Increased timeout from 30s to 60s and added retry logic.
Handles network failures gracefully.

Fixes ECOM-999
```

```
hotfix(security): patch SQL injection vulnerability

Updated input validation to prevent SQL injection attacks.
Applied prepared statements across all database queries.

CRITICAL: Deploy immediately
Fixes ECOM-888
```

## Branch Protection Rules

### `main` Branch Protection

```yaml
Protection Rules:
  - Require pull request before merging: true
  - Require approvals: 2
  - Dismiss stale reviews: true
  - Require review from code owners: true
  - Require status checks to pass: true
    - Jenkins CI build
    - Unit tests
    - SonarQube quality gate
    - Trivy security scan
  - Require branches to be up to date: true
  - Require signed commits: true
  - Include administrators: true
  - Allow force pushes: false
  - Allow deletions: false
```

### `develop` Branch Protection

```yaml
Protection Rules:
  - Require pull request before merging: true
  - Require approvals: 1
  - Require status checks to pass: true
    - Jenkins CI build
    - Unit tests
  - Allow force pushes: false
  - Allow deletions: false
```

## Version Tagging Strategy

### Semantic Versioning

- Format: `vMAJOR.MINOR.PATCH`
- Example: `v1.2.3`

#### Version Components

- **MAJOR**: Breaking changes (v1.0.0 → v2.0.0)
- **MINOR**: New features, backward compatible (v1.0.0 → v1.1.0)
- **PATCH**: Bug fixes (v1.0.0 → v1.0.1)

#### Tagging Process

```bash
# After merging release to main
git checkout main
git pull origin main
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0
```

#### Tag Protection

- ✅ Only maintainers can create tags
- ✅ Tags are immutable (cannot be deleted)
- ✅ Signed tags required for production

## Workflow Examples

### Scenario 1: Developing a New Feature

```bash
# Start from develop
git checkout develop
git pull origin develop

# Create feature branch
git checkout -b feature/ECOM-123-user-auth

# Make changes and commit
git add .
git commit -m "feat(auth): implement user login endpoint"

# Push to remote
git push origin feature/ECOM-123-user-auth

# Create Pull Request on GitHub/Azure Repos
# Target: develop
# Get approval → Merge → Delete branch
```

### Scenario 2: Preparing a Release

```bash
# Create release branch from develop
git checkout develop
git pull origin develop
git checkout -b release/v1.0.0

# Update version
# Update changelog
git commit -m "chore(release): prepare v1.0.0"

# Push to remote (auto-deploys to QA)
git push origin release/v1.0.0

# QA finds bugs → Fix in release branch
git commit -m "fix: resolve QA bug #1"

# After QA approval, merge to main
# Create PR: release/v1.0.0 → main
# Merge → Deploy to production → Tag

# Merge back to develop
git checkout develop
git merge release/v1.0.0
git push origin develop

# Delete release branch
git branch -d release/v1.0.0
```

### Scenario 3: Emergency Hotfix

```bash
# Create hotfix from main
git checkout main
git pull origin main
git checkout -b hotfix/ECOM-999-payment-failure

# Fix the issue
git commit -m "hotfix(payment): fix gateway timeout"

# Push and deploy to QA for verification
git push origin hotfix/ECOM-999-payment-failure

# After QA verification, merge to main
# Create PR: hotfix/ECOM-999-payment-failure → main
# Fast-track approval → Merge → Deploy to production

# Tag hotfix
git tag -a v1.0.1 -m "Hotfix: Payment gateway timeout"
git push origin v1.0.1

# Merge back to develop
git checkout develop
git merge hotfix/ECOM-999-payment-failure
git push origin develop
```

## Merge Strategies

| Source → Target   | Strategy         | Reason                   |
| ----------------- | ---------------- | ------------------------ |
| feature → develop | Squash and merge | Clean history            |
| release → main    | Merge commit     | Preserve release history |
| release → develop | Merge commit     | Keep fix commits         |
| hotfix → main     | Merge commit     | Track emergency fixes    |
| hotfix → develop  | Merge commit     | Preserve hotfix          |

## Best Practices

### DO ✅

- Keep feature branches small and focused
- Commit frequently with meaningful messages
- Pull latest changes before creating branch
- Delete branches after merging
- Tag all production releases
- Document breaking changes
- Write descriptive PR descriptions
- Review your own code before requesting review

### DON'T ❌

- Push directly to `main` or `develop`
- Leave branches open for weeks
- Commit commented-out code
- Mix features in one branch
- Force push to shared branches
- Skip writing tests
- Merge without approval
- Ignore merge conflicts

## Interview Key Points

> **Q: Explain your Git branching strategy**
>
> - Main: Production code only
> - Develop: Integration branch for features
> - Feature branches: Individual feature development
> - Release branches: QA and pre-production
> - Hotfix branches: Emergency production fixes

> **Q: How do you prevent accidental production changes?**
>
> - Branch protection rules on `main`
> - Required PR approvals (minimum 2)
> - CI/CD checks must pass
> - No direct push allowed
> - Manual approval gates in pipeline

> **Q: How do you handle urgent production bugs?**
>
> - Create hotfix branch from `main`
> - Fix and test in QA
> - Fast-track approval process
> - Merge to `main` (production)
> - Backport to `develop`
> - Tag as patch version (v1.0.1)

> **Q: What if a release needs to be rolled back?**
>
> - Use Git tags to identify previous version
> - Use Helm rollback command
> - Revert commits if necessary
> - Create new hotfix if code changes needed

> **Q: How do you ensure code quality before merge?**
>
> - Code review required (PR approval)
> - Automated CI checks (build, test, scan)
> - SonarQube quality gates
> - Security scans (Trivy)
> - Unit test coverage > 80%
