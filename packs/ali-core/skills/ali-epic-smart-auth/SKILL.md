---
name: ali-epic-smart-auth
description: |
  Epic's SMART on FHIR OAuth 2.0 implementation, CDS Hooks, and FHIRcast patterns.
  Use when:

  PLANNING: Designing SMART EHR launch or standalone launch flows for Epic, architecting
  backend service authorization, planning CDS Hooks integration, designing FHIRcast
  context synchronization

  IMPLEMENTATION: Writing Epic OAuth 2.0 authorization code flows, implementing backend
  service JWT assertion auth, handling Epic token refresh, building CDS Hooks services,
  subscribing to FHIRcast topics in Epic

  GUIDANCE: Asking about Epic SMART scope requirements, Epic OAuth client registration,
  which SMART scopes Epic supports, how Epic CDS Hooks differ from the base spec

  REVIEW: Auditing SMART authorization implementations against Epic requirements,
  checking scope configuration, validating token refresh handling, reviewing CDS
  Hooks response formats for Epic compatibility
---

# Epic SMART on FHIR, CDS Hooks, and FHIRcast

## Overview

Epic's authorization layer is built on SMART on FHIR (OAuth 2.0 with HL7 extensions). All patient-context FHIR API calls require a valid SMART token. System-to-system integrations use backend service authorization with asymmetric JWT assertions. This skill covers the Epic-specific behavior that differs from the base SMART on FHIR specification.

---

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing a SMART EHR launch flow for an Epic-embedded app
- Designing a standalone launch for a patient-facing or provider-facing app outside Epic
- Architecting backend service (system-to-system) authorization for Epic FHIR calls
- Planning token storage, refresh, and session management for Epic integrations
- Designing a CDS Hooks service for Epic clinical decision support
- Planning FHIRcast topic subscriptions for context synchronization in Epic

**Implementation:**
- Writing OAuth 2.0 authorization code exchange for Epic
- Implementing JWT assertion generation for Epic backend services
- Handling Epic token responses (access token, patient context, encounter context)
- Building a CDS Hooks service that handles Epic's hook invocation format
- Subscribing to and receiving FHIRcast events from Epic

**Guidance/Best Practices:**
- Asking what SMART scopes are required for specific Epic FHIR resources
- Asking the difference between Epic confidential and public clients
- Asking how Epic's CDS Hooks implementation differs from the specification
- Asking about Epic App Orchard or Showroom registration requirements
- Asking about FHIRcast hub configuration in Epic

**Review/Validation:**
- Reviewing OAuth flows for Epic-specific compliance gaps
- Checking scope configuration against Epic's supported scope list
- Validating token refresh handling for long-running sessions
- Auditing CDS Hook response format for Epic compatibility
- Checking FHIRcast subscription handling

## When NOT to Use This Skill

- For general OAuth 2.0 patterns not specific to Epic — use `ali-secure-coding`
- For Epic FHIR resource-level questions (Patient, Observation, etc.) — use `ali-epic-fhir-r4`
- For base FHIR R4 specification questions — use `ali-fhir-r4-core`
- For Epic HL7 v2 interface questions — use `ali-epic-hl7v2`

---

## Key Principles

- All Epic FHIR API calls require a valid OAuth 2.0 bearer token — there is no unauthenticated FHIR access in Epic production
- Epic enforces scopes at the resource level — requesting `patient/*.read` does not guarantee access to every resource; Epic evaluates each resource type against its configured scope list
- Confidential clients use client_secret (for server-side apps); public clients use PKCE (for native/mobile apps) — Epic does not support client_secret flows for public clients
- Backend service auth uses asymmetric key pairs (RS384 or ES384 JWT assertion) — symmetric secrets are not supported for system-to-system
- Token expiry is short in Epic (typically 3600 seconds for access tokens) — always implement token refresh; never cache indefinitely
- Epic's CDS Hooks implementation follows the base spec closely but has Epic-specific context extension fields — always check the Epic CDS Hooks documentation for the specific hook type
- FHIRcast requires app registration in Epic's CDS configuration — apps cannot self-register at runtime
- App Orchard (production) and the Epic FHIR sandbox are separate environments with separate client registrations — do not reuse credentials between them

---

## Epic OAuth 2.0 Client Types

### Confidential Client

- Server-side applications (web apps with a backend, backend services)
- Credentials: `client_id` + `client_secret`
- Client secret is registered in Epic and presented at token exchange
- Used for: EHR launch apps with server-side backends, backend service flows (with JWT instead of secret for system-to-system)

**Epic registration note:** Confidential client secrets are managed in Epic's app configuration (Hyperdrive or UserWeb App Orchard); rotating a secret requires re-publishing the app or updating the configuration.

