# Azure Pipeline Design Instructions

This document defines the standard patterns and conventions for creating Azure DevOps pipelines in this organization. AI assistants should follow these guidelines when creating or modifying pipelines.

## Pipeline Location

- All pipeline files must be placed in the `pipeline/` folder
- Main pipeline file: `pipeline/azure-pipelines.yml`

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

## Stage Design Pattern

### Stage Naming Convention
- `WhatIf_<Environment>` - For what-if/preview stages
- `Deploy_<Environment>` - For deployment stages
- `Approval_<Environment>` - For manual approval gates

### Required Stages (in order)

1. **WhatIf_Nonprod** - Preview changes for nonprod
2. **Deploy_Nonprod** - Deploy to nonprod (depends on WhatIf_Nonprod)
3. **WhatIf_Prod** - Preview changes for prod (depends on Deploy_Nonprod)
4. **Approval_Prod** - Manual approval gate (depends on WhatIf_Prod)
5. **Deploy_Prod** - Deploy to prod (depends on Approval_Prod)

### Stage Flow
```
WhatIf_Nonprod → Deploy_Nonprod → WhatIf_Prod → Approval_Prod → Deploy_Prod
```

## Azure PowerShell Task Configuration

### Task Type
- Always use `AzurePowerShell@5` (NOT AzureCLI)
- Use PowerShell cmdlets, not Azure CLI commands

### Required Settings
```yaml
- task: AzurePowerShell@5
  displayName: '<descriptive-name>'
  inputs:
    azureSubscription: $(serviceConnection<Environment>)
    ScriptType: 'InlineScript'
    Inline: |
      Set-AzContext -SubscriptionId '$(subscriptionId<Environment>)'
      # PowerShell commands here
    azurePowerShellVersion: 'LatestVersion'
```

### Context Switching
- Always call `Set-AzContext -SubscriptionId` at the beginning of each PowerShell script
- This ensures deployment targets the correct subscription

## Deployment Stacks

### Use Deployment Stacks (NOT regular deployments)
- Use `Set-AzSubscriptionDeploymentStack` for subscription-scoped deployments
- Use `Set-AzResourceGroupDeploymentStack` for resource group-scoped deployments

### Required Parameters for Deployment Stacks
```powershell
Set-AzSubscriptionDeploymentStack `
  -Name "<stack-name>" `
  -Location '<location>' `
  -TemplateFile '<path-to-bicep>' `
  -TemplateParameterFile '<path-to-bicepparam>' `
  -DenySettingsMode 'None' `
  -ActionOnUnmanage 'DeleteResources'
```

### What-If Analysis
- Include `-WhatIf` flag for preview stages
- Include `-ActionOnUnmanage 'DeleteResources'` in what-if to show accurate preview of deletions

## Manual Approval Pattern

Use a dedicated approval stage with deployment job:
```yaml
- stage: Approval_<Environment>
  displayName: 'Approve <Environment> Deployment'
  dependsOn: WhatIf_<Environment>
  jobs:
    - deployment: Approval
      displayName: 'Manual Approval'
      environment: '<environment-name>'
      strategy:
        runOnce:
          deploy:
            steps:
              - script: echo "<Environment> deployment approved"
                displayName: 'Approval confirmed'
```

Configure approval checks on the environment in Azure DevOps (Pipelines → Environments).

## Bicep Parameter File Structure

### Folder Structure (Mimic Management Group Hierarchy)
```
<mg-root>/
├── <mg-child-nonprod>/
│   └── <name>.bicepparam
└── <mg-child-prod>/
    └── <name>.bicepparam
```

### Parameter File Format
```bicep
using '<relative-path-to-main.bicep>'

param environment = '<environment>'
param location = '<location>'

param tags = {
  project: '<project-name>'
  costCenter: '<cost-center>'
}
```

## Complete Pipeline Template

