# GitHub Copilot Instructions for DevOps Engineers

## Overview

This document provides guidelines and standards for DevOps engineers using GitHub Copilot when working with Azure infrastructure, Bicep templates, PowerShell scripts, and Azure DevOps pipelines. These standards ensure consistency, security, and maintainability across all infrastructure-as-code (IaC) projects.

**Target Technologies:**
- Azure Cloud Platform
- Bicep (Infrastructure as Code)
- PowerShell (Automation & Scripting)
- YAML (Pipeline & Configuration)
- Azure DevOps (CI/CD Pipelines)

---

## 1. Coding Standards

### 1.1 General Code Quality

- **Modularity**: Break code into reusable, single-responsibility modules
- **Comments**: Include meaningful comments for complex logic; use inline comments for "why" not "what"
- **Error Handling**: Implement comprehensive error handling with descriptive messages
- **Logging**: Log significant operations at appropriate levels (Info, Warning, Error)
- **DRY Principle**: Avoid code duplication; create shared functions and modules

### 1.2 Bicep Standards

```bicep
// Use descriptive parameter names with clear purposes
param environment string
param location string = resourceGroup().location
param tags object = {}

// Include metadata for parameters
@description('Environment name: dev, staging, or prod')
param environment string

@minLength(1)
@maxLength(24)
param storageAccountName string

// Use symbolic references instead of hardcoded values
var resourceGroupName = resourceGroup().name
var uniqueSuffix = uniqueString(resourceGroup().id)

// Organize outputs logically
output resourceId string = resource.id
output resourceName string = resource.name
```

**Bicep Conventions:**
- Use camelCase for variables and parameters
- Prefix internal variables with underscore: `_internalVar`
- Group related resources using symbolic references
- Always include descriptions for parameters and outputs
- Use symbolic names instead of string names for dependencies
- Implement consistent naming patterns (see Section 5)

### 1.3 PowerShell Standards

```powershell
# Use proper comment-based help
<#
.SYNOPSIS
    Deploys Azure resources using Bicep templates.

.DESCRIPTION
    Comprehensive description of what the script does.

.PARAMETER TemplatePath
    Path to the Bicep template file.

.EXAMPLE
    .\Deploy-Infrastructure.ps1 -TemplatePath './main.bicep' -Environment 'prod'

.NOTES
    Author: DevOps Team
    Version: 1.0
#>

[CmdletBinding()]
param(
    [Parameter(Mandatory = $true)]
    [ValidateScript({ Test-Path -Path $_ -PathType Leaf })]
    [string]$TemplatePath,

    [Parameter(Mandatory = $true)]
    [ValidateSet('dev', 'staging', 'prod')]
    [string]$Environment
)

# Use Set-StrictMode for enhanced error checking
Set-StrictMode -Version Latest
$ErrorActionPreference = 'Stop'

# Implement logging function
function Write-Log {
    param(
        [ValidateSet('Info', 'Warning', 'Error')]
        [string]$Level = 'Info',
        [string]$Message
    )
    
    $timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    Write-Host "[$timestamp] [$Level] $Message"
}

# Use consistent naming: Verb-Noun format
function Deploy-ResourceGroup {
    param([string]$ResourceGroupName)
    
    try {
        Write-Log -Level 'Info' -Message "Deploying resource group: $ResourceGroupName"
        # Implementation
    }
    catch {
        Write-Log -Level 'Error' -Message "Failed to deploy resource group: $_"
        throw
    }
}
```

**PowerShell Conventions:**
- Use PascalCase for function names following Verb-Noun pattern
- Use camelCase for variables
- Always use `[CmdletBinding()]` attribute
- Include parameter validation and type hints
- Use `Set-StrictMode -Version Latest` and `$ErrorActionPreference = 'Stop'`
- Implement structured error handling with try-catch
- Use Write-Verbose, Write-Warning, Write-Error for messaging

### 1.4 YAML Standards (Azure DevOps)

