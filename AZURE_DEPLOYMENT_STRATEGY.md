# Azure Deployment Strategy for WhatsX

## Overview
The **WhatsX** project is a full‑stack WhatsApp messaging automation platform consisting of:
- **Backend** (Node.js/Express) – Dockerized, located in `backend/`
- **Frontend** (Vite React) – Dockerized, located in `frontend/`
- **Email API** – Serverless function used for SMTP email delivery
- **Compose** files and Dockerfiles for local development
- CI workflow (`.github/workflows/testci.yml`) for testing

We will map each component to Azure services that provide a production‑grade, scalable, and cost‑effective environment.

---

## 1. Services Mapping
| Component | Azure Service | Reason |
|---|---|---|
| **Backend API** | **Azure Container Apps** (or **Azure App Service** for simple deployments) | Fully managed, supports HTTP autoscaling, easy Docker image integration. |
| **Frontend** | **Azure Static Web Apps** (or **Azure App Service** for node‑based builds) | Serves static assets, integrates with GitHub Actions for CI/CD, custom domain & SSL. |
| **MongoDB** | **Azure Cosmos DB (MongoDB API)** | Fully managed, globally distributed, compatible with existing connection strings. |
| **Email Sending** | **Azure Functions (HTTP trigger)** + **SendGrid** (or Azure Communication Services) | Serverless, no long‑running containers, secret‑managed via Key Vault. |
| **CI/CD** | **GitHub Actions** → Azure **Web Apps Deploy** action | Re‑uses existing workflow, adds Azure login step. |
| **Secrets / Config** | **Azure Key Vault** & **App Service Settings** | Centralised secret management, versioned, integrates with App Service & Functions. |
| **Logging & Monitoring** | **Azure Monitor** + **Log Analytics** | Centralised metrics, alerts, and tracing across all services. |
| **Container Registry** | **Azure Container Registry (ACR)** | Stores built Docker images for backend & frontend. |

---

## 2. Prerequisites
1. **Azure Subscription** – Owner or Contributor rights.
2. **Azure CLI** installed locally (`az`).
3. **Docker** installed (for building images).
4. **GitHub repository** linked to Azure (service principal). 
5. **SendGrid account** (or other SMTP) – will be used by the email function.

---

## 3. Infrastructure Setup (One‑time)
```bash
# Log in to Azure
az login

# Set variables
RESOURCE_GROUP=whatsx-rg
LOCATION=eastus
ACR_NAME=whatsxacr$(date +%s)   # unique name

# Create resource group
az group create -n $RESOURCE_GROUP -l $LOCATION

# Create Azure Container Registry
az acr create -n $ACR_NAME -g $RESOURCE_GROUP --sku Basic --admin-enabled true

# Create Azure Cosmos DB (MongoDB API)
az cosmosdb create -n whatsx-cosmos -g $RESOURCE_GROUP --kind MongoDB --capabilities EnableMongo

# Create Azure Key Vault
az keyvault create -n whatsx-kv -g $RESOURCE_GROUP -l $LOCATION

# Enable secret permissions for the current user
az keyvault set-policy -n whatsx-kv --secret-permissions get list set delete
```
The above commands provision the core resources. Store the output values (ACR login server, Cosmos DB connection string, etc.) for later steps.

---

## 4. Build & Push Docker Images
```bash
# Backend image
cd d:/new/WhatsX-Advanced-WhatsApp-Messaging-Automation/backend
az acr login -n $ACR_NAME
docker build -t $ACR_NAME.azurecr.io/whatsx-backend:latest .
docker push $ACR_NAME.azurecr.io/whatsx-backend:latest

# Frontend image (if using Container Apps, otherwise Static Web Apps will build from source)
cd d:/new/WhatsX-Advanced-WhatsApp-Messaging-Automation/frontend
docker build -t $ACR_NAME.azurecr.io/whatsx-frontend:latest .
docker push $ACR_NAME.azurecr.io/whatsx-frontend:latest
```
If you prefer **Static Web Apps**, you can skip the frontend Docker build and let Azure build from the repo.

---

## 5. Deploy Backend to Azure Container Apps
```bash
# Create Container Apps environment
az containerapp env create -n whatsx-env -g $RESOURCE_GROUP --location $LOCATION

# Deploy backend
az containerapp create \
  -n whatsx-backend \
  -g $RESOURCE_GROUP \
  --environment whatsx-env \
  --image $ACR_NAME.azurecr.io/whatsx-backend:latest \
  --cpu 0.5 --memory 1.0Gi \
  --target-port 10000 \
  --ingress 'external' \
  --env-vars NODE_ENV=production \
             PORT=10000 \
             MONGO_URI=$(az cosmosdb keys list -n whatsx-cosmos -g $RESOURCE_GROUP --type connection-strings --query "connectionStrings[?description=='Primary MongoDB Connection String'].connectionString" -o tsv) \
             JWT_SECRET=$(az keyvault secret show -n jwt-secret --vault-name whatsx-kv --query value -o tsv) \
             SENDGRID_API_KEY=$(az keyvault secret show -n sendgrid-key --vault-name whatsx-kv --query value -o tsv) \
             VERCEL_EMAIL_ENDPOINT=$(az keyvault secret show -n vercel-email-endpoint --vault-name whatsx-kv --query value -o tsv)
```
> **Note**: Store secret values (JWT_SECRET, SENDGRID_API_KEY, etc.) in **Azure Key Vault** first:
```bash
az keyvault secret set --vault-name whatsx-kv -n jwt-secret -v <your-secret>
az keyvault secret set --vault-name whatsx-kv -n sendgrid-key -v <your-key>
# optionally set VERCEL_EMAIL_ENDPOINT if you still use Vercel for email
```