```yaml
trigger:
  branches:
    include:
      - main

pool: default

variables:
  serviceConnectionNonprod: '<nonprod-service-connection>'
  serviceConnectionProd: '<prod-service-connection>'
  subscriptionIdNonprod: '<nonprod-subscription-id>'
  subscriptionIdProd: '<prod-subscription-id>'

stages:
  # Nonprod What-If
  - stage: WhatIf_Nonprod
    displayName: 'What-If Nonprod'
    jobs:
      - job: WhatIfAnalysis
        displayName: 'Run What-If'
        steps:
          - task: AzurePowerShell@5
            displayName: 'What-If Analysis'
            inputs:
              azureSubscription: $(serviceConnectionNonprod)
              ScriptType: 'InlineScript'
              Inline: |
                Set-AzContext -SubscriptionId '$(subscriptionIdNonprod)'
                Set-AzSubscriptionDeploymentStack `
                  -Name "<stack-name>-nonprod" `
                  -Location '<location>' `
                  -TemplateFile '<template>.bicep' `
                  -TemplateParameterFile '<path>/nonprod.bicepparam' `
                  -DenySettingsMode 'None' `
                  -ActionOnUnmanage 'DeleteResources' `
                  -WhatIf
              azurePowerShellVersion: 'LatestVersion'

  # Nonprod Deploy
  - stage: Deploy_Nonprod
    displayName: 'Deploy Nonprod'
    dependsOn: WhatIf_Nonprod
    jobs:
      - job: DeployStack
        displayName: 'Deploy Deployment Stack'
        steps:
          - task: AzurePowerShell@5
            displayName: 'Deploy'
            inputs:
              azureSubscription: $(serviceConnectionNonprod)
              ScriptType: 'InlineScript'
              Inline: |
                Set-AzContext -SubscriptionId '$(subscriptionIdNonprod)'
                Set-AzSubscriptionDeploymentStack `
                  -Name "<stack-name>-nonprod" `
                  -Location '<location>' `
                  -TemplateFile '<template>.bicep' `
                  -TemplateParameterFile '<path>/nonprod.bicepparam' `
                  -DenySettingsMode 'None' `
                  -ActionOnUnmanage 'DeleteResources'
              azurePowerShellVersion: 'LatestVersion'

  # Prod What-If
  - stage: WhatIf_Prod
    displayName: 'What-If Prod'
    dependsOn: Deploy_Nonprod
    jobs:
      - job: WhatIfAnalysis
        displayName: 'Run What-If'
        steps:
          - task: AzurePowerShell@5
            displayName: 'What-If Analysis'
            inputs:
              azureSubscription: $(serviceConnectionProd)
              ScriptType: 'InlineScript'
              Inline: |
                Set-AzContext -SubscriptionId '$(subscriptionIdProd)'
                Set-AzSubscriptionDeploymentStack `
                  -Name "<stack-name>-prod" `
                  -Location '<location>' `
                  -TemplateFile '<template>.bicep' `
                  -TemplateParameterFile '<path>/prod.bicepparam' `
                  -DenySettingsMode 'None' `
                  -ActionOnUnmanage 'DeleteResources' `
                  -WhatIf
              azurePowerShellVersion: 'LatestVersion'

  # Manual Approval for Prod
  - stage: Approval_Prod
    displayName: 'Approve Prod Deployment'
    dependsOn: WhatIf_Prod
    jobs:
      - deployment: Approval
        displayName: 'Manual Approval'
        environment: 'prod'
        strategy:
          runOnce:
            deploy:
              steps:
                - script: echo "Prod deployment approved"
                  displayName: 'Approval confirmed'

  # Prod Deploy
  - stage: Deploy_Prod
    displayName: 'Deploy Prod'
    dependsOn: Approval_Prod
    jobs:
      - job: DeployStack
        displayName: 'Deploy Deployment Stack'
        steps:
          - task: AzurePowerShell@5
            displayName: 'Deploy'
            inputs:
              azureSubscription: $(serviceConnectionProd)
              ScriptType: 'InlineScript'
              Inline: |
                Set-AzContext -SubscriptionId '$(subscriptionIdProd)'
                Set-AzSubscriptionDeploymentStack `
                  -Name "<stack-name>-prod" `
                  -Location '<location>' `
                  -TemplateFile '<template>.bicep' `
                  -TemplateParameterFile '<path>/prod.bicepparam' `
                  -DenySettingsMode 'None' `
                  -ActionOnUnmanage 'DeleteResources'
              azurePowerShellVersion: 'LatestVersion'
```

## Checklist for AI

When creating a pipeline:
- [ ] Use `pool: default`
- [ ] Use `AzurePowerShell@5` task (not AzureCLI)
- [ ] Use Deployment Stacks (`Set-AzSubscriptionDeploymentStack`)
- [ ] Include `Set-AzContext` in every PowerShell script
- [ ] Include `-ActionOnUnmanage 'DeleteResources'` in all stages
- [ ] Include `-WhatIf` in What-If stages
- [ ] Create separate stages: WhatIf → Deploy for each environment
- [ ] Add Approval stage before Prod Deploy
- [ ] Use separate service connections and subscription IDs per environment
- [ ] Place parameter files in management group hierarchy structure