```yaml
# Use descriptive trigger conditions
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - 'infrastructure/**'

# Use meaningful names for stages and jobs
stages:
  - stage: Validate
    displayName: 'Validate Infrastructure Code'
    jobs:
      - job: BicepValidation
        displayName: 'Validate Bicep Templates'
        steps:
          - task: AzureCLI@2
            displayName: 'Validate Bicep Syntax'
            inputs:
              azureSubscription: '$(AzureSubscription)'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file main.bicep

# Use variables for repeated values
variables:
  buildConfiguration: 'Release'
  artifactName: 'infrastructure-artifacts'
  vmImage: 'ubuntu-latest'

# Include conditions for conditional steps
steps:
  - task: PublishBuildArtifacts@1
    condition: succeeded()
    displayName: 'Publish Artifacts'
```

**YAML Conventions:**
- Use 2-space indentation consistently
- Include `displayName` for all stages, jobs, and tasks for clarity
- Use template variables for reusable values
- Organize stages in logical order: Validate → Build → Test → Deploy
- Include conditions for conditional execution
- Keep YAML files under 500 lines; split large pipelines into templates

---

## 2. Security Standards

### 2.1 General Security Principles

- **Principle of Least Privilege**: Grant minimal required permissions
- **No Hardcoded Secrets**: Always use Azure Key Vault, secure variables, or managed identities
- **Encryption**: Enable encryption at rest and in transit for all data
- **Audit Logging**: Enable diagnostic logging for all deployed resources
- **Access Control**: Use role-based access control (RBAC) with specific roles

### 2.2 Azure Best Practices

```bicep
// Use managed identities instead of credentials
resource userManagedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'id-${environment}-${uniqueSuffix}'
  location: location
}

// Enable diagnostic settings
resource diagnosticSetting 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  scope: storageAccount
  name: 'diag-${environment}'
  properties: {
    logs: [
      {
        category: 'StorageRead'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'Transaction'
        enabled: true
      }
    ]
    workspaceId: logAnalyticsWorkspace.id
  }
}

// Use network security with NSGs and firewalls
resource networkSecurityGroup 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-${environment}'
  location: location
  properties: {
    securityRules: [
      {
        name: 'DenyAllInbound'
        properties: {
          protocol: '*'
          sourcePortRange: '*'
          destinationPortRange: '*'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Deny'
          priority: 4096
          direction: 'Inbound'
        }
      }
      {
        name: 'AllowHttpsInbound'
        properties: {
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
          access: 'Allow'
          priority: 100
          direction: 'Inbound'
        }
      }
    ]
  }
}
```

### 2.3 PowerShell Security

```powershell
# Use secure string handling
function Get-SecureValue {
    param(
        [Parameter(Mandatory = $true)]
        [string]$KeyVaultName,
        
        [Parameter(Mandatory = $true)]
        [string]$SecretName
    )
    
    $secret = Get-AzKeyVaultSecret -VaultName $KeyVaultName -Name $SecretName -AsPlainText
    return $secret
}

# Validate input to prevent injection attacks
function Deploy-Resource {
    param(
        [ValidatePattern('^[a-zA-Z0-9-]{1,24}$')]
        [string]$ResourceName
    )
    
    # Implementation
}

# Use parameterized queries and avoid string interpolation in sensitive contexts
$deploymentName = "deploy-$(Get-Date -Format 'yyyyMMddHHmmss')"

# Never log sensitive information
Write-Log -Level 'Info' -Message "Deploying resource with managed identity"
# Instead of: Write-Log -Level 'Info' -Message "Using password: $password"

# Use TLS 1.2 minimum
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
```

### 2.4 Azure DevOps Pipeline Security

```yaml
# Use secure files for certificates and keys
steps:
  - task: DownloadSecureFile@1
    name: downloadCert
    displayName: 'Download Certificate'
    inputs:
      secureFile: 'app.pfx'

# Use service connection with least privilege
- task: AzureCLI@2
  displayName: 'Azure CLI Task'
  inputs:
    azureSubscription: 'Production-ServiceConnection'  # Audit-able, scoped connection
    scriptType: 'bash'

# Never output secrets in logs
- script: |
    echo "##vso[task.setvariable variable=MySecret;issecret=true]$(secretValue)"
  displayName: 'Set Secret Variable'

# Enable continuous deployment protection for production
- ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
  - task: ManualValidation@0
    displayName: 'Approve Production Deployment'
    inputs:
      instructions: 'Review and approve production deployment'
      onTimeout: 'reject'
```

---

## 3. Infrastructure as Code (IaC) Best Practices

### 3.1 Bicep IaC Standards

