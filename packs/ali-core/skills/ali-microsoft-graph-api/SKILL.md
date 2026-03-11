---
name: ali-microsoft-graph-api
description: |
  Microsoft Graph API v1.0 and beta patterns for Aliunde projects. Use when:

  PLANNING: Designing SharePoint, OneDrive, Teams, or Outlook integrations;
  selecting MSAL authentication strategy; planning permission scope architecture;
  evaluating batch vs individual request trade-offs; designing webhook subscriptions

  IMPLEMENTATION: Writing Graph API clients with MSAL; implementing SharePoint
  checkout/checkin workflows; building pagination with nextLink/deltaLink; handling
  429 throttling; implementing $batch requests; registering webhook subscriptions

  GUIDANCE: Asking about Graph API scope selection; asking how to authenticate
  against SharePoint vs OneDrive; asking about throttling limits and retry patterns;
  asking about change notification patterns; asking about delegated vs application
  permissions

  REVIEW: Reviewing Graph API client code for security and resilience; checking
  MSAL scope-to-endpoint alignment; validating 429 handling strategy; auditing
  auth manager selection; checking pagination completeness; verifying webhook
  validation token handling

  Do NOT use for: Power Platform CoPilot Studio connectors (use ali-copilot-studio),
  Azure AD/Entra ID RBAC policy configuration (use ali-security-expert), or
  Graph PowerShell module usage (use ali-backend-developer)
---

# Microsoft Graph API

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing integrations with SharePoint Online, OneDrive, Teams, or Outlook
- Selecting MSAL authentication strategy (delegated vs application permissions)
- Architecting permission scopes for least-privilege access
- Evaluating batch vs individual API call trade-offs
- Planning webhook subscription and change notification handling

**Implementation:**
- Writing Python Graph API clients with MSAL authentication
- Implementing SharePoint document library operations (checkout, checkin, upload)
- Building pagination with nextLink and deltaLink patterns
- Handling 429 throttling with Retry-After and exponential backoff
- Implementing $batch requests for multi-resource operations
- Registering and renewing webhook subscriptions

**Guidance/Best Practices:**
- Asking which OAuth scope to use for a given endpoint
- Asking how to authenticate for SharePoint vs personal OneDrive
- Asking about Graph API throttling limits and retry strategies
- Asking about change notification patterns and deltaLink
- Asking how to handle co-authoring locks (423) vs explicit checkouts

**Review/Validation:**
- Reviewing Graph API client implementations for security
- Checking MSAL scope alignment with endpoints being called
- Validating 429 retry depth against integration SLA requirements
- Auditing auth manager selection (AzCliAuthManager vs MsalAuthManager)
- Verifying pagination loops handle multi-page results

## When NOT to Use This Skill

- Power Platform CoPilot Studio connector OAuth patterns (use ali-copilot-studio)
- Azure AD / Entra ID RBAC policy design (use ali-security-expert)
- Graph PowerShell module usage (use ali-backend-developer)
- SharePoint on-premises (Graph API covers SharePoint Online only)

---

## Key Principles

- **Scope-to-endpoint must match**: Files.ReadWrite is personal OneDrive only; SharePoint site drives require Sites.ReadWrite.All. Mismatch causes 403 that looks like auth failure
- **Delegated vs application is a security gate**: Delegated = user context (correct for user-facing tools). Application = tenant-wide daemon (never for per-user scoping)
- **AzCliAuthManager is dev only**: Production deployments require MsalAuthManager with Secrets Manager credentials
- **Retry-After is mandatory**: 429 responses carry a Retry-After header specifying wait duration. Ignoring it and using fixed sleep is a policy violation
- **Single retry vs exponential backoff depends on SLA**: Pipeline tools can retry once. Long-running services need exponential backoff with jitter across multiple attempts
- **deltaLink must be persisted**: Discard deltaLink = full enumeration on next sync. Store and reuse between cycles
- **$batch limit is 20**: Each batch envelope holds at most 20 requests. Response status must be checked per-item, not just on the envelope
- **Connection pooling matters at scale**: Module-level requests.request() creates a new TCP+TLS connection per call; use requests.Session for multi-request clients
- **423 does not expose lock type**: Graph API returns 423 for explicit checkout, browser co-authoring, and desktop sync locks alike. Code must not branch on assumed lock type

---

## Authentication

### Two Auth Managers

