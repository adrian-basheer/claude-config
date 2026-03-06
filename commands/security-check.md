# Security check

Run a comprehensive security review of the current project against OWASP Top 10 and common web application vulnerabilities.

## What this command does

Audits the codebase for security issues, produces a findings report with severity ratings, and offers to fix issues found. This is not a penetration test — it's a static code review focused on common vulnerability patterns.

## Steps

1. **Identify the project type** by reading `CLAUDE.md`, `*.csproj`, `package.json`, etc. Adapt checks to the tech stack found.

2. **Run automated checks:**
   - `dotnet list package --vulnerable` — known CVEs in NuGet packages
   - `npm audit` (in portal directory) — known CVEs in npm packages
   - Search for `.env` files that might be committed with real secrets

3. **Review each OWASP Top 10 category** by reading relevant source files:

### A01: Broken Access Control
- [ ] All API endpoints require authentication (no accidental `[AllowAnonymous]`)
- [ ] Authorization checks on every controller action (not just the controller level)
- [ ] Resource ownership verified before operations (`IOwnedResource` pattern)
- [ ] Admin-only endpoints check admin role/group membership
- [ ] `UserApprovalMiddleware` cannot be bypassed
- [ ] CORS allows only expected origins (check `Program.cs` and pipeline CORS)
- [ ] No direct object references exposable to users (e.g., sequential IDs in URLs)
- [ ] File download endpoints validate the requested blob belongs to the user's accessible resources

### A02: Cryptographic Failures
- [ ] No secrets in source code (connection strings, client secrets, API keys)
- [ ] No secrets in `appsettings.json` (should be in Key Vault or user-secrets)
- [ ] HTTPS enforced (`UseHttpsRedirection`)
- [ ] Tokens stored securely (MSAL uses localStorage — document the tradeoff)
- [ ] SAS tokens have short expiry and minimal permissions
- [ ] No sensitive data in URL query parameters or logs

### A03: Injection
- [ ] No raw SQL or string concatenation in queries
- [ ] Cosmos DB / Table Storage queries use parameterized inputs
- [ ] No `eval()`, `new Function()`, or `dangerouslySetInnerHTML` in React code
- [ ] File names sanitized before use in blob storage paths
- [ ] No command injection via user input passed to `Process.Start` or shell commands
- [ ] API responses don't reflect unsanitized user input (XSS via API)

### A04: Insecure Design
- [ ] Rate limiting on authentication endpoints
- [ ] User approval workflow cannot be self-approved
- [ ] Role changes require admin privilege (not just any authenticated user)
- [ ] File upload size limits configured
- [ ] File type validation (not just extension — check content type / magic bytes for FSA/CSV)

### A05: Security Misconfiguration
- [ ] Swagger UI disabled in production (or behind auth)
- [ ] Stack traces not exposed in production error responses
- [ ] Default error handler returns generic messages
- [ ] `ASPNETCORE_ENVIRONMENT` is not `Development` in production App Service
- [ ] Key Vault access uses managed identity, not connection strings
- [ ] Storage account access uses managed identity or SAS, not account keys in code
- [ ] No unnecessary HTTP methods enabled
- [ ] Response headers don't leak server info (X-Powered-By, Server)

### A06: Vulnerable and Outdated Components
- [ ] `dotnet list package --vulnerable` reports no issues
- [ ] `npm audit` reports no critical/high issues
- [ ] .NET SDK and runtime are current
- [ ] Node.js version is LTS

### A07: Identification and Authentication Failures
- [ ] JWT validation is properly configured (issuer, audience, signing key)
- [ ] Token expiry is reasonable
- [ ] `MapInboundClaims = false` is set (prevents claim name mapping issues)
- [ ] User identity extracted from `oid` claim (not `sub` which is pairwise)
- [ ] No custom auth schemes that bypass Entra validation
- [ ] Logout properly clears MSAL cache

### A08: Software and Data Integrity Failures
- [ ] CI/CD pipeline uses pinned versions for actions/tasks
- [ ] No `npm install` with `--ignore-scripts` disabled carelessly
- [ ] Pipeline secrets are in variable groups (not inline)
- [ ] Deployment uses zip deploy (not FTP or direct file copy)

### A09: Security Logging and Monitoring Failures
- [ ] Authentication failures are logged
- [ ] Authorization failures are logged (403 responses)
- [ ] User approval/rejection actions are logged with admin ID
- [ ] No sensitive data in logs (tokens, passwords, PII)
- [ ] Application Insights or equivalent configured for production

### A10: Server-Side Request Forgery (SSRF)
- [ ] File download proxy validates blob URIs against an allow-list of hosts/containers
- [ ] No endpoints that fetch arbitrary URLs based on user input
- [ ] Graph API calls use fixed endpoints, not user-supplied URLs

4. **Check project-specific concerns:**
   - [ ] Dev personas / `DevPersonaMiddleware` cannot activate in production
   - [ ] `DevController` not accessible in production
   - [ ] Blob containers set to `PublicAccessType.None`
   - [ ] SAS URL generation validates container allow-list
   - [ ] CSV parsing handles malformed input gracefully (no crash on bad data)
   - [ ] FSA/ABIF file parsing handles malformed input (no buffer overflows or infinite loops)

5. **Produce a findings report** with this format:

   ```
   ## Security Check Results — [Project Name]

   ### Summary
   - Critical: X
   - High: X
   - Medium: X
   - Low: X
   - Info: X

   ### Findings

   #### [CRITICAL] Finding title
   **File**: path/to/file.cs:line
   **Category**: OWASP A0X
   **Description**: What the issue is
   **Impact**: What could go wrong
   **Fix**: How to fix it
   ```

6. **Offer to fix** — after presenting findings, ask the user if they'd like you to fix any or all issues.

## Notes

- Read files before reporting issues — don't assume vulnerabilities from file names alone
- False positives are worse than missed findings — only report issues you've confirmed in the code
- For borderline cases, report as "Info" with context so the user can decide
- If the project has tests, check that security-relevant paths are tested
