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
  "version": "1.0.0"
}
```

**Note**: Metadata files contain only version information. All other metadata (parameters, scope, description) is documented in the Bicep file itself using `@description` decorators and parameter definitions.

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
- Examples: `role-assignment/`, `managed-identity/`, `role-definition/`, `rg/`

### Module Hierarchy

```
product/rbac/main.bicep
    └── services/iam-resources/main.bicep
            ├── resources/rg/main.bicep
            ├── resources/role-definition/main.bicep
            ├── resources/managed-identity/main.bicep
            └── resources/role-assignment/main.bicep
```

## IAM Architecture

This repository implements a comprehensive Identity and Access Management (IAM) solution using three managed identities, three custom roles, and automatic role assignments.

### Managed Identities

**1. `iamRbacMI` - IAM/RBAC Deployment Identity**
- **Purpose**: Deploys and manages all IAM resources
- **Assigned Role**: IAM Deployment Operator (custom role)
- **Permissions**:
  - Role Assignments: `Microsoft.Authorization/roleAssignments/*`
  - Role Definitions: `Microsoft.Authorization/roleDefinitions/*`
  - Managed Identities: `Microsoft.ManagedIdentity/userAssignedIdentities/*`
  - Deployment Stacks: `Microsoft.Resources/deploymentStacks/*`
  - Resource Groups: `Microsoft.Resources/subscriptions/resourceGroups/*`
  - Deployments: `Microsoft.Resources/deployments/*`
  - Subscriptions Read: `Microsoft.Resources/subscriptions/read`

**2. `mgCreateMI` - Management Group Identity**
- **Purpose**: Manages management group operations and deployment stacks
- **Assigned Roles**:
  - MG Contributor with Deployment Stack (custom role)
  - Additional roles via `mgRoleAssignments` array parameter
- **Permissions** (via MG Contributor role):
  - Management Groups: `Microsoft.Management/managementGroups/*`
  - Deployment Stacks: `Microsoft.Resources/deploymentStacks/*`

**3. `subVendingMI` - Subscription Vending Identity**
- **Purpose**: Creates and manages subscriptions
- **Assigned Roles**: Configured via `subVendingRoleAssignments` array parameter
- **Expected Permissions** (via external role assignments):
  - Subscription operations: `Microsoft.Subscription/subscriptions/*`
  - Other permissions as needed

### Custom Roles

**1. IAM Deployment Operator**
- **Assigned to**: `iamRbacMI`
- **Purpose**: Complete control over IAM resource deployment
- **Key Actions**: Role assignments, role definitions, managed identities, deployment stacks, resource groups
- **Scope**: Subscription level

**2. MG Contributor with Deployment Stack**
- **Assigned to**: `mgCreateMI`
- **Purpose**: Management group operations and deployment stack management
- **Key Actions**: Management group operations, deployment stacks
- **Scope**: Subscription level

**3. Subscription Vending Operator**
- **Assigned to**: None (available for external assignment)
- **Purpose**: Subscription vending operations
- **Key Actions**: Subscription creation and management
- **Scope**: Subscription level

### Role Assignment Pattern

The architecture supports two types of role assignments:

**1. Automatic Custom Role Assignments** (defined in service module)
```bicep
// Automatically assigns custom roles to MIs
module mgContributorRoleAssignment = {
  principalId: mgCreateMI.principalId
  roleDefinitionId: mgContributorRole.roleDefinitionGuid
}

module iamDeploymentRoleAssignment = {
  principalId: iamRbacMI.principalId
  roleDefinitionId: iamDeploymentRole.roleDefinitionGuid
}
```

**2. Parameterized External Role Assignments** (defined in parameter files)
```bicep
// Allows assignment of built-in or external roles via arrays
param mgRoleAssignments = [
  {
    subscriptionId: '<subscription-id>'
    roleDefinitionId: '5d58bcaf-24a5-4b20-bdb6-eed9f69fbe4c' // MG Contributor
  }
]

param subVendingRoleAssignments = [
  {
    subscriptionId: '<subscription-id>'
    roleDefinitionId: 'b24988ac-6180-42a0-ab88-20f7382dd24c' // Contributor
  }
]
```

### Deployment Flow

1. **Bootstrap Phase** (using existing service principal/MI):
   - Deploys resource group
   - Creates three custom role definitions
   - Creates three managed identities

2. **Role Assignment Phase**:
   - Assigns `iamDeploymentRole` to `iamRbacMI`
   - Assigns `mgContributorRole` to `mgCreateMI`
   - Assigns external roles from parameter arrays

3. **Future Deployments** (using `iamRbacMI`):
   - Use `iamRbacMI` service connection in pipelines
   - This MI has full permissions to manage all IAM resources
   - No need for highly privileged service principals

### Parameter File Configuration

Each environment requires these parameters:

```bicep
// Managed identity names
param mgManagedIdentityName = 'mg-contributor'
param subVendingManagedIdentityName = 'sub-vending'
param iamRbacManagedIdentityName = 'iam-rbac'

// External role assignments
param mgRoleAssignments = [
  {
    subscriptionId: '<target-subscription-id>'
    roleDefinitionId: '<built-in-role-definition-id>'
  }
]

param subVendingRoleAssignments = [
  {
    subscriptionId: '<target-subscription-id>'
    roleDefinitionId: '<built-in-role-definition-id>'
  }
]
```

### Security Considerations

1. **Least Privilege**: Each MI has only the permissions needed for its specific purpose
2. **Custom Roles**: Custom roles provide minimal required permissions vs. broad built-in roles
3. **Separation of Duties**: Three distinct identities for three different operational areas
4. **Automatic Assignment**: Role assignments are codified and versioned in Bicep
5. **Deployment Stacks**: All resources managed via deployment stacks for consistency and drift detection

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
     "version": "1.1.0"
   }
   ```

2. **Create git tag** for releases:
   ```bash
   git tag <module-name>/v1.1.0
   git push origin <module-name>/v1.1.0
   ```

3. **Tag naming convention**: `<module-name>/v<version>`
   - Examples: `role-assignment/v1.0.0`, `rg/v1.1.0`, `role-definition/v1.0.0`

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
│       ├── rg/
│       │   ├── main.bicep
│       │   └── metadata.json
│       ├── role-assignment/
│       │   ├── main.bicep
│       │   └── metadata.json
│       ├── managed-identity/
│       │   ├── main.bicep
│       │   └── metadata.json
│       └── role-definition/
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
