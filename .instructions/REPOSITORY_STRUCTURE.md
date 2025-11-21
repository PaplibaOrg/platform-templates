# Repository Structure Instructions (Bicep IaC)

> **Note:** This instruction file is specifically for **Bicep IaC repositories**. Other repository types (e.g., applications, Terraform, etc.) will have separate instruction files.

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

The `modules/` folder contains all Bicep templates organized by type. Each module has its own folder containing `main.bicep` and `metadata.json`:

```
modules/
├── product/           # Main deployment templates (entry points)
│   └── <module-name>/
│       ├── main.bicep
│       └── metadata.json
├── services/          # Service-level modules (APIs, apps, etc.)
│   └── <module-name>/
│       ├── main.bicep
│       └── metadata.json
└── resources/         # Resource-level modules (storage, VMs, etc.)
    └── <module-name>/
        ├── main.bicep
        └── metadata.json
```

### Module Structure

Each module folder must contain:
- `main.bicep` - The Bicep template
- `metadata.json` - Version and module information

#### metadata.json Format
```json
{
  "name": "<module-name>",
  "version": "1.0.0",
  "description": "Description of the module",
  "scope": "subscription|resourceGroup",
  "dependencies": [],
  "parameters": ["param1", "param2"]
}
```

### Module Types

#### Product (`modules/product/`)
- Main entry point Bicep files
- Orchestrates services modules
- Called directly by pipelines
- References modules in `services/`
- Example: `modules/product/rbac/main.bicep`

#### Services (`modules/services/`)
- Service-specific modules
- Combines multiple resource modules into a logical service
- References modules in `resources/`
- Example: `modules/services/iam-resources/main.bicep`

#### Resources (`modules/resources/`)
- Individual Azure resource modules
- Reusable across services
- Standalone modules (no dependencies)
- Examples: `role-assignment/`, `managed-identity/`, `custom-role-definition/`

### Module Hierarchy

```
product/rbac/main.bicep
    └── services/iam-resources/main.bicep
            └── resources/iam-rg/main.bicep
```

### Module Versioning

Use **Git Tags + metadata.json** for version management:

#### Semantic Versioning
- **Major** (1.0.0 → 2.0.0): Breaking changes (parameter removed, scope changed)
- **Minor** (1.0.0 → 1.1.0): New features, backward compatible (new optional parameter)
- **Patch** (1.0.0 → 1.0.1): Bug fixes, no API changes

#### Versioning Workflow

1. **Update metadata.json** when making changes:
   ```json
   {
     "name": "role-assignment",
     "version": "1.1.0",
     ...
   }
   ```

2. **Create git tag** for releases:
   ```bash
   git tag <module-name>/v1.1.0
   git push origin <module-name>/v1.1.0
   ```

3. **Tag naming convention**: `<module-name>/v<version>`
   - Examples: `role-assignment/v1.0.0`, `iam-rg/v1.1.0`

#### Version History
- Git history serves as version history
- No need to create version folders or file copies
- Use `git log --oneline modules/resources/<module-name>/` to view changes

#### Future: Azure Container Registry (ACR)
For centralized module sharing, publish to ACR:
```bash
bicep publish main.bicep --target br:myregistry.azurecr.io/bicep/modules/<module-name>:<version>
```

Reference in Bicep:
```bicep
module example 'br:myregistry.azurecr.io/bicep/modules/role-assignment:1.0.0' = { }
```

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
using '../../modules/product/<module-name>/main.bicep'

param environment = '<environment>'
param application = '<application>'
param region = '<region>'
param sequence = '<sequence>'
param location = '<location>'
param tags = {
  environment: '<environment>'
  managedBy: 'bicep'
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

4. **Each module must have its own folder with `main.bicep` and `metadata.json`**

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
│   │   └── rbac/
│   │       ├── main.bicep
│   │       └── metadata.json
│   ├── services/
│   │   └── iam-resources/
│   │       ├── main.bicep
│   │       └── metadata.json
│   └── resources/
│       ├── iam-rg/
│       │   ├── main.bicep
│       │   └── metadata.json
│       ├── role-assignment/
│       │   ├── main.bicep
│       │   └── metadata.json
│       ├── managed-identity/
│       │   ├── main.bicep
│       │   └── metadata.json
│       └── custom-role-definition/
│           ├── main.bicep
│           └── metadata.json
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
