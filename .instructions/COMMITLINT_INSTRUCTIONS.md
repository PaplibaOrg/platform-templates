# Commitlint and Husky Setup Instructions

All repositories in this organization must have commitlint and husky configured to enforce conventional commit messages.

## Required Setup

Every new repository must include:
1. **Husky** - Git hooks manager
2. **Commitlint** - Commit message linter with conventional commit rules

## Installation Steps

### 1. Initialize npm (if not already done)
```bash
npm init -y
```

### 2. Install Dependencies
```bash
npm install --save-dev husky @commitlint/cli @commitlint/config-conventional
```

### 3. Initialize Husky
```bash
npx husky init
```

### 4. Create Commitlint Config
Create `commitlint.config.js` in the repo root:
```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
};
```

### 5. Create Commit-msg Hook
Create `.husky/commit-msg`:
```bash
npx --no -- commitlint --edit $1
```

### 6. Update .gitignore
Add the following to `.gitignore`:
```
# Node
node_modules/
package-lock.json
```

## Required Files

After setup, the repository should have:
```
.husky/
└── commit-msg
commitlint.config.js
package.json
.gitignore (with node_modules/)
```

## Commit Message Format

All commits must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

### Allowed Types
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation changes
- `style` - Code style changes (formatting, semicolons, etc.)
- `refactor` - Code refactoring
- `perf` - Performance improvements
- `test` - Adding or updating tests
- `build` - Build system or external dependencies
- `ci` - CI/CD configuration changes
- `chore` - Other changes (maintenance tasks)
- `revert` - Reverting a previous commit

### Examples
```bash
# Feature
feat: add user authentication module

# Bug fix
fix: resolve null pointer in login handler

# With scope
feat(api): add rate limiting to endpoints

# Breaking change
feat!: change API response format

# With body
fix: correct calculation in billing module

The previous formula was using incorrect tax rates
for international customers.
```

## Validation

Commits that don't follow the format will be rejected:
```
❌ git commit -m "fixed stuff"
✔️ git commit -m "fix: resolve login timeout issue"
```

## AI Instructions

When setting up a new repository:
1. Always configure husky and commitlint as the first step
2. Use the exact configuration shown above
3. Do not skip or modify the conventional commit rules
4. Ensure `.gitignore` includes `node_modules/` and `package-lock.json`
5. Remove any default husky pre-commit hook if not needed

## Quick Setup Script

For AI assistants, run these commands in sequence:
```bash
npm init -y
npm install --save-dev husky @commitlint/cli @commitlint/config-conventional
npx husky init
```

Then create the config files as specified above.