---

## 6. Deploy Frontend
### Option A – Azure Static Web Apps (recommended)
1. In Azure Portal, create a **Static Web App**.
2. Connect to the GitHub repo, select the **frontend** folder as the *app location*.
3. Build preset: **Custom** → Build command `npm run build`, Output location `dist`.
4. Add **Application settings** (environment variables) for the API URL:
   - `VITE_API_URL=https://<backend‑fqdn>/api`
   - SMTP variables if the frontend still calls the serverless email function.

### Option B – Azure Container Apps (if you need a custom container)
```bash
az containerapp create \
  -n whatsx-frontend \
  -g $RESOURCE_GROUP \
  --environment whatsx-env \
  --image $ACR_NAME.azurecr.io/whatsx-frontend:latest \
  --cpu 0.5 --memory 1.0Gi \
  --target-port 80 \
  --ingress external \
  --env-vars VITE_API_URL=https://<backend‑fqdn>/api
```
---

## 7. Serverless Email Function (Azure Functions)
Create an HTTP‑triggered Function App (Node.js runtime).
```bash
az functionapp create \
  -g $RESOURCE_GROUP \
  -n whatsx-email-func \
  --storage-account <storage_name> \
  --runtime node \
  --functions-version 4 \
  --consumption-plan-location $LOCATION
```
Deploy the `frontend/api/send-email.js` (or refactor to a dedicated folder). Add the following **Application settings**:
- `SMTP_HOST`
- `SMTP_PORT`
- `SMTP_SECURE`
- `SMTP_USER`
- `SMTP_PASS`
- `SMTP_FROM`
These can be pulled from Key Vault via **Azure Managed Identity**.
Update the backend environment variable:
```bash
az containerapp env set -n whatsx-backend -g $RESOURCE_GROUP \
  --env-vars VERCEL_EMAIL_ENDPOINT=https://<function-app-name>.azurewebsites.net/api/send-email
```
---

## 8. CI/CD Integration (GitHub Actions)
Add a new job to `.github/workflows/testci.yml` (or create `azure-deploy.yml`):
```yaml
name: Azure Deploy
on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Azure CLI
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}   # service principal JSON
      - name: Build & Push Backend Image
        run: |
          az acr login -n ${{ env.ACR_NAME }}
          docker build -t ${{ env.ACR_NAME }}.azurecr.io/whatsx-backend:latest ./backend
          docker push ${{ env.ACR_NAME }}.azurecr.io/whatsx-backend:latest
      - name: Deploy Backend to Container Apps
        run: |
          az containerapp update -n whatsx-backend -g ${{ env.RESOURCE_GROUP }} \
            --image ${{ env.ACR_NAME }}.azurecr.io/whatsx-backend:latest
      # Add similar steps for frontend if using Container Apps
```
Create the `AZURE_CREDENTIALS` secret in GitHub (service principal with `Contributor` role on the resource group).
---

## 9. Secrets Management Workflow
1. **Add secrets to Key Vault** (as shown earlier).
2. **Enable managed identity** on Container Apps and Functions.
3. **Grant the identity access to Key Vault**:
```bash
az identity create -g $RESOURCE_GROUP -n whatsx-id
IDENTITY_CLIENT_ID=$(az identity show -g $RESOURCE_GROUP -n whatsx-id --query clientId -o tsv)
az keyvault set-policy -n whatsx-kv --object-id $IDENTITY_CLIENT_ID --secret-permissions get list
```
4. Reference secrets via environment variables using the `--secret` flag or the `AZURE_KEYVAULT_*` convention.
---

## 10. Monitoring & Alerts
- **Enable Application Insights** on Container Apps and Functions:
```bash
az monitor app-insights component create -a whatsx-ai -g $RESOURCE_GROUP -l $LOCATION
az containerapp update -n whatsx-backend -g $RESOURCE_GROUP --instrumentation-key <key>
```
- Set **Log Analytics workspace** for logs.
- Create **alerts** for high CPU, error rates, or failed email sends.
---

## 11. Cost Optimisation Tips
| Tip | How |
|---|---|
| Use **Consumption Plan** for Functions – pay per execution. |
| Choose **Basic SKU** for ACR and Cosmos DB if traffic is low. |
| Enable **auto‑scale** on Container Apps (scale‑to‑zero). |
| Periodically review **Azure Advisor** recommendations. |
---

## 12. Optional Enhancements
- **Azure Front Door** for global load‑balancing and WAF protection.
- **Azure CDN** for static assets.
- **Custom domain & TLS** via Azure DNS and Managed Certificates.
- **GitHub Actions matrix** to build both ARM and Bicep templates for IaC.

---

## 13. Summary Checklist
- [ ] Create Resource Group, ACR, Cosmos DB, Key Vault.
- [ ] Store all secrets (JWT, SendGrid, SMTP) in Key Vault.
- [ ] Build Docker images and push to ACR.
- [ ] Deploy backend to Container Apps (or App Service).
- [ ] Deploy frontend to Static Web Apps (or Container Apps).
- [ ] Deploy email function to Azure Functions.
- [ ] Wire environment variables and Key Vault access.
- [ ] Add Azure login and deploy steps to GitHub Actions.
- [ ] Enable monitoring (App Insights, Log Analytics).
- [ ] Test end‑to‑end flow (registration → verification → WhatsApp connect → send message).

**All services are fully managed, Azure‑native, and ready for production scaling.**
