# Azure Data Factory CI/CD with Azure DevOps

This repository implements a **full CI/CD pipeline** for Azure Data Factory (ADF) using **Azure DevOps YAML pipelines**.  
It shows how to:

- Build and validate ADF resources  
- Generate ARM templates  
- Deploy to **DEV**, **QA**, and **PROD** environments  
- Use **approval gates** for Production  
- Customize each environment with **parameter files**
<img width="857" height="575" alt="d1" src="https://github.com/user-attachments/assets/5e06dc8f-c54f-4d3f-8332-b8b14d841502" />

---

## 1. High-Level Flow

### CI/CD Conceptual Flow

1. Developer commits ADF changes to the **main** branch.
2. **CI stage**:
   - Installs Node.js and ADF npm utilities.
   - Validates ADF JSON.
   - Exports **ARM templates** and publishes them as artifacts.
3. **CD stages**:
   - **Deploy to DEV** automatically.
   - **Deploy to QA** automatically (if DEV/build succeed).
   - **Deploy to PROD** only after **manual approval**.
4. For each deployment:
   - Stop ADF triggers (pre-deployment).
   - Deploy ARM template with environment-specific parameters.
   - Start ADF triggers again (post-deployment).

---

## 2. Repository Structure
<img width="357" height="720" alt="d3" src="https://github.com/user-attachments/assets/33e64b43-d395-4b60-810c-9619753b39e9" />


```text
.
├── cicd
│   ├── ARMParams
│   │   ├── dev.json          # DEV environment parameters
│   │   ├── qa.json           # QA environment parameters
│   │   └── pd.json           # PROD environment parameters
│   ├── cd_deploy.yml         # Reusable deployment template (DEV/QA/PROD)
│   ├── ci_build.yml          # Reusable CI build template
│   └── cicd_pipeline.yml     # Main pipeline orchestrating all stages
│
├── dataset/                  # ADF dataset definitions
├── factory/                  # ADF factory configuration
├── linkedService/            # ADF linked service definitions
├── pipeline/                 # ADF pipelines
├── trigger/                  # ADF triggers
├── package.json              # Node/npm configuration for ADF build utilities
└── publish_config.json       # ADF publish configuration (from ADF Git integration)
```

Each `dataset`, `factory`, `linkedService`, `pipeline`, and `trigger` folder contains ADF JSON definitions that are exported into ARM templates during the CI stage.

---

## 3. CI Build Template (`cicd/ci_build.yml`)

This YAML file defines the **Continuous Integration (CI)** logic. It is used as a **template** in the main pipeline.

### Parameters

```yaml
parameters:
- name: DataFactory
  type: string
- name: ResourceGroup
  type: string
- name: SubscriptionId
  type: string
```

These parameters are passed from the main pipeline and identify which ADF instance to validate and export from.

### Steps

#### 3.1 Install Node.js

```yaml
- task: UseNode@1
  inputs:
    version: '18.x'
  displayName: 'Install Node.js'
```

Ensures the pipeline uses Node.js 18.x for running npm-based ADF utilities.

#### 3.2 Install npm Packages

```yaml
- task: Npm@1
  inputs:
    command: 'install'
    workingDir: '$(Build.Repository.LocalPath)'
    verbose: true
  displayName: 'Install npm package'
```

Runs `npm install` in the repo root to install ADF utility tools defined in `package.json` (for example the `build` and `export` commands).

#### 3.3 Validate ADF

```yaml
- task: Npm@1
  inputs:
    command: 'custom'
    workingDir: '$(Build.Repository.LocalPath)'
    customCommand: 'run build validate $(Build.Repository.LocalPath)/ /subscriptions/${{ parameters.SubscriptionId }}/resourceGroups/${{ parameters.ResourceGroup }}/providers/Microsoft.DataFactory/factories/${{ parameters.DataFactory }}'
  displayName: 'Validate'
```

Runs a custom npm script `npm run build validate ...` to validate the ADF JSON files in the repo against the ADF instance (schema, references, etc.).