**Module Structure:**
```
infrastructure/
├── modules/
│   ├── compute/
│   │   ├── vm.bicep
│   │   └── vmss.bicep
│   ├── networking/
│   │   ├── vnet.bicep
│   │   └── nsg.bicep
│   ├── storage/
│   │   ├── storageAccount.bicep
│   │   └── managedDisk.bicep
│   └── monitoring/
│       ├── logAnalytics.bicep
│       └── applicationInsights.bicep
├── environments/
│   ├── dev.bicep
│   ├── staging.bicep
│   └── prod.bicep
├── main.bicep
└── README.md
```

**Module Best Practices:**

```bicep
// Reusable module with comprehensive parameters
metadata description = 'Creates a managed storage account with security settings'

@description('Environment name for resource naming')
param environment string

@description('Azure region for resource deployment')
param location string = resourceGroup().location

@description('Tags to apply to all resources')
param tags object = {}

@minLength(3)
@maxLength(24)
@description('Storage account name')
param storageAccountName string

@description('Storage account kind')
@allowed([
  'StorageV2'
  'BlobStorage'
  'FileStorage'
])
param kind string = 'StorageV2'

@description('Storage account tier')
@allowed([
  'Standard'
  'Premium'
])
param tier string = 'Standard'

// Implement variables for reusable values
var resourceNamePrefix = '${environment}-${location}'
var skuName = '${tier}_${kind == 'StorageV2' ? 'GRSV2' : 'GRS'}'

// Create resource with security best practices
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  kind: kind
  sku: {
    name: skuName
  }
  tags: tags
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    accessTier: 'Hot'
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
    allowBlobPublicAccess: false
    networkAcls: {
      bypass: 'AzureServices'
      defaultAction: 'Deny'
    }
  }
}

// Disable public access blob
resource blobServices 'Microsoft.Storage/storageAccounts/blobServices@2023-01-01' = {
  name: 'default'
  parent: storageAccount
  properties: {
    deleteRetentionPolicy: {
      enabled: true
      days: 7
    }
  }
}

// Output important resource information
@description('Storage account resource ID')
output storageAccountId string = storageAccount.id

@description('Storage account primary blob endpoint')
output primaryBlobEndpoint string = storageAccount.properties.primaryEndpoints.blob

@description('Storage account managed identity principal ID')
output managedIdentityPrincipalId string = storageAccount.identity.principalId
```

### 3.2 Configuration as Code

```bicep
// Use parameter files for environment-specific values
// main.bicep references environment-specific parameters

// parameters-dev.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "value": "dev"
    },
    "vmSize": {
      "value": "Standard_B2s"
    },
    "storageAccountTier": {
      "value": "Standard"
    }
  }
}

// parameters-prod.json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": {
      "value": "prod"
    },
    "vmSize": {
      "value": "Standard_D4s_v3"
    },
    "storageAccountTier": {
      "value": "Premium"
    }
  }
}
```

### 3.3 Versioning and Dependencies

- **Semantic Versioning**: Follow semver for infrastructure components (MAJOR.MINOR.PATCH)
- **Module Versions**: Tag Bicep modules with versions in Git
- **API Versions**: Always use latest stable API versions; document API version selection rationale
- **Change Documentation**: Include CHANGELOG.md documenting all changes

---

## 4. Pipeline Standards

### 4.1 Pipeline Structure

```yaml
# Define trigger strategy
trigger:
  branches:
    include:
      - main
      - develop
      - feature/*
  paths:
    include:
      - 'infrastructure/**'
    exclude:
      - 'infrastructure/docs/**'

# Define pull request triggers
pr:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - 'infrastructure/**'

# Define variables for pipeline
variables:
  - group: 'DevOps-Variables'  # Variable group with common values
  - name: 'buildConfiguration'
    value: 'Release'
  - name: 'artifactName'
    value: 'bicep-artifacts'
  - name: 'vmImage'
    value: 'ubuntu-latest'

# Define stages with clear progression
stages:
  # Stage 1: Validation
  - stage: Validate
    displayName: 'Validate Infrastructure Code'
    jobs:
      - job: CodeQuality
        displayName: 'Code Quality and Validation'
        
  # Stage 2: Build
  - stage: Build
    displayName: 'Build Infrastructure Artifacts'
    dependsOn: Validate
    
  # Stage 3: Test
  - stage: Test
    displayName: 'Test Infrastructure'
    dependsOn: Build
    
  # Stage 4: Deploy Dev
  - stage: DeployDev
    displayName: 'Deploy to Development'
    dependsOn: Test
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/develop')
    
  # Stage 5: Deploy Prod
  - stage: DeployProd
    displayName: 'Deploy to Production'
    dependsOn: DeployDev
    condition: eq(variables['Build.SourceBranch'], 'refs/heads/main')
```

