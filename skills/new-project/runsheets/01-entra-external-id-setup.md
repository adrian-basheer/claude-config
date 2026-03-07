# Runsheet 1: Entra External ID Tenant Setup

## Purpose

Create the Microsoft Entra External ID (CIAM) tenant that handles user sign-up, sign-in, and identity management for your application.

## Prerequisites

- Azure subscription with permissions to create resources
- Global Administrator or Azure AD Privileged Role Administrator

## Steps

### 1. Create the Entra External ID Tenant

1. Go to [Azure Portal](https://portal.azure.com) → **Microsoft Entra ID** → **Manage tenants**
2. Click **+ Create**
3. Select **Customer** (this is the External ID / CIAM type)
4. Fill in:
   - **Tenant name**: `{{projectName}}tenant` (e.g. `myapptenant`)
   - **Domain name**: `{{projectName}}tenant.onmicrosoft.com`
   - **Location**: Choose your region
5. Click **Review + Create** → **Create**

### 2. Record Tenant Information

After creation, switch to the new tenant directory and note:

| Field | Value | Config Key |
|-------|-------|------------|
| Tenant subdomain | `{{projectName}}tenant` | `entra.tenantSubdomain` |
| Tenant ID | (GUID from Overview page) | `entra.tenantId` |

### 3. Create Admin Security Group

1. In the new tenant: **Microsoft Entra ID** → **Groups** → **+ New group**
2. Fill in:
   - **Group type**: Security
   - **Group name**: `{{projectName}}Admins`
   - **Group description**: Site administrators
   - **Membership type**: Assigned
3. Add yourself as a member
4. Click **Create**
5. Note the **Object ID** of the group → `entra.adminGroup.objectId`

### 4. Configure Token Claims

1. Go to **App registrations** (you'll create these in Runsheet 2)
2. For the API app registration → **Token configuration** → **Add groups claim**
3. Select **Security groups**
4. Under **Customize token properties by type**, for **Access** token, check **Group ID**
5. Save

### 5. Enable Social Login (Optional)

1. **External Identities** → **All identity providers**
2. Enable **Microsoft Account** (or Google, Facebook, etc.)
3. For Microsoft Account: use the default Microsoft configuration

### 6. Create User Flow

1. **External Identities** → **User flows** → **+ New user flow**
2. Name: `signupsignin1`
3. Select identity providers (email + any social providers)
4. Under **User attributes**, select what to collect:
   - Display Name (required)
   - Email Address (required)
5. Link the API app registration (after creating it in Runsheet 2)

## Verification

- [ ] Tenant created and accessible
- [ ] Tenant ID recorded
- [ ] Admin security group created with Object ID recorded
- [ ] User flow created
- [ ] Social login configured (if needed)

## Common Issues

- **Can't find "Customer" option**: Ensure you have the right subscription tier. External ID requires at least a free tier.
- **Tenant creation fails**: Check subscription limits — you may have a max number of tenants.

## Output → `project-config.json`

```json
{
  "entra": {
    "tenantSubdomain": "{{projectName}}tenant",
    "tenantId": "<from Azure Portal>",
    "adminGroup": {
      "objectId": "<from step 3>",
      "displayName": "{{projectName}}Admins"
    }
  }
}
```
