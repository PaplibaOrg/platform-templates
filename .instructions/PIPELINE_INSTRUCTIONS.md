# Azure Pipeline Design Instructions

This document defines the standard patterns and conventions for creating Azure DevOps pipelines using **Azure Deployment Stacks** in this organization. AI assistants should follow these guidelines when creating or modifying pipelines.

## Pipeline Location

- All pipeline files must be placed in the `pipeline/` folder
- Main pipeline file: `pipeline/azure-pipelines.yml`
- Reusable templates: `pipeline/templates/` folder

## Core Principles

### 1. Pool Configuration
- Always use self-hosted agent pool: `pool: default`
- Never use Microsoft-hosted agents (no `vmImage`)

### 2. Trigger Configuration
```yaml
trigger:
  branches:
    include:
      - main
```

### 3. Variables Structure
Define separate variables for each environment:
```yaml
variables:
  serviceConnectionNonprod: '<nonprod-service-connection-name>'
  serviceConnectionProd: '<prod-service-connection-name>'
  subscriptionIdNonprod: '<nonprod-subscription-guid>'
  subscriptionIdProd: '<prod-subscription-guid>'
```

## Template-Based Pipeline Design

### Why Templates?
- **DRY Principle**: Avoid code duplication for multiple environments
- **Consistency**: Same deployment logic across all environments
- **Maintainability**: Update once, apply everywhere
- **Flexibility**: Support different configurations (approval gates, dependencies)

### Folder Structure
```
pipeline/
├── azure-pipelines.yml           # Main pipeline (orchestration)
└── templates/
    └── deploy-stack.yml          # Reusable deployment template
```

## Main Pipeline File (azure-pipelines.yml)

The main pipeline orchestrates deployments by calling the template with environment-specific parameters.

### Structure
```yaml
trigger:
  branches:
    include:
      - main

pool: default

variables:
  serviceConnectionNonprod: 'id-iam-rbac-us-nonprod-01'
  serviceConnectionProd: 'mi-rbac-admin-us-prod-01'
  subscriptionIdNonprod: '243c3cf8-1a50-4b3c-a45e-0d1f0a82a37b'
  subscriptionIdProd: '00000000-0000-0000-0000-000000000000'

stages:
  # Nonprod deployment (WhatIf → Deploy)
  - template: templates/deploy-stack.yml
    parameters:
      environment: 'Nonprod'
      serviceConnection: $(serviceConnectionNonprod)
      subscriptionId: $(subscriptionIdNonprod)
      parameterFile: 'plb-root/plb-platform-nonprod-01/nonprod.bicepparam'
      stackName: 'iam-stack-nonprod'

  # Prod deployment (WhatIf → Approval → Deploy, depends on Nonprod)
  - template: templates/deploy-stack.yml
    parameters:
      environment: 'Prod'
      serviceConnection: $(serviceConnectionProd)
      subscriptionId: $(subscriptionIdProd)
      parameterFile: 'plb-root/plb-platform-prod-01/prod.bicepparam'
      stackName: 'iam-stack-prod'
      dependsOn: 'Deploy_Nonprod'
      requiresApproval: true
```

### Key Features
- **Nonprod**: Automatic deployment (WhatIf → Deploy)
- **Prod**: Manual approval gate (WhatIf → Approval → Deploy)
- **Sequential**: Prod waits for Nonprod to complete
- **Parameterized**: All environment-specific values via template parameters

## Deployment Template (templates/deploy-stack.yml)

The template implements the WhatIf → Deploy (or WhatIf → Approval → Deploy) pattern.

### Template Parameters
```yaml
parameters:
  - name: environment            # Environment name (e.g., 'Nonprod', 'Prod')
    type: string
  - name: serviceConnection      # Azure service connection name
    type: string
  - name: subscriptionId         # Target subscription GUID
    type: string
  - name: parameterFile          # Path to .bicepparam file
    type: string
  - name: stackName              # Deployment stack name
    type: string
  - name: dependsOn              # Optional: stage dependency
    type: string
    default: ''
  - name: requiresApproval       # Optional: manual approval gate
    type: boolean
    default: false
```

### Template Structure
The template creates 2-3 stages per environment:

1. **WhatIf Stage**: Preview deployment changes
2. **Approval Stage** (optional): Manual approval gate for production
3. **Deploy Stage**: Execute deployment

