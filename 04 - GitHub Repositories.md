## **4\. Migrating GitHub Repositories and CI/CD from AWS to Azure**

Migrating scientific projects from an AWS-centric environment to Azure involves more than just moving code. It requires adapting build and deployment pipelines, updating SDKs, and reconfiguring authentication mechanisms. This section focuses on these aspects, particularly for projects using GitHub Actions for Continuous Integration and Continuous Delivery (CI/CD).

### **4.1. Key Considerations for Migrating Repositories**

When moving GitHub repositories that interact with cloud services from AWS to Azure, several key areas need attention:

* **SDK Changes:**  
  * Code interacting with AWS services (e.g., S3, EC2, Lambda) typically uses AWS SDKs, such as Boto3 for Python. 45  
  * When targeting Azure, this code must be refactored to use the corresponding Azure SDKs. For Python, this includes libraries like azure-storage-blob for Blob Storage, azure-mgmt-compute for managing VMs, azure-functions for Azure Functions, and azure-ai-ml for Azure Machine Learning. 47  
  * This involves changes in client initialization, authentication methods, and the specific API calls for service operations (e.g., uploading files, launching compute jobs).  
* **Credential Management:**  
  * In AWS, GitHub Actions often authenticate using IAM roles (via OIDC) or IAM user access keys stored as GitHub secrets.  
  * In Azure, the preferred methods are **OpenID Connect (OIDC) with Azure Service Principals** or **Managed Identities** (for self-hosted runners on Azure VMs). 45 Storing long-lived secrets like service principal passwords in GitHub secrets is generally discouraged in favor of OIDC. 51  
* **CI/CD Pipeline Adjustments (GitHub Actions):**  
  * **Authentication Actions:** AWS workflows use actions like aws-actions/configure-aws-credentials. Azure workflows use azure/login to authenticate. 45  
  * **CLI Commands:** Scripts using AWS CLI commands (e.g., aws s3 sync, aws lambda update-function-code) need to be replaced with Azure CLI commands (e.g., az storage blob sync, az functionapp deployment source config-zip). 45  
  * **Service-Specific Actions:** While some generic CLI steps can be adapted, you might also leverage Azure-specific GitHub Actions for deploying to services like Azure App Service, Azure Functions, or Azure Container Apps. 54  
  * **Environment Variables:** Names and sources of environment variables for configuring cloud access (e.g., subscription IDs, tenant IDs, client IDs) will change. 51  
  * **Resource Identifiers:** References to AWS ARNs will need to be updated to Azure Resource IDs.  
  * **Triggers, Dependencies, and Secrets:** These need to be mapped to Azure DevOps Pipelines if migrating fully to Azure DevOps, or reconfigured within GitHub Actions for Azure targets. 45  
* **Infrastructure as Code (IaC):**  
  * If using CloudFormation, these templates will need to be translated to Azure Resource Manager (ARM) templates, Bicep files, or Terraform configurations compatible with Azure.  
* **Repository Size and LFS:**  
  * Be aware of any repository size limits (Azure DevOps has a 250 GB limit). 45  
  * Git Large File Storage (LFS) objects need separate migration and reconfiguration. 45

### **4.2. Secure Authentication for GitHub Actions to Azure**

Storing long-lived secrets (like passwords or service principal keys) in GitHub secrets is a security risk. Azure and GitHub offer more secure alternatives:

* **OpenID Connect (OIDC) with Azure Service Principals:**  
  * This is the **recommended method** for GitHub Actions to access Azure resources without needing to store Azure credentials as long-lived GitHub secrets. 51  
  * **How it works:** GitHub Actions can request an OIDC JSON Web Token (JWT) from GitHub's OIDC provider. This token can then be presented to Microsoft Entra ID to obtain a short-lived access token for an Azure Service Principal. The service principal is granted permissions to Azure resources via RBAC.  
  * **Setup Steps:** 51  
    1. **Create a Microsoft Entra application registration and an associated service principal** in your Azure tenant (e.g., using az ad app create and az ad sp create).  
    2. **Configure a federated identity credential** on the Entra ID application. This establishes a trust relationship between your Entra ID application and your GitHub repository/branch/environment. You'll specify the GitHub repository, and a "subject identifier" (e.g., repo:myorganization/myrepository:ref:refs/heads/main for the main branch, or repo:myorganization/myrepository:pull\_request for pull requests). 51 The audience setting is typically api://AzureADTokenExchange. 57  
    3. **Grant the service principal appropriate RBAC roles** (e.g., Contributor, Reader, Storage Blob Data Contributor) on the Azure resources it needs to access. Use the principle of least privilege. 51  
    4. **Store Azure configuration details as GitHub secrets:** AZURE\_CLIENT\_ID (the Application (client) ID of the Entra ID app), AZURE\_TENANT\_ID, and AZURE\_SUBSCRIPTION\_ID. 51  
    5. **Update the GitHub Actions workflow:**  
       * Add permissions: id-token: write to the workflow or job to allow it to request an OIDC token from GitHub. 51  
       * Use the azure/login@v1 (or newer, e.g., azure/login@v2) action with the client-id, tenant-id, and subscription-id parameters populated from the GitHub secrets. This action handles the OIDC token exchange. 51  
