---
name: ali-secure-coding
description: |
  Secure coding practices, compliance patterns, and data privacy regulations (GDPR, CCPA) for Aliunde projects. Use when:

  PLANNING: Designing features involving authentication, authorization, sensitive data,
  external APIs, regulatory compliance, data privacy requirements, GDPR/CCPA obligations,
  cross-border data flows, or privacy impact assessments; considering security requirements;
  evaluating security trade-offs between approaches; planning data residency strategy

  IMPLEMENTATION: Writing code that handles user input, authentication, PII/PHI/SSN/tax data,
  secrets, encryption, API endpoints, file uploads, webhooks, or audit logging; implementing
  right-to-erasure workflows; building data retention enforcement; tagging data with
  classification metadata; implementing DSARs or consent management

  GUIDANCE: Asking about security best practices, OWASP Top 10, how others handle security,
  what's standard for compliance (WISP, HIPAA, IRS Pub 4557, GDPR, CCPA); asking about
  data subject rights, lawful basis, privacy by design, data minimization, or retention
  schedules

  REVIEW: Checking if code is secure, reviewing for vulnerabilities, validating compliance
  requirements, auditing authentication flows, checking for injection risks, evaluating
  GDPR/CCPA compliance posture, reviewing PII classification coverage, checking data
  residency enforcement
---

# Secure Coding

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing authentication or authorization flows
- Planning features that handle PII, PHI, SSN, or tax data
- Architecting API endpoints or external integrations
- Considering encryption requirements (at-rest, in-transit, field-level)
- Evaluating compliance requirements for a feature
- Planning GDPR/CCPA compliance posture or data privacy obligations
- Designing data residency or cross-border data transfer controls
- Scoping a Data Privacy Impact Assessment (DPIA)

**Implementation:**
- Writing code that accepts user input
- Implementing login, session management, or password handling
- Building API endpoints (especially public-facing)
- Handling file uploads or downloads
- Integrating with external services (webhooks, OAuth)
- Writing audit logging or access control logic
- Implementing right-to-erasure across data layers
- Building data retention enforcement or deletion workflows

**Guidance/Best Practices:**
- Asking about OWASP Top 10 vulnerabilities
- Asking how to securely handle sensitive data
- Asking what's required for WISP, HIPAA, IRS, GDPR, or CCPA compliance
- Asking about secrets management or encryption patterns
- Asking about lawful basis, consent, data subject rights, or privacy by design

**Review/Validation:**
- Reviewing code for security vulnerabilities
- Checking if input validation is sufficient
- Validating compliance with regulatory requirements
- Auditing authentication or authorization logic
- Checking GDPR/CCPA compliance posture or PII classification coverage

---

## Key Principles

- **Defense in depth**: Multiple layers of security; don't rely on a single control
- **Least privilege**: Grant minimum permissions needed; deny by default
- **Fail secure**: On error, deny access rather than allow
- **Input validation everywhere**: All external input is untrusted (users, APIs, files)
- **Secrets never in code**: Environment variables or secrets manager only
- **Audit everything sensitive**: Who accessed what, when, and why
- **Encrypt sensitive data**: At-rest (database) and in-transit (HTTPS/TLS)
- **Immutable audit logs**: Append-only; never delete or modify
- **7-year retention**: Tax data and audit logs per IRS requirements
- **Privacy by design**: Embed privacy into system design; default to maximum privacy
- **Data minimization**: Collect only what is necessary; do not retain beyond need

---

## Patterns We Use

### Input Validation Pattern (Python)

Validate all external input at system boundaries. Use Pydantic for structured validation:

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional
import re

class ClientInput(BaseModel):
    """Validates client creation input."""
    email: str = Field(..., max_length=255)
    ssn: Optional[str] = Field(None, min_length=9, max_length=11)
    phone: Optional[str] = Field(None, max_length=20)

    @field_validator('email')
    @classmethod
    def validate_email(cls, v: str) -> str:
        # Simple validation - let email provider validate deliverability
        if not re.match(r'^[^@]+@[^@]+\.[^@]+$', v):
            raise ValueError('Invalid email format')
        return v.lower().strip()

    @field_validator('ssn')
    @classmethod
    def validate_ssn(cls, v: Optional[str]) -> Optional[str]:
        if v is None:
            return None
        # Strip formatting, validate digits only
        digits = re.sub(r'\D', '', v)
        if len(digits) != 9:
            raise ValueError('SSN must be 9 digits')
        # Return masked for logging safety
        return digits
```

### SQL Injection Prevention

Always use parameterized queries. Never concatenate user input into SQL:

```python
# CORRECT - Parameterized query
cursor.execute(
    "SELECT * FROM clients WHERE email = %s AND status = %s",
    (email, status)
)

# CORRECT - SQLAlchemy ORM (automatically parameterized)
client = session.query(Client).filter(
    Client.email == email,
    Client.status == status
).first()

# WRONG - SQL injection vulnerability
cursor.execute(f"SELECT * FROM clients WHERE email = '{email}'")  # NEVER DO THIS
```

### Snowflake Query Safety

For Snowflake SQL generation (ali-ai-ingestion-engine), identifiers need quoting:

```python
def safe_identifier(name: str) -> str:
    """Quote identifier to prevent SQL injection in DDL/DML generation."""
    # Snowflake identifiers: alphanumeric and underscore only
    if not re.match(r'^[A-Za-z_][A-Za-z0-9_]*$', name):
        raise ValueError(f"Invalid identifier: {name}")
    return f'"{name.upper()}"'

