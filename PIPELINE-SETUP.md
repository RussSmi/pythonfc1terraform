# Azure DevOps Pipeline Setup Instructions

This guide explains how to set up the Azure DevOps pipeline for deploying your Python Function App with Terraform infrastructure.

## Prerequisites

### 1. Azure DevOps Project
- Create or have access to an Azure DevOps project
- Ensure you have permissions to create pipelines and service connections

### 2. Azure Service Connection
Create a service connection in Azure DevOps:

1. Go to **Project Settings** → **Service connections**
2. Click **New service connection**
3. Select **Azure Resource Manager**
4. Choose **Service principal (automatic)**
5. Configure:
   - **Scope level**: Subscription
   - **Subscription**: Your Azure subscription
   - **Resource group**: Leave empty (or specify if you want to restrict)
   - **Service connection name**: `azure-terraform-connection` (or update in pipeline)
6. Save the service connection

### 3. Pipeline Variables to Update

Before running the pipeline, update these variables in `azure-pipelines.yml`:

```yaml
# Replace these values:
azureServiceConnection: 'your-service-connection-name'  # Name from step 2
subscriptionId: 'your-subscription-id'                 # Your Azure subscription ID
```

## Pipeline Stages

### Stage 1: ValidateAndPlan
- **Purpose**: Validates Terraform configuration and creates execution plan
- **Actions**:
  - Installs Terraform
  - Runs `terraform init`, `validate`, and `fmt`
  - Creates and publishes Terraform plan

### Stage 2: DeployInfrastructure
- **Purpose**: Deploys Azure infrastructure using Terraform
- **Actions**:
  - Downloads Terraform plan
  - Runs `terraform apply`
  - Outputs function app name and resource group for next stage

### Stage 3: BuildAndDeployFunction
- **Purpose**: Builds and deploys Python function code
- **Jobs**:
  - **BuildFunction**: Creates deployment package with Python 3.12
  - **DeployFunction**: Deploys to Azure Function App using ZIP deployment

### Stage 4: PostDeploymentValidation
- **Purpose**: Validates the deployment was successful
- **Actions**:
  - Tests function endpoint
  - Verifies function is responding

## Terraform Backend Configuration (Recommended)

For production use, configure a Terraform backend to store state remotely:

1. Create a storage account for Terraform state
2. Add backend configuration to your Terraform:

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "terraformstatestorage"
    container_name       = "tfstate"
    key                  = "python-function.terraform.tfstate"
  }
}
```

## Environment Configuration

The pipeline supports multiple environments:
- **main branch** → production environment
- **other branches** → development environment

You can customize this in the variables section of the pipeline.

## Pipeline Triggers

Current triggers:
- **Branches**: main, develop
- **Paths**: All files (`**`)

## Security Considerations

1. **Service Principal**: The pipeline uses a service principal with least-privilege access
2. **Secrets**: No secrets are hardcoded in the pipeline
3. **Environment Gates**: Consider adding approval gates for production deployments

## Setup Steps

1. **Create Service Connection** (see prerequisites)
2. **Update Pipeline Variables** (azureServiceConnection, subscriptionId)
3. **Create Pipeline** in Azure DevOps pointing to `azure-pipelines.yml`
4. **Configure Environment** (optional: add approval gates)
5. **Run Pipeline**

## Troubleshooting

### Common Issues:

1. **Terraform Init Fails**
   - Check service connection permissions
   - Verify subscription ID is correct

2. **Function Deployment Fails**
   - Ensure function app was created successfully in previous stage
   - Check Python version compatibility

3. **Validation Stage Fails**
   - Function might need time to warm up
   - Check function logs in Azure portal

## Monitoring

After deployment, monitor your function:
- **Azure Portal**: Function App → Functions → Monitor
- **Application Insights**: Linked automatically via Terraform
- **Pipeline Logs**: Azure DevOps pipeline execution logs

## Next Steps

1. **Add Tests**: Include unit tests in the BuildFunction job
2. **Environment Gates**: Add manual approval for production
3. **Notifications**: Configure build notifications
4. **Security Scanning**: Add security scanning stages