The reference implementation (ali-reqs/src/feedback/graph/auth.py) supports two auth managers with distinct deployment contexts:

**AzCliAuthManager (development only)**
- No app registration required
- Relies on local `az` CLI session (`az login`)
- Acquires token from Azure CLI token cache
- Not suitable for production, CI, or headless deployments
- Use this for local development and testing against real SharePoint

**MsalAuthManager (production / CI)**
- Requires Azure app registration with client ID
- Uses PublicClientApplication with device code or interactive flow
- Token cache persisted to disk; cache file must use 0o600 permissions with atomic rename
- Needs GRAPH_CLIENT_ID and GRAPH_TENANT_ID from Secrets Manager
- validate_authority=False is acceptable only for CI with known-good tenant IDs

```python
# Development: no app registration needed
auth = AzCliAuthManager()

# Production: requires app registration
auth = MsalAuthManager(
    client_id=get_secret("GRAPH_CLIENT_ID"),
    tenant_id=get_secret("GRAPH_TENANT_ID"),
)
```

### Delegated vs Application Permissions

**Delegated permissions** (user context):
- Token scoped to the authenticated user's access
- Correct for desktop tools, user-facing APIs, interactive sessions
- User's own SharePoint/OneDrive access enforced by Graph
- Reference implementation uses delegated flow (PublicClientApplication)

**Application permissions** (daemon / tenant-wide):
- Token grants access across the entire tenant
- Required for background jobs with no user context
- Must NEVER be used to impersonate per-user access
- Requires admin consent; client secret or certificate required

**Review gate**: When reviewing code, require explicit documentation of which permission type is used and why. Flag any case where application permissions are used for work that should be delegated.

### Token Cache Security

The correct pattern from the reference implementation (auth.py):

```python
# Write to temp, chmod, atomic rename — prevents corruption and race conditions
tmp_path = cache_path.with_suffix(".tmp")
tmp_path.write_text(serialized)
tmp_path.chmod(0o600)
tmp_path.rename(cache_path)
```

Anti-patterns to flag:
- Writing directly to cache file (no atomic rename — risk of corruption)
- Cache file with 0o644 or world-readable permissions
- Bearer token values appearing in log statements

### MSAL Token Acquisition Errors

MSAL error responses may include error_description containing user UPN or account details. This is acceptable in desktop tools writing to local logs. It is NOT acceptable in web services where RuntimeError may surface in HTTP responses or unfiltered error tracking (Sentry, Datadog).

```python
# Flag this pattern in web service contexts:
raise RuntimeError(f"MSAL token acquisition failed: {error} - {description}")
# error_description may contain "AADSTS70011: user@domain.com failed..."
```

---

## Scope Reference

### Scope-to-Endpoint Mapping

| Access Pattern | Correct Delegated Scope | Wrong Scope |
|----------------|------------------------|-------------|
| Personal OneDrive files | Files.ReadWrite | Sites.ReadWrite.All |
| SharePoint site drive (/sites/{id}/drive/...) | Sites.ReadWrite.All | Files.ReadWrite |
| SharePoint site drive (read-only) | Sites.Read.All | Files.Read |
| Mail read/write | Mail.ReadWrite | - |
| Mail send only | Mail.Send | - |
| Calendar read/write | Calendars.ReadWrite | - |
| Teams channel messages | ChannelMessage.Read.All | - |
| Teams channel list | Channel.ReadBasic.All | - |
| User profile | User.Read | - |
| All users (admin) | User.Read.All | - |

**The critical mismatch to catch**: Code using Files.ReadWrite scope but calling /sites/{site_id}/drive/items/... endpoints. MSAL silently acquires a token with the OneDrive scope and Graph returns 403 Forbidden, which appears to be a permission or auth failure rather than a scope mismatch.

### Common Scope Declarations

```python
# Personal OneDrive pipeline
OAUTH_SCOPES = [
    "Files.ReadWrite",
    "offline_access",
]

# SharePoint site integration
OAUTH_SCOPES = [
    "Sites.ReadWrite.All",
    "offline_access",
]

# Mail + Calendar integration
OAUTH_SCOPES = [
    "Mail.ReadWrite",
    "Calendars.ReadWrite",
    "offline_access",
]
```

---

## SharePoint Operations

### Site ID Resolution

SharePoint Graph paths require the composite site_id in format `hostname,site-guid,web-guid`. Resolve once and cache; it does not change.

