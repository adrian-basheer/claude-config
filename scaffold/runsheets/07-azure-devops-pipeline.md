# Runsheet 7: Azure DevOps CI/CD Pipeline

## Purpose

Set up an Azure DevOps pipeline that automatically builds, tests, and deploys the API and portal to Azure App Service on every push to the trigger branch.

## Prerequisites

- Azure DevOps organization and project
- All Azure resources created (Runsheets 1-6)
- Service principal for deployment

## Steps

### 1. Create a Service Principal

```bash
# Create the service principal (in your MAIN Azure AD, not the CIAM tenant)
az ad sp create-for-rbac \
  --name "{{projectName}}-deploy" \
  --role Contributor \
  --scopes /subscriptions/{{azure.subscriptionId}}/resourceGroups/{{azure.resourceGroup}}
```

This outputs:
```json
{
  "appId": "<SP_APP_ID>",
  "displayName": "{{projectName}}-deploy",
  "password": "<SP_PASSWORD>",
  "tenant": "<SP_TENANT>"
}
```

Save all three values — you'll need them for the variable group.

### 2. Create the Variable Group in Azure DevOps

1. Go to your Azure DevOps project → **Pipelines** → **Library**
2. Click **+ Variable group**
3. Name: `{{pipeline.variableGroup}}` (e.g. `azure-deploy`)
4. Add these variables:

| Variable | Value | Lock? |
|----------|-------|-------|
| `AZURE_SP_APP_ID` | `<from step 1>` | No |
| `AZURE_SP_PASSWORD` | `<from step 1>` | **Yes** (secret) |
| `AZURE_SP_TENANT` | `<from step 1>` | No |
| `VITE_TENANT_SUBDOMAIN` | `{{entra.tenantSubdomain}}` | No |
| `VITE_TENANT_ID` | `{{entra.tenantId}}` | No |
| `VITE_CLIENT_ID` | `{{entra.portalAppRegistration.clientId}}` | No |
| `VITE_API_BASE_URL` | `{{appServices.api.url}}` | No |
| `VITE_API_CLIENT_ID` | `{{entra.apiAppRegistration.clientId}}` | No |

5. Click **Save**

### 3. Set Up the Agent

#### Option A: Self-Hosted Agent (No Parallelism Approval Needed)

1. In Azure DevOps → **Project settings** → **Agent pools**
2. Create a new pool (e.g. `{{pipeline.agentPool}}`)
3. Download and install the agent on your machine:
   ```bash
   # Follow: https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/windows-agent
   ```
4. Register the agent with name `{{pipeline.agentName}}`

#### Option B: Microsoft-Hosted Agent

1. Request free parallelism: [Azure DevOps Parallelism Request](https://aka.ms/azpipelines-parallelism-request)
2. Wait for approval (can take 2-3 business days)
3. Change pipeline pool to:
   ```yaml
   pool:
     vmImage: 'windows-latest'
   ```

### 4. Create the Pipeline

1. Go to **Pipelines** → **+ New pipeline**
2. Select your repo source (GitHub, Azure Repos, etc.)
3. Select **Existing Azure Pipelines YAML file**
4. Point to `azure-pipelines.yml` in the repo root
5. Click **Run** to test

### 5. Authorize the Variable Group

On first run, the pipeline will ask to authorize access to the variable group. Click **Permit**.

## Pipeline Structure

The generated `azure-pipelines.yml` has this structure:

```
Build Stage (parallel jobs):
├── BuildApi:    .NET restore → build → test → publish → artifact
└── BuildPortal: npm install → lint → test → build (VITE_* env) → copy web.config → artifact

Deploy Stage (after Build succeeds):
├── DeployApi:    az login → download → zip subfolder/* → az webapp deploy → az webapp cors add
└── DeployPortal: az login → download → zip → az webapp deploy
```

### Critical Pipeline Gotchas

| Gotcha | What Goes Wrong | The Fix |
|--------|----------------|---------|
| `zipAfterPublish` default | `DotNetCoreCLI@2` publish zips by default → pipeline zips again → double-zip | Set `zipAfterPublish: false` |
| Project subfolder | Publish output goes to `api-drop/{{projectName}}Api/` → Azure extracts to `wwwroot/{{projectName}}Api/` | Zip contents of subfolder: `api-drop/{{projectName}}Api/*` |
| CORS after deploy | IIS intercepts OPTIONS before ASP.NET Core | `az webapp cors add` step after API deployment |
| VITE_* at build time | Vite bakes env vars into JS bundle at build time | Pass as `env:` on the build script step |
| web.config | IIS needs it for SPA routing, not in dist by default | `CopyFiles@2` task copies it before artifact upload |

## Verification

- [ ] Service principal created with Contributor role
- [ ] Variable group created with all variables
- [ ] Agent pool and agent running (self-hosted) or parallelism approved (Microsoft-hosted)
- [ ] Pipeline created and linked to YAML file
- [ ] First run succeeds — build + deploy
- [ ] API accessible at production URL with Swagger
- [ ] Portal accessible at production URL

## Common Issues

- **Pipeline hangs waiting for agent**: Self-hosted agent isn't running. Start the agent service.
- **`az login` fails**: Check service principal credentials in variable group. Make sure `AZURE_SP_TENANT` is your main Azure AD tenant (not the CIAM tenant).
- **Portal build fails**: VITE_* variables not in variable group, or variable group not linked to Build stage.
- **API 404 after deploy**: Check the zip structure (see Debugging section in Runsheet 5).
- **CORS errors after deploy**: The `az webapp cors add` step may have failed. Re-run it manually.

## Debugging Tips

```bash
# Check what's deployed
az rest --method get \
  --url "https://{{appServices.api.name}}.scm.<region>.azurewebsites.net/api/vfs/site/wwwroot/" \
  --resource "https://management.azure.com/"

# Check CORS config
az webapp cors show --resource-group {{azure.resourceGroup}} --name {{appServices.api.name}}

# View deployment logs
az webapp log download --resource-group {{azure.resourceGroup}} --name {{appServices.api.name}} --log-file logs.zip
```
