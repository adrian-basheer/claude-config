# Scaffold Project Specification

This document defines the full specification for generating a new project from the patterns established in the Mixshare codebase. The scaffold produces a working **ASP.NET Core 8.0 Web API + React SPA** with **Entra External ID** authentication, **user approval workflow**, **role-based access control**, and **CI/CD pipeline** — all wired up and ready to go.

## Table of Contents

1. [Input](#input)
2. [Output Structure](#output-structure)
3. [Layered Architecture](#layered-architecture)
4. [API Project](#api-project)
5. [React Portal Project](#react-portal-project)
6. [CI/CD Pipeline](#cicd-pipeline)
7. [Architecture Patterns](#architecture-patterns)
8. [Lessons Learned (Baked In)](#lessons-learned-baked-in)

---

## Input

The scaffold takes two arguments:

1. **Project name** (string) — e.g. `MyApp`. Used to name the solution, projects, and replace all `Mixshare` references.
2. **Config JSON file** (path) — see `project-config.template.json` for the schema. Contains all Azure resource IDs, URLs, and customization options.

---

## Output Structure

```
<ProjectName>/
├── <ProjectName>.sln
├── CLAUDE.md                           # Pre-filled project instructions
├── lessons-learned.md                  # Seeded with all known gotchas
├── .gitignore
├── azure-pipelines.yml
├── project-config.json                 # Copy of the input config (for reference)
│
├── <ProjectName>Core/                  # Domain layer — no dependencies on other layers
│   ├── <ProjectName>Core.csproj
│   ├── Entities/
│   │   └── UserProfile.cs              # Pure domain model (not a Table entity)
│   └── Interfaces/
│       └── IUserProfileRepository.cs   # Abstraction defined here, implemented in Infra
│
├── <ProjectName>Application/           # Application layer — references Core only
│   ├── <ProjectName>Application.csproj
│   ├── Dtos/
│   │   └── UserProfileDto.cs
│   └── Services/
│       └── IUserProfileService.cs      # Service interface (implementation in Infra)
│
├── <ProjectName>Infra/                 # Infrastructure layer — references Core + Application
│   ├── <ProjectName>Infra.csproj
│   ├── Entities/
│   │   └── UserProfileEntity.cs        # Azure Table Storage entity (ITableEntity)
│   └── Services/
│       ├── GraphService.cs             # Microsoft Graph implementation
│       └── UserProfileService.cs       # Azure Table Storage implementation
│
├── <ProjectName>Api/                   # Web API layer — references Application + Infra (for DI)
│   ├── <ProjectName>Api.csproj
│   ├── Program.cs
│   ├── appsettings.json
│   ├── appsettings.Development.json    # Gitignored but generated with dev values
│   ├── Authorization/
│   │   ├── IOwnedResource.cs
│   │   ├── Operations.cs
│   │   └── ResourceOwnerAuthorizationHandler.cs
│   ├── Controllers/
│   │   ├── UserProfilesController.cs
│   │   └── UsersController.cs
│   └── Middleware/
│       └── UserApprovalMiddleware.cs
│
├── <ProjectName>Api.Tests/
│   └── <ProjectName>Api.Tests.csproj
│
├── <ProjectName>-portal/
│   ├── package.json
│   ├── vite.config.ts
│   ├── tsconfig.json
│   ├── tsconfig.app.json
│   ├── tsconfig.node.json
│   ├── eslint.config.js
│   ├── .prettierrc
│   ├── .env.example
│   ├── .env.development
│   ├── .env.production
│   ├── .gitignore
│   ├── web.config                      # IIS SPA routing
│   ├── index.html
│   ├── public/
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── index.css
│       ├── api/
│       │   ├── index.ts
│       │   ├── fetchClient.ts
│       │   └── clients/
│       │       ├── index.ts
│       │       ├── adminClient.ts
│       │       └── userProfileClient.ts
│       ├── components/
│       │   ├── index.ts
│       │   ├── common/
│       │   │   ├── index.ts
│       │   │   ├── ApiErrorAlert.tsx
│       │   │   ├── PermissionGate.tsx
│       │   │   └── ProtectedRoute.tsx
│       │   └── layout/
│       │       ├── index.ts
│       │       ├── AppLayout.tsx
│       │       └── Header.tsx
│       ├── config/
│       │   ├── index.ts
│       │   └── authConfig.ts
│       ├── hooks/
│       │   ├── index.ts
│       │   └── useAuthSync.ts
│       ├── pages/
│       │   ├── index.ts
│       │   ├── HomePage.tsx
│       │   ├── ProfilePage.tsx
│       │   ├── PendingApprovalPage.tsx
│       │   ├── AdminPage.tsx
│       │   └── UsersPage.tsx
│       ├── stores/
│       │   ├── index.ts
│       │   ├── authStore.ts
│       │   └── userProfileStore.ts
│       └── types/
│           ├── index.ts
│           ├── apiState.ts
│           ├── userProfile.ts
│           └── entraUser.ts
│
└── user-documents/
    ├── 01-entra-external-id-setup.md
    ├── 02-app-registrations.md
    ├── 03-storage-account.md
    ├── 04-key-vault.md
    ├── 05-app-services.md
    ├── 06-microsoft-graph-api.md
    └── 07-azure-devops-pipeline.md
```

---

## Layered Architecture

The scaffold generates a **Clean Architecture** solution with four .NET projects. Each layer has a single responsibility and a strict dependency direction: outer layers depend on inner layers, never the reverse.

### Dependency Graph

```
<ProjectName>Api
   ├── references <ProjectName>Application  (uses service interfaces, DTOs)
   ├── references <ProjectName>Infra        (for DI registration in Program.cs)
   └── references <ProjectName>Core         (for domain models, IOwnedResource)

<ProjectName>Infra
   ├── references <ProjectName>Application  (implements service interfaces)
   └── references <ProjectName>Core         (implements repository interfaces)

<ProjectName>Application
   └── references <ProjectName>Core         (uses domain models and repo interfaces)

<ProjectName>Core
   └── (no project references — pure domain)
```

### Layer Responsibilities

#### `<ProjectName>Core` — Domain Layer
- **Purpose**: Pure domain — no framework dependencies, no infrastructure concerns.
- **What lives here**:
  - Domain models (POCOs, not EF or Table Storage entities)
  - Repository interfaces (`IUserProfileRepository`, etc.)
  - Domain-specific value types, enums, exceptions
- **What does NOT live here**: Database entities, service implementations, DTOs, HTTP concerns
- **NuGet packages**: Minimal. No Azure, no ASP.NET Core.
- **Starter file**: `Entities/UserProfile.cs` — plain C# record with domain properties.

**Example `UserProfile.cs`:**
```csharp
namespace <ProjectName>Core.Entities;

public class UserProfile
{
    public string Id { get; set; } = string.Empty;       // Entra Object ID
    public string Email { get; set; } = string.Empty;
    public string DisplayName { get; set; } = string.Empty;
    public string Status { get; set; } = "pending";       // pending | approved | rejected | suspended
    public string? ApprovedBy { get; set; }
    public DateTimeOffset? ApprovedAt { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset LastLoginAt { get; set; }
    public string? Notes { get; set; }
    public List<string> Roles { get; set; } = [];
}
```

**Example `IUserProfileRepository.cs`:**
```csharp
namespace <ProjectName>Core.Interfaces;

public interface IUserProfileRepository
{
    Task<UserProfile?> GetAsync(string userId);
    Task<UserProfile> GetOrCreateAsync(string userId, string email, string displayName);
    Task<IEnumerable<UserProfile>> GetAllAsync();
    Task<IEnumerable<UserProfile>> GetPendingAsync();
    Task UpdateAsync(UserProfile profile);
}
```

---

#### `<ProjectName>Application` — Application Layer
- **Purpose**: Business logic and use-case orchestration. Defines service contracts; doesn't know how they're implemented.
- **What lives here**:
  - Service interfaces (`IUserProfileService`, `IGraphService`)
  - DTOs (data transfer objects for API responses/requests)
  - Use-case logic that crosses repositories
- **What does NOT live here**: HTTP controllers, database entities, Azure SDK calls
- **NuGet packages**: None beyond framework + Core project reference.

**Example `IUserProfileService.cs`:**
```csharp
namespace <ProjectName>Application.Services;

public interface IUserProfileService
{
    Task<UserProfileDto> GetOrCreateProfileAsync(string userId, string email, string displayName, bool isAdmin);
    Task<UserProfileDto?> GetProfileAsync(string userId);
    Task<IEnumerable<UserProfileDto>> GetAllProfilesAsync();
    Task<IEnumerable<UserProfileDto>> GetPendingProfilesAsync();
    Task ApproveAsync(string userId, string adminId);
    Task RejectAsync(string userId, string adminId, string? reason);
    Task SuspendAsync(string userId, string adminId);
    Task UpdateRolesAsync(string userId, List<string> roles);
    Dictionary<string, RoleDefinition> GetRoleDefinitions();
}
```

---

#### `<ProjectName>Infra` — Infrastructure Layer
- **Purpose**: Implements all infrastructure concerns — Azure Storage, Graph API, external HTTP calls.
- **What lives here**:
  - Azure Table Storage entities (`ITableEntity` implementations)
  - Concrete service implementations (`UserProfileService`, `GraphService`)
  - Repository implementations
  - Any external SDK usage (Azure SDK, Graph SDK, etc.)
- **What does NOT live here**: HTTP controllers, business rules, domain logic
- **NuGet packages**: Full Azure SDK packages (same as what was previously in Api):
  ```xml
  <PackageReference Include="Azure.Data.Tables" Version="12.10.0" />
  <PackageReference Include="Azure.Identity" Version="..." />
  <PackageReference Include="Microsoft.Graph" Version="5.101.0" />
  ```

**`UserProfileEntity.cs`** (Table Storage): Same as current Mixshare implementation — implements `ITableEntity`, maps to/from `UserProfile` domain model.

**`UserProfileService.cs`**: Implements `IUserProfileService`, takes `TableServiceClient` via constructor DI, maps between `UserProfileEntity` ↔ `UserProfile` ↔ `UserProfileDto`.

---

#### `<ProjectName>Api` — Web API Layer
- **Purpose**: HTTP surface only. Thin controllers that delegate to Application services. Auth, middleware, DI wiring.
- **What lives here**:
  - Controllers (thin — call services, return HTTP results)
  - `Program.cs` (DI composition root — registers Infra services against Application interfaces)
  - Authorization handlers and operations
  - Middleware (`UserApprovalMiddleware`)
  - `appsettings.json` / `appsettings.Development.json`
- **What does NOT live here**: Business logic, database access, domain models

**Key DI registrations in `Program.cs`**:
```csharp
// Register infrastructure implementations against application interfaces
builder.Services.AddSingleton<IUserProfileService, UserProfileService>();
builder.Services.AddSingleton<IGraphService, GraphService>();
```

---

#### `<ProjectName>Api.Tests` — Tests
- References `<ProjectName>Api`, `<ProjectName>Application`, `<ProjectName>Core`
- Tests controllers and services in isolation using mocked interfaces

---

### `.csproj` Project References

**`<ProjectName>Core.csproj`** — no project references

**`<ProjectName>Application.csproj`**:
```xml
<ItemGroup>
  <ProjectReference Include="..\<ProjectName>Core\<ProjectName>Core.csproj" />
</ItemGroup>
```

**`<ProjectName>Infra.csproj`**:
```xml
<ItemGroup>
  <ProjectReference Include="..\<ProjectName>Core\<ProjectName>Core.csproj" />
  <ProjectReference Include="..\<ProjectName>Application\<ProjectName>Application.csproj" />
</ItemGroup>
```

**`<ProjectName>Api.csproj`**:
```xml
<ItemGroup>
  <ProjectReference Include="..\<ProjectName>Core\<ProjectName>Core.csproj" />
  <ProjectReference Include="..\<ProjectName>Application\<ProjectName>Application.csproj" />
  <ProjectReference Include="..\<ProjectName>Infra\<ProjectName>Infra.csproj" />
</ItemGroup>
```

---

### Migration Note (Mixshare → Scaffold)

The Mixshare project grew organically with `Entities/`, `Services/`, and `Dtos/` directly in `MixshareApi`. The scaffold establishes the clean separation from day one. Mapping:

| Currently in `MixshareApi` | Moves to |
|----------------------------|----------|
| `Entities/UserProfileEntity.cs` | `<ProjectName>Infra/Entities/` |
| `Services/UserProfileService.cs` | `<ProjectName>Infra/Services/` |
| `Services/GraphService.cs` | `<ProjectName>Infra/Services/` |
| `Dtos/UserProfileDto.cs` | `<ProjectName>Application/Dtos/` |
| `Authorization/IOwnedResource.cs` | `<ProjectName>Core/` (or stays in Api) |
| `Controllers/`, `Middleware/`, `Authorization/` | Stay in `<ProjectName>Api/` |

---

## API Project

### NuGet Packages

With the layered architecture, packages are distributed across projects by responsibility:

**`<ProjectName>Api.csproj`** — HTTP + auth surface only:
```xml
<PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" Version="1.4.0" />
<PackageReference Include="Microsoft.Extensions.Azure" Version="1.11.0" />
<PackageReference Include="Microsoft.Identity.Web" Version="4.2.0" />
<PackageReference Include="Swashbuckle.AspNetCore" Version="6.6.2" />
```

**`<ProjectName>Infra.csproj`** — infrastructure/storage packages:
```xml
<PackageReference Include="Azure.Data.Tables" Version="12.10.0" />
<PackageReference Include="Azure.Identity" Version="1.13.2" />
<PackageReference Include="Azure.Storage.Blobs" Version="12.24.0" />
<PackageReference Include="Azure.Storage.Files.Shares" Version="12.22.0" />
<PackageReference Include="Azure.Storage.Queues" Version="12.22.0" />
<PackageReference Include="Microsoft.Graph" Version="5.101.0" />
```

### Program.cs Pattern

The Program.cs follows this exact order:

1. **Azure Key Vault** — loaded in non-Development only
2. **Entra config** — read from config with dev fallbacks
3. **AddControllers**
4. **CORS** — local dev + production portal URL
5. **Authentication** — `AddMicrosoftIdentityWebApi` with `MapInboundClaims = false`
6. **Authorization** — `ResourceOwnerAuthorizationHandler` singleton
7. **Graph service** — singleton
8. **User profile service** — singleton
9. **Swagger** — OAuth2 authorization code flow with PKCE
10. **Azure Storage clients** — URI overloads + `DefaultAzureCredential`
11. **Build app**
12. **Middleware pipeline**: Swagger → HTTPS → CORS → Auth → Authz → UserApprovalMiddleware → MapControllers

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `MapInboundClaims = false` | Keeps JWT claim names as-is (`oid`, `sub`, `groups`) instead of mapping to long URIs |
| `oid` claim preferred over `sub` | `oid` is the Entra Object ID, stable across apps, matches Graph API. `sub` is pairwise per app. |
| URI overloads for storage clients | String overload tries to parse as connection string. URI + `DefaultAzureCredential` works for both dev and prod. |
| Key Vault in production only | Dev uses `appsettings.Development.json` + user-secrets. Key Vault needs managed identity (App Service). |
| Singleton services | `GraphService` and `UserProfileService` are stateless (hold only clients). Table client creation is idempotent. |

### Authorization Pattern

Three-layer auth check in `ResourceOwnerAuthorizationHandler`:

1. **Admin bypass** — checks `extension_Role`, `IsInRole("Admin")`, and Entra group membership
2. **Resource ownership** — `IOwnedResource.OwnerId == userId`
3. **Specific permissions** — maps operations (Read/Write/Delete) to permission claims

Admin detection is replicated in `UserApprovalMiddleware` and `UserProfilesController.IsAdmin()`. All three check:
- `groups` claim against `Entra:AdminGroupId` from config
- `extension_Role` custom claim
- Standard role claim

### User Profile & Approval Workflow

**Entity**: `UserProfileEntity` implements `ITableEntity` (Azure Table Storage)
- PartitionKey: `"user"` (fixed)
- RowKey: Entra Object ID
- Status: `pending` | `approved` | `rejected` | `suspended`
- Roles: comma-separated string (Table Storage limitation — no arrays)

**Flow**:
1. User signs in → `GET /api/UserProfiles/me` → `GetOrCreateProfileAsync()`
2. New users get status `pending`, except admins (Entra group members) who are auto-approved with `Admin` role
3. Existing admins are also auto-approved if they were pending
4. `UserApprovalMiddleware` blocks non-approved users from all endpoints except `/api/userprofiles/me`, `/api/userprofiles/roles`, and `/swagger`
5. Admin approves via `POST /api/UserProfiles/{id}/approve` → assigns `Viewer` role if none set

**Roles & Permissions**:
- Roles are defined in `UserProfileService.RoleDefinitions` dictionary
- Each role maps to a list of permission strings
- Permissions are computed at read time, not stored
- Admin role includes all permissions by default (checked in `hasPermission`)

### API Endpoints

```
GET    /api/UserProfiles/me           — Get/create current user profile
GET    /api/UserProfiles              — List all profiles (admin)
GET    /api/UserProfiles/pending      — List pending profiles (admin)
GET    /api/UserProfiles/roles        — Get role definitions
POST   /api/UserProfiles/{id}/approve — Approve user (admin)
POST   /api/UserProfiles/{id}/reject  — Reject user (admin)
POST   /api/UserProfiles/{id}/suspend — Suspend user (admin)
PUT    /api/UserProfiles/{id}/roles   — Update user roles (admin)
GET    /api/Users                     — List Entra directory users (admin)
GET    /api/Users/{id}                — Get Entra user by ID (admin)
```

### Graph Service

Uses `ClientSecretCredential` (not managed identity) because Graph API requires app-level permissions:
- `User.Read.All` — list/read users
- `GroupMember.Read.All` — read admin group members

Config: `Graph:ClientId` (falls back to `Entra:ClientId`), `Graph:ClientSecret` from Key Vault.

---

## React Portal Project

### Dependencies

**Runtime:**
- `@azure/msal-browser` + `@azure/msal-react` — Entra External ID auth
- `antd` — UI component library
- `react` + `react-dom` — UI framework
- `react-router-dom` — client-side routing
- `zustand` — state management

**Dev:**
- `vite` + `@vitejs/plugin-react` — build tool
- `typescript` — type safety
- `vitest` + `@testing-library/react` + `jsdom` — testing
- `eslint` + `eslint-config-prettier` + `prettier` — code quality
- `openapi-typescript` — generate TS types from API swagger.json

### Environment Variables

```env
VITE_TENANT_SUBDOMAIN=myapptenant
VITE_TENANT_ID=<guid>
VITE_CLIENT_ID=<portal-client-id>
VITE_API_BASE_URL=http://localhost:5098   # .env.development
VITE_API_BASE_URL=https://myapp-api...    # .env.production
VITE_API_CLIENT_ID=<api-client-id>
```

### MSAL Configuration Pattern

```typescript
// authConfig.ts
export const msalConfig: Configuration = {
  auth: {
    clientId,
    authority: `https://${tenantSubdomain}.ciamlogin.com/${tenantId}`,
    redirectUri: window.location.origin,
    postLogoutRedirectUri: window.location.origin,
    knownAuthorities: [`${tenantSubdomain}.ciamlogin.com`],
  },
  cache: { cacheLocation: 'localStorage' },
};

export const loginRequest = {
  scopes: ['openid', 'offline_access', `api://${apiClientId}/access_as_user`],
};

export const apiConfig = {
  baseUrl: apiBaseUrl,
  scopes: [`api://${apiClientId}/access_as_user`],
};
```

### main.tsx Bootstrap Pattern

```typescript
const msalInstance = new PublicClientApplication(msalConfig);

async function initializeApp() {
  await msalInstance.initialize();
  await msalInstance.handleRedirectPromise(); // Handle auth redirect
  createRoot(document.getElementById('root')!).render(
    <StrictMode>
      <MsalProvider instance={msalInstance}>
        <App />
      </MsalProvider>
    </StrictMode>
  );
}
initializeApp();
```

### API Client Pattern (`fetchClient.ts`)

- Module-level `msalInstance` set via `setMsalInstance()` from `useAuthSync`
- `getAccessToken()` — acquires token silently from MSAL cache
- `fetchClient` object with `get<T>`, `post<T>`, `put<T>`, `delete<T>` methods
- Returns `ApiResult<T>` = `{ ok: boolean; data?: T; errors?: Record<string, string[]> }`
- Parses ASP.NET validation errors, problem details, and generic error messages
- Includes `uploadWithProgress<T>` using XMLHttpRequest for file uploads

### ApiState Pattern (`types/apiState.ts`)

Discriminated union for tracking async state in Zustand stores:

```typescript
type ApiState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; errors: Record<string, string[]> }
  | { status: 'success'; data: T };
```

Helper constructors: `idle()`, `loading()`, `success(data)`, `error(errors)`
Type guards: `isIdle()`, `isLoading()`, `isError()`, `isSuccess()`
Accessors: `getData()`, `getDataOrDefault()`

### Store Pattern (Zustand)

**authStore**: Syncs MSAL account state
- `account`, `isAuthenticated`, `setAccount()`, `getInitials()`, `getDisplayName()`

**userProfileStore**: Manages user profile + permissions
- `profile: ApiState<UserProfile>`
- `isApproved()`, `isAdmin()`, `getStatus()`, `getRoles()`
- `hasPermission(p)`, `hasAnyPermission(...ps)` — admin bypass built-in
- `fetchProfile()`, `reset()`

### useAuthSync Hook

Central hook (called once in `AppContent`) that:
1. Sets MSAL instance for `fetchClient`
2. Syncs account state to `authStore` when MSAL interaction completes
3. Fetches user profile once on login (ref-guarded to prevent duplicates)
4. Listens for `LOGIN_SUCCESS` / `LOGOUT_SUCCESS` events for mid-session auth changes
5. Resets profile store on logout

### Component Patterns

**PermissionGate**: Conditional rendering based on permissions
```tsx
<PermissionGate permission="admin:users">
  <AdminLink />
</PermissionGate>

<PermissionGate anyOf={['content:create', 'content:edit']}>
  <EditButton />
</PermissionGate>
```

**ProtectedRoute**: Route wrapper that enforces auth + approval + permission
- Waits for MSAL initialization (shows spinner)
- Handles cached-but-not-yet-synced accounts
- Redirects unauthenticated to home, unapproved to `/pending-approval`
- Checks specific permission

**ApiErrorAlert**: Renders `ApiState` errors or simple string errors as Ant Design alerts

**AppLayout**: `Layout` → `Header` + `Content` with `Outlet` for nested routes

**Header**: Navigation with permission-gated menu items + avatar dropdown

### Page Patterns

**HomePage**: Public landing page
**ProfilePage**: Shows user info and JWT claims (debugging)
**PendingApprovalPage**: Status-specific messages for pending/rejected/suspended
**AdminPage**: Landing page with admin tiles
**UsersPage**: Full admin page with:
- Merged view (Entra users + local profiles)
- Approval actions (approve/reject/suspend)
- Role management drawer with checkboxes
- Table with filters and sorting

### Routing

```tsx
// Public routes
<Route index element={<HomePage />} />
<Route path="profile" element={<ProfilePage />} />
<Route path="pending-approval" element={<PendingApprovalPage />} />

// Permission-gated routes
<Route path="admin" element={<ProtectedRoute permission="admin:users"><AdminPage /></ProtectedRoute>} />
<Route path="admin/users" element={<ProtectedRoute permission="admin:users"><UsersPage /></ProtectedRoute>} />

// Catch-all redirect
<Route path="*" element={<Navigate to="/" replace />} />
```

### UI Design System

The scaffold produces a consistent look and feel. All colours come from `project-config.json` so each project can be themed without editing component code.

#### Global CSS (`index.css`)

```css
* { margin: 0; padding: 0; box-sizing: border-box; }
body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif; }
#root { min-height: 100vh; }
```

#### Ant Design Theme

```tsx
<ConfigProvider theme={{ token: { colorPrimary: config.portal.primaryColor } }}>
```

This sets the primary colour across all Ant Design components (buttons, links, tags, etc.).

#### Colour Palette (from config)

| Token | Default | Used In |
|-------|---------|---------|
| `primaryColor` | `#3182ce` | Ant Design `colorPrimary`, stat values, active nav |
| `logoLetter` | `"M"` | Header logo square |
| `logoColor` | `#48bb78` | Logo background, avatar background |
| `headerGradient` | `["#1a365d", "#2c5282"]` | Header `linear-gradient(135deg, ...)` |
| `headingColor` | `#1a365d` | Page titles (`<Title>` colour) |
| `subtextColor` | `#718096` | Descriptions, secondary text |
| `accentColors.success` | `#38a169` | Success stats, positive indicators |
| `accentColors.purple` | `#805ad5` | Tertiary stat colour |
| `accentColors.adminTileUsers` | `#1890ff` | Admin tile: Users (border + icon) |
| `accentColors.adminTileSettings` | `#52c41a` | Admin tile: Settings (border + icon) |

#### Layout Pattern

```
┌─────────────────────────────────────────────────┐
│ Header (gradient bg, full width)                │
│   Logo [Letter] + App Name    Nav Items   Avatar│
├─────────────────────────────────────────────────┤
│ Content area (#f5f5f5 background)               │
│  ┌───────────────────────────────────────────┐  │
│  │ White card (padding 24px, border-radius 8)│  │
│  │                                           │  │
│  │   <Outlet /> (page content)               │  │
│  │                                           │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

#### Header Pattern

- **Background**: `linear-gradient(135deg, headerGradient[0] 0%, headerGradient[1] 100%)`
- **Logo**: Coloured square (`logoColor` background, 32x32, border-radius 6) with white letter (`logoLetter`)
- **App title**: White, 20px, bold, next to logo
- **Navigation**: Ant Design `<Menu>` horizontal, dark theme, transparent background
- **Nav items**: Conditionally shown based on `hasPermission()` — only show links the user can access
- **Admin gear icon**: Shown when `hasPermission('admin:users')`, top-right before avatar
- **Avatar**: `logoColor` background, user initials from `getInitials()`, click opens dropdown
- **Sign in button**: Primary button with `LoginOutlined` icon (shown when not authenticated)

#### Admin Page Tile Pattern

- Cards in responsive `Row/Col` grid (xs=24, sm=12, lg=8)
- Each card: `hoverable`, coloured top border (`borderTop: 4px solid ${tile.color}`), centred content
- Icon (48px) → Title (level 4) → Description (secondary text)
- Clicks navigate to the tile's path

#### Home Page Pattern

- Centred layout with welcome text
- Heading in `headingColor`, subtitle in `subtextColor`
- Authenticated: personalised greeting + CTA buttons
- Unauthenticated: "please sign in" message
- Stats row: 3 `<Statistic>` cards in `Row/Col` (xs=24, sm=8), each with a different accent colour

#### Page Header Pattern (Admin subpages)

```tsx
<Breadcrumb items={[
  { title: <Link to="/admin"><HomeOutlined /> Admin</Link> },
  { title: 'Page Name' },
]} />
<Title level={2}><IconOutlined /> Page Title</Title>
<Text type="secondary">Page description.</Text>
```

### web.config (IIS SPA Routing)

```xml
<rewrite>
  <rules>
    <rule name="SPA Routes" stopProcessing="true">
      <match url=".*" />
      <conditions logicalGrouping="MatchAll">
        <add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
        <add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
      </conditions>
      <action type="Rewrite" url="/index.html" />
    </rule>
  </rules>
</rewrite>
```

---

## CI/CD Pipeline

### Structure

```yaml
trigger: master
pool: self-hosted

stages:
  - Build:
      jobs:
        - BuildApi: restore → build → test → publish → artifact
        - BuildPortal: npm install → lint → test → build (with VITE_* env) → copy web.config → artifact
  - Deploy (depends on Build):
      jobs:
        - DeployApi: az login → download artifact → zip subfolder contents → az webapp deploy → az webapp cors add
        - DeployPortal: az login → download artifact → zip → az webapp deploy
```

### Critical Gotchas (Baked Into Template)

1. `zipAfterPublish: false` on `DotNetCoreCLI@2` publish
2. Zip the **project-named subfolder contents** (`api-drop/<ProjectName>Api/*`), not the parent
3. `az webapp cors add` after API deployment (IIS intercepts OPTIONS before ASP.NET Core)
4. VITE_* variables passed as `env:` on the build script step
5. `web.config` copied to `dist/` before artifact upload
6. Service principal login (`az login --service-principal`) for deployment

### Variable Group (`azure-deploy`)

Required variables:
- `AZURE_SP_APP_ID` — Service principal app ID
- `AZURE_SP_PASSWORD` — Service principal password (secret)
- `AZURE_SP_TENANT` — Azure AD tenant ID (not the CIAM tenant)
- `VITE_TENANT_SUBDOMAIN`
- `VITE_TENANT_ID`
- `VITE_CLIENT_ID`
- `VITE_API_BASE_URL`
- `VITE_API_CLIENT_ID`

---

## Architecture Patterns

### Config Flow

```
Production:   Azure Key Vault → IConfiguration → services
Development:  appsettings.Development.json + user-secrets → IConfiguration → services
```

Key Vault secrets use `--` delimiter for nesting: `Entra--TenantId` → `Entra:TenantId`

### Request Flow

```
Production:
  Browser → MSAL loginRedirect → Entra External ID → redirect back with code
         → MSAL acquireTokenSilent → Bearer token
         → API Management (gateway) → passes Authorization header through
         → API App Service validates JWT → UserApprovalMiddleware → Controller

Local Development:
  Browser → MSAL → Bearer token → API directly (no APIM)
```

### API Management (APIM)

APIM sits between the portal and the API in production:

- **Tier**: Consumption (serverless, pay-per-call, ~$3.50/million calls)
- **Purpose**: Analytics, rate limiting, API versioning, caching, developer portal (future)
- **Auth**: Passes JWT through to backend — API still does its own validation
- **CORS**: Configured on APIM for production; API CORS for local dev
- **Subscription**: Disabled (JWT auth is sufficient)
- Portal's `VITE_API_BASE_URL` points to APIM gateway URL in production, API directly in dev
- API imported into APIM from its Swagger/OpenAPI endpoint

### User Lifecycle

```
Sign up → First API call → Profile created (pending) → Admin approves → Full access
                                                     → Admin rejects  → Blocked
                                                     → Admin suspends → Blocked
```

Admin group members are auto-approved with Admin role.

### Permission Model

```
Entra Group (admin detection)
  └── Checked via JWT "groups" claim against config "Entra:AdminGroupId"

Local Roles (stored in Table Storage, comma-separated)
  └── Admin, Editor, Viewer (extensible)
      └── Each maps to permission strings
          └── "users:read", "content:create", "admin:users", etc.

Permission checks:
  API:    IsAdmin() || profile.status == "approved" (middleware)
  Portal: hasPermission(p) — admin bypasses, then checks computed permissions
```

---

## Lessons Learned (Baked In)

The scaffold generates code that already accounts for all these:

| # | Gotcha | How the scaffold handles it |
|---|--------|---------------------------|
| 1 | IIS intercepts CORS preflight on Azure App Service | Pipeline includes `az webapp cors add`; Program.cs CORS for local dev |
| 2 | DotNetCoreCLI publish creates project subfolder | Pipeline zips subfolder contents |
| 3 | DotNetCoreCLI zipAfterPublish double-zip | Set to `false` in pipeline |
| 4 | Browser CORS errors masking server errors | Documented in lessons-learned.md |
| 5 | Azurite not running causes SocketException | Dev config points to real Azure Storage |
| 6 | Admin group not detected in dev | `appsettings.Development.json` has AdminGroupId fallback |
| 7 | Vite env vars need restart | Documented |
| 8 | `oid` vs `sub` claim confusion | All `GetUserId()` methods prefer `oid` |
| 9 | Graph API needs GroupMember.Read.All | Documented in runsheet |
| 10 | Storage needs URI overloads not string | Program.cs uses `new Uri(...)` overloads |
| 11 | MapInboundClaims remaps claim names | Set to `false` in JWT config |
| 12 | Managed identity needs RBAC on storage | Documented in runsheet |

---

## Future: Bicep Infrastructure as Code

The runsheets currently require manual `az` CLI commands and portal clicks. A future enhancement is to add a **Bicep file** (`infrastructure/main.bicep`) that creates all automatable Azure resources in a single deployment.

### What Bicep Can Automate

| Resource | Bicep? | Notes |
|----------|--------|-------|
| Storage Account | Yes | Including RBAC role assignments |
| Key Vault | Yes | Including secrets and access policies |
| App Service Plans | Yes | Both API and Portal |
| App Services (API + Portal) | Yes | Including managed identity, app settings |
| APIM | Yes | Including API import, CORS policy, subscription settings |
| RBAC Role Assignments | Yes | Storage + Key Vault for managed identity and dev user |
| Key Vault Secrets | Yes | Entra config, storage URIs (values from parameters) |

### What Still Requires Manual Steps (Runsheets)

| Resource | Why Not Bicep |
|----------|--------------|
| Entra External ID Tenant | Different resource provider, tenant-level operation |
| App Registrations (API + Portal) | Entra/Graph API objects, not ARM resources |
| API Scopes & Permissions | Entra app registration config |
| Admin Consent for Graph | Requires interactive admin approval |
| Client Secret creation | Entra operation, value must be captured |
| User Flow configuration | Entra External ID feature |
| Azure DevOps Pipeline | Not an ARM resource (Azure DevOps REST API) |
| Service Principal | Azure AD operation |
| Variable Group | Azure DevOps Library |

### Proposed Bicep Structure

```
infrastructure/
├── main.bicep              # Orchestrator — calls modules
├── main.bicepparam         # Parameter file with project-specific values
├── modules/
│   ├── storage.bicep       # Storage account + RBAC
│   ├── keyvault.bicep      # Key Vault + secrets + RBAC
│   ├── appservice-api.bicep    # API App Service + managed identity
│   ├── appservice-portal.bicep # Portal App Service
│   └── apim.bicep          # API Management + API import + CORS policy
└── README.md               # How to deploy
```

### Deployment Command

```bash
az deployment group create \
  --resource-group MyResourceGroup \
  --template-file infrastructure/main.bicep \
  --parameters infrastructure/main.bicepparam
```

### Reduced Runsheet Flow With Bicep

```
Manual:  1. Entra Tenant → 2. App Registrations → 6. Graph API permissions
Bicep:   One command creates Storage + Key Vault + App Services + APIM + RBAC
Manual:  7. Pipeline setup (Azure DevOps)
```

This cuts the manual work from ~90-110 min down to ~40-50 min (Entra + Graph + Pipeline) plus a single Bicep deployment.