```python
def resolve_site_id(auth_manager, site_url: str) -> str:
    """Resolve SharePoint site URL to Graph API site_id."""
    # Parse hostname and server-relative path from URL
    parsed = urlparse(site_url)
    hostname = parsed.hostname
    site_path = parsed.path  # e.g., /sites/DACommunityOfPractice

    url = f"https://graph.microsoft.com/v1.0/sites/{hostname}:{site_path}"
    response = _request("GET", url, auth_manager)
    return response["id"]  # "hostname,site-guid,web-guid"
```

### Drive Item Operations

All drive operations for SharePoint use site-scoped paths:

```
GET  /v1.0/sites/{site_id}/drive/items/{item_id}
PUT  /v1.0/sites/{site_id}/drive/items/{item_id}/content
POST /v1.0/sites/{site_id}/drive/items/{item_id}/checkout
POST /v1.0/sites/{site_id}/drive/items/{item_id}/checkin
```

### Checkout / Checkin Pattern

SharePoint checkout prevents concurrent edits. The reference implementation (client.py) demonstrates:

1. POST /checkout — acquire exclusive lock (returns 204 No Content on success)
2. GET /content — download file content
3. PUT /content — upload modified content
4. POST /checkin — release lock with version comment

**423 Locked response**: Graph API returns 423 for any of: explicit checkout by another user, browser co-authoring session, or desktop sync (OneDrive folder open in Word). The API does NOT differentiate lock types. The 423 body may contain `checkedOutBy` for explicit checkouts only.

```python
# Pattern from client.py — 423 handling
if response.status_code == 423:
    # Try to identify lock holder from metadata
    # checkedOutBy only populated for explicit checkouts, not co-authoring
    lock_holder = _extract_lock_holder(item_metadata)
    raise LockAcquisitionError(f"File locked by {lock_holder or 'unknown'}")
```

**Checkin returning 400**: Graph API may return 400 "file is not checked out" when another user holds the lock. This is a confusing API behavior — the file IS checked out, just not by the caller.

**Resource unit cost**: Checkout costs 2 resource units (vs 1 for standard GET). Under throttling pressure, loops that checkout many files exhaust quota faster than expected.

### Two-Step Operation Idempotency

Upload-then-checkin is a two-step sequence. If upload succeeds but checkin fails, the file is modified but has no version label. This partial failure must be explicitly documented as an acceptable design choice or guarded with retry logic.

The reference implementation (client.py:343-349) deliberately does not raise on checkin failure — acceptable for a pipeline tool but must be flagged in production web services.

---

## OneDrive Operations

OneDrive uses the /me/drive namespace for the authenticated user:

```
GET /v1.0/me/drive/root/children          # List root items
GET /v1.0/me/drive/items/{item_id}        # Get item metadata
GET /v1.0/me/drive/items/{item_id}/content # Download file
PUT /v1.0/me/drive/items/{item_id}/content # Upload/replace file
```

Upload resumable sessions for files over 4MB:

```python
# Create upload session
session_url = f"/v1.0/me/drive/items/{item_id}/createUploadSession"
response = requests.post(session_url, json={"item": {}})
upload_url = response.json()["uploadUrl"]  # Preauthenticated, 15-minute expiry

# Upload in chunks (recommended 5-10MB per chunk)
for chunk, start, end, total in chunks:
    requests.put(
        upload_url,
        data=chunk,
        headers={"Content-Range": f"bytes {start}-{end}/{total}"}
    )
```

---

## Teams API

Teams operations require different scopes depending on action:

```python
# Read channel messages
GET /v1.0/teams/{team_id}/channels/{channel_id}/messages

# Post message
POST /v1.0/teams/{team_id}/channels/{channel_id}/messages
# Body: {"body": {"content": "message text"}}

# List channels
GET /v1.0/teams/{team_id}/channels
```

**Scopes**: ChannelMessage.Read.All (read), ChannelMessage.Send (post), Channel.ReadBasic.All (list)

Teams API does not support $batch for all operations. Check endpoint-specific documentation before assuming batch compatibility.

---

## Outlook / Mail API

```python
# List messages with filter
GET /v1.0/me/messages?$filter=isRead eq false&$top=50

# Send message
POST /v1.0/me/sendMail
# Body: {"message": {"subject": "...", "body": {...}, "toRecipients": [...]}}

# Move to folder
POST /v1.0/me/messages/{id}/move
# Body: {"destinationId": "inbox"}
```

