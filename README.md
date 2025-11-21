# Platform Templates

Central repository for organization-wide standards, templates, and AI instructions.

## Purpose

This repository contains instruction files that define how projects should be structured and configured. AI assistants should read these files to understand organizational conventions.

## Structure

```
platform-templates/
├── .instructions/
│   ├── PIPELINE_INSTRUCTIONS.md    # Azure DevOps pipeline patterns
│   └── COMMITLINT_INSTRUCTIONS.md  # Husky + commitlint setup
└── README.md
```

## Instructions

### Pipeline Instructions
Defines standard patterns for Azure DevOps pipelines including:
- Stage design (WhatIf → Deploy → Approval)
- Deployment Stacks usage
- AzurePowerShell task configuration
- Environment separation (nonprod/prod)

### Commitlint Instructions
Defines commit message standards including:
- Husky and commitlint setup
- Conventional commit format
- Required configuration files

## Usage

### For New Repositories
1. Copy the `.instructions/` folder to your new repository
2. Follow the instructions to set up pipelines and commitlint
3. AI assistants will automatically read these files for context

### For AI Assistants
When working on repositories in this organization:
1. Check for `.instructions/` folder
2. Read all instruction files before making changes
3. Follow the patterns and conventions defined

## Adding New Instructions

1. Create a new markdown file in `.instructions/`
2. Use clear headings and code examples
3. Include an "AI Instructions" section with explicit guidance
4. Update this README with the new file description