### Public Client

- Native mobile apps, single-page applications (no server-side backend)
- Credentials: `client_id` only; no client_secret
- Uses PKCE (Proof Key for Code Exchange) to prevent authorization code interception
- `code_verifier` and `code_challenge` must be generated per session — Epic validates the PKCE pair at token exchange

---

## SMART Launch Flows

### EHR Launch (App Launched from Inside Epic)

The EHR launch flow is used when Epic initiates the app launch from within the clinical workflow (Hyperdrive, MyChart provider portal, or embedded within an Epic app).

**Flow:**
1. Epic sends the app a launch URL with `launch` and `iss` parameters:
   ```
   https://your-app.example.com/launch?launch=abc123&iss=https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4
   ```
2. App exchanges `launch` token for authorization:
   ```
   GET https://fhir.epic.com/interconnect-fhir-oauth/oauth2/authorize
     ?response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=https://your-app.example.com/callback
     &scope=launch openid fhirUser patient/Patient.read
     &state=RANDOM_STATE
     &launch=abc123
     &aud=https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4
   ```
3. Epic authenticates the user and returns an authorization code to `redirect_uri`
4. App exchanges code for access token:
   ```
   POST https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token
   Content-Type: application/x-www-form-urlencoded

   grant_type=authorization_code
   &code=AUTH_CODE
   &redirect_uri=https://your-app.example.com/callback
   &client_id=YOUR_CLIENT_ID
   &client_secret=YOUR_CLIENT_SECRET  (confidential clients only)
   &code_verifier=PKCE_VERIFIER       (public clients only)
   ```
5. Token response includes `access_token`, `patient` (patient FHIR ID), `encounter` (if in encounter context)

**Epic EHR launch key points:**
- The `iss` parameter identifies the Epic FHIR base URL — always validate this matches an expected Epic instance; this is a security check
- The `launch` token is single-use and short-lived (minutes)
- Epic does not always include `encounter` context even if the user is in an encounter — apps must handle its absence

### Standalone Launch (App Initiated Outside Epic)

Used when the app starts from outside Epic (patient portal integration, provider workflow tool, data analytics app).

**Flow:**
1. App discovers the Epic authorization endpoint via SMART Well-Known:
   ```
   GET https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4/.well-known/smart-configuration
   ```
2. App builds the authorization URL without a `launch` parameter:
   ```
   GET https://fhir.epic.com/interconnect-fhir-oauth/oauth2/authorize
     ?response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=https://your-app.example.com/callback
     &scope=openid fhirUser patient/Patient.read
     &state=RANDOM_STATE
     &aud=https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4
   ```
3. Epic presents login; user authenticates
4. Authorization code exchange is the same as EHR launch step 4
5. Token response will NOT include `patient` context unless the user selects a patient — handle this case

### Backend Service Authorization (System-to-System)

For services that need FHIR access without a user present (batch jobs, analytics pipelines, integrations).

**Flow (Client Credentials with JWT Assertion):**
1. Generate a JWT signed with your private key:
   ```json
   Header: { "alg": "RS384", "typ": "JWT" }
   Payload: {
     "iss": "YOUR_CLIENT_ID",
     "sub": "YOUR_CLIENT_ID",
     "aud": "https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token",
     "jti": "UNIQUE_UUID",
     "exp": CURRENT_TIME + 300
   }
   ```
2. Exchange JWT for access token:
   ```
   POST https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials
   &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
   &client_assertion=SIGNED_JWT
   ```
3. Token response: `access_token`, `token_type=bearer`, `expires_in`

**Epic backend service key points:**
- Public key must be registered in Epic before the first token request — self-registration at runtime is not supported
- `jti` (JWT ID) must be unique per request — Epic rejects replayed JTIs within the expiry window
- `exp` must be within 5 minutes of the server time — clock skew failures are common on first deployment; verify NTP synchronization
- Backend service tokens are not patient-scoped; they have system-level scopes (e.g., `system/Patient.read`)

---

## SMART Scopes for Epic FHIR Resources

Epic enforces SMART scopes at the resource type level. The scope format follows the SMART on FHIR specification with some Epic-specific nuances.

**Scope format:**
```
{context}/{resource}.{permission}
```

| Context | When Used |
|---------|-----------|
| `patient` | EHR launch with patient context; access limited to the launched patient |
| `user` | Access to resources the authenticated user can access |
| `system` | Backend service; access controlled by Epic app configuration |

**Permission values Epic supports:**

| Permission | Meaning |
|------------|---------|
| `read` | GET (read single resource, search) |
| `write` | POST, PUT, PATCH |
| `*` | Read and write (use with caution; Epic may not support for all resources) |