**Scopes**: Mail.Read (list), Mail.ReadWrite (modify/move), Mail.Send (send)

Mail API supports $batch for read operations. Send operations must be individual requests.

---

## Batch Requests

The $batch endpoint sends up to 20 requests in a single HTTP call:

```python
batch_url = "https://graph.microsoft.com/v1.0/$batch"

batch_body = {
    "requests": [
        {
            "id": "1",  # Unique string ID per request
            "method": "GET",
            "url": "/me/drive/items/abc123"
        },
        {
            "id": "2",
            "method": "GET",
            "url": "/sites/site_id/drive/items/def456"
        }
    ]
}

response = requests.post(batch_url, json=batch_body, headers=auth_headers)
batch_result = response.json()

# CRITICAL: HTTP 200 on the batch envelope does NOT mean all items succeeded
# Check each response individually
for item_response in batch_result["responses"]:
    if item_response["status"] != 200:
        logger.warning(
            "Batch item %s failed with status %s",
            item_response["id"],
            item_response["status"]
        )
```

**Limits**:
- Maximum 20 requests per $batch call
- Each request must have a unique `id` string
- Requests within a batch may run in parallel on the server; do not assume order
- Use `dependsOn` array to declare ordering dependencies between requests

**Batching opportunity**: Fetching metadata for N documents via N individual GET calls can be replaced with ceil(N/20) batch calls. Flag unbatched loops over 5+ identical GET requests as a recommendation.

---

## Pagination

### nextLink (Standard Pagination)

Graph API truncates large result sets and includes @odata.nextLink in the response. You MUST follow this until absent.

```python
def get_all_items(url: str, auth_manager) -> list[dict]:
    """Follow nextLink pagination until exhausted."""
    items = []
    next_url = url

    while next_url:
        response = _request("GET", next_url, auth_manager)
        items.extend(response.get("value", []))
        next_url = response.get("@odata.nextLink")  # None when last page

    return items
```

**Anti-pattern to flag**: Assuming first response contains all results. Any list endpoint can return a partial page.

### deltaLink (Change Tracking)

deltaLink enables incremental sync — only items changed since the last delta are returned.

```python
# First sync: use /delta to get initial set + deltaLink
initial_url = "/v1.0/me/drive/root/delta"

# Follow nextLink through all pages of initial sync
# The LAST page's response contains @odata.deltaLink (not nextLink)

# CRITICAL: Store deltaLink between runs
store_delta_link(delta_link_token)

# Subsequent syncs: use stored deltaLink
# Returns only items changed since that checkpoint
next_sync_url = load_delta_link()
response = _request("GET", next_sync_url, auth_manager)
```

**The critical mistake**: Discarding deltaLink after a sync. This forces a full enumeration on every run instead of incremental deltas. Flag any implementation that does not persist the deltaLink token to durable storage.

---

## Throttling and Error Handling

### Graph API Throttle Limits

- Per-user: 3,000 requests per 5-minute window
- Checkout operations: 2 resource units each (vs 1 for GET)
- Application-level: Microsoft may impose additional limits; Retry-After can be up to 300 seconds

### 429 Handling: Two Distinct Patterns

**Pattern 1: Retry-After honor (single retry)**

Correct for pipeline tools and scripts with low request volume:

```python
# From client.py — single retry pattern
if response.status_code == 429:
    retry_after_raw = response.headers.get("Retry-After")
    try:
        wait_seconds = int(retry_after_raw) if retry_after_raw else _DEFAULT_RETRY_SECONDS
    except ValueError:
        wait_seconds = _DEFAULT_RETRY_SECONDS

    logger.warning("Graph API throttled. Waiting %d seconds.", wait_seconds)
    time.sleep(wait_seconds)  # Acceptable in synchronous pipelines

    # Retry once
    response = _request(...)
    if response.status_code == 429:
        raise FeedbackCycleError("Rate limited after retry")
```

**Pattern 2: Exponential backoff with jitter (multi-retry)**

Required for FastAPI services, long-running jobs, and sustained high-volume integrations:

```python
import asyncio
import random

async def graph_request_with_backoff(
    method: str,
    url: str,
    max_retries: int = 5,
    base_delay: float = 1.0,
) -> dict:
    for attempt in range(max_retries):
        response = await async_client.request(method, url)

        if response.status_code == 429:
            # Always honor Retry-After if present
            retry_after = response.headers.get("Retry-After")
            if retry_after:
                wait = int(retry_after)
            else:
                # Exponential backoff with jitter
                wait = min(base_delay * (2 ** attempt), 300.0)
                wait *= (0.5 + random.random())

            await asyncio.sleep(wait)
            continue

        response.raise_for_status()
        return response.json()

    raise RuntimeError(f"Graph API rate limited after {max_retries} attempts")
```

**Review gate**: For every 429 handler, ask:
1. Does it read Retry-After header? (If not, BLOCKING)
2. Is a single retry sufficient for the integration's SLA? (If not, flag as warning)
3. Does the handler use time.sleep() in an async context? (If yes, BLOCKING — use asyncio.sleep)

### 401 Token Refresh

A 401 may mean the token expired mid-session. Retry once with a fresh token (not from cache):

```python
if response.status_code == 401 and retry_count < _MAX_AUTH_RETRIES:
    # Force fresh token acquisition — do NOT use cached token
    token = auth_manager.acquire_token(force_refresh=True)
    headers["Authorization"] = f"Bearer {token}"
    # Retry the request
```

### Structured Error Hierarchy

Prefer typed exceptions over bare RuntimeError for caller-distinguishable failures:

```python
# Good — callers can catch specific errors
class GraphApiError(Exception): pass
class DownloadError(GraphApiError): pass
class UploadError(GraphApiError): pass
class LockAcquisitionError(GraphApiError): pass
class ThrottleError(GraphApiError): pass

# Bad — callers must parse error message strings
raise RuntimeError("Upload failed: 403 Forbidden")
```

---

## Webhooks / Change Notifications

### Subscription Creation

```python
subscription_body = {
    "changeType": "created,updated,deleted",
    "notificationUrl": "https://your-app.com/api/graph/notifications",
    "resource": "/me/drive/root",
    "expirationDateTime": (datetime.utcnow() + timedelta(hours=72)).isoformat() + "Z",
    "clientState": secrets.token_urlsafe(32),  # Store for verification
}

response = requests.post(
    "https://graph.microsoft.com/v1.0/subscriptions",
    json=subscription_body,
    headers=auth_headers,
)
```

### Validation Token Handshake

On subscription creation, Graph sends a GET request to your notificationUrl with a `validationToken` query parameter. Your endpoint must echo it back with 200 OK within 10 seconds:

```python
from fastapi import Request

@router.get("/api/graph/notifications")
async def validate_subscription(validationToken: str = None):
    if validationToken:
        # Echo token back as plain text — MUST respond within 10 seconds
        return Response(content=validationToken, media_type="text/plain")
```

### Processing Notifications

```python
@router.post("/api/graph/notifications")
async def handle_notification(request: Request):
    body = await request.json()

    for notification in body.get("value", []):
        # Verify clientState matches what was set at subscription creation
        if notification.get("clientState") != STORED_CLIENT_STATE:
            logger.warning("Notification clientState mismatch — possible spoofing")
            continue

        # Process change notification
        resource_id = notification["resourceData"]["id"]
        change_type = notification["changeType"]
        await process_change(resource_id, change_type)

    return Response(status_code=202)
```

### Subscription Renewal

Graph subscriptions expire (max 4,230 minutes for OneDrive/SharePoint). Renew before expiry:

```python
# PATCH /v1.0/subscriptions/{subscription_id}
# Body: {"expirationDateTime": new_expiry_iso}
```

---

## HTTP Client Patterns

### Connection Pooling (requests.Session)

Module-level requests.request() creates a new TCP+TLS connection per call. Use requests.Session for clients that make multiple calls to the same host:

```python
class GraphApiClient:
    def __init__(self, auth_manager):
        self._auth_manager = auth_manager
        self._session = requests.Session()  # Reuses connections

    def _request(self, method: str, url: str, **kwargs) -> requests.Response:
        token = self._auth_manager.acquire_token()
        self._session.headers.update({"Authorization": f"Bearer {token}"})
        return self._session.request(
            method,
            url,
            timeout=_REQUEST_TIMEOUT_SECONDS,
            **kwargs,
        )
```

### Async Clients (FastAPI Context)