### 4.2 Validation Stage

```yaml
- stage: Validate
  displayName: 'Validate Infrastructure Code'
  jobs:
    - job: BicepValidation
      displayName: 'Bicep Template Validation'
      steps:
        - checkout: self
          fetchDepth: 0
          
        - task: AzureCLI@2
          displayName: 'Validate Bicep Syntax'
          inputs:
            azureSubscription: '$(AzureSubscription)'
            scriptType: 'bash'
            scriptLocation: 'inlineScript'
            inlineScript: |
              for file in $(find infrastructure -name '*.bicep'); do
                echo "Validating: $file"
                az bicep build --file "$file" || exit 1
              done
              
        - task: PowerShell@2
          displayName: 'PSScriptAnalyzer'
          inputs:
            targetType: 'inline'
            script: |
              Install-Module -Name PSScriptAnalyzer -Force -Scope CurrentUser
              $results = Invoke-ScriptAnalyzer -Path './scripts' -Recurse
              if ($results) {
                $results | Format-Table
                exit 1
              }
              
    - job: SecurityScanning
      displayName: 'Security Scanning'
      steps:
        - task: AzureCLI@2
          displayName: 'Scan for Security Issues'
          inputs:
            azureSubscription: '$(AzureSubscription)'
            scriptType: 'bash'
            scriptLocation: 'inlineScript'
            inlineScript: |
              # Example: Scan for hardcoded secrets
              grep -r "password\|apiKey\|secret" infrastructure/ && exit 1 || true
              echo "No obvious secrets detected"
```

### 4.3 Deployment Stage

```yaml
- stage: DeployDev
  displayName: 'Deploy to Development'
  dependsOn: Build
  jobs:
    - deployment: DeployInfra
      displayName: 'Deploy Infrastructure'
      environment: 'Development'
      strategy:
        runOnce:
          deploy:
            steps:
              - checkout: self
              
              - task: DownloadPipelineArtifact@2
                displayName: 'Download Build Artifacts'
                inputs:
                  artifactName: '$(artifactName)'
                  targetPath: '$(Pipeline.Workspace)/artifacts'
                  
              - task: AzureCLI@2
                displayName: 'Deploy Bicep Template'
                inputs:
                  azureSubscription: 'Dev-ServiceConnection'
                  scriptType: 'bash'
                  scriptLocation: 'inlineScript'
                  inlineScript: |
                    az deployment group create \
                      --name "deploy-$(Build.BuildNumber)" \
                      --resource-group "rg-dev-$(location)" \
                      --template-file "main.bicep" \
                      --parameters "parameters-dev.json" \
                      --output table
                      
              - task: AzureCLI@2
                displayName: 'Run Post-Deployment Validation'
                inputs:
                  azureSubscription: 'Dev-ServiceConnection'
                  scriptType: 'bash'
                  scriptLocation: 'inlineScript'
                  inlineScript: |
                    # Validate deployment success
                    az deployment group show \
                      --name "deploy-$(Build.BuildNumber)" \
                      --resource-group "rg-dev-$(location)" \
                      --query "properties.provisioningState"
```

### 4.4 Approval Gates

```yaml
# Add manual approval for production deployments
- stage: DeployProd
  displayName: 'Deploy to Production'
  dependsOn: DeployStaging
  jobs:
    # Approval step before deployment
    - job: ApprovalWait
      displayName: 'Wait for Approval'
      pool: server
      steps:
        - task: ManualValidation@0
          timeoutInMinutes: 1440  # 24 hours
          inputs:
            notifyUsers: '$(ApprovalGroup)'
            instructions: |
              Please review the staging deployment and approve production deployment.
              - Verify infrastructure changes are correct
              - Confirm no security issues
              - Validate monitoring and alerting
            onTimeout: 'reject'
            
    - deployment: DeployToProduction
      displayName: 'Deploy Production Infrastructure'
      dependsOn: ApprovalWait
      environment: 'Production'
      strategy:
        runOnce:
          deploy:
            steps:
              # Deployment steps here
```