# Use for table/column names in generated SQL
table_ref = f"{safe_identifier(schema)}.{safe_identifier(table)}"
```

### Authentication Pattern (FastAPI + JWT)

```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from jose import JWTError, jwt

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    """Extract and validate user from JWT token."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        # Secret from environment, never hardcoded
        payload = jwt.decode(
            token,
            os.environ["JWT_SECRET_KEY"],
            algorithms=["HS256"]
        )
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await get_user_by_id(user_id)
    if user is None:
        raise credentials_exception
    return user
```

### Secrets Management Pattern

Secrets come from environment variables or AWS Secrets Manager:

```python
import os
from functools import lru_cache

@lru_cache
def get_secret(secret_name: str) -> str:
    """Get secret from environment or AWS Secrets Manager."""
    # First check environment (for local dev)
    env_value = os.environ.get(secret_name)
    if env_value:
        return env_value

    # Production: fetch from AWS Secrets Manager
    import boto3
    client = boto3.client('secretsmanager')
    response = client.get_secret_value(SecretId=secret_name)
    return response['SecretString']

# Usage
db_password = get_secret("DB_PASSWORD")
stripe_key = get_secret("STRIPE_SECRET_KEY")
```

### PII Masking Pattern

Mask sensitive data in logs and non-essential displays:

```python
def mask_ssn(ssn: str) -> str:
    """Mask SSN for display/logging: XXX-XX-1234"""
    if not ssn or len(ssn) < 4:
        return "***"
    return f"XXX-XX-{ssn[-4:]}"

def mask_email(email: str) -> str:
    """Mask email for logging: j***@example.com"""
    if not email or '@' not in email:
        return "***"
    local, domain = email.split('@', 1)
    if len(local) <= 1:
        return f"*@{domain}"
    return f"{local[0]}***@{domain}"

def mask_phone(phone: str) -> str:
    """Mask phone for display: (***) ***-1234"""
    digits = re.sub(r'\D', '', phone)
    if len(digits) < 4:
        return "***"
    return f"(***) ***-{digits[-4:]}"
```

### Audit Logging Pattern

Immutable, structured audit logs for compliance:

```python
import json
import logging
from datetime import datetime, UTC
from typing import Any

audit_logger = logging.getLogger("audit")

def audit_log(
    action: str,
    user_id: str,
    resource_type: str,
    resource_id: str,
    details: dict[str, Any] = None,
    outcome: str = "success"
) -> None:
    """Write immutable audit log entry.

    Never log PII values - only IDs and masked references.
    """
    entry = {
        "timestamp": datetime.now(UTC).isoformat(),
        "action": action,
        "user_id": user_id,
        "resource_type": resource_type,
        "resource_id": resource_id,
        "outcome": outcome,
        "details": details or {}
    }
    # Audit logs go to separate, append-only destination
    audit_logger.info(json.dumps(entry))

# Usage
audit_log(
    action="client.view",
    user_id=current_user.id,
    resource_type="client",
    resource_id=client.id,
    details={"fields_accessed": ["name", "email", "ssn_masked"]}
)
```

### Webhook Signature Verification

Always verify webhook signatures before processing:

```python
import hmac
import hashlib

def verify_stripe_signature(payload: bytes, signature: str, secret: str) -> bool:
    """Verify Stripe webhook signature."""
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

def verify_generic_hmac(payload: bytes, signature: str, secret: str) -> bool:
    """Verify generic HMAC-SHA256 webhook signature."""
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    # Use compare_digest to prevent timing attacks
    return hmac.compare_digest(expected, signature)
```

### Secure File Upload Pattern

```python
import magic
from pathlib import Path

ALLOWED_TYPES = {
    'application/pdf': '.pdf',
    'image/jpeg': '.jpg',
    'image/png': '.png',
}
MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB

async def validate_upload(file: UploadFile) -> tuple[bytes, str]:
    """Validate uploaded file type and size."""
    # Read file content
    content = await file.read()

    # Check size
    if len(content) > MAX_FILE_SIZE:
        raise HTTPException(400, "File too large")

    # Check actual file type (not just extension)
    detected_type = magic.from_buffer(content, mime=True)
    if detected_type not in ALLOWED_TYPES:
        raise HTTPException(400, f"File type not allowed: {detected_type}")

    # Generate safe filename (never use user-provided filename directly)
    safe_ext = ALLOWED_TYPES[detected_type]
    safe_name = f"{uuid.uuid4()}{safe_ext}"

    return content, safe_name
```

### Field-Level Encryption Pattern

For highly sensitive fields (SSN, EIN) that need encryption at rest:

```python
from cryptography.fernet import Fernet
import base64
import os

class FieldEncryptor:
    """Encrypt/decrypt sensitive fields using Fernet (AES-128-CBC)."""

    def __init__(self):
        # Key from AWS Secrets Manager or environment
        key = get_secret("FIELD_ENCRYPTION_KEY")
        self._fernet = Fernet(key.encode())

    def encrypt(self, plaintext: str) -> str:
        """Encrypt and return base64-encoded ciphertext."""
        return self._fernet.encrypt(plaintext.encode()).decode()

    def decrypt(self, ciphertext: str) -> str:
        """Decrypt base64-encoded ciphertext."""
        return self._fernet.decrypt(ciphertext.encode()).decode()

# Usage in model
class Client(Base):
    __tablename__ = "clients"

    id = Column(UUID, primary_key=True)
    name = Column(String(255))
    ssn_encrypted = Column(String(500))  # Encrypted at application layer

    @property
    def ssn(self) -> str:
        if self.ssn_encrypted:
            return field_encryptor.decrypt(self.ssn_encrypted)
        return None

    @ssn.setter
    def ssn(self, value: str):
        if value:
            self.ssn_encrypted = field_encryptor.encrypt(value)
```

---

## Anti-Patterns to Catch

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| String concatenation in SQL | SQL injection risk | Use parameterized queries |
| `eval()` or `exec()` on user input | Code injection | Use safe parsing (JSON, etc.) |
| Logging PII values | Compliance violation, data leak | Log IDs and masked values only |
| Secrets in code or config files | Secrets leak via git | Use env vars or Secrets Manager |
| `pickle.loads()` on untrusted data | Remote code execution | Use JSON for serialization |
| Trusting file extensions | File type spoofing | Check actual content type (magic) |
| Hardcoded encryption keys | Key compromise | Key from secrets manager |
| MD5 or SHA1 for passwords | Easily cracked | Use bcrypt or Argon2 |
| Processing webhooks without signature | Spoofed requests | Always verify signatures |
| Returning detailed errors to users | Information disclosure | Generic errors to users, details to logs |
| `allow_pickle=True` in pandas | Code execution risk | Use safer formats (CSV, Parquet) |
| Using `shell=True` in subprocess | Command injection | Use list args, avoid shell |

---

## Quick Reference

### OWASP Top 10 (2021) Checklist

| # | Category | Key Mitigations |
|---|----------|-----------------|
| A01 | Broken Access Control | Check permissions on every request; deny by default |
| A02 | Cryptographic Failures | TLS everywhere; encrypt PII at rest; no weak algorithms |
| A03 | Injection | Parameterized queries; input validation; no eval() |
| A04 | Insecure Design | Threat modeling; secure defaults; defense in depth |
| A05 | Security Misconfiguration | Minimal permissions; disable unused features; update deps |
| A06 | Vulnerable Components | Regular dependency updates; vulnerability scanning |
| A07 | Auth Failures | MFA; rate limiting; secure session management |
| A08 | Software/Data Integrity | Verify signatures; signed artifacts; CI/CD security |
| A09 | Logging Failures | Audit sensitive ops; monitor for attacks; don't log PII |
| A10 | SSRF | Validate URLs; allowlist destinations; no internal access |

### Password Hashing

```python
import bcrypt

def hash_password(password: str) -> str:
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(password.encode(), hashed.encode())
```

### Secure Random Values

```python
import secrets

# Generate secure token
token = secrets.token_urlsafe(32)  # 256 bits

# Generate secure integer
code = secrets.randbelow(1000000)  # 6-digit code
```

### HTTP Security Headers

```python
# FastAPI middleware
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.aliunde.com"],  # Never use "*" in production
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
)

# Additional headers (via middleware or proxy)
# X-Content-Type-Options: nosniff
# X-Frame-Options: DENY
# Content-Security-Policy: default-src 'self'
# Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

## Compliance Quick Reference

### WISP Requirements (Written Information Security Plan)

Required for tax preparers per IRS Pub 4557:

- [ ] Designate security coordinator
- [ ] Risk assessment documented
- [ ] Employee security training
- [ ] Access controls (need-to-know basis)
- [ ] Encryption for PII in transit and at rest
- [ ] Secure disposal of client data
- [ ] Incident response plan
- [ ] Annual review and update

### HIPAA Security (if handling PHI)

- [ ] Access controls with unique user IDs
- [ ] Automatic logoff
- [ ] Encryption at rest and in transit
- [ ] Audit controls (who accessed what)
- [ ] Integrity controls (prevent unauthorized changes)
- [ ] Transmission security (TLS 1.2+)

### IRS Pub 4557 Highlights

- Minimum 7-year retention for tax records
- Background checks for employees with data access
- Secure disposal (shred physical, secure delete digital)
- Report breaches affecting 250+ clients to IRS
- Annual security training required

### Form 7216 Consent

Required before disclosing client tax information to third parties:
- Must be signed before disclosure
- Cannot be buried in engagement letter
- Separate consent for each third party
- Retain signed consent with client file

---

## Security Testing

### Testing Frameworks

| Framework | Purpose | Language |
|-----------|---------|----------|
| pytest + hypothesis | Property-based input fuzzing | Python |
| bandit | Static security analysis | Python |
| safety | Dependency vulnerability scan | Python |
| OWASP ZAP | Dynamic application security testing | Any |
| SpotBugs + FindSecBugs | Static security analysis | Java |
| OWASP Dependency-Check | Dependency vulnerability scan | Java/Python |

### Security Unit Tests

Test input validation rejects malicious input:

```python
import pytest
from hypothesis import given, strategies as st

class TestInputValidation:
    """Security-focused input validation tests."""

    def test_sql_injection_blocked(self):
        """Verify SQL injection attempts are rejected."""
        malicious_inputs = [
            "'; DROP TABLE users; --",
            "1 OR 1=1",
            "admin'--",
            "1; SELECT * FROM passwords",
        ]
        for payload in malicious_inputs:
            with pytest.raises(ValueError):
                validate_client_id(payload)

    def test_xss_payload_escaped(self):
        """Verify XSS payloads are escaped in output."""
        xss_inputs = [
            "<script>alert('xss')</script>",
            "javascript:alert(1)",
            "<img src=x onerror=alert(1)>",
        ]
        for payload in xss_inputs:
            result = sanitize_display_text(payload)
            assert "<script>" not in result
            assert "javascript:" not in result

    @given(st.text(max_size=1000))
    def test_arbitrary_input_no_crash(self, text):
        """Property test: arbitrary input should not crash validation."""
        try:
            validate_input(text)
        except ValueError:
            pass  # Validation rejection is fine
        # No unhandled exceptions should escape
```

### Authentication Tests

```python
class TestAuthentication:
    """Authentication security tests."""

    def test_invalid_token_rejected(self):
        """Invalid JWT tokens must be rejected."""
        invalid_tokens = [
            "",
            "not-a-token",
            "eyJ.invalid.token",
            create_expired_token(),
        ]
        for token in invalid_tokens:
            with pytest.raises(HTTPException) as exc:
                get_current_user(token)
            assert exc.value.status_code == 401

    def test_password_timing_attack_resistant(self):
        """Password comparison should use constant-time comparison."""
        import time
        # Timing should not vary significantly based on password content
        times = []
        for _ in range(100):
            start = time.perf_counter()
            verify_password("wrong_password", stored_hash)
            times.append(time.perf_counter() - start)
        # Check variance is low (constant time)
        assert max(times) - min(times) < 0.01

    def test_rate_limiting_enforced(self, client):
        """Verify brute force protection."""
        for _ in range(10):
            client.post("/auth/login", json={"email": "test@test.com", "password": "wrong"})
        response = client.post("/auth/login", json={"email": "test@test.com", "password": "wrong"})
        assert response.status_code == 429  # Too Many Requests
```

### Static Analysis Commands

```bash
# Python - Bandit security scan
bandit -r app/ -ll  # Report medium+ severity

# Python - Safety dependency check
safety check --full-report

# Python - pip-audit (newer alternative to safety)
pip-audit

# Java - SpotBugs with FindSecBugs plugin
mvn spotbugs:check -Dspotbugs.plugins=findsecbugs

# Java - OWASP Dependency Check
mvn dependency-check:check
```

### CI/CD Security Gates

```yaml
# GitHub Actions security job
security-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4

    - name: Run Bandit
      run: |
        pip install bandit
        bandit -r app/ -ll -f json -o bandit-report.json

    - name: Run Safety
      run: |
        pip install safety
        safety check --full-report

    - name: Upload SARIF
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: bandit-report.sarif
```

### Penetration Testing Checklist

Manual security testing for releases:

- [ ] Test all input fields with SQL injection payloads
- [ ] Test file uploads with malicious file types
- [ ] Verify HTTPS enforced (no HTTP)
- [ ] Check for sensitive data in responses/logs
- [ ] Test authentication bypass attempts
- [ ] Verify authorization on all endpoints (IDOR)
- [ ] Test for CSRF on state-changing operations
- [ ] Check rate limiting on sensitive endpoints
- [ ] Verify webhook signature validation
- [ ] Test session timeout and invalidation

---

## Rate Limiting

Protect against abuse and brute force attacks:

```python
from datetime import datetime, timedelta
from collections import defaultdict
import asyncio

class RateLimiter:
    """Token bucket rate limiter."""

    def __init__(self, requests_per_minute: int = 60):
        self.rate = requests_per_minute
        self.tokens = defaultdict(lambda: requests_per_minute)
        self.last_update = defaultdict(datetime.utcnow)

    async def check(self, key: str) -> bool:
        """Check if request is allowed."""
        now = datetime.utcnow()
        elapsed = (now - self.last_update[key]).total_seconds()

        # Refill tokens
        self.tokens[key] = min(
            self.rate,
            self.tokens[key] + elapsed * (self.rate / 60)
        )
        self.last_update[key] = now

        if self.tokens[key] >= 1:
            self.tokens[key] -= 1
            return True
        return False

# FastAPI middleware
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware

class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, limiter: RateLimiter):
        super().__init__(app)
        self.limiter = limiter

    async def dispatch(self, request: Request, call_next):
        # Use IP or user ID as key
        key = request.client.host
        if not await self.limiter.check(key):
            raise HTTPException(
                status_code=429,
                detail="Too many requests",
                headers={"Retry-After": "60"}
            )
        return await call_next(request)
```

---

## CORS Configuration

```python
from fastapi.middleware.cors import CORSMiddleware

# Strict production configuration
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://app.aliunde.com",
        "https://admin.aliunde.com"
    ],  # Never use "*" in production
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Authorization", "Content-Type"],
    expose_headers=["X-Request-ID"],
    max_age=600,  # Cache preflight for 10 minutes
)

# Development (more permissive)
if os.environ.get("ENV") == "development":
    origins = ["http://localhost:3000", "http://localhost:5173"]
```

---

## CSRF Protection

For state-changing operations with cookie-based auth:

```python
import secrets
from fastapi import Request, HTTPException, Depends
from fastapi.responses import JSONResponse

def generate_csrf_token() -> str:
    return secrets.token_urlsafe(32)

async def verify_csrf_token(request: Request):
    """Verify CSRF token in header matches cookie."""
    if request.method in ("GET", "HEAD", "OPTIONS"):
        return  # Safe methods don't need CSRF

    cookie_token = request.cookies.get("csrf_token")
    header_token = request.headers.get("X-CSRF-Token")

    if not cookie_token or not header_token:
        raise HTTPException(403, "CSRF token missing")

    if not secrets.compare_digest(cookie_token, header_token):
        raise HTTPException(403, "CSRF token invalid")

# Set token in response
@app.get("/csrf-token")
async def get_csrf_token():
    token = generate_csrf_token()
    response = JSONResponse({"token": token})
    response.set_cookie(
        "csrf_token",
        token,
        httponly=True,
        samesite="strict",
        secure=True
    )
    return response
```

---

## Role-Based Access Control (RBAC)

```python
from enum import Enum
from functools import wraps
from typing import Set

class Permission(Enum):
    READ_CLIENTS = "read:clients"
    WRITE_CLIENTS = "write:clients"
    DELETE_CLIENTS = "delete:clients"
    READ_DOCUMENTS = "read:documents"
    UPLOAD_DOCUMENTS = "upload:documents"
    ADMIN = "admin:all"

class Role(Enum):
    VIEWER = "viewer"
    PREPARER = "preparer"
    REVIEWER = "reviewer"
    ADMIN = "admin"

ROLE_PERMISSIONS: dict[Role, Set[Permission]] = {
    Role.VIEWER: {Permission.READ_CLIENTS, Permission.READ_DOCUMENTS},
    Role.PREPARER: {
        Permission.READ_CLIENTS, Permission.WRITE_CLIENTS,
        Permission.READ_DOCUMENTS, Permission.UPLOAD_DOCUMENTS
    },
    Role.REVIEWER: {
        Permission.READ_CLIENTS, Permission.WRITE_CLIENTS,
        Permission.READ_DOCUMENTS, Permission.UPLOAD_DOCUMENTS
    },
    Role.ADMIN: {Permission.ADMIN},  # ADMIN grants all
}

def has_permission(user_roles: list[Role], required: Permission) -> bool:
    """Check if user has required permission."""
    for role in user_roles:
        permissions = ROLE_PERMISSIONS.get(role, set())
        if Permission.ADMIN in permissions or required in permissions:
            return True
    return False

# FastAPI dependency
def require_permission(permission: Permission):
    async def check_permission(current_user: User = Depends(get_current_user)):
        if not has_permission(current_user.roles, permission):
            raise HTTPException(
                status_code=403,
                detail=f"Permission denied: {permission.value}"
            )
        return current_user
    return check_permission

# Usage
@app.delete("/clients/{client_id}")
async def delete_client(
    client_id: str,
    user: User = Depends(require_permission(Permission.DELETE_CLIENTS))
):
    ...
```

---

## Session Security

```python
from datetime import datetime, timedelta
import secrets

class SessionManager:
    """Secure session management."""

    def __init__(self, store, timeout_minutes: int = 30):
        self.store = store
        self.timeout = timedelta(minutes=timeout_minutes)

    async def create_session(self, user_id: str, metadata: dict = None) -> str:
        """Create new session with secure token."""
        session_id = secrets.token_urlsafe(32)
        session_data = {
            "user_id": user_id,
            "created_at": datetime.utcnow().isoformat(),
            "last_activity": datetime.utcnow().isoformat(),
            "metadata": metadata or {}
        }
        await self.store.set(session_id, session_data, ttl=self.timeout)
        return session_id

    async def validate_session(self, session_id: str) -> dict:
        """Validate and refresh session."""
        session = await self.store.get(session_id)
        if not session:
            raise SessionExpiredError()

        last_activity = datetime.fromisoformat(session["last_activity"])
        if datetime.utcnow() - last_activity > self.timeout:
            await self.invalidate_session(session_id)
            raise SessionExpiredError()

        # Refresh session
        session["last_activity"] = datetime.utcnow().isoformat()
        await self.store.set(session_id, session, ttl=self.timeout)
        return session

    async def invalidate_session(self, session_id: str):
        """Invalidate session (logout)."""
        await self.store.delete(session_id)

    async def invalidate_all_user_sessions(self, user_id: str):
        """Invalidate all sessions for user (password change, etc.)."""
        # Implementation depends on store supporting user-indexed sessions
        pass
```

---

## Data Privacy & Regulatory Compliance

---

### GDPR Core Requirements

The General Data Protection Regulation applies to any processing of EU/EEA residents' personal data, regardless of where the processor is located.

#### The Seven Principles (Article 5)

| Principle | Meaning | Implementation |
|-----------|---------|----------------|
| Lawfulness, fairness, transparency | Process only with a lawful basis; tell subjects how data is used | Privacy notice, consent records, lawful-basis tagging |
| Purpose limitation | Collect for specified, explicit purposes; do not repurpose | Tag data with purpose at ingestion; block secondary use without new basis |
| Data minimization | Collect only what is necessary | Review each field: "Do we need this?" before schema design |
| Accuracy | Keep data accurate and up to date | Provide user correction flows; schedule accuracy reviews |
| Storage limitation | Do not keep data longer than necessary | Automated retention enforcement; deletion/anonymization on schedule |
| Integrity and confidentiality | Protect against unauthorized access, loss, or destruction | Encryption at rest and in transit, access controls, audit logging |
| Accountability | Controller must demonstrate compliance | DPIA records, processing register, privacy policies, DPO designation |

#### Lawful Basis Types

A lawful basis must be identified **before** processing begins and documented.

| Basis | When It Applies | Example |
|-------|----------------|---------|
| Consent | Subject has given clear, affirmative consent | Marketing emails, optional analytics |
| Contract | Processing is necessary to perform a contract with the subject | Account creation, order fulfillment |
| Legal obligation | Processing is required by law | Tax reporting, AML/KYC |
| Vital interests | Necessary to protect life | Emergency medical situations |
| Public task | Necessary for official public authority tasks | Government services |
| Legitimate interests | Necessary for legitimate interests of controller/third party (unless overridden by subject rights) | Fraud prevention, network security, B2B marketing |

**Implementation pattern:**

```python
from enum import Enum

class LawfulBasis(Enum):
    CONSENT = "consent"
    CONTRACT = "contract"
    LEGAL_OBLIGATION = "legal_obligation"
    VITAL_INTERESTS = "vital_interests"
    PUBLIC_TASK = "public_task"
    LEGITIMATE_INTERESTS = "legitimate_interests"

# Tag every processing activity in your data catalog
PROCESSING_REGISTER = {
    "user_account_creation": {
        "lawful_basis": LawfulBasis.CONTRACT,
        "purpose": "Provide service account",
        "retention_days": 2555,  # 7 years post-account-close
        "data_categories": ["name", "email", "password_hash"],
    },
    "marketing_email": {
        "lawful_basis": LawfulBasis.CONSENT,
        "purpose": "Promotional communications",
        "retention_days": None,  # Until consent withdrawn
        "data_categories": ["email", "name"],
        "consent_withdrawal_url": "/unsubscribe",
    },
}
```

#### Data Subject Access Requests (DSAR)

Under GDPR Article 15, individuals have the right to obtain confirmation of processing and a copy of their personal data.

**Response SLA:** 30 calendar days (extendable to 3 months for complex/numerous requests, with notice within first 30 days).

**What must be provided:**
- Confirmation that their data is processed
- Copy of personal data (in portable format if requested)
- Purposes of processing
- Categories of data
- Recipients or categories of recipients
- Retention periods
- Information on subject rights (rectification, erasure, restriction, objection)
- Right to lodge complaint with supervisory authority
- Source of data (if not collected from subject)
- Existence of automated decision-making, including profiling

**Implementation pattern:**

```python
async def handle_dsar(subject_id: str) -> dict:
    """Collect all personal data for a DSAR response."""
    # Query each data layer
    profile_data = await db.fetch_profile(subject_id)
    activity_logs = await db.fetch_activity(subject_id)
    bronze_records = await data_lake.fetch_raw(subject_id)
    silver_records = await data_lake.fetch_cleansed(subject_id)
    # Gold/aggregated: if individual is identifiable, include
    gold_records = await data_lake.fetch_aggregated(subject_id)

    return {
        "subject_id": subject_id,
        "request_date": datetime.utcnow().isoformat(),
        "response_deadline": (datetime.utcnow() + timedelta(days=30)).isoformat(),
        "data": {
            "profile": profile_data,
            "activity": activity_logs,
            "bronze_layer": bronze_records,
            "silver_layer": silver_records,
            "gold_layer": gold_records,
        },
        "processing_activities": PROCESSING_REGISTER,
    }

    # Log the DSAR itself (required for accountability)
    audit_log(
        action="dsar.response_compiled",
        user_id="system",
        resource_type="data_subject",
        resource_id=subject_id,
        details={"layers_searched": ["profile", "activity", "bronze", "silver", "gold"]}
    )
```

#### Right to Erasure (Right to be Forgotten)

Article 17 grants data subjects the right to have their personal data deleted.

**When it applies:**
- Data is no longer necessary for the original purpose
- Subject withdraws consent (and no other lawful basis)
- Subject objects and no overriding legitimate grounds exist
- Data was unlawfully processed
- Deletion required to comply with a legal obligation
- Data collected in relation to offering information society services to a child

**Exceptions (erasure can be refused):**
- Exercising freedom of expression and information
- Compliance with a legal obligation (e.g., tax record retention)
- Public interest in public health
- Archiving in the public interest, scientific/historical research, or statistics
- Establishment, exercise, or defense of legal claims

**Decision tree:**

```
Erasure request received
    │
    ├─ Is there a legal obligation to retain? ──Yes──> Refuse; notify subject; log
    │  (tax records, AML/KYC, court orders)
    │
    ├─ Is data needed for legal claims? ──Yes──> Place on legal hold; notify subject
    │
    ├─ Is subject in public interest category? ──Yes──> Assess public interest; may refuse
    │
    └─ No exceptions apply ──> Proceed with erasure across all layers
```

---

### CCPA Requirements

The California Consumer Privacy Act (effective Jan 1, 2020; amended by CPRA 2023) applies to for-profit businesses that collect California residents' personal information and meet any of: $25M+ annual gross revenue, buy/sell/share 100K+ consumers/households/devices annually, or derive 50%+ revenue from selling/sharing personal information.

#### Key Consumer Rights

| Right | Description | Response SLA |
|-------|-------------|--------------|
| Right to Know | Know what personal information is collected, used, disclosed, and sold | 45 days (extendable 45 more) |
| Right to Delete | Request deletion of personal information | 45 days (extendable 45 more) |
| Right to Opt-Out of Sale/Sharing | Stop sale or sharing of personal information | Honored within 15 business days |
| Right to Non-Discrimination | Cannot be denied service or charged more for exercising CCPA rights | Ongoing |
| Right to Correct (CPRA) | Request correction of inaccurate personal information | 45 days (extendable 45 more) |
| Right to Limit Use of Sensitive PI (CPRA) | Limit use/disclosure of sensitive personal information | Honored within 15 business days |

#### GDPR vs CCPA Comparison

| Dimension | GDPR | CCPA/CPRA |
|-----------|------|-----------|
| Jurisdiction | EU/EEA residents | California residents |
| Lawful basis required | Yes (before processing) | No (opt-out model) |
| Default stance | Opt-in (need basis) | Opt-out (can collect until objection) |
| "Sale" definition | Not a core concept | Broadly defined (includes sharing for cross-context behavioral ads) |
| Response SLA | 30 days | 45 days |
| Fines | Up to 4% global revenue or 20M EUR | Up to $7,500 per intentional violation |
| Children | 16 (or 13 with parental consent) | Under 16 requires opt-in; under 13 parental consent |

#### "Sale" Under CCPA

CCPA defines "sale" broadly: selling, renting, releasing, disclosing, disseminating, making available, transferring, or otherwise communicating personal information to a third party for monetary OR other valuable consideration.

This includes:
- Targeted advertising networks receiving data for ad targeting (even without cash payment)
- Analytics providers that use data across clients
- Data brokers

**Implementation:** "Do Not Sell or Share My Personal Information" link must appear prominently on homepage and in privacy policy. Global Privacy Control (GPC) browser signal must be honored automatically.

```python
def check_do_not_sell(request: Request, user_id: str) -> bool:
    """Return True if user has opted out of sale/sharing."""
    # Check GPC header (automatic browser signal)
    if request.headers.get("Sec-GPC") == "1":
        return True
    # Check stored user preference
    prefs = get_user_privacy_preferences(user_id)
    return prefs.get("do_not_sell", False)

def should_share_with_ad_network(request: Request, user_id: str) -> bool:
    """Determine if data can be shared with advertising networks."""
    return not check_do_not_sell(request, user_id)
```

---

### Data Privacy Impact Assessment (DPIA)

A DPIA (Article 35 GDPR) is required before processing that is likely to result in high risk to individuals.

#### When a DPIA is Required

A DPIA is mandatory when processing involves at least two of these criteria, or when supervisory authority guidance specifies it:

| Criterion | Example |
|-----------|---------|
| Systematic and extensive automated profiling | Credit scoring, behavioral targeting, HR performance analytics |
| Large-scale processing of special category data | Health records, biometrics, race, political opinions, religion |
| Systematic monitoring of publicly accessible areas | CCTV, location tracking, IoT sensor networks |
| Large-scale processing of criminal conviction data | Background check services |
| Innovative technology with unknown risks | New AI/ML models applied to personal data |
| Denial of service or access based on automated processing | Loan denials, job screening, benefits eligibility |
| Vulnerable data subjects | Children, employees, patients |

**Decision tree:**

```
Is the processing:
├─ New or changed significantly? ──No──> DPIA may not be needed (document rationale)
│
└─ Yes ──> Does it involve:
    ├─ Systematic/extensive profiling with significant effects? ──Yes──> DPIA required
    ├─ Large-scale special category data? ──Yes──> DPIA required
    ├─ Systematic monitoring of public areas? ──Yes──> DPIA required
    └─ None of the above but novel risk? ──Yes──> Consult DPO; DPIA recommended
```

#### DPIA Template Elements

```markdown
## DPIA: [Processing Activity Name]

### 1. Description of Processing
- What data is collected and from whom?
- How is it collected (source)?
- How is it used (purpose)?
- Who has access?
- How long is it retained?
- Who are the third-party recipients?

### 2. Necessity and Proportionality Assessment
- Is this processing necessary for the stated purpose?
- Could a less privacy-intrusive approach achieve the same goal?
- Is the lawful basis appropriate?
- Are data minimization principles applied?

### 3. Risks to Individuals
| Risk | Likelihood | Severity | Overall |
|------|------------|----------|---------|
| Unauthorized access | Medium | High | High |
| Data breach | Low | High | Medium |
| Re-identification | Low | Medium | Low |
| Discrimination from profiling | Low | High | Medium |

### 4. Mitigations
| Risk | Mitigation | Owner | Target Date |
|------|------------|-------|-------------|
| Unauthorized access | Role-based access control + MFA | Engineering | YYYY-MM-DD |
| Data breach | Encryption at rest and in transit | Engineering | YYYY-MM-DD |

### 5. Residual Risk Assessment
- Are residual risks acceptable?
- If high residual risk remains: consult supervisory authority before proceeding

### 6. Sign-off
- DPO review date:
- Decision: Proceed / Do Not Proceed / Proceed with conditions
```

---

### Cross-Border Data Residency

Personal data of EU/EEA residents may only be transferred outside the EEA where adequate protection is ensured.

#### Transfer Mechanisms

| Mechanism | Description | Use Case |
|-----------|-------------|----------|
| Adequacy decision | European Commission has deemed destination country adequate | UK (current), Switzerland, Canada (commercial), Japan, South Korea |
| Standard Contractual Clauses (SCCs) | EU-approved contract clauses between exporter and importer | Most common; updated 2021 SCCs required |
| Binding Corporate Rules (BCRs) | Intra-group transfers; requires supervisory authority approval | Large multinationals with global data flows |
| Derogations (Article 49) | Limited cases: explicit consent, contract performance, vital interests, public interest | One-off transfers only; cannot be habitual |

#### APAC Data Residency Patterns

| Country | Regulation | Key Requirement |
|---------|------------|-----------------|
| China | PIPL (Personal Information Protection Law) | Personal information of Chinese citizens must be stored in China unless security assessment passed; cross-border transfers require user consent and government assessment for large-scale transfers |
| Japan | APPI (Act on Protection of Personal Information) | Transfer to third countries requires equivalent protection or user consent |
| Australia | Privacy Act 1988 | Australian Privacy Principles apply; cross-border disclosure requires equivalent protection or consent |
| India | DPDP Act 2023 | Data localization for certain categories; government may restrict cross-border transfers |

#### Implementation Patterns

```python
from enum import Enum
from dataclasses import dataclass

class DataRegion(Enum):
    EU_EEA = "eu-eea"       # GDPR
    US = "us"               # CCPA + state laws
    CN = "cn"               # PIPL
    JP = "jp"               # APPI
    AU = "au"               # Privacy Act
    IN = "in"               # DPDP

@dataclass
class DataResidencyPolicy:
    subject_region: DataRegion
    allowed_storage_regions: list[DataRegion]
    requires_scc: bool
    requires_adequacy_decision: bool
    requires_user_consent: bool

RESIDENCY_POLICIES = {
    DataRegion.EU_EEA: DataResidencyPolicy(
        subject_region=DataRegion.EU_EEA,
        allowed_storage_regions=[DataRegion.EU_EEA, DataRegion.US],  # US with SCCs
        requires_scc=True,     # If storing in US
        requires_adequacy_decision=False,
        requires_user_consent=False,
    ),
    DataRegion.CN: DataResidencyPolicy(
        subject_region=DataRegion.CN,
        allowed_storage_regions=[DataRegion.CN],  # Must stay in China by default
        requires_scc=False,
        requires_adequacy_decision=False,
        requires_user_consent=True,  # For any cross-border transfer
    ),
}

def validate_data_residency(subject_region: DataRegion, storage_region: DataRegion) -> bool:
    """Check if storing data in storage_region is allowed for given subject."""
    policy = RESIDENCY_POLICIES.get(subject_region)
    if not policy:
        return True  # No specific restriction known
    return storage_region in policy.allowed_storage_regions
```

**Encryption-key-per-region pattern:** Store data in a single physical location but use region-specific encryption keys so that data is cryptographically inaccessible outside its home region:

```python
def get_regional_key(region: DataRegion) -> str:
    """Retrieve the encryption key for a specific data region."""
    return get_secret(f"FIELD_ENCRYPTION_KEY_{region.value.upper()}")

def encrypt_for_region(plaintext: str, subject_region: DataRegion) -> str:
    """Encrypt data using region-specific key."""
    key = get_regional_key(subject_region)
    fernet = Fernet(key.encode())
    return fernet.encrypt(plaintext.encode()).decode()
```

**Data residency metadata tagging:** Tag all records at ingestion with the data subject's region so queries and access controls can enforce residency:

```python
# Iceberg/Spark metadata pattern
spark.sql("""
    ALTER TABLE bronze.customer_events
    SET TBLPROPERTIES (
        'data.residency.region' = 'eu-eea',
        'data.residency.regulation' = 'gdpr',
        'data.residency.enforced_at' = '2026-01-01'
    )