### Complete Template Example
```yaml
parameters:
  - name: environment
    type: string
  - name: serviceConnection
    type: string
  - name: subscriptionId
    type: string
  - name: parameterFile
    type: string
  - name: stackName
    type: string
  - name: dependsOn
    type: string
    default: ''
  - name: requiresApproval
    type: boolean
    default: false

stages:
  # Stage 1: What-If Analysis
  - stage: WhatIf_${{ parameters.environment }}
    displayName: 'What-If ${{ parameters.environment }}'
    ${{ if ne(parameters.dependsOn, '') }}:
      dependsOn: ${{ parameters.dependsOn }}
    jobs:
      - job: WhatIfAnalysis
        displayName: 'Run What-If'
        steps:
          - task: AzurePowerShell@5
            displayName: 'What-If Stack'
            inputs:
              azureSubscription: ${{ parameters.serviceConnection }}
              ScriptType: 'InlineScript'
              Inline: |
                Set-AzContext -SubscriptionId '${{ parameters.subscriptionId }}'
                Set-AzSubscriptionDeploymentStack `
                  -Name "${{ parameters.stackName }}" `
                  -Location 'eastus' `
                  -TemplateFile 'modules/product/rbac/main.bicep' `
                  -TemplateParameterFile '${{ parameters.parameterFile }}' `
                  -DenySettingsMode 'None' `
                  -ActionOnUnmanage 'DeleteAll' `
                  -Force `
                  -WhatIf
              azurePowerShellVersion: 'LatestVersion'

  # Stage 2: Manual Approval (only if requiresApproval = true)
  - ${{ if eq(parameters.requiresApproval, true) }}:
    - stage: Approval_${{ parameters.environment }}
      displayName: 'Approve ${{ parameters.environment }} Deployment'
      dependsOn: WhatIf_${{ parameters.environment }}
      jobs:
        - deployment: Approval
          displayName: 'Manual Approval'
          environment: ${{ lower(parameters.environment) }}
          strategy:
            runOnce:
              deploy:
                steps:
                  - script: echo "${{ parameters.environment }} deployment approved"
                    displayName: 'Approval confirmed'

  # Stage 3: Deployment
  - stage: Deploy_${{ parameters.environment }}
    displayName: 'Deploy ${{ parameters.environment }}'
    ${{ if eq(parameters.requiresApproval, true) }}:
      dependsOn: Approval_${{ parameters.environment }}
    ${{ else }}:
      dependsOn: WhatIf_${{ parameters.environment }}
    jobs:
      - job: DeployStack
        displayName: 'Run Deploy'
        steps:
          - task: AzurePowerShell@5
            displayName: 'Deploy Stack'
            inputs:
              azureSubscription: ${{ parameters.serviceConnection }}
              ScriptType: 'InlineScript'
              Inline: |
                Set-AzContext -SubscriptionId '${{ parameters.subscriptionId }}'
                Set-AzSubscriptionDeploymentStack `
                  -Name "${{ parameters.stackName }}" `
                  -Location 'eastus' `
                  -TemplateFile 'modules/product/rbac/main.bicep' `
                  -TemplateParameterFile '${{ parameters.parameterFile }}' `
                  -DenySettingsMode 'None' `
                  -ActionOnUnmanage 'DeleteAll' `
                  -Force
              azurePowerShellVersion: 'LatestVersion'
```

## Azure Deployment Stacks

### Why Deployment Stacks (NOT regular deployments)?

**Deployment Stacks provide:**
- **Lifecycle Management**: Manage resources as a unit
- **Drift Detection**: Detect and prevent configuration drift
- **Deletion Control**: Control what happens to resources when removed from template
- **Resource Locking**: Prevent accidental deletions
- **Unified Management**: Track all resources created by a deployment

### PowerShell Cmdlets
```powershell
# Subscription-scoped deployment stack
Set-AzSubscriptionDeploymentStack

# Resource group-scoped deployment stack
Set-AzResourceGroupDeploymentStack

# Management group-scoped deployment stack
Set-AzManagementGroupDeploymentStack
```

### Required Parameters
```powershell
Set-AzSubscriptionDeploymentStack `
  -Name "<stack-name>" `              # Unique stack name
  -Location '<location>' `             # Azure region
  -TemplateFile '<path>.bicep' `       # Bicep template
  -TemplateParameterFile '<path>.bicepparam' `  # Parameter file
  -DenySettingsMode 'None' `           # Deny settings (None, DenyDelete, DenyWriteAndDelete)
  -ActionOnUnmanage 'DeleteAll' `      # What to do with unmanaged resources
  -Force                               # Skip confirmation prompts
```

### ActionOnUnmanage Options
- **`DeleteAll`**: Delete all resources when removed from template
- **`DeleteResources`**: Delete resources but keep resource groups
- **`DetachAll`**: Keep all resources, remove from stack tracking