---

## 5. Naming Conventions

### 5.1 General Naming Rules

- **Format**: Use lowercase alphanumeric characters with hyphens as separators
- **Length**: Keep names concise but descriptive (max 24 characters for resources with limitations)
- **Pattern**: `[resourceType]-[environment]-[project]-[instance]`
- **Avoid**: Underscores, special characters (except hyphens), spaces
- **Uniqueness**: Ensure global uniqueness for public-facing resources (storage accounts, app names)

### 5.2 Azure Resource Naming Convention

```
Resource Type               | Abbreviation | Example
------------------------------------------------------
Resource Group              | rg           | rg-prod-webapp-001
Storage Account            | st           | stprodwebapp001
Virtual Network            | vnet         | vnet-prod-core-001
Subnet                      | snet         | snet-prod-app-001
Network Security Group      | nsg          | nsg-prod-app-001
Virtual Machine            | vm           | vm-prod-web-001
Virtual Machine Scale Set  | vmss         | vmss-prod-app-001
App Service Plan           | asp          | asp-prod-api-001
App Service                | app          | app-prod-api-001
Application Gateway        | agw          | agw-prod-web-001
Load Balancer              | lb           | lb-prod-api-001
Key Vault                  | kv           | kv-prod-core-001
Application Insights       | appi         | appi-prod-app-001
Log Analytics Workspace    | law          | law-prod-core-001
Azure SQL Database Server  | sql          | sql-prod-data-001
Azure SQL Database         | sqldb        | sqldb-prod-app-001
Managed Identity           | id           | id-prod-app-001
```

### 5.3 Environment Designations

```
Environment | Abbreviation | Usage
------------------------------------------------------
Development | dev          | Development and testing
Staging     | staging      | Pre-production validation
Production  | prod         | Live production environment
```

### 5.4 Bicep Variable Naming

```bicep
// Local variables (internal use)
var resourceNamePrefix = 'rg-${environment}'
var uniqueSuffix = uniqueString(resourceGroup().id)
var location = resourceGroup().location

// Parameters (external input)
param environment string
param location string
param tags object

// Outputs (exported values)
output resourceGroupId string = resourceGroup().id
output storageAccountId string = storageAccount.id

// Constants (fixed values)
var requiredTlsVersion = 'TLS1_2'
var minReplicaCount = 2
```

### 5.5 Script and Function Naming

```powershell
# PowerShell Functions: Verb-Noun format
Deploy-Infrastructure
Get-ResourceGroupStatus
Test-BicepTemplate
Set-ResourceTags
Invoke-DeploymentValidation

# Script Files: Verb-Noun.ps1 format
Deploy-Infrastructure.ps1
Validate-Templates.ps1
Monitor-Resources.ps1

# Variables: camelCase
$resourceGroupName
$deploymentStatus
$isProduction
```

---

## 6. Code Review Checklist

### 6.1 Pre-Review Checklist (Author)

**General Code Quality:**
- [ ] Code follows all coding standards outlined in Section 1
- [ ] No commented-out code remains (except temporary debugging)
- [ ] Error handling is comprehensive with meaningful messages
- [ ] Logging is implemented appropriately
- [ ] Code is DRY (no unnecessary duplication)
- [ ] Comments explain "why" not "what"

**Bicep-Specific:**
- [ ] All parameters have @description metadata
- [ ] All outputs have @description metadata
- [ ] Parameter validation (minLength, maxLength, allowed values) is appropriate
- [ ] Symbolic references used instead of hardcoded resource names
- [ ] Module is reusable and not environment-specific
- [ ] API versions are latest stable (as of submission date)
- [ ] No hardcoded values except for constants
- [ ] Consistent naming conventions applied
- [ ] Module has been validated with `az bicep build`