**Common scope patterns:**
```
patient/Patient.read          - Read the launched patient's demographics
patient/Observation.read      - Read observations for the launched patient
patient/MedicationRequest.read - Read medication requests
user/Patient.read             - Read patients the user has access to
system/Patient.read           - Backend service read of patient data
openid fhirUser               - OpenID Connect + FHIR user identity claim
launch                        - Required for EHR launch flow
launch/patient                - Request patient context in the launch
launch/encounter              - Request encounter context in the launch
```

**Epic-specific scope limitations:**
- Not all FHIR resource types support all permission levels — check open.epic.com for the specific resource
- `offline_access` scope is supported for refresh token issuance — omit it if refresh tokens are not needed
- Epic validates that requested scopes match the scopes registered for the client in App Orchard — requesting unregistered scopes results in an authorization error, not a narrowed token

---

## Token Refresh

Epic access tokens expire (typically 3600 seconds). Request `offline_access` scope to receive a refresh token.

**Refresh flow:**
```
POST https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=REFRESH_TOKEN
&client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET  (confidential clients)
```

**Key behaviors:**
- Epic refresh tokens have a longer but still finite lifetime — do not assume they are permanent
- Refresh tokens are single-use in some Epic configurations — always replace the stored refresh token with the new one returned in the refresh response
- Background services should refresh proactively (e.g., when token is within 5 minutes of expiry) rather than reactively on 401 responses
- If the refresh token has expired, the user must re-authenticate through the full authorization flow

---

## Epic CDS Hooks Implementation

CDS Hooks is a standard for embedding clinical decision support into EHR workflows. Epic's implementation follows the base specification with specific configuration requirements.

### Supported Hooks in Epic

Epic supports a subset of the CDS Hooks specification. Commonly supported hooks include:

| Hook | When It Fires |
|------|--------------|
| `patient-view` | When a provider opens a patient chart |
| `order-select` | When a provider selects an order to place |
| `order-sign` | When a provider signs an order |
| `encounter-discharge` | When a discharge is being processed |
| `appointment-book` | When an appointment is being scheduled |

**Epic-specific behavior:** Not all hooks listed in the CDS Hooks specification are available in all Epic versions. Check the Epic release notes for the specific hooks available in the target Epic version.

### CDS Service Response Format

Epic expects a valid CDS Hooks response. Epic-specific considerations:

```json
{
  "cards": [
    {
      "summary": "Brief summary (140 char max in Epic display)",
      "detail": "Longer markdown detail text",
      "indicator": "info",
      "source": {
        "label": "Your Service Name",
        "url": "https://your-service.example.com"
      },
      "suggestions": [
        {
          "label": "Apply this suggestion",
          "actions": [
            {
              "type": "create",
              "description": "Create a MedicationRequest",
              "resource": { ... FHIR resource ... }
            }
          ]
        }
      ],
      "links": [
        {
          "label": "Learn more",
          "url": "https://your-app.example.com/launch",
          "type": "smart",
          "appContext": "{\"key\":\"value\"}"
        }
      ]
    }
  ]
}
```

**Epic CDS Hooks key points:**
- `indicator` values: `info`, `warning`, `critical` — Epic renders each differently in the UI
- `summary` is the headline shown in the card — keep under 140 characters for full display
- SMART app links (type `"smart"`) allow a CDS card to launch a SMART app with context from the hook
- Epic ignores cards with invalid structure silently — errors are not surfaced to the user; use Epic's CDS test harness to validate
- Response time matters: Epic has a configurable timeout (typically 5 seconds) after which it discards the CDS response and shows nothing — optimize for low latency

### CDS Service Registration in Epic

CDS services must be registered in Epic's CDS configuration (not self-registered via the CDS discovery endpoint). The service URL, supported hooks, and security settings are configured by Epic administrators.

---

## FHIRcast Context Synchronization

FHIRcast is a standard for sharing context (open patient, open encounter) across clinical apps within the same EHR session.

**Use case:** A third-party app embedded in Epic's Hyperdrive browser needs to know when the provider navigates to a different patient without requiring a full page reload.

### Epic FHIRcast Configuration

Epic acts as the FHIRcast hub. Apps subscribe to context topics when they launch (typically via the SMART launch context which includes the hub URL and topic).

**Context keys in the SMART launch token:**
```
smart_style_url       - Style URL (optional)
need_patient_banner   - Whether app should show patient banner
hub.url               - FHIRcast hub URL
hub.topic             - FHIRcast topic ID for this session
```

### Subscribing to FHIRcast Events

