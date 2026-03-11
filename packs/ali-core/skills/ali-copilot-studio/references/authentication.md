# ali-copilot-studio - Authentication Reference

This reference provides detailed authentication configuration patterns for Microsoft CoPilot Studio connectors and actions.

---

## Authentication Methods

### Comparison Table

| Method | Use Case | Setup Complexity | Security Level |
|--------|----------|------------------|----------------|
| **No Auth** | Public APIs, read-only data | None | Low |
| **API Key** | Simple service auth | Low | Medium |
| **OAuth 2.0** | User-delegated access | High | High |
| **Service Account** | System-to-system calls | Medium | High |

---

## No Authentication

### When to Use
- Public APIs (weather, news, public data)
- Read-only endpoints with no sensitive data
- Internal APIs behind firewall (not exposed to internet)

### Configuration
No configuration required - simply provide API endpoint URL

### Limitations
- No user context
- Cannot access protected resources
- Vulnerable to abuse without rate limiting

---

## API Key Authentication

### When to Use
- Simple service-to-service authentication
- APIs requiring identification but not user-specific permissions
- Internal tool integrations

### Configuration

**In CoPilot Studio:**
1. Navigate to Connections
2. Create new connection
3. Select "API Key" authentication
4. Choose header or query parameter
5. Store API key value

**Header example:**
```
X-API-Key: {api_key_value}
```

**Query parameter example:**
```
https://api.example.com/data?api_key={api_key_value}
```

### Best Practices
- **Never hardcode keys**: Use environment variables or Key Vault
- **Rotate keys regularly**: Every 90 days minimum
- **Use different keys per environment**: Separate dev/test/prod keys
- **Monitor usage**: Track API key usage for anomalies
- **Implement rate limiting**: Protect against abuse

### Storage
Store API keys in Azure Key Vault:
```
Key Vault: company-secrets-prod
Secret: external-api-key
Value: sk_live_abc123...
```

Reference in Power Automate or connector configuration by Key Vault URI.

---

## OAuth 2.0 Authentication

### When to Use
- User-delegated access (act on behalf of user)
- Third-party API integrations (Google, Salesforce, etc.)
- Microsoft 365 services (Teams, SharePoint, Outlook)
- Any scenario requiring user consent

### Setup Steps

**1. Register app in Azure AD**
- Navigate to Azure Portal > Azure Active Directory > App Registrations
- Click "New registration"
- Enter name: "CoPilot Studio Connector - [Service Name]"
- Select supported account types
- Set redirect URI: `https://global.consent.azure-apim.net/redirect`

**2. Configure redirect URIs**
- Add CoPilot Studio callback URL
- Common pattern: `https://oauth.powerplatform.com/redirect`
- Ensure HTTPS protocol

**3. Set API permissions**

**Delegated permissions**: User acts on their own behalf
- Example: `Mail.Read`, `Files.ReadWrite.All`

**Application permissions**: App acts without user context
- Example: `User.Read.All`, `Sites.ReadWrite.All`
- Requires admin consent

**4. Create client secret**
- Navigate to Certificates & secrets
- Click "New client secret"
- Set expiration (recommend 12 months, rotate before expiry)
- Copy secret value immediately (shown only once)

**5. Configure in CoPilot Studio connector**
- Navigate to Connections
- Select OAuth 2.0
- Enter:
  - Authorization URL: `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize`
  - Token URL: `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token`
  - Client ID: From app registration
  - Client Secret: From step 4
  - Scope: Space-separated permissions (e.g., `User.Read Mail.Send`)

**6. Test authentication flow**
- Trigger connection from CoPilot Studio
- Consent screen appears
- User approves permissions
- Token stored securely

---

## OAuth Security Requirements

### Transport Security
- **HTTPS required**: All OAuth endpoints must use HTTPS
- **Validate certificates**: Ensure valid SSL/TLS certificates
- **No insecure redirects**: Block http:// redirect URIs

### Token Management
- **Store tokens securely**: Never log or expose access tokens
- **Implement token refresh**: Handle token expiration gracefully
- **Use least-privilege scopes**: Request only necessary permissions
- **Revoke on logout**: Invalidate tokens when user logs out

### User Consent
- **Clear permission descriptions**: Explain what app will access
- **Request consent per user**: Don't share tokens across users
- **Allow consent withdrawal**: Provide mechanism to revoke access
- **Log consent events**: Audit trail of user approvals