**PowerShell-Specific:**
- [ ] Script includes comment-based help with all sections
- [ ] [CmdletBinding()] attribute is used
- [ ] Parameters have [Parameter()] attributes with Mandatory and Validation
- [ ] Parameter type hints specified
- [ ] Set-StrictMode -Version Latest is set
- [ ] $ErrorActionPreference = 'Stop' is set
- [ ] Try-catch blocks for error handling
- [ ] Functions follow Verb-Noun naming pattern
- [ ] Script has been tested locally
- [ ] PSScriptAnalyzer passes without warnings

**YAML-Specific:**
- [ ] Consistent 2-space indentation throughout
- [ ] All stages, jobs, and tasks have displayName
- [ ] Template variables used for repeated values
- [ ] Trigger conditions are appropriate for branch
- [ ] Pipeline stages ordered logically
- [ ] Conditions specified for conditional steps
- [ ] No hardcoded secrets or credentials
- [ ] Pipeline validated with YAML schema

**Security Review:**
- [ ] No hardcoded secrets, passwords, or API keys
- [ ] Managed identities used instead of credentials
- [ ] RBAC roles follow principle of least privilege
- [ ] Encryption enabled at rest and in transit
- [ ] Network security configured appropriately
- [ ] Diagnostic logging enabled
- [ ] Service connections use appropriate scope and permissions
- [ ] Secure variables used for sensitive data
- [ ] No sensitive information in logs

**IaC Best Practices:**
- [ ] Resources organized into logical modules
- [ ] Parameter files separate from templates
- [ ] Dependencies correctly defined
- [ ] Tags applied to all resources
- [ ] Resource naming follows conventions
- [ ] Documentation updated
- [ ] Version/CHANGELOG updated

### 6.2 Reviewer Checklist

**Functionality & Correctness:**
- [ ] Does the code accomplish the intended purpose?
- [ ] Are all requirements met?
- [ ] Have edge cases been considered?
- [ ] Is error handling appropriate?
- [ ] Would this code work in production?

**Code Quality:**
- [ ] Does code follow established standards?
- [ ] Is code maintainable and understandable?
- [ ] Are there any code smells or anti-patterns?
- [ ] Could this be refactored for better clarity?
- [ ] Is there unnecessary complexity?

**Security:**
- [ ] No secrets in code or configuration?
- [ ] Are permissions appropriately restricted?
- [ ] Is data properly encrypted?
- [ ] Are there any security vulnerabilities?
- [ ] Does this follow security best practices?

**Performance:**
- [ ] Will this scale appropriately?
- [ ] Are there obvious performance issues?
- [ ] Is resource usage optimized?
- [ ] Are there any bottlenecks?

**Testing:**
- [ ] Has code been tested locally?
- [ ] Are there appropriate automated tests?
- [ ] Have edge cases been tested?
- [ ] Does CI/CD pipeline pass all checks?

**Documentation:**
- [ ] Is documentation clear and complete?
- [ ] Are complex sections explained?
- [ ] Is README updated if needed?
- [ ] Are code comments helpful?
- [ ] Is CHANGELOG updated?

### 6.3 Azure-Specific Review Points

**Bicep Templates:**
- [ ] Template validates without errors
- [ ] What-if deployment reviewed (if applicable)
- [ ] No deprecation warnings in outputs
- [ ] Idempotent (can be deployed multiple times safely)
- [ ] Works in target Azure environment/region
- [ ] Naming conventions prevent conflicts
- [ ] Proper use of copy loops if needed
- [ ] Symbolic references throughout (no string names)

**PowerShell Scripts:**
- [ ] Script runs without errors
- [ ] Error messages are clear and helpful
- [ ] Exit codes appropriate
- [ ] Verbose output useful for debugging
- [ ] No Azure CLI/PowerShell version conflicts
- [ ] Handles authentication correctly
- [ ] Cleans up resources on failure (if applicable)

**Azure DevOps Pipelines:**
- [ ] Triggers appropriate for use case
- [ ] Pipeline gated with appropriate approval
- [ ] Service connections properly scoped
- [ ] Secrets not exposed in pipeline logs
- [ ] Deployment order correct
- [ ] Rollback strategy documented
- [ ] Notifications configured for failures

### 6.4 Review Comment Template

```markdown
## Review Comment Template

**Category**: [Security | Performance | Code Quality | Documentation | IaC Best Practice]

**Issue**: Brief description of the issue found

**Location**: File path and line number(s)

**Current Code**:
```
code snippet
```

**Suggested Change**:
```
improved code
```

**Rationale**: Explanation of why this change is recommended

**Resources**: Links to relevant documentation or standards
```