Synchronous requests.Session blocks the event loop. In FastAPI async handlers, use httpx.AsyncClient:

```python
import httpx

async def fetch_drive_item(item_id: str, token: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"https://graph.microsoft.com/v1.0/me/drive/items/{item_id}",
            headers={"Authorization": f"Bearer {token}"},
            timeout=30.0,
        )
        response.raise_for_status()
        return response.json()
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Files.ReadWrite for SharePoint endpoints | 403 Forbidden that looks like auth failure | Use Sites.ReadWrite.All for /sites/{id}/drive/... paths |
| AzCliAuthManager in production | Requires local az CLI session; fails in CI/headless | MsalAuthManager with Secrets Manager credentials |
| Application permissions for per-user work | Grants tenant-wide access; violates least privilege | Delegated permissions with user context |
| Fixed sleep on 429 (ignoring Retry-After) | May under-wait (triggers more 429s) or over-wait unnecessarily | Read Retry-After header; use that duration |
| time.sleep() in async context | Blocks the event loop | asyncio.sleep() |
| Single 429 retry for long-running services | Insufficient for sustained throttling | Exponential backoff with jitter, multiple retries |
| Discarding deltaLink after sync | Forces full enumeration every run | Persist deltaLink to durable storage |
| Checking only $batch envelope status | Partial failures silently ignored | Check status per item in batch response |
| $batch with >20 requests | Graph API rejects the request | Split into ceil(N/20) batch calls |
| Token value in log statements | Credential leakage in log aggregators | Log token presence/absence only, never value |
| validate_authority=False in multi-tenant code | Misconfigured tenant ID not caught at startup | Only use in CI with known-good tenant IDs |
| requests.request() in multi-call clients | New TCP+TLS connection per request | requests.Session stored on client instance |
| 401 retry using cached token | Stale token causes second 401 | Force fresh token acquisition on retry |

---

## Quick Reference

### Base URL and Common Paths

```
Base:   https://graph.microsoft.com/v1.0

# Personal OneDrive
/me/drive/root
/me/drive/items/{item_id}/content

# SharePoint site drive
/sites/{site_id}/drive/items/{item_id}/content
/sites/{site_id}/drive/items/{item_id}/checkout
/sites/{site_id}/drive/items/{item_id}/checkin

# Mail
/me/messages
/me/sendMail

# Teams
/teams/{team_id}/channels/{channel_id}/messages

# Batch
/$batch

# Webhooks
/subscriptions
```

### Resolve Site ID

```bash
# Using az CLI (development)
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/sites/hostname.sharepoint.com:/sites/SiteName"
```

### HTTP Status Reference

| Status | Meaning in Graph API |
|--------|---------------------|
| 200 | Success with body |
| 204 | Success, no body (checkout, delete) |
| 400 | Bad request; also: "file not checked out by you" (confusing) |
| 401 | Token expired or invalid — retry with fresh token |
| 403 | Insufficient permissions — check scope-to-endpoint alignment |
| 404 | Item not found |
| 409 | Conflict (item already checked out by caller) |
| 423 | Locked (checkout, co-authoring, or desktop sync) |
| 429 | Throttled — read Retry-After header |
| 503 | Service unavailable — retry with backoff |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Reference Graph API client | projects/ali-reqs/src/feedback/graph/client.py |
| MSAL auth managers | projects/ali-reqs/src/feedback/graph/auth.py |
| Graph API Protocol (interface) | projects/ali-reqs/src/feedback/graph/protocol.py |
| Pipeline orchestrator | projects/ali-reqs/src/feedback/pipeline/runner.py |
| Graph API return code research | projects/ali-reqs/.tmp/graph-api-checkout-return-codes.md |

---

## References

- [Microsoft Graph API Overview](https://docs.microsoft.com/en-us/graph/overview)
- [Graph API v1.0 Reference](https://docs.microsoft.com/en-us/graph/api/overview)
- [MSAL Python Documentation](https://msal-python.readthedocs.io/)
- [Graph API Throttling](https://docs.microsoft.com/en-us/graph/throttling)
- [Change Notifications](https://docs.microsoft.com/en-us/graph/webhooks)
- [Batch Requests](https://docs.microsoft.com/en-us/graph/json-batching)
- [Delta Queries](https://docs.microsoft.com/en-us/graph/delta-query-overview)

---

**Last Updated:** 2026-02-20