### Client Secret Protection
- **Store in Azure Key Vault**: Never hardcode in configuration
- **Rotate regularly**: Every 6-12 months
- **Use different secrets per environment**: Separate dev/prod secrets
- **Monitor secret access**: Alert on unauthorized access attempts

---

## Service Account Authentication

### When to Use
- System-to-system API calls
- Scheduled background jobs
- Automation without user interaction

### Managed Identity (Preferred)

**Setup:**
1. Enable managed identity for Power Automate/Logic App
2. Grant identity permissions to target resource
3. Authenticate using Azure AD token endpoint

**Benefits:**
- No credentials to manage
- Automatic rotation
- Tied to resource lifecycle
- Auditable in Azure AD logs

**Example: Access Azure SQL Database**
```
Managed Identity: flow-prod-sqlaccess
Grant: db_datareader role on SQL database
Connection: Uses Azure AD authentication (no password)
```

### Certificate-Based Authentication

**Setup:**
1. Generate certificate (X.509)
2. Upload to app registration
3. Configure connector to use certificate thumbprint

**Use case:** High-security environments requiring mutual TLS

---

## Power Platform Connectors

### Pre-Built Connectors

**Microsoft 365:**
- Outlook (Mail.Send, Calendar.ReadWrite)
- Teams (ChatMessage.Send, Team.ReadBasic)
- OneDrive (Files.ReadWrite.All)
- SharePoint (Sites.Read.All, Sites.ReadWrite.All)

**Dynamics 365:**
- Sales (CRM records read/write)
- Customer Service (Case management)
- Field Service (Work orders, scheduling)

**Azure:**
- SQL Database (Query, insert, update)
- Cosmos DB (Document operations)
- Key Vault (Secret retrieval)
- Storage (Blob upload/download)

**Third-Party:**
- Salesforce (OAuth, API key)
- ServiceNow (Basic auth, OAuth)
- Slack (OAuth)
- Zendesk (API token)

### Custom Connectors

**Creation steps:**
1. Define OpenAPI specification
2. Configure authentication method
3. Test operations
4. Publish to organization
5. Share with team

**Example: Custom REST API Connector**

```yaml
openapi: 3.0.0
info:
  title: Internal CRM API
  version: 1.0.0
servers:
  - url: https://crm.company.com/api/v1
components:
  securitySchemes:
    OAuth2:
      type: oauth2
      flows:
        authorizationCode:
          authorizationUrl: https://auth.company.com/oauth/authorize
          tokenUrl: https://auth.company.com/oauth/token
          scopes:
            crm:read: Read CRM data
            crm:write: Write CRM data
paths:
  /customers:
    get:
      summary: List customers
      security:
        - OAuth2: [crm:read]
```

---

## Troubleshooting

### OAuth Consent Screen Not Appearing

**Symptoms**: User clicks authenticate, nothing happens

**Solutions**:
- Check redirect URI matches app registration exactly
- Verify callback URL is HTTPS
- Clear browser cache and cookies
- Check if pop-ups are blocked
- Verify app registration status (active, not disabled)

### Token Refresh Failing

**Symptoms**: "Token expired" errors after period of inactivity

**Solutions**:
- Verify refresh token scope included
- Check token lifetime settings in Azure AD
- Implement exponential backoff for retries
- Handle refresh failures gracefully (prompt re-authentication)

### API Key Not Working

**Symptoms**: "Unauthorized" or "Forbidden" errors

**Solutions**:
- Verify API key is correct (copy-paste, no extra spaces)
- Check key is sent in correct location (header vs. query)
- Ensure key hasn't expired or been revoked
- Verify API key has necessary permissions
- Check rate limiting (may be temporarily blocked)

### Managed Identity Permission Issues

**Symptoms**: "Access denied" when using managed identity

**Solutions**:
- Verify managed identity is enabled
- Check role assignments (must be granted explicitly)
- Wait 5-10 minutes after granting permissions (propagation delay)
- Verify target resource supports managed identities
- Check Azure AD logs for detailed error

---

## Best Practices Summary

### Security
1. Use OAuth 2.0 for user-delegated access
2. Use managed identities for system accounts
3. Store secrets in Azure Key Vault
4. Rotate credentials regularly
5. Implement least-privilege access

### Reliability
1. Handle token expiration gracefully
2. Implement retry logic with exponential backoff
3. Monitor authentication failures
4. Log consent events for audit
5. Test authentication in all environments

### Compliance
1. Document permission requirements
2. Obtain user consent explicitly
3. Provide consent withdrawal mechanism
4. Audit access to sensitive resources
5. Comply with data residency requirements