""")
```

---

### Enterprise PII Classification & Auto-Identification

Classifying personal data consistently at ingestion prevents compliance gaps downstream.

#### PII Taxonomy

| Category | Type | Examples | Sensitivity |
|----------|------|---------|-------------|
| Direct identifiers | Name, SSN, email, passport number, driver's license | John Smith, 123-45-6789, john@example.com | Critical |
| Quasi-identifiers | DOB, gender, ZIP code, race (alone, can combine to re-identify) | 1985-03-14, Male, 90210 | High |
| Special category (GDPR Art. 9) | Health, biometric, genetic, race/ethnicity, religion, political, sexual orientation, trade union | Medical records, fingerprints, DNA | Critical |
| Financial | Credit card numbers, bank accounts, income | 4111 1111 1111 1111, routing number | Critical |
| Behavioral | Browsing history, purchase history, location history (if linked to individual) | Clickstream, GPS trace | High |
| Professional | Employer, job title, performance reviews | Works at ACME Inc., Performance: Exceeds | Medium |

#### Auto-Identification Patterns

**Regex-based scanning** (fast, low false-negative rate for structured PII):

```python
import re
from dataclasses import dataclass

@dataclass
class PiiMatch:
    field_name: str
    pii_type: str
    confidence: float
    sample: str  # Masked sample for audit

PII_PATTERNS = {
    "ssn": (r'\b\d{3}-\d{2}-\d{4}\b|\b\d{9}\b', 0.9),
    "email": (r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', 0.95),
    "credit_card": (r'\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14}|3[47][0-9]{13})\b', 0.85),
    "phone_us": (r'\b(?:\+1[-.\s]?)?\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}\b', 0.75),
    "ip_address": (r'\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b', 0.6),
    "passport": (r'\b[A-Z]{1,2}[0-9]{6,9}\b', 0.5),
}

def scan_for_pii(text: str, field_name: str = "unknown") -> list[PiiMatch]:
    """Scan a text value for PII patterns."""
    matches = []
    for pii_type, (pattern, confidence) in PII_PATTERNS.items():
        if re.search(pattern, text):
            matches.append(PiiMatch(
                field_name=field_name,
                pii_type=pii_type,
                confidence=confidence,
                sample=mask_for_audit(text, pii_type)
            ))
    return matches
```

**Classification-at-ingestion pattern for data pipelines:**

```python
# Spark/PySpark classification at Bronze ingestion
from pyspark.sql import functions as F
from pyspark.sql.types import StringType

def classify_column(col_name: str, col_value) -> str:
    """Return PII classification for a column value."""
    # Run regex scan; return highest-confidence match
    matches = scan_for_pii(str(col_value), col_name)
    if not matches:
        return "non-pii"
    return max(matches, key=lambda m: m.confidence).pii_type

classify_udf = F.udf(classify_column, StringType())

# Apply at ingestion
df_classified = df_raw.withColumn(
    "_pii_classification",
    classify_udf(F.lit(col_name), F.col(col_name))
).withColumn(
    "_residency_region",
    F.lit(subject_region.value)
).withColumn(
    "_ingestion_ts",
    F.current_timestamp()
)
```

**Integration with Collibra sensitivity tagging:** After classification, push tags to Collibra data catalog:

```python
import requests

def tag_collibra_asset(asset_id: str, pii_type: str, residency_region: str):
    """Apply sensitivity tags to a Collibra data asset."""
    collibra_url = get_secret("COLLIBRA_API_URL")
    token = get_secret("COLLIBRA_API_TOKEN")

    # Map internal PII type to Collibra sensitivity classification
    sensitivity_map = {
        "ssn": "Highly Confidential",
        "credit_card": "Highly Confidential",
        "email": "Confidential",
        "phone_us": "Confidential",
        "ip_address": "Internal",
        "non-pii": "Public",
    }
    sensitivity = sensitivity_map.get(pii_type, "Confidential")

    response = requests.patch(
        f"{collibra_url}/api/v1/assets/{asset_id}/tags",
        json={
            "tags": [
                {"name": f"sensitivity:{sensitivity}"},
                {"name": f"pii-type:{pii_type}"},
                {"name": f"residency:{residency_region}"},
            ]
        },
        headers={"Authorization": f"Bearer {token}"}
    )
    response.raise_for_status()
```

---

### Right to be Forgotten Implementation

Implementing erasure across a Medallion Architecture (Bronze/Silver/Gold) is non-trivial because columnar formats like Parquet and Iceberg are immutable by design.

#### The Challenge

| Layer | Format | Mutability | Erasure Approach |
|-------|--------|------------|-----------------|
| Bronze | Raw Parquet / Iceberg | Immutable files | Delete vector (Iceberg v2) or file rewrite |
| Silver | Cleansed Parquet / Iceberg | Immutable files | Delete vector or file rewrite |
| Gold | Aggregated tables / views | Immutable files | Recompute aggregation without subject; or anonymization if k-anonymity holds |
| Operational DB | PostgreSQL / MySQL | Mutable rows | Standard DELETE + cascade |
| Backups | Compressed archives | Immutable | Purge after retention period; note in response |
| Logs | Append-only | Immutable | Anonymize (replace identifiers with hashed/null values) |

#### Iceberg Delete Vector Pattern (preferred for large datasets)

Apache Iceberg v2 supports positional and equality deletes without full file rewrite:

```python
from pyiceberg.catalog import load_catalog

def iceberg_erasure(catalog_name: str, table_path: str, subject_id: str):
    """Delete all records for subject_id using Iceberg equality delete."""
    catalog = load_catalog(catalog_name)
    table = catalog.load_table(table_path)

    # Write equality delete file - marks rows for logical deletion
    # Physical file rewrite happens during compaction
    with table.new_overwrite() as overwrite:
        overwrite.delete_where(f"subject_id = '{subject_id}'")
        overwrite.commit()

    # Log the deletion for audit
    audit_log(
        action="gdpr.erasure.iceberg",
        user_id="system",
        resource_type="iceberg_table",
        resource_id=f"{table_path}/{subject_id}",
        details={"table": table_path, "method": "equality_delete"}
    )
```

#### Full Erasure Workflow

```python
async def execute_right_to_erasure(subject_id: str, request_id: str) -> dict:
    """
    Execute GDPR Right to Erasure across all data layers.
    Returns audit trail of what was deleted/anonymized.
    """
    audit_trail = {"request_id": request_id, "subject_id": subject_id, "layers": {}}

    # 1. Check for legal hold or retention obligation
    obligations = check_retention_obligations(subject_id)
    if obligations:
        return {
            "status": "refused",
            "reason": "legal_obligation",
            "obligations": obligations,
            "note": "Data retained per legal requirement; subject notified"
        }

    # 2. Operational database
    rows_deleted = await db.execute(
        "DELETE FROM users WHERE id = $1", subject_id
    )
    audit_trail["layers"]["operational_db"] = {"deleted_rows": rows_deleted}

    # 3. Bronze layer
    iceberg_erasure("production", "bronze.raw_events", subject_id)
    audit_trail["layers"]["bronze"] = {"method": "iceberg_equality_delete"}

    # 4. Silver layer
    iceberg_erasure("production", "silver.cleansed_events", subject_id)
    audit_trail["layers"]["silver"] = {"method": "iceberg_equality_delete"}

    # 5. Gold layer - recompute or anonymize
    recompute_gold_aggregates(subject_id)
    audit_trail["layers"]["gold"] = {"method": "aggregate_recompute"}

    # 6. Logs - anonymize (replace identifiers, keep event structure)
    anonymize_audit_logs(subject_id, replacement="[ERASED]")
    audit_trail["layers"]["audit_logs"] = {"method": "anonymized"}

    # 7. Notify downstream systems
    notify_downstream_of_erasure(subject_id)

    # 8. Record the erasure itself (ironically, must be retained)
    erasure_record = {
        "request_id": request_id,
        "subject_id": subject_id,  # Retain ID to prove erasure happened
        "erased_at": datetime.utcnow().isoformat(),
        "audit_trail": audit_trail,
    }
    await store_erasure_record(erasure_record)  # Retained for compliance evidence

    return {"status": "completed", "audit_trail": audit_trail}
```

#### Downstream Notification Pattern

When personal data is deleted, all downstream recipients who received the data must be notified (GDPR Article 19):

```python
def notify_downstream_of_erasure(subject_id: str):
    """Notify all downstream data recipients of erasure."""
    recipients = get_data_recipients(subject_id)
    for recipient in recipients:
        send_erasure_notification(
            recipient=recipient,
            subject_reference=hash_for_notification(subject_id),
            required_action="delete_personal_data",
            deadline_days=30,
        )
```

---

### Data Retention & Deletion Policies

#### Retention Schedule by Data Category

| Data Category | Regulation | Retention Period | Action at Expiry |
|---------------|------------|-----------------|-----------------|
| Tax records | IRS Pub 4557 | 7 years from filing | Secure deletion |
| Financial transactions | SOX / FINRA | 7 years | Secure deletion |
| EU personal data (general) | GDPR | Duration of purpose + legal period | Deletion or anonymization |
| Health records (US) | HIPAA | 6 years from creation or last effective date | Secure deletion |
| Employee records | FLSA / state law | 3 years (payroll), up to 7 years (benefits) | Secure deletion |
| Marketing consent records | GDPR / CCPA | Duration of relationship + 1 year (evidence) | Deletion |
| Security logs | Internal / SOC 2 | 1-3 years | Secure deletion |
| Backup tapes | Follows primary record policy | Same as source | Overwrite schedule |

#### Automated Retention Enforcement

```python
from datetime import datetime, timedelta
from dataclasses import dataclass

@dataclass
class RetentionPolicy:
    category: str
    retention_days: int
    action: str  # "delete" or "anonymize"
    legal_basis: str

RETENTION_POLICIES = {
    "tax_record": RetentionPolicy("tax_record", 2555, "delete", "IRS Pub 4557"),
    "eu_personal_data": RetentionPolicy("eu_personal_data", 365, "anonymize", "GDPR Art 5(1)(e)"),
    "health_record": RetentionPolicy("health_record", 2190, "delete", "HIPAA"),
    "security_log": RetentionPolicy("security_log", 1095, "delete", "SOC2"),
}

async def enforce_retention():
    """Scheduled job: delete or anonymize expired records."""
    for category, policy in RETENTION_POLICIES.items():
        cutoff = datetime.utcnow() - timedelta(days=policy.retention_days)
        expired_records = await db.fetch_expired(category, cutoff)

        for record in expired_records:
            if policy.action == "delete":
                await db.delete(record.id)
                audit_log("retention.delete", "system", category, record.id,
                          {"policy": policy.legal_basis, "cutoff": cutoff.isoformat()})
            elif policy.action == "anonymize":
                await db.anonymize(record.id)
                audit_log("retention.anonymize", "system", category, record.id,
                          {"policy": policy.legal_basis, "cutoff": cutoff.isoformat()})
```

#### Legal Hold Pattern

Legal holds suspend normal retention schedules when litigation or investigation is anticipated:

```python
async def apply_legal_hold(subject_ids: list[str], hold_reason: str, hold_id: str):
    """Apply legal hold - suspends deletion schedule."""
    for subject_id in subject_ids:
        await db.execute(
            """INSERT INTO legal_holds (subject_id, hold_id, reason, applied_at)
               VALUES ($1, $2, $3, $4)
               ON CONFLICT (subject_id, hold_id) DO NOTHING""",
            subject_id, hold_id, hold_reason, datetime.utcnow()
        )
        audit_log("legal_hold.applied", "system", "data_subject", subject_id,
                  {"hold_id": hold_id, "reason": hold_reason})

def check_retention_obligations(subject_id: str) -> list[dict]:
    """Return active legal holds and regulatory retention requirements."""
    holds = db.fetch_active_holds(subject_id)
    regulations = check_regulatory_minimums(subject_id)
    return holds + regulations
```

#### Secure Deletion vs Anonymization Decision Tree

```
Data expiry reached
    │
    ├─ Is the data in a mutable system (RDBMS)? ──Yes──> DELETE rows; overwrite if needed
    │
    ├─ Is the data in an immutable format (Parquet, Iceberg)?
    │   ├─ Does the regulation accept anonymization? ──Yes──> Anonymize (replace identifiers)
    │   └─ Must data be fully deleted? ──Yes──> Rewrite Parquet files excluding subject
    │
    ├─ Is the data in backups?
    │   ├─ Can backup be restored and re-purged? ──Yes──> Do so; log the operation
    │   └─ Cannot access backup? ──> Document in erasure record; purge backup at scheduled rotation
    │
    └─ Is the data in logs?
        ├─ Structured logs: Replace identifiers with hashed/null values
        └─ Unstructured logs: If possible extract and redact; else document retention justification
```

---

### Privacy by Design

Privacy by Design (PbD) is both a GDPR requirement (Article 25) and a foundational engineering philosophy.

#### The Seven Foundational Principles

| Principle | Meaning | Engineering Application |
|-----------|---------|------------------------|
| 1. Proactive not Reactive | Prevent privacy issues before they occur | Threat model for privacy in design phase; DPIA before launch |
| 2. Privacy as the Default | Maximum privacy without subject action | Collect minimum data by default; opt-in for additional collection |
| 3. Privacy Embedded into Design | Privacy is a core feature, not an add-on | Data classification schema defined before schema design; PII fields marked in ERD |
| 4. Full Functionality | Privacy and functionality are not zero-sum | Design for both; if trade-off required, document and justify |
| 5. End-to-End Security | Security across full data lifecycle | Encryption at ingestion, at rest, in transit, at deletion; key management lifecycle |
| 6. Visibility and Transparency | Open about policies and practices | Machine-readable privacy notices; audit-accessible processing records |
| 7. Respect for User Privacy | Keep it user-centric | Simple consent UX; easy opt-out; accessible DSAR portal |

#### How to Apply in Data Platform Design

**Schema design:**
- Before creating any table or field, ask: "Is this personal data? Which regulation applies? What is the lawful basis?"
- Mark PII fields in ERD/schema with classification tags
- Never store raw SSN without encryption; never store passwords in plaintext

**Pipeline design:**
- Classify data at Bronze ingestion (see PII Classification section above)
- Apply data minimization transforms at Silver (strip fields not needed for purpose)
- Apply access controls per classification at Gold

**Access control:**
- Use column-level security in Snowflake/Starburst for sensitive columns
- Do not include PII in default SELECT *; require explicit column request
- Log all access to PII columns (GDPR accountability)

```python
# Snowflake column-level security example
snowflake_cursor.execute("""
    CREATE MASKING POLICY ssn_mask AS (val STRING) RETURNS STRING ->
        CASE
            WHEN CURRENT_ROLE() IN ('DATA_ANALYST_ROLE') THEN 'XXX-XX-' || RIGHT(val, 4)
            WHEN CURRENT_ROLE() IN ('COMPLIANCE_ROLE') THEN val
            ELSE '***-**-****'
        END;