```
POST {hub.url}
Content-Type: application/x-www-form-urlencoded

hub.channel.type=websub
&hub.mode=subscribe
&hub.topic={hub.topic}
&hub.events=Patient-open,Encounter-open,Patient-close
&hub.callback=https://your-app.example.com/fhircast-callback
&hub.secret=SHARED_SECRET
```

**Epic FHIRcast key points:**
- Epic supports WebSub (HTTP callback) and WebSocket subscription models
- Subscription requires the `hub.topic` value from the SMART launch — this is session-scoped, not reusable
- Epic fires `Patient-open` when the provider opens a patient chart, `Patient-close` when leaving it
- Apps must validate the hub signature (`X-Hub-Signature` header) on incoming notifications — Epic signs with the `hub.secret` provided at subscription time

---

## Epic App Orchard and Showroom Registration

**App Orchard** is Epic's app marketplace for production applications. Apps must pass Epic's review process before being listed.

**Epic Showroom** (also called Connection Hub) is the newer branding for the same marketplace infrastructure.

**Registration requirements:**
- OAuth client credentials provisioned per Epic customer instance — there is no single global credential
- Apps register their redirect URIs, scopes, and public keys (for backend services) at registration time
- Production apps must complete Epic's App Orchard review (security review, FHIR conformance testing)
- Sandbox testing uses the Epic FHIR sandbox at `fhir.epic.com` with separate sandbox credentials

**Epic sandbox:**
- Available at `https://fhir.epic.com` without customer-specific credentials for public API testing
- Uses synthetic patients — test patient IDs are documented on fhir.epic.com
- Sandbox credentials differ from production credentials — registrations are separate

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| Requesting `patient/*.read` and assuming it grants access to all resource types | Epic evaluates scopes per resource type; `patient/*.read` may not include resources that require explicit scope registration | Request specific resource scopes; test each resource type in the sandbox; review the open.epic.com scope table |
| Caching access tokens past their expiry | Epic tokens expire (default 3600s); using an expired token results in 401 errors | Store token expiry time; refresh proactively before expiry |
| Not handling the case where `patient` is absent from the token response | Standalone launch does not guarantee patient context; assuming it is always present causes NullPointerException-class failures | Check for `patient` in the token response before using it; handle the no-patient-context case explicitly |
| Using the same `jti` value across backend service JWT assertions | Epic rejects replayed JTIs within the expiry window | Generate a new UUID for `jti` on every token request |
| Using client_secret for public clients | Epic does not support client_secret for native/mobile apps | Use PKCE for public clients; generate `code_verifier` and `code_challenge` per session |
| Assuming refresh tokens are permanent | Epic refresh tokens have a finite lifetime — duration is instance-configurable | Handle refresh token expiry by redirecting the user to re-authenticate |
| Not validating the `iss` parameter in EHR launch | A malicious redirect could supply a fake `iss` pointing to an attacker's OAuth server | Validate `iss` against a list of known Epic instance URLs before initiating the authorization flow |
| Building CDS Hook responses that take more than 5 seconds | Epic discards CDS responses past its timeout — the card is never shown | Cache relevant data; pre-compute recommendations; keep CDS service response time under 2 seconds |

---

## Quick Reference

### Well-Known SMART Configuration URL

```
GET {FHIR_BASE_URL}/.well-known/smart-configuration
```

### Epic Sandbox FHIR Base URL

```
https://fhir.epic.com/interconnect-fhir-oauth/api/FHIR/R4
```

### Token Endpoint (Sandbox)

```
https://fhir.epic.com/interconnect-fhir-oauth/oauth2/token
```

### Authorization Endpoint (Sandbox)

```
https://fhir.epic.com/interconnect-fhir-oauth/oauth2/authorize
```

### JWT Assertion Algorithm Support

```
RS384 (RSA-SHA384) — most common
ES384 (ECDSA with P-384) — supported on newer Epic builds
```

### Scope Quick List

```
launch                     - EHR launch (required for EHR launch flow)
launch/patient             - Request patient context
launch/encounter           - Request encounter context
openid                     - OpenID Connect identity
fhirUser                   - FHIR identity resource claim
offline_access             - Refresh token
patient/{Resource}.read    - Patient-level read
user/{Resource}.read       - User-level read
system/{Resource}.read     - System-level read (backend service)
```

---

## References

- [SMART on FHIR App Launch Framework](https://hl7.org/fhir/smart-app-launch/)
- [Epic SMART on FHIR Documentation (open.epic.com)](https://open.epic.com/Interface/FHIR)
- [CDS Hooks Specification](https://cds-hooks.hl7.org/)
- [FHIRcast Specification](https://fhircast.org/)
- [Epic App Orchard](https://appmarket.epic.com/)
- [Epic FHIR Sandbox](https://fhir.epic.com)