**Best Practice**: Use `DeleteAll` for complete lifecycle management

### What-If Flag
- Add `-WhatIf` flag for preview/analysis stages
- Shows what resources will be created, modified, or deleted
- No actual changes are made

## Stage Naming Convention

### Automatic Stage Names (from template)
- `WhatIf_<Environment>` - Preview deployment changes
- `Approval_<Environment>` - Manual approval gate (if requiresApproval = true)
- `Deploy_<Environment>` - Execute deployment

### Example Stage Flow

**Nonprod (no approval)**:
```
WhatIf_Nonprod → Deploy_Nonprod
```

**Prod (with approval)**:
```
WhatIf_Prod → Approval_Prod → Deploy_Prod
```

**Full Pipeline**:
```
WhatIf_Nonprod → Deploy_Nonprod → WhatIf_Prod → Approval_Prod → Deploy_Prod
```

## Azure PowerShell Task Configuration

### Always Use AzurePowerShell@5
```yaml
- task: AzurePowerShell@5
  displayName: '<descriptive-name>'
  inputs:
    azureSubscription: ${{ parameters.serviceConnection }}
    ScriptType: 'InlineScript'
    Inline: |
      # PowerShell commands here
    azurePowerShellVersion: 'LatestVersion'
```

**Never use**:
- ❌ AzureCLI task
- ❌ Azure CLI commands (`az` commands)
- ❌ Older PowerShell task versions

### Context Switching
Always call `Set-AzContext` at the beginning:
```powershell
Set-AzContext -SubscriptionId '${{ parameters.subscriptionId }}'
```

This ensures the deployment targets the correct subscription.

## Manual Approval Gates

### Configuring Approvals

1. **In Pipeline Template**: Set `requiresApproval: true`
2. **In Azure DevOps**:
   - Go to Pipelines → Environments
   - Create environment (e.g., "prod", "nonprod")
   - Add approval checks
   - Add approvers

### Approval Stage Pattern
```yaml
- deployment: Approval
  displayName: 'Manual Approval'
  environment: ${{ lower(parameters.environment) }}  # Links to Azure DevOps environment
  strategy:
    runOnce:
      deploy:
        steps:
          - script: echo "${{ parameters.environment }} deployment approved"
            displayName: 'Approval confirmed'
```

**Key Points**:
- Uses `deployment` job type (not `job`)
- Links to Azure DevOps environment via `environment:` property
- Actual approval configured in Azure DevOps UI

## Parameter Files Structure

### Folder Structure (Mimic Management Group Hierarchy)
```
plb-root/                              # Root management group
├── plb-platform-nonprod-01/           # Nonprod MG/subscription
│   └── nonprod.bicepparam
└── plb-platform-prod-01/              # Prod MG/subscription
    └── prod.bicepparam
```

### Parameter File Format
```bicep
using '../../modules/product/rbac/main.bicep'

param environment = 'nonprod'
param application = 'platform-mi'
param region = 'us'
param sequence = '01'
param location = 'eastus'

param tags = {
  environment: 'nonprod'
  managedBy: 'bicep'
}

param mgManagedIdentityName = 'mg-contributor'
param subVendingManagedIdentityName = 'sub-vending'
param iamRbacManagedIdentityName = 'iam-rbac'

param mgRoleAssignments = [
  {
    subscriptionId: '243c3cf8-1a50-4b3c-a45e-0d1f0a82a37b'
    roleDefinitionId: '5d58bcaf-24a5-4b20-bdb6-eed9f69fbe4c'
  }
]
```

## Adding New Environments

To add a new environment (e.g., "QA"), add a template call to the main pipeline:

```yaml
stages:
  # Existing Nonprod and Prod deployments...

  # New QA deployment
  - template: templates/deploy-stack.yml
    parameters:
      environment: 'QA'
      serviceConnection: $(serviceConnectionQA)
      subscriptionId: $(subscriptionIdQA)
      parameterFile: 'plb-root/plb-platform-qa-01/qa.bicepparam'
      stackName: 'iam-stack-qa'
      dependsOn: 'Deploy_Nonprod'    # Optional: run after Nonprod
      requiresApproval: false         # Optional: add approval gate
```

**Steps**:
1. Add variables (`serviceConnectionQA`, `subscriptionIdQA`)
2. Create parameter file in `plb-root/`
3. Add template call to `stages` section
4. Configure dependencies and approval as needed

## Best Practices

### 1. Template Reusability
- ✅ Create reusable templates for common patterns
- ✅ Use parameters for environment-specific values
- ❌ Don't duplicate stages for each environment

