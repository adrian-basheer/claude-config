# Runsheet 8: Azure API Management (APIM)

## Purpose

Place an API Management gateway in front of the API App Service. The portal calls APIM instead of the API directly. This gives you analytics, rate limiting, API versioning, caching, a developer portal, and a single front door for future APIs — without changing any API code.

## Architecture

```
Portal (React SPA)
    │
    ▼
Azure API Management (gateway)       ← analytics, rate limiting, policies
    │
    ▼
API App Service (ASP.NET Core)       ← JWT validation, approval, permissions
    │
    ▼
Azure Table Storage / Graph API
```

The API still validates JWTs and checks permissions. APIM passes the `Authorization` header through to the backend unchanged.

## Prerequisites

- API App Service deployed and working (Runsheet 5)
- Resource group created

## Steps

### 1. Create the APIM Instance

```bash
az apim create \
  --name {{apim.name}} \
  --resource-group {{azure.resourceGroup}} \
  --location {{azure.location}} \
  --publisher-name "{{apim.publisherName}}" \
  --publisher-email "{{apim.publisherEmail}}" \
  --sku-name Consumption
```

> **SKU choice**: `Consumption` tier is serverless (pay-per-call, no idle cost, ~$3.50/million calls). Good for dev and low-traffic apps. Upgrade to `Developer` or `Standard` later if you need a custom domain, developer portal, or virtual network integration.

> **Note**: APIM creation can take 30-45 minutes for non-Consumption tiers. Consumption tier is usually ready in 2-5 minutes.

Record the gateway URL → `apim.gatewayUrl` (format: `https://{{apim.name}}.azure-api.net`)

### 2. Import the API from App Service

#### Option A: Import from OpenAPI (Recommended)

1. Go to Azure Portal → API Management → **APIs** → **+ Add API**
2. Select **OpenAPI**
3. Enter the Swagger URL: `{{appServices.api.url}}/swagger/v1/swagger.json`
4. Fill in:
   - **Display name**: `{{projectName}} API`
   - **Name**: `{{projectName}}-api` (URL-safe)
   - **API URL suffix**: `api` (so endpoints become `https://apim-name.azure-api.net/api/...`)
5. Click **Create**

#### Option B: Import from App Service

1. **APIs** → **+ Add API** → **App Service**
2. Browse and select `{{appServices.api.name}}`
3. APIM auto-discovers the endpoints

### 3. Configure the Backend

1. Go to your imported API → **Settings** tab
2. Under **Web service URL**, verify it points to: `{{appServices.api.url}}`
3. Under **Subscription required**: **Uncheck this** (we're using JWT auth, not APIM subscriptions)
   - Or leave it checked if you want an additional API key layer

### 4. Configure CORS Policy on APIM

1. Go to your API → **All operations** → **Inbound processing** → **+ Add policy**
2. Select **CORS** (or use the policy editor)
3. Add the portal origins:

```xml
<cors allow-credentials="true">
    <allowed-origins>
        <origin>http://localhost:5173</origin>
        <origin>https://localhost:5173</origin>
        <origin>https://{{appServices.portal.name}}.azurewebsites.net</origin>
    </allowed-origins>
    <allowed-methods>
        <method>*</method>
    </allowed-methods>
    <allowed-headers>
        <header>*</header>
    </allowed-headers>
</cors>
```

> **Important**: With APIM in front, CORS should be handled at the APIM level. You may still want CORS on the API App Service for direct access during local development.

### 5. Pass Through the Authorization Header

By default, APIM forwards all headers. Verify this is working:

1. Go to your API → **All operations** → **Inbound processing**
2. Check that no policy strips the `Authorization` header
3. The default behaviour is to pass it through — no action needed unless you added custom policies

### 6. Update the Portal to Call APIM

Update the portal environment to point to APIM instead of the API directly:

**`.env.production`:**
```env
VITE_API_BASE_URL=https://{{apim.name}}.azure-api.net/api
```

**`.env.development`** (keep pointing to API directly for local dev):
```env
VITE_API_BASE_URL=http://localhost:5098
```

**Pipeline variable group** — update `VITE_API_BASE_URL`:
```
VITE_API_BASE_URL = https://{{apim.name}}.azure-api.net/api
```

### 7. Update CORS on API App Service

With APIM in front, the API now receives requests from APIM's IP, not the browser. You have two options:

**Option A: Keep API CORS open** (simpler, recommended for now)
- Leave the existing `az webapp cors add` in the pipeline as-is
- Local dev still calls the API directly and needs CORS

**Option B: Restrict API to APIM only** (more secure, future)
- Remove browser origins from API CORS
- Add the APIM gateway URL to API CORS
- Configure APIM with a managed identity and validate it in the API
- Use APIM's **IP restriction** policy

### 8. Test the Flow

1. Open the portal at the production URL
2. Sign in
3. Open browser DevTools → Network tab
4. Verify API calls go to `https://{{apim.name}}.azure-api.net/api/...` (not the API directly)
5. Check APIM analytics: Portal → API Management → **Analytics**

## APIM Features Available After Setup

| Feature | How to Enable |
|---------|--------------|
| **Analytics** | Built-in — view in Portal → APIM → Analytics |
| **Rate Limiting** | Add `rate-limit` or `rate-limit-by-key` inbound policy |
| **Caching** | Add `cache-lookup` / `cache-store` policies |
| **API Versioning** | Create version sets in APIM |
| **Request/Response Logging** | Enable Application Insights integration |
| **IP Filtering** | Add `ip-filter` inbound policy |
| **Mock Responses** | Add `mock-response` policy (useful for frontend dev) |
| **Developer Portal** | Available on Developer tier and above (not Consumption) |

## Example: Add Rate Limiting

```xml
<inbound>
    <rate-limit calls="100" renewal-period="60" />
</inbound>
```

## Verification

- [ ] APIM instance created (Consumption tier)
- [ ] API imported from Swagger/OpenAPI
- [ ] Subscription requirement disabled (using JWT auth)
- [ ] CORS policy configured on APIM
- [ ] Authorization header passes through to backend
- [ ] Portal `.env.production` updated to APIM URL
- [ ] Pipeline variable group updated
- [ ] Portal calls go through APIM (check Network tab)
- [ ] APIM Analytics shows traffic

## Common Issues

- **CORS errors after switching to APIM**: CORS must be configured on APIM now, not just the API. Add the CORS policy as shown in step 4.
- **401 Unauthorized**: APIM is stripping or modifying the Authorization header. Check inbound policies.
- **404 on APIM**: The API URL suffix may be wrong. If suffix is `api`, then endpoints are at `https://apim.azure-api.net/api/UserProfiles/me` (not `https://apim.azure-api.net/UserProfiles/me`).
- **Subscription key required**: Uncheck "Subscription required" in the API settings, or pass `Ocp-Apim-Subscription-Key` header.
- **Swagger not accessible**: Swagger is on the API directly, not through APIM. Use the API URL for Swagger testing.

## Cost

| Tier | Cost | Best For |
|------|------|----------|
| Consumption | ~$3.50/million calls, no idle cost | Dev, low-traffic production |
| Developer | ~$50/month | Development with developer portal |
| Basic | ~$150/month | Small production workloads |
| Standard | ~$700/month | Medium production with VNet |

## Output → `project-config.json`

```json
{
  "apim": {
    "name": "{{apim.name}}",
    "gatewayUrl": "https://{{apim.name}}.azure-api.net",
    "publisherName": "Your Name",
    "publisherEmail": "you@example.com",
    "sku": "Consumption",
    "apiSuffix": "api"
  }
}
```