""")

snowflake_cursor.execute("""
    ALTER TABLE silver.customers
    MODIFY COLUMN ssn
    SET MASKING POLICY ssn_mask;
""")
```

---

### Regulatory Quick Reference

| Regulation | Jurisdiction | Key Rights | Response SLA | Max Penalty |
|------------|-------------|------------|--------------|-------------|
| GDPR | EU/EEA residents (global reach) | Access, erasure, portability, rectification, restriction, objection | 30 days | 4% global revenue or 20M EUR |
| CCPA/CPRA | California residents | Know, delete, opt-out, correct, non-discrimination | 45 days | $7,500 per intentional violation |
| HIPAA | US health data | Access, amendment, accounting of disclosures | 30-60 days | Up to $1.9M per violation category per year |
| China PIPL | Chinese residents (global reach) | Access, correction, deletion, portability, explanation | 15 working days | 5% China revenue or 50M RMB |
| Japan APPI | Japanese residents | Disclosure, correction, suspension of use | Prompt (not specified) | 100M JPY |
| Australia Privacy Act | Australian residents | Access, correction | 30 days | AUD 50M (or 30% of adjusted turnover) |
| India DPDP | Indian citizens | Access, correction, erasure, grievance redress | TBD (rules pending) | INR 250 crore |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| Input validation models | `ali-ai-acctg/app/schemas/` |
| Authentication logic | `ali-ai-acctg/app/auth/` |
| Audit logging | `ali-ai-acctg/app/services/audit.py` |
| Secrets configuration | `ali-ai-acctg/.env.example` |
| Field encryption | `ali-ai-acctg/app/utils/encryption.py` |
| Webhook handlers | `ali-ai-acctg/app/webhooks/` |
| Snowflake identifier safety | `ali-ai-ingestion-engine/sql_gen_build/config.py` |
| Secure file handling | `ali-ai-acctg/app/services/documents.py` |

---

## References

- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [IRS Publication 4557](https://www.irs.gov/pub/irs-pdf/p4557.pdf) - Safeguarding Taxpayer Data
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/index.html)
- [CWE/SANS Top 25](https://cwe.mitre.org/top25/)
- [Python Security Best Practices](https://python-security.readthedocs.io/)
- [GDPR Full Text](https://gdpr-info.eu/)
- [CCPA/CPRA Text](https://oag.ca.gov/privacy/ccpa)
- [EDPB Guidelines on DPIA](https://www.edpb.europa.eu/our-work-tools/our-documents/guidelines/guidelines-92017-data-protection-impact-assessment-dpia_en)
- [EU Standard Contractual Clauses (2021)](https://commission.europa.eu/law/law-topic/data-protection/international-dimension-data-protection/standard-contractual-clauses-scc_en)
- [ICO Right to Erasure Guidance](https://ico.org.uk/for-organisations/guide-to-data-protection/guide-to-the-general-data-protection-regulation-gdpr/individual-rights/right-to-erasure/)
- [China PIPL (English translation)](https://digichina.stanford.edu/work/translation-personal-information-protection-law-of-the-peoples-republic-of-china/)

---

## Claude Code /security-review

### What It Is

`/security-review` is an AI-powered security scanning feature built into Claude Code. It performs static analysis of your codebase against common vulnerability patterns, data flow analysis, and compliance checks — surfacing findings directly in the conversation.

### How to Invoke

Run `/security-review` in any Claude Code session from the project directory:

```
/security-review
```

No arguments required. Claude Code scans the project in context and returns a structured findings report.

### What It Detects

- **OWASP Top 10 vulnerabilities** — injection (including XSS), identification and authentication failures, cryptographic failures, insecure design, security misconfiguration (including XXE), vulnerable and outdated components, broken access control, software and data integrity failures, security logging and monitoring failures, SSRF
- **Data flow tracing** — follows untrusted input from entry point to output, flags unvalidated paths
- **Authentication and authorization flaws** — missing checks, privilege escalation paths, session issues
- **Secrets and credential exposure** — hardcoded API keys, tokens, passwords in source
- **Dependency vulnerabilities** — known CVEs in imported packages (where resolvable from context)

### Workflow

1. **Update code** — make your changes as normal
2. **Run** — invoke `/security-review` in the Claude Code session
3. **Review findings** — Claude returns severity-ranked findings (Critical, High, Medium, Low)
4. **Request fixes** — for each finding, ask Claude to suggest a remediation (e.g., "Suggest a fix for finding C1")
5. **Apply with approval** — review every suggested fix before applying; do NOT auto-apply
6. **Document** — log findings and resolutions in your audit trail for compliance purposes

### GitHub Actions Integration

Automated security review on every pull request via the official action:

```yaml
# .github/workflows/security-review.yml
name: Security Review

on:
  pull_request:
    branches: [main]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: anthropics/claude-code-security-review@v1
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

The action posts findings as PR review comments. Configure severity thresholds to fail the build on Critical or High findings.

### Availability

- **Enterprise and Team plans**: Available by default
- **Open-source projects**: Expedited access available via Anthropic's limited research preview program — apply through the Claude Code documentation
- **Individual Pro**: Not currently included; check release notes for updates

### Important: Human Approval Required

`/security-review` surfaces findings and can suggest fixes. It does NOT automatically apply changes. Every remediation requires explicit human review and approval before application. This is non-negotiable for audit trail integrity — particularly for projects under IRS, HIPAA, or SOC 2 compliance requirements.

Never accept a security fix suggestion without reading the diff and understanding the change.