* **Managed Identities with Self-Hosted Runners on Azure VMs:**  
  * If you are using self-hosted GitHub runners that are Azure Virtual Machines, you can configure a **managed identity** (either system-assigned or user-assigned) for the VM. 52  
  * The Azure VM's managed identity can be granted RBAC permissions to other Azure resources.  
  * The azure/login action can then use this managed identity to authenticate to Azure without needing any explicit credentials stored in GitHub. 52  
  * **Setup for User-Assigned Managed Identity:**  
    1. Create a user-assigned managed identity in Azure. 59  
    2. Assign it to the Azure VM running the self-hosted runner.  
    3. Grant the managed identity RBAC roles to the target Azure resources.  
    4. In GitHub secrets, store AZURE\_CLIENT\_ID (Client ID of the user-assigned managed identity), AZURE\_TENANT\_ID, and AZURE\_SUBSCRIPTION\_ID. 52  
    5. Use azure/login@v1 with these secrets. The action detects it's running on an Azure VM with a matching managed identity.  
  * **Setup for System-Assigned Managed Identity:**  
    1. Enable system-assigned managed identity on the Azure VM.  
    2. Grant this identity RBAC roles.  
    3. In GitHub secrets, store AZURE\_TENANT\_ID and AZURE\_SUBSCRIPTION\_ID. The client ID is not needed as it's implicit to the VM. 52  
    4. Use azure/login@v1.

OIDC is generally preferred for its flexibility as it doesn't tie you to self-hosted runners on Azure VMs.

### **4.3. Example: GitHub Actions Workflow Changes (AWS CLI to Azure CLI)**

Migrating workflow steps from AWS CLI to Azure CLI involves changing both the authentication mechanism and the command syntax.

**Conceptual Workflow Migration:**

**1\. Authentication Step:**

* **AWS:**  
  YAML  
  \- name: Configure AWS credentials  
    uses: aws-actions/configure-aws-credentials@v4 \# or earlier versions  
    with:  
      aws-access-key-id: ${{ secrets.AWS\_ACCESS\_KEY\_ID }}  
      aws-secret-access-key: ${{ secrets.AWS\_SECRET\_ACCESS\_KEY }}  
      aws-region: us-east-1  
  \# Or using OIDC with AWS  
  \# \- name: Configure AWS credentials  
  \#   uses: aws-actions/configure-aws-credentials@v4  
  \#   with:  
  \#     role-to-assume: arn:aws:iam::ACCOUNT\_ID:role/GITHUB\_OIDC\_ROLE  
  \#     aws-region: us-east-1

* **Azure (using OIDC):**  
  YAML  
  permissions:  
    id-token: write \# Required for OIDC  
    contents: read  \# Usually needed to checkout code

  jobs:  
    deploy:  
      runs-on: ubuntu-latest  
      steps:  
      \- name: Checkout code  
        uses: actions/checkout@v3

      \- name: Azure Login  
        uses: azure/login@v2 \# Or newer  
        with:  
          client-id: ${{ secrets.AZURE\_CLIENT\_ID }}  
          tenant-id: ${{ secrets.AZURE\_TENANT\_ID }}  
          subscription-id: ${{ secrets.AZURE\_SUBSCRIPTION\_ID }}

  The azure/login action, when configured with OIDC parameters, handles the token exchange and makes the Azure CLI (and other Azure tools) available and authenticated for subsequent steps. 51

**2\. Storage Interaction (Syncing Files):**

* **AWS S3 Sync:**  
  YAML  
  \- name: Sync files to S3  
    run: aws s3 sync./my-local-data/ s3://my-aws-bucket/data/ \--delete

