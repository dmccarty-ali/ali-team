---
name: ali-google-workspace-addins
description: |
  Google Workspace Add-ins and Drive UI integration patterns. Use when:

  PLANNING: Designing Drive "Open with" handlers, planning Workspace add-ons,
  architecting editor add-ons, considering OAuth consent and verification

  IMPLEMENTATION: Registering Drive UI integrations, configuring OAuth scopes,
  building add-on manifests, implementing Drive file handlers

  GUIDANCE: Asking about Workspace Marketplace publishing, OAuth verification
  requirements, add-on types, or Drive integration patterns

  REVIEW: Checking add-on configuration, validating OAuth scopes,
  reviewing manifest files, auditing security settings
---

# Google Workspace Add-ins

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Drive "Open with" handlers
- Planning Workspace Marketplace add-ons
- Architecting editor add-ons (Docs, Sheets, Slides)
- Considering OAuth consent screen requirements

**Implementation:**
- Registering Drive UI integrations in Cloud Console
- Configuring OAuth scopes and consent screens
- Building add-on manifests (appsscript.json)
- Implementing file type handlers

**Guidance/Best Practices:**
- Understanding Workspace Marketplace publishing
- OAuth verification requirements
- Add-on types and their capabilities
- Drive API integration patterns

**Review/Validation:**
- Checking add-on configuration
- Validating OAuth scope selection
- Reviewing security settings
- Auditing manifest configuration

---

## Key Principles

- **Drive UI Integration**: Register in Cloud Console to appear in "Open with" menu
- **OAuth scopes**: Use drive.file for per-file access (non-sensitive, easier verification)
- **State parameter**: Drive passes file info via URL-encoded JSON state parameter
- **Verification requirements**: Sensitive scopes require Google security review
- **Manifest configuration**: appsscript.json defines add-on behavior and scopes

---

## Patterns We Use

### Drive UI Integration Setup

Steps to register an app for Drive "Open with":

1. **Create Google Cloud Project**
   - Go to console.cloud.google.com
   - Create new project or select existing

2. **Enable APIs**
   - Google Drive API
   - Google Apps Script API (if using Apps Script)

3. **Configure OAuth Consent Screen**
   - Set app name, support email
   - Add scopes (prefer drive.file)
   - Add test users during development

4. **Register Drive UI Integration**
   - APIs & Services > Drive UI Integration
   - Set Open URL (your web app URL)
   - Configure MIME types and file extensions

### State Parameter Structure

When Drive opens your app, it appends a state parameter:

```javascript
// URL: https://your-app.com/?state=%7B%22ids%22%3A%5B...
const state = JSON.parse(decodeURIComponent(e.parameter.state));

// State object structure:
{
  "ids": ["fileId1", "fileId2"],        // Selected file IDs
  "resourceKeys": {                      // For link-shared files
    "fileId1": "resourceKey1"
  },
  "action": "open",                      // "open" | "create"
  "userId": "user123",                   // Google profile ID
  "folderId": "parentFolderId"           // For create actions
}
```

### OAuth Scope Selection

| Scope | Sensitivity | Use Case | Verification |
|-------|-------------|----------|--------------|
| drive.file | Non-sensitive | Per-file access (opened via UI) | Simplified |
| drive.readonly | Sensitive | Read all Drive files | Required |
| drive | Sensitive | Full Drive access | Required |
| drive.metadata.readonly | Sensitive | File metadata only | Required |

**Recommendation**: Use drive.file for "Open with" apps - it grants access only to files the user explicitly opens with your app.

### Apps Script Manifest (appsscript.json)

```json
{
  "timeZone": "America/New_York",
  "dependencies": {},
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "oauthScopes": [
    "https://www.googleapis.com/auth/drive.file",
    "https://www.googleapis.com/auth/script.container.ui"
  ],
  "webapp": {
    "executeAs": "USER_DEPLOYING",
    "access": "ANYONE"
  }
}
```

### MIME Type Configuration

For markdown viewer, register these MIME types:

| Type | MIME | Extension | Priority |
|------|------|-----------|----------|
| Markdown | text/markdown | .md | Default |
| Plain text | text/plain | .txt | Secondary |
| Text (generic) | text/x-markdown | .markdown | Secondary |

### Handling Resource Keys (Link-Shared Files)

Files shared via link require resource keys:

```javascript
function getFileWithResourceKey(fileId, resourceKey) {
  // For Apps Script, resource keys are handled automatically
  // when accessing files opened via Drive UI

  // For direct API calls, include resource key in request:
  const url = `https://www.googleapis.com/drive/v3/files/${fileId}?`
    + `key=${API_KEY}&resourceKey=${resourceKey}`;
}
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Using drive scope for "Open with" | Overly broad, triggers verification | Use drive.file scope |
| Hardcoding file IDs | Won't work for user files | Parse from state parameter |
| Ignoring resourceKeys | Link-shared files won't load | Check state.resourceKeys |
| Missing error for invalid state | Confusing UX | Validate state on entry |
| Publishing without verification | Blocked by Google | Complete OAuth verification |
| Requesting unnecessary scopes | User distrust, verification delays | Minimal scopes |

---

## Quick Reference

### Cloud Console URLs

```
Project Dashboard: https://console.cloud.google.com/home/dashboard
Drive UI Integration: https://console.cloud.google.com/apis/credentials/consent
OAuth Consent: https://console.cloud.google.com/apis/credentials/consent
Enable APIs: https://console.cloud.google.com/apis/library
```

### Drive UI Integration Fields

| Field | Description |
|-------|-------------|
| Application Name | Displayed in "Open with" menu |
| Open URL | Your web app URL (receives state param) |
| Default MIME types | File types you're built for |
| Secondary MIME types | File types you can also handle |
| Short description | Shown in Drive UI |
| Icons | 16x16, 32x32, 128x128 PNG |

### Verification Requirements

| Scope Type | Requirement |
|------------|-------------|
| Non-sensitive | Self-verification (immediate) |
| Sensitive | Google security review (weeks) |
| Restricted | Enhanced security review + annual audit |

Non-sensitive scopes: drive.file, drive.appdata, drive.install

### Add-on Types

| Type | Trigger | Capability |
|------|---------|------------|
| Drive UI Integration | Right-click > Open with | Open/create files |
| Editor Add-on | Menu in Docs/Sheets/Slides | Sidebar, dialog, menu items |
| Gmail Add-on | Compose/view email | Sidebar, compose actions |
| Calendar Add-on | View event | Sidebar, conference solutions |

---

## Your Codebase

When building Drive integrations for Aliunde projects:

| Purpose | Pattern |
|---------|---------|
| File viewer web app | Apps Script + Drive UI Integration |
| Internal tools | Web app + Workspace domain restriction |
| Public tools | Marketplace publishing + verification |

---

## References

- [Drive UI Integration Guide](https://developers.google.com/workspace/drive/api/guides/enable-sdk)
- [Open with Context Menu](https://developers.google.com/workspace/drive/api/guides/integrate-open)
- [OAuth Scopes for Drive](https://developers.google.com/workspace/drive/api/guides/api-specific-auth)
- [Apps Script Manifest](https://developers.google.com/apps-script/manifest)
- [Workspace Marketplace Publishing](https://developers.google.com/workspace/marketplace/how-to-publish)