#### 3.4 Export ARM Template

```yaml
- task: Npm@1
  inputs:
    command: 'custom'
    workingDir: '$(Build.Repository.LocalPath)'
    customCommand: 'run build export $(Build.Repository.LocalPath)/ /subscriptions/${{ parameters.SubscriptionId }}/resourceGroups/${{ parameters.ResourceGroup }}/providers/Microsoft.DataFactory/factories/${{ parameters.DataFactory }} "ArmTemplate"'
  displayName: 'Validate and Generate ARM template'
```

Runs `npm run build export ...` to generate:

- `ARMTemplateForFactory.json`
- `ARMTemplateParametersForFactory.json`

The generated files are stored in an `ArmTemplate` folder.

#### 3.5 Publish ARM Template Artifact

```yaml
- task: PublishPipelineArtifact@1
  inputs:
    targetPath: '$(Build.Repository.LocalPath)/ArmTemplate'
    artifact: 'ArmTemplate'
    publishLocation: 'pipeline'
```

Publishes the generated ARM templates as a pipeline artifact named `ArmTemplate`, which will be used by the deployment stages.

---

## 4. CD Deploy Template (`cicd/cd_deploy.yml`)

This YAML file defines a **generic deployment process**. It is reused for DEV, QA, and PROD.

### Parameters

```yaml
parameters:
- name: AzureResourceManagerConnection
  type: string
- name: DataFactory
  type: string
- name: ResourceGroup
  type: string
- name: SubscriptionId
  type: string
- name: Environment
  type: string
```

`Environment` controls which parameter file is used: `dev`, `qa`, or `pd`.

### Steps

#### 4.1 Download ARM Template Artifact

```yaml
- task: DownloadPipelineArtifact@2
  displayName: "Download The ADF Artifact"
  inputs:
    buildType: "current"
    artifactName: "ARMTemplate"
    targetpath: "$(Pipeline.Workspace)/ARMTemplate"
```

Downloads the `ARMTemplateForFactory.json` (and related files) produced in the CI stage.

#### 4.2 Stop Current ADF Triggers (Pre-Deployment)

```yaml
- task: AzurePowerShell@5
  displayName: "Stop Current ADF Triggers"
  inputs: 
    azureSubscription: ${{ parameters.AzureResourceManagerConnection }}
    pwsh: true
    azurePowerShellVersion: "LatestVersion"
    ScriptType: "FilePath"
    ScriptPath: "$(Pipeline.Workspace)/ARMTemplate/PrePostDeploymentScript.ps1"
    scriptArguments:
      -ArmTemplate "$(Pipeline.Workspace)/ARMTemplate/ARMTemplateForFactory.json"
      -ArmTemplateParameters $(Build.Repository.LocalPath)/cicd/ARMParams/${{ parameters.Environment }}.json
      -ResourceGroupName ${{ parameters.ResourceGroup }}
      -DataFactoryName ${{ parameters.DataFactory }}
      -predeployment $true
      -deleteDeployment $false
```

Uses the pre/post deployment script to:

- Read the ARM template and parameter file.
- Identify ADF triggers.
- Stop them before deployment to avoid overlapping runs.

#### 4.3 Deploy ARM Template

```yaml
- task: AzureResourceManagerTemplateDeployment@3
  displayName: "Deploy ADF ARM Template"
  inputs:
    deploymentScope: 'Resource Group'
    azureResourceManagerConnection: ${{ parameters.AzureResourceManagerConnection }}
    subscriptionId: ${{ parameters.SubscriptionId }}
    action: 'Create Or Update Resource Group'
    resourceGroupName: ${{ parameters.ResourceGroup }}
    location: 'canadacentral'
    templateLocation: 'Linked artifact'
    csmFile: '$(Pipeline.Workspace)/ARMTemplate/ARMTemplateForFactory.json'
    csmParametersFile: '$(Build.Repository.LocalPath)/cicd/ARMParams/${{ parameters.Environment }}.json'
    deploymentMode: 'Incremental'
```

Performs an ARM template deployment:

- Uses `ARMTemplateForFactory.json` as the base template.
- Uses the environment-specific parameter file for values.
- Uses **Incremental** mode so that only changed resources are updated.

#### 4.4 Start ADF Triggers (Post-Deployment)

```yaml
- task: AzurePowerShell@5
  displayName: "Start ADF Triggers"
  inputs:
    azureSubscription: ${{ parameters.AzureResourceManagerConnection }}
    pwsh: true
    azurePowerShellVersion: "LatestVersion"
    ScriptType: "FilePath"
    ScriptPath: "$(Pipeline.Workspace)/ARMTemplate/PrePostDeploymentScript.ps1"
    scriptArguments:
      -ArmTemplate "$(Pipeline.Workspace)/ARMTemplate/ARMTemplateForFactory.json"
      -ArmTemplateParameters $(Build.Repository.LocalPath)/cicd/ARMParams/${{ parameters.Environment }}.json
      -ResourceGroupName ${{ parameters.ResourceGroup }}
      -DataFactoryName ${{ parameters.DataFactory }}
      -predeployment $false
      -deleteDeployment $true
```

Runs the same script in post-deployment mode to:

- Restart the ADF triggers that were stopped earlier.
- Optionally clean up previous deployment entries.

---

## 5. Main Pipeline (`cicd/cicd_pipeline.yml`)
<img width="1193" height="458" alt="d2" src="https://github.com/user-attachments/assets/d6dfb83e-5af1-4ab5-b749-327e0e6def7f" />


This file orchestrates the whole CI/CD process and defines the stages for the different environments.

### 5.1 Trigger

```yaml
trigger:
- main
```

The pipeline runs automatically when new commits are pushed to the `main` branch.

### 5.2 Build Stage – `BuildARMTemplate`

```yaml
- stage: BuildARMTemplate
  displayName: "Build ARM Template"

  variables:
  - group: devgroup

  jobs:
  - job: BuildARM
    displayName: "Build ARM"
    steps:
    - template: ci_build.yml
      parameters:
        DataFactory: $(DataFactory)
        ResourceGroup: $(ResourceGroup)
        SubscriptionId: $(SubscriptionId)
```

- Uses Azure DevOps variable group `devgroup` for `DataFactory`, `ResourceGroup`, and `SubscriptionId`.
- Calls `ci_build.yml` to validate and generate ARM templates from the DEV ADF instance.

### 5.3 Deploy to DEV – `DeployToDEV`

```yaml
- stage: DeployToDEV
  dependsOn: BuildARMTemplate
  condition: succeeded()
  displayName: "Deploy To DEV"

  variables:
  - group: devgroup

  jobs:
  - job: DeployToDEV
    displayName: "Deploy To DEV"
    steps:
    - template: cd_deploy.yml
      parameters:
        AzureResourceManagerConnection: adf-azure-service-connection
        DataFactory: $(DataFactory)
        SubscriptionId: $(SubscriptionId)
        ResourceGroup: $(ResourceGroup)
        Environment: dev
```

- Runs automatically after a successful build stage.
- Uses `devgroup` and `dev.json`.
- Deploys the ARM template to the DEV Data Factory.

### 5.4 Deploy to QA – `DeployToQA`

```yaml
- stage: DeployToQA
  dependsOn: BuildARMTemplate
  condition: succeeded()
  displayName: "Deploy To QA"

  variables:
  - group: qagroup

  jobs:
  - job: DeployToQA
    displayName: "Deploy To QA"
    steps:
    - template: cd_deploy.yml
      parameters:
        AzureResourceManagerConnection: adf-azure-service-connection
        DataFactory: $(DataFactory)
        SubscriptionId: $(SubscriptionId)
        ResourceGroup: $(ResourceGroup)
        Environment: qa
```

- Also runs after a successful build.
- Uses `qagroup` and `qa.json`.
- Deploys to the QA Data Factory.

> If you want strict sequencing (DEV → QA), you can change `dependsOn: BuildARMTemplate` to `dependsOn: DeployToDEV`.

