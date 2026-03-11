---
name: ali-google-apps-script
description: |
  Google Apps Script development patterns for Workspace integrations. Use when:

  PLANNING: Designing Apps Script web apps, planning Drive integrations,
  architecting Workspace add-ons, considering OAuth scopes and permissions

  IMPLEMENTATION: Writing Apps Script code, creating HTML templates,
  building doGet/doPost handlers, implementing Drive API calls

  GUIDANCE: Asking about Apps Script best practices, quotas, limitations,
  deployment options, or how to structure Apps Script projects

  REVIEW: Checking Apps Script code for security issues, quota concerns,
  or alignment with Google's best practices
---

# Google Apps Script

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing Apps Script web apps or add-ons
- Planning Google Workspace integrations (Drive, Docs, Sheets)
- Considering OAuth scopes and permission models
- Architecting client-server communication in Apps Script

**Implementation:**
- Writing Apps Script server-side code (Code.gs)
- Creating HTML templates with HtmlService
- Implementing doGet/doPost web app handlers
- Using DriveApp, SpreadsheetApp, DocumentApp services
- Building custom UI with sidebar or dialog

**Guidance/Best Practices:**
- Asking about Apps Script quotas and limitations
- Understanding deployment options (web app, add-on, library)
- Managing script properties and user properties
- Error handling and logging patterns

**Review/Validation:**
- Checking for security issues (injection, XSS)
- Validating OAuth scope usage (least privilege)
- Reviewing quota-friendly patterns
- Verifying deployment configuration

---

## Key Principles

- **Server-client separation**: Code.gs runs server-side; HTML/JS runs client-side
- **google.script.run**: Bridge between client and server (async only)
- **HtmlService sandboxing**: IFRAME mode is default; no direct DOM from server
- **Quota awareness**: Know your limits (execution time, API calls, triggers)
- **Least privilege scopes**: Request only what you need
- **Error handling**: Client can't catch server exceptions directly

---

## Patterns We Use

### Web App Entry Point (doGet)

Standard pattern for web app that receives parameters:

```javascript
function doGet(e) {
  // Parse query parameters
  const params = e.parameter;
  const fileId = params.fileId || params.state;

  if (!fileId) {
    return HtmlService.createHtmlOutput('Missing required parameter');
  }

  // Create template with data
  const template = HtmlService.createTemplateFromFile('index');
  template.fileId = fileId;
  template.data = fetchData(fileId);

  return template.evaluate()
    .setTitle('My App')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}
```

### Template with Server Data

Passing data from server to HTML template:

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
  <base target="_top">
  <?!= include('styles'); ?>
</head>
<body>
  <div id="content"><?= data ?></div>
  <script>
    const fileId = '<?= fileId ?>';
  </script>
  <?!= include('scripts'); ?>
</body>
</html>
```

Include pattern for modular HTML:

```javascript
// Code.gs
function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}
```

### Client-Server Communication

Calling server functions from client JavaScript:

```javascript
// Client-side (in HTML)
google.script.run
  .withSuccessHandler(onSuccess)
  .withFailureHandler(onError)
  .serverFunction(param1, param2);

function onSuccess(result) {
  console.log('Server returned:', result);
}

function onError(error) {
  console.error('Server error:', error.message);
}
```

### Drive File Access

Reading file content with proper error handling:

```javascript
function getFileContent(fileId) {
  try {
    const file = DriveApp.getFileById(fileId);
    const content = file.getBlob().getDataAsString();
    const metadata = {
      name: file.getName(),
      mimeType: file.getMimeType(),
      size: file.getSize(),
      lastUpdated: file.getLastUpdated()
    };
    return { content, metadata, success: true };
  } catch (error) {
    return {
      success: false,
      error: error.toString(),
      message: 'Cannot access file. Check permissions.'
    };
  }
}
```

### State Parameter Parsing (Drive UI Integration)

When app is opened via Drive "Open with":

```javascript
function doGet(e) {
  // State is URL-encoded JSON from Drive
  const stateParam = e.parameter.state;

  if (!stateParam) {
    return HtmlService.createHtmlOutput('Direct access not supported');
  }

  const state = JSON.parse(decodeURIComponent(stateParam));

  // state.ids = array of file IDs
  // state.action = "open" | "create"
  // state.userId = user's Google profile ID

  const fileId = state.ids[0];
  // ... proceed with file
}
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Synchronous client calls | google.script.run is always async | Use callbacks or promises |
| Large data in template | Bloats initial page load | Fetch data after page load via google.script.run |
| Hardcoded API keys | Security risk, visible in source | Use PropertiesService.getScriptProperties() |
| No error handling on server | Errors don't propagate cleanly | Return {success, error, data} objects |
| Using innerHTML without sanitizing | XSS vulnerability | Use textContent or sanitize HTML |
| Long-running doGet | Times out, bad UX | Return page immediately, load data async |

---

## Quick Reference

### Quotas (Consumer Accounts)

| Limit | Value |
|-------|-------|
| Script runtime | 6 minutes/execution |
| Daily trigger runtime | 90 minutes total |
| UrlFetch calls | 20,000/day |
| Properties storage | 500KB total |
| Email sending | 100/day |

### Common Services

```javascript
// Drive
DriveApp.getFileById(id)
DriveApp.createFile(name, content, mimeType)
file.getBlob().getDataAsString()

// Spreadsheet
SpreadsheetApp.getActiveSpreadsheet()
sheet.getRange('A1:B10').getValues()

// Properties (key-value storage)
PropertiesService.getScriptProperties().getProperty('key')
PropertiesService.getUserProperties().setProperty('key', 'value')

// URL Fetch
UrlFetchApp.fetch(url, options)

// Utilities
Utilities.base64Encode(data)
Utilities.formatDate(date, timezone, format)
```

### Deployment Types

| Type | Use Case | Access |
|------|----------|--------|
| Web app | Standalone web interface | URL with /exec or /dev |
| Editor add-on | Extends Docs/Sheets/Slides | Installed via Marketplace |
| Drive add-on | Drive context menu integration | Via Drive UI |
| Library | Shared code across projects | Import via script ID |

---

## References

- [Apps Script Overview](https://developers.google.com/apps-script/overview)
- [Web Apps Guide](https://developers.google.com/apps-script/guides/web)
- [HTML Service](https://developers.google.com/apps-script/guides/html)
- [DriveApp Reference](https://developers.google.com/apps-script/reference/drive)
- [Quotas](https://developers.google.com/apps-script/guides/services/quotas)