### 2. Naming Conventions
- **Stack Names**: `<product>-stack-<environment>` (e.g., `iam-stack-nonprod`)
- **Service Connections**: `id-<identity-name>-<region>-<environment>-<sequence>`
- **Parameter Files**: `<environment>.bicepparam`

### 3. Security
- ✅ Use managed identities for service connections
- ✅ Use separate service connections per environment
- ✅ Apply least privilege to service connections
- ❌ Don't hardcode credentials or secrets

### 4. Deployment Stacks
- ✅ Use consistent stack names across environments
- ✅ Use `DeleteAll` for complete lifecycle management
- ✅ Always include WhatIf stage before Deploy
- ❌ Don't skip WhatIf analysis

### 5. Approvals
- ✅ Require approval for production deployments
- ✅ Configure approval in Azure DevOps environments
- ✅ Use environment-specific approvers
- ❌ Don't skip approvals for production

## Troubleshooting

### Common Issues

**Issue**: "Stack not found" error
- **Solution**: First deployment will create the stack, ignore this warning

**Issue**: "Subscription context not set"
- **Solution**: Ensure `Set-AzContext` is called at the start of every script

**Issue**: "WhatIf shows unexpected deletions"
- **Solution**: Check `ActionOnUnmanage` setting and review template changes

**Issue**: "Approval stage never triggers"
- **Solution**: Verify environment exists in Azure DevOps and has approval checks configured

## Checklist for AI

When creating or updating a pipeline:

- [ ] Use `pool: default` (self-hosted agents)
- [ ] Use `AzurePowerShell@5` task (not AzureCLI)
- [ ] Use Deployment Stacks (`Set-AzSubscriptionDeploymentStack`)
- [ ] Create reusable template in `pipeline/templates/`
- [ ] Include `Set-AzContext` in every PowerShell script
- [ ] Include `-ActionOnUnmanage 'DeleteAll'` in all stages
- [ ] Include `-WhatIf` flag in WhatIf stages
- [ ] Include `-Force` flag in Deploy stages
- [ ] Use template parameters for environment-specific values
- [ ] Add approval stage for production (`requiresApproval: true`)
- [ ] Set stage dependencies using `dependsOn` parameter
- [ ] Use separate service connections per environment
- [ ] Follow parameter file naming and location conventions
- [ ] Document any custom parameters or modifications

## Example: Complete Pipeline Setup

### 1. Create Template
File: `pipeline/templates/deploy-stack.yml`
```yaml
# See "Complete Template Example" section above
```

### 2. Create Main Pipeline
File: `pipeline/azure-pipelines.yml`
```yaml
trigger:
  branches:
    include:
      - main

pool: default

variables:
  serviceConnectionNonprod: 'id-iam-rbac-us-nonprod-01'
  serviceConnectionProd: 'id-iam-rbac-us-prod-01'
  subscriptionIdNonprod: '243c3cf8-1a50-4b3c-a45e-0d1f0a82a37b'
  subscriptionIdProd: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890'

stages:
  - template: templates/deploy-stack.yml
    parameters:
      environment: 'Nonprod'
      serviceConnection: $(serviceConnectionNonprod)
      subscriptionId: $(subscriptionIdNonprod)
      parameterFile: 'plb-root/plb-platform-nonprod-01/nonprod.bicepparam'
      stackName: 'iam-stack-nonprod'

  - template: templates/deploy-stack.yml
    parameters:
      environment: 'Prod'
      serviceConnection: $(serviceConnectionProd)
      subscriptionId: $(subscriptionIdProd)
      parameterFile: 'plb-root/plb-platform-prod-01/prod.bicepparam'
      stackName: 'iam-stack-prod'
      dependsOn: 'Deploy_Nonprod'
      requiresApproval: true
```

### 3. Create Parameter Files
File: `plb-root/plb-platform-nonprod-01/nonprod.bicepparam`
```bicep
using '../../modules/product/<product>/main.bicep'

param environment = 'nonprod'
param location = 'eastus'
# ... other parameters
```

### 4. Configure Azure DevOps
1. Create service connections in Azure DevOps
2. Create "nonprod" and "prod" environments
3. Add approval checks to "prod" environment
4. Run pipeline

## Summary

This template-based pipeline design provides:
- ✅ **Consistency**: Same deployment logic everywhere
- ✅ **Maintainability**: Update once, apply everywhere
- ✅ **Flexibility**: Easy to add environments or customize behavior
- ✅ **Safety**: WhatIf analysis and approval gates
- ✅ **Lifecycle Management**: Full control via Deployment Stacks
- ✅ **Scalability**: Easy to extend to multiple products/environments