### 5.5 Deploy to PROD with Approval – `DeployToPROD`

```yaml
- stage: DeployToPROD
  dependsOn: DeployToQA
  condition: succeeded()
  displayName: "Deploy To Production"

  variables:
  - group: prodgroup

  jobs:
  - deployment: DeployToPD
    displayName: "Deploy To Production"
    environment: 'pd' # This links to the Environment in Pipelines > Environments
    strategy:
      runOnce:
        deploy:
          steps:
          - checkout: self
          - template: cd_deploy.yml
            parameters:
              AzureResourceManagerConnection: adf-azure-service-connection
              DataFactory: $(DataFactory)
              SubscriptionId: $(SubscriptionId)
              ResourceGroup: $(ResourceGroup)
              Environment: pd
```

Key points:

- Depends on `DeployToQA`, so PROD runs only if QA succeeds.
- Uses variable group `prodgroup` and parameter file `pd.json`.
- Uses a **deployment job** with `environment: 'pd'`.

In Azure DevOps:

- The environment named `pd` can be configured with **Approvals and Checks**.
- When this stage is reached, the pipeline **pauses** and waits for a manual approval.
- After approval is granted, the deployment steps run and deploy to the PROD Data Factory.

---

## 6. Environment Parameter Files (`cicd/ARMParams/dev.json`, `qa.json`, `pd.json`)

Each environment has its own parameter file to override values in the ARM template.

### Example Structure

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "factoryName": {
      "value": "yourfactoryname"
    },
    "AzureKeyVault1_properties_typeProperties_baseUrl": {
      "value": "https://yourkeyvault.vault.azure.net/"
    },
    "ls_adls_properties_typeProperties_url": {
      "value": "https://yourstorageaccount.dfs.core.windows.net/"
    },
    "ls_http_properties_typeProperties_url": {
      "value": "https://raw.githubusercontent.com/"
    }
  }
}
```

You have three such files:

- `dev.json`
- `qa.json`
- `pd.json`

Each file should contain environment-specific values for:

- `factoryName`  
- Key Vault URL (`AzureKeyVault1_properties_typeProperties_baseUrl`)  
- Storage/Data Lake URL (`ls_adls_properties_typeProperties_url`)  
- HTTP base URL (`ls_http_properties_typeProperties_url`)  

### How the Correct File Is Selected

In `cd_deploy.yml`:

```yaml
csmParametersFile: '$(Build.Repository.LocalPath)/cicd/ARMParams/${{ parameters.Environment }}.json'
```

The `Environment` parameter is set in the main pipeline:

- DEV stage: `Environment: dev` → uses `dev.json`
- QA stage: `Environment: qa` → uses `qa.json`
- PROD stage: `Environment: pd` → uses `pd.json`

This allows you to deploy the same ADF template to different environments but point to different resources (storage accounts, key vaults, etc.).

---

## 7. Multi-Environment + Approval Workflow Summary

End-to-end behavior:

1. **Developer pushes to `main`.**
2. `BuildARMTemplate` stage runs:
   - Validates ADF JSON.
   - Generates ARM templates.
   - Publishes `ArmTemplate` artifact.
3. `DeployToDEV`:
   - Uses `devgroup` variables and `dev.json`.
   - Stops DEV triggers → deploys ARM → restarts triggers.
4. `DeployToQA`:
   - Uses `qagroup` and `qa.json`.
   - Stops QA triggers → deploys ARM → restarts triggers.
5. `DeployToPROD`:
   - Depends on successful QA deployment.
   - Uses `prodgroup` and `pd.json`.
   - Uses Azure DevOps environment `pd` with an approval gate.
   - Pipeline waits for an approver to approve/reject.
   - If approved, stops PROD triggers → deploys ARM → restarts triggers.

This setup gives you:

- Automated deployments to DEV and QA.  
- A controlled, auditable deployment to PROD with a **manual approval gate**.  

---
**Working Video**


https://github.com/user-attachments/assets/9d978236-6b32-4aca-a38d-05f2b6aad61f





