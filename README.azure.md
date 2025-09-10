# Deploying Dockerized Node.js App to Azure with GitHub Actions

## Provision Resources and Service Principal via Azure Portal (Control Panel Path)

### 1. Provision Azure Resources via Azure Portal

**a. Create Azure Container Registry (ACR):**
1. Go to the Azure Portal: https://portal.azure.com
2. Search for “Container Registries” and click “Create”
3. Fill in the Resource Group, Registry Name, and select SKU (e.g., Basic).
4. Click “Review + create” and then “Create.”

**b. Create Azure Web App for Containers:**
1. In the Portal, search for “App Services” and click “Create - Web App”
2. Fill in the Resource Group, App Name, and select “Docker Container” for Publish.
3. Under “Docker,” set the image source to “Azure Container Registry.”
4. Select your registry and specify the image and tag.
5. Click “Review + create” and then “Create.”

---

### 2. Create Service Principal & Set GitHub Secrets via Azure Portal

**a. Create Service Principal:**
1. In the Portal, search for “Microsoft Entra ID” (formerly Azure Active Directory).
1. Go to “App registrations” > “New registration.”
1. Name your app (e.g., github-actions-deploy) and register.
1. After registration, go to “Certificates & secrets” > “New client secret.” Copy the value.
1. Go to “Overview” to get the Application (client) ID and Directory (tenant) ID.

**b. Assign Role to Resource Group:**
1. Go to your Resource Group.
1. Click “Access control (IAM)” > “Add” > “Add role assignment.”
1. Select “Website Contributor” and assign it to your registered app.

**c. Assign Role to Azure Container Registry:**
1. Go to your Azure Container Registry
1. Click “Access control (IAM)” > “Add” > “Add role assignment”
1. Select "Acr Pull" and assign it to your registered app.
1. Click “Access control (IAM)” > “Add” > “Add role assignment”
1. Select "Acr Push" and assign it to your registered app.

**c. Add GitHub Environment:**
- In your GitHub repo, go to “Settings” > “Environments” > “Actions”
- Add the environment you are deploying to and 
- Add the following secrets:
	- `AZURE_CREDENTIALS` (JSON with clientId, clientSecret, tenantId, subscriptionId)
    - `AZURE_ACR_PASSWORD` (Same as clientSecret - can use admin login)
- Add the following environment variables:
	- `AZURE_ACR_LOGIN_SERVER` (Azure Container Repository Login Server URL)
    - `AZURE_ACR_USERNAME` (Same as clientId - can use admin login)
	- `AZURE_WEBAPP_NAME`

---

## Provision Resources and Service Principal via Azure CLI (Command Line Path) 

### 1. Provision Azure Resources

- **Create Azure Container Registry (ACR):**
	```sh
	az acr create --resource-group <ResourceGroup> --name <ACRName> --sku Basic
	```

- **Create Azure Web App for Containers:**
	```sh
	az webapp create --resource-group <ResourceGroup> --plan <AppServicePlan> --name <WebAppName> --deployment-container-image-name <ACRName>.azurecr.io/<ImageName>:latest
	```

---

### 2. Create Azure Service Principal & Set GitHub Secrets

**Create Service Principal and Assign Contributor Role:**
```sh
az ad sp create-for-rbac --name "github-actions-deploy" --role contributor --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<ResourceGroup>
```

**Assign AcrPull and AcrPush Roles to Azure Container Registry:**
```sh
az role assignment create --assignee <APP_ID> --role "AcrPull" --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<ResourceGroup>/providers/Microsoft.ContainerRegistry/registries/<ACR_NAME>
az role assignment create --assignee <APP_ID> --role "AcrPush" --scope /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<ResourceGroup>/providers/Microsoft.ContainerRegistry/registries/<ACR_NAME>
```
Replace `<APP_ID>` with your service principal's Application (client) ID, `<SUBSCRIPTION_ID>` with your subscription ID, `<ResourceGroup>` with your resource group name, and `<ACR_NAME>` with your Azure Container Registry name.

**Add GitHub Environemnt:** 
- In your GitHub repo, go to “Settings” > “Environments” > “Actions”
- Add the environment you are deploying to and 
- Add the following secrets:
	- `AZURE_CREDENTIALS` (JSON with clientId, clientSecret, tenantId, subscriptionId)
    - `AZURE_ACR_PASSWORD` (Same as clientServer - can use admin login)
- Add the following environment variables:
	- `AZURE_ACR_LOGIN_SERVER` (Azure Container Repository Login Server URL)
    - `AZURE_ACR_USERNAME` (Same as clientId - can use admin login)
	- `AZURE_WEBAPP_NAME`

---

### 3. Add GitHub Actions Workflow

Create `.github/workflows/azure-deploy.yml`, and make sure the environment is the same as the name you use in settings:

```yaml
name: Build and Deploy to Azure Web App for Containers

on:
	push:
		branches:
			- main

jobs:
	build-and-deploy:
		runs-on: ubuntu-latest
        environment: production

		steps:
		- name: Checkout code
			uses: actions/checkout@v4

		- name: Log in to Azure
			uses: azure/login@v2
			with:
				creds: ${{ secrets.AZURE_CREDENTIALS }}

		- name: Log in to Azure Container Registry
			uses: azure/docker-login@v2
			with:
				login-server: ${{ vars.AZURE_ACR_LOGIN_SERVER }}
				username: ${{ vars.AZURE_ACR_USERNAME }}
				password: ${{ secrets.AZURE_ACR_PASSWORD }}

		- name: Build and push Docker image
			run: |
				docker build -t ${{ vars.AZURE_ACR_LOGIN_SERVER }}/app:${{ github.sha }} .
				docker push ${{ vars.AZURE_ACR_LOGIN_SERVER }}/app:${{ github.sha }}

		- name: Deploy to Azure Web App
			uses: azure/webapps-deploy@v3
			with:
				app-name: ${{ vars.AZURE_WEBAPP_NAME }}
				images: ${{ vars.AZURE_ACR_LOGIN_SERVER }}/app:${{ github.sha }}
```

---

## 4. Update Azure Web App to Use Latest Image

The workflow will update the Web App to use the new image after each push to `main`.

---
