# Repository Structure Instructions

This document defines the standard folder structure for platform repositories using Bicep IaC.

## Root Structure

```
<repo-root>/
├── .instructions/           # AI instruction files
├── .husky/                  # Git hooks
├── modules/                 # Bicep modules
├── pipeline/                # Azure DevOps pipelines
├── plb-root/                # Parameter files (MG hierarchy)
├── scripts/                 # Helper scripts (optional)
├── .gitignore
├── commitlint.config.js
└── package.json
```

## Modules Folder

The `modules/` folder contains all Bicep templates organized by type:

```
modules/
├── product/     # Main deployment templates (entry points)
├── services/    # Service-level modules (APIs, apps, etc.)
└── resources/   # Resource-level modules (storage, VMs, etc.)
```

### Module Types

#### Product (`modules/product/`)
- Main entry point Bicep files
- Orchestrates services and resources
- Called directly by pipelines
- Example: `main.bicep` for IAM deployment

#### Services (`modules/services/`)
- Service-specific modules
- Combines multiple resources into a logical service
- Examples: `api-management.bicep`, `web-app.bicep`

#### Resources (`modules/resources/`)
- Individual Azure resource modules
- Reusable across services
- Examples: `roleAssignment.bicep`, `managedIdentity.bicep`, `storageAccount.bicep`

## Parameter Files Structure

Parameter files follow the management group hierarchy:

```
plb-root/                              # Root management group
├── plb-platform-nonprod-01/           # Nonprod subscription/MG
│   └── <name>.bicepparam
└── plb-platform-prod-01/              # Prod subscription/MG
    └── <name>.bicepparam
```

### Naming Convention
- Folder names match management group or subscription names
- Parameter files use environment name: `nonprod.bicepparam`, `prod.bicepparam`

### Parameter File Format
```bicep
using '../../modules/product/main.bicep'

param environment = '<environment>'
param location = '<location>'

param tags = {
  project: '<project-name>'
  costCenter: '<cost-center>'
}
```

## Pipeline Folder

```
pipeline/
└── azure-pipelines.yml    # Main pipeline definition
```

- All pipeline files go in `pipeline/` folder
- Main pipeline file is always `azure-pipelines.yml`
- See `PIPELINE_INSTRUCTIONS.md` for pipeline design patterns

## Instructions Folder

```
.instructions/
├── PIPELINE_INSTRUCTIONS.md      # Pipeline design patterns
├── COMMITLINT_INSTRUCTIONS.md    # Commit message standards
└── REPOSITORY_STRUCTURE.md       # This file
```

- Contains AI instruction files
- Prefix with dot (`.`) to keep at top of directory listing
- Use UPPERCASE names for visibility

## Required Configuration Files

### .gitignore
```gitignore
# Bicep compiled ARM templates
*.json
!bicepconfig.json

# Azure CLI
.azure/

# Environment files
.env
*.env.local

# IDE
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Secrets
**/secrets/
*.pem
*.key

# Node
node_modules/
package-lock.json
```

### bicepconfig.json
```json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-hardcoded-env-urls": { "level": "warning" },
        "no-unused-params": { "level": "warning" },
        "no-unused-vars": { "level": "warning" },
        "prefer-interpolation": { "level": "warning" },
        "secure-parameter-default": { "level": "error" },
        "simplify-interpolation": { "level": "warning" }
      }
    }
  }
}
```

### commitlint.config.js
```javascript
module.exports = {
  extends: ['@commitlint/config-conventional'],
};
```

### package.json
Minimal package.json for husky and commitlint:
```json
{
  "name": "<repo-name>",
  "version": "1.0.0",
  "private": true,
  "devDependencies": {
    "@commitlint/cli": "^latest",
    "@commitlint/config-conventional": "^latest",
    "husky": "^latest"
  }
}
```

## Creating a New Platform Repository

### Step-by-step Setup

1. **Create repository**
   ```bash
   gh repo create <org>/<repo-name> --public
   ```

2. **Initialize structure**
   ```bash
   mkdir -p modules/{product,services,resources}
   mkdir -p pipeline
   mkdir -p plb-root/{plb-platform-nonprod-01,plb-platform-prod-01}
   mkdir -p .instructions
   ```

3. **Setup commitlint and husky**
   ```bash
   npm init -y
   npm install --save-dev husky @commitlint/cli @commitlint/config-conventional
   npx husky init
   ```

4. **Create commit-msg hook**
   ```bash
   echo 'npx --no -- commitlint --edit $1' > .husky/commit-msg
   ```

5. **Create config files**
   - `.gitignore`
   - `bicepconfig.json`
   - `commitlint.config.js`

6. **Copy instruction files**
   Copy from `platform-templates` repo:
   - `.instructions/PIPELINE_INSTRUCTIONS.md`
   - `.instructions/COMMITLINT_INSTRUCTIONS.md`
   - `.instructions/REPOSITORY_STRUCTURE.md`

7. **Create Bicep templates**
   - Main template in `modules/product/`
   - Parameter files in `plb-root/<mg-name>/`

8. **Create pipeline**
   - Pipeline file in `pipeline/azure-pipelines.yml`
   - Follow patterns in `PIPELINE_INSTRUCTIONS.md`

## AI Instructions

When creating a new platform repository:

1. **Always create this folder structure:**
   - `modules/product/`, `modules/services/`, `modules/resources/`
   - `pipeline/`
   - `plb-root/<nonprod-mg>/`, `plb-root/<prod-mg>/`
   - `.instructions/`

2. **Always setup commitlint and husky first**

3. **Always copy instruction files from platform-templates**

4. **Main Bicep files go in `modules/product/`**

5. **Parameter file paths must match management group hierarchy**

6. **Never put Bicep files in root** - use `modules/` folder

7. **Never put pipelines in root** - use `pipeline/` folder

## Example Complete Structure

```
azure-iam/
├── .husky/
│   └── commit-msg
├── .instructions/
│   ├── COMMITLINT_INSTRUCTIONS.md
│   ├── PIPELINE_INSTRUCTIONS.md
│   └── REPOSITORY_STRUCTURE.md
├── modules/
│   ├── product/
│   │   └── main.bicep
│   ├── services/
│   └── resources/
├── pipeline/
│   └── azure-pipelines.yml
├── plb-root/
│   ├── plb-platform-nonprod-01/
│   │   └── nonprod.bicepparam
│   └── plb-platform-prod-01/
│       └── prod.bicepparam
├── .gitignore
├── bicepconfig.json
├── commitlint.config.js
└── package.json
```
