# Deploying Dockerized Node.js App to Azure with GitHub Actions

## Provision Resources and Service Principal via Azure Portal (Control Panel Path)

### 1. Provision Azure Resources via Azure Portal

**a. Create Azure Container Registry (ACR):**
1. Go to the Azure Portal: https://portal.azure.com
2. Search for “Container Registries” and click “Create.”
3. Fill in the Resource Group, Registry Name, and select SKU (e.g., Basic).
4. Click “Review + create” and then “Create.”

**b. Create Azure Web App for Containers:**
1. In the Portal, search for “Web App” and click “Create.”
2. Fill in the Resource Group, App Name, and select “Docker Container” for Publish.
3. Under “Docker,” set the image source to “Azure Container Registry.”
4. Select your registry and specify the image and tag.
5. Click “Review + create” and then “Create.”

---

### 2. Create Service Principal & Set GitHub Secrets via Azure Portal

**a. Create Service Principal:**
1. In the Portal, search for “Azure Active Directory.”
2. Go to “App registrations” > “New registration.”
3. Name your app (e.g., github-actions-deploy) and register.
4. After registration, go to “Certificates & secrets” > “New client secret.” Copy the value.
5. Go to “Overview” to get the Application (client) ID and Directory (tenant) ID.

**b. Assign Role:**
1. Go to your Resource Group.
2. Click “Access control (IAM)” > “Add” > “Add role assignment.”
3. Select “Contributor” and assign it to your registered app.

**c. Add GitHub Secrets:**
- In your GitHub repo, go to “Settings” > “Secrets and variables” > “Actions.”
- Add the following secrets:
	- `AZURE_CREDENTIALS` (JSON with clientId, clientSecret, tenantId, subscriptionId)
	- `AZURE_WEBAPP_NAME`
	- `AZURE_ACR_NAME`
	- `AZURE_RESOURCE_GROUP`
	- `AZURE_SUBSCRIPTION_ID`

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

- **Create Service Principal:**
	```sh
	az ad sp create-for-rbac --name "github-actions-deploy" --role contributor --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<ResourceGroup>
	```

- **Add these as GitHub secrets:**
	- `AZURE_CREDENTIALS` (JSON output from above)
	- `AZURE_WEBAPP_NAME`
	- `AZURE_ACR_NAME`
	- `AZURE_RESOURCE_GROUP`
	- `AZURE_SUBSCRIPTION_ID`

---

### 3. Add GitHub Actions Workflow

Create `.github/workflows/azure-deploy.yml`:

```yaml
name: Build and Deploy to Azure Web App for Containers

on:
	push:
		branches:
			- main

jobs:
	build-and-deploy:
		runs-on: ubuntu-latest

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
				login-server: ${{ secrets.AZURE_ACR_NAME }}.azurecr.io
				username: ${{ secrets.AZURE_ACR_NAME }}
				password: ${{ secrets.AZURE_CREDENTIALS }}

		- name: Build and push Docker image
			run: |
				docker build -t ${{ secrets.AZURE_ACR_NAME }}.azurecr.io/app:${{ github.sha }} .
				docker push ${{ secrets.AZURE_ACR_NAME }}.azurecr.io/app:${{ github.sha }}

		- name: Deploy to Azure Web App
			uses: azure/webapps-deploy@v3
			with:
				app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
				images: ${{ secrets.AZURE_ACR_NAME }}.azurecr.io/app:${{ github.sha }}
```

---

## 4. Update Azure Web App to Use Latest Image

The workflow will update the Web App to use the new image after each push to `main`.

---