---

## 7. Additional Resources

### 7.1 Learning and Documentation

- [Azure Bicep Documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Resource Manager Best Practices](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/best-practices)
- [PowerShell Best Practices](https://learn.microsoft.com/en-us/powershell/scripting/development/best-practices/best-practices)
- [Azure DevOps Documentation](https://learn.microsoft.com/en-us/azure/devops/)
- [Azure Security Best Practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/)

### 7.2 Tools and Linters

```bash
# Bicep validation
az bicep build --file template.bicep
az bicep lint --file template.bicep

# PowerShell analysis
Install-Module -Name PSScriptAnalyzer
Invoke-ScriptAnalyzer -Path './scripts' -Recurse

# YAML validation
yamllint azure-pipelines.yml

# Azure naming validation tool
# https://github.com/Azure/azure-naming-tool
```

### 7.3 Bicep Linter Rules to Enable

Create `bicepconfig.json`:
```json
{
  "analyzers": {
    "core": {
      "enabled": true,
      "rules": {
        "no-unused-params": "error",
        "no-unused-variables": "error",
        "no-hardcoded-env-urls": "error",
        "no-hardcoded-locations": "warning",
        "admin-username-should-not-be-literal": "error",
        "no-loc-expr-outside-params": "error",
        "outputs-should-not-contain-secrets": "error"
      }
    }
  }
}
```

### 7.4 Pre-Commit Hooks

```bash
# Install pre-commit
pip install pre-commit

# Create .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: bicep-validate
        name: Bicep Validate
        entry: az bicep build
        language: system
        files: \.bicep$
        
      - id: bicep-lint
        name: Bicep Lint
        entry: az bicep lint
        language: system
        files: \.bicep$
        
      - id: pslint
        name: PowerShell Script Analyzer
        entry: pwsh -NoProfile -Command "Invoke-ScriptAnalyzer"
        language: system
        files: \.ps1$
        
      - id: yaml-lint
        name: YAML Lint
        entry: yamllint
        language: system
        files: \.ya?ml$

# Install hooks
pre-commit install
```

---

## 8. Compliance and Governance

### 8.1 Deployment Approvals

- **Dev**: Auto-deploy on merge to develop branch
- **Staging**: Auto-deploy on successful dev deployment
- **Production**: Manual approval required before deployment to main
- **Rollback**: Document and execute rollback procedures for failed deployments

### 8.2 Audit and Logging

- Enable Activity Log for all resource group deployments
- Enable Azure Monitor diagnostic settings
- Configure log retention per organizational policy
- Regular audits of RBAC assignments
- Review of deployment history quarterly

### 8.3 Change Management

- All infrastructure changes must go through Pull Requests
- Minimum 2 approvals for production changes
- Link pull requests to Azure DevOps work items
- Document breaking changes in CHANGELOG.md
- Notify stakeholders of major infrastructure changes

---

## 9. Troubleshooting and Support

### 9.1 Common Issues

**Bicep Build Failures:**
```bash
# Check for syntax errors
az bicep build --file main.bicep --verbose

# Validate against deployment target
az deployment group validate \
  --resource-group "my-rg" \
  --template-file main.bicep \
  --parameters parameters.json
```

**PowerShell Execution Issues:**
```powershell
# Check script for issues
Get-Content script.ps1 | Out-Host
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Debug script execution
$DebugPreference = 'Continue'
& .\script.ps1
```

**Pipeline Failures:**
- Check agent logs in Azure DevOps
- Verify service connection permissions
- Validate YAML syntax
- Check trigger conditions
- Review variable group permissions

### 9.2 Getting Help

- **Internal**: Contact DevOps team in Teams channel
- **Azure Support**: Use Azure DevOps service connections with support plan
- **Community**: Check StackOverflow with tags [azure], [bicep], [azure-devops]

---

## Feedback and Updates

This document is living and should be updated as standards evolve. 

**To suggest changes:**
1. Create a pull request with proposed updates
2. Include rationale for changes
3. Request review from DevOps team lead
4. Update version number in document header
5. Announce changes to team

**Last Updated**: [Date]
**Version**: 1.0
**Owner**: DevOps Team