* Azure Blob Storage Sync/Upload:  
  Azure CLI offers az storage blob sync (though its availability and features have evolved; direct use of az storage blob upload-batch is often more robust for full uploads).  
  YAML  
  \- name: Upload files to Azure Blob Storage  
    uses: azure/CLI@v1  
    with:  
      inlineScript: |  
        az storage blob upload-batch \\  
          \--account-name mystorageaccountname \\  
          \--destination '$web' \\ \# Example container, often used for static sites  
          \--source./my-local-data/ \\  
          \--overwrite true \# or use \--auth-mode login if service principal has RBAC  
          \# For general data, use a specific container: \--destination mydatacontainer  
          \# The \--auth-mode key is shown in \[53\], but login with RBAC is preferred with OIDC

  The az storage blob upload-batch command is a common way to upload multiple files. 53 The \--auth-mode login parameter can be used if the logged-in service principal (via azure/login) has the necessary RBAC permissions (e.g., Storage Blob Data Contributor) on the storage account. Alternatively, an account key can be used (less secure). azcopy login \--identity can be used with managed identities for AzCopy operations, and AZCOPY\_AUTO\_LOGIN\_TYPE=AZCLI can allow AzCopy to use the Azure CLI's token. 60

**3\. Deploying to Serverless (Lambda vs. Functions):**

* **AWS Lambda Deployment (example using AWS CLI):**  
  YAML  
  \- name: Update Lambda function code  
    run: |  
      zip \-r function.zip.  
      aws lambda update-function-code \\  
        \--function-name my-lambda-function \\  
        \--zip-file fileb://function.zip

* **Azure Functions Deployment (example using Azure Functions Action):**  
  YAML  
  \- name: Deploy to Azure Functions  
    uses: Azure/functions-action@v1  
    with:  
      app-name: 'my-azure-function-app'  
      package: ${{ env.AZURE\_FUNCTIONAPP\_PACKAGE\_PATH }} \# Path to the zipped package  
      \# publish-profile: ${{ secrets.AZURE\_FUNCTIONAPP\_PUBLISH\_PROFILE }} \# Alternative auth

  The Azure/functions-action simplifies deployment. 61 The package input usually points to a directory or a zip file containing the function app code. Authentication is handled by the preceding azure/login step if not using a publish profile. For Python apps, ensure dependencies are handled either by pre-building them in the workflow or enabling remote build on Azure. 62

**4\. Deploying to Container Services (ECS/EKS vs. Container Apps/AKS):**

* AWS ECS/EKS Deployment (conceptual):  
  Typically involves building a Docker image, pushing it to Amazon ECR, and then updating an ECS service definition or EKS deployment manifest.  
  YAML  
  \# Example steps (simplified)  
  \# \- name: Build and push Docker image to ECR  
  \#   uses: aws-actions/amazon-ecr-login@v1  
  \#  ... then docker build and push...  
  \# \- name: Deploy to ECS  
  \#   uses: aws-actions/amazon-ecs-deploy-task-definition@v1  
  \#  ...

* **Azure Container Apps Deployment (example using Azure Container Apps Deploy Action):**  
  YAML  
  \- name: Build and deploy to Azure Container Apps  
    uses: Azure/container-apps-deploy-action@v1  
    with:  
      appSourcePath: ${{ github.workspace }}/app\_source\_directory \# Path to app source or Dockerfile  
      acrName: myazurecontainerregistry  
      containerAppName: my-container-app  
      resourceGroup: my-resource-group  
      \# Other parameters like imageName, dockerfilePath, etc.

  The Azure/container-apps-deploy-action can build an image from source (using Oryx++ if no Dockerfile is found) or use an existing Dockerfile, push it to Azure Container Registry (ACR), and deploy/update the Container App. 54 It authenticates using the azure/login step.  
  Alternatively, one can use Azure CLI commands like az acr build and az containerapp update. 55

**Key Changes & Considerations:**

* **Authentication Actions:** Shift from aws-actions/configure-aws-credentials to azure/login.  
* **CLI Tool:** aws CLI calls are replaced by az CLI calls. Syntax and command structures differ significantly.  
* **Service Endpoints & Resource Naming:** AWS ARNs vs. Azure Resource IDs/names.  
* **Azure-Specific Actions:** Leverage actions like Azure/functions-action or Azure/container-apps-deploy-action for streamlined deployments to specific Azure services.  
* **Permissions:** Ensure the Azure service principal used by azure/login has the correct RBAC roles (e.g., Website Contributor for App Service, Contributor on the resource group for creating resources, Storage Blob Data Contributor for blob uploads).  
* **Environment Variables:** Update GitHub secrets to store Azure-specific credentials (AZURE\_CLIENT\_ID, AZURE\_TENANT\_ID, AZURE\_SUBSCRIPTION\_ID) instead of AWS credentials.

Migrating CI/CD pipelines requires careful mapping of each step, understanding the authentication flow for Azure, and translating service interaction commands. Using OIDC and Azure-specific GitHub Actions can simplify this process and enhance security.
