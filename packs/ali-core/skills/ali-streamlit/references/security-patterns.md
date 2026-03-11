# ali-streamlit - Security Patterns Reference

Comprehensive security implementations for Streamlit applications. Load when implementing authentication, audit logging, file uploads, or security compliance features.

---

## Authentication

### Database-Backed Authentication (from tax-ai-chat)

```python
def authenticate_user(email: str, password: str) -> tuple[dict | None, str | None]:
    """
    Authenticate user against database using bcrypt.

    Returns tuple of (user_dict, error_message).
    - Success: (user_dict, None)
    - Failed credentials: (None, None)
    - System error: (None, error_message)
    """
    try:
        import psycopg2
        from psycopg2.extras import RealDictCursor

        conn = psycopg2.connect(...)

        with conn.cursor(cursor_factory=RealDictCursor) as cur:
            # Get user by email and verify password using DB function
            cur.execute(
                """
                SELECT id, email, first_name, last_name, role, is_active,
                       verify_password(%s, password_hash) as password_valid
                FROM users
                WHERE email = %s AND is_active = true
                """,
                (password, email.strip().lower()),
            )

            row = cur.fetchone()

            if row and row.get("password_valid"):
                return {
                    "id": str(row["id"]),
                    "email": row["email"],
                    "display_name": f"{row['first_name']} {row['last_name']}",
                    "role": row["role"],
                }, None

        conn.close()
        return None, None  # Invalid credentials
    except Exception as e:
        return None, f"System error: {str(e)}"
```

---

## Lockout Protection

**Track failed attempts in session state:**

```python
# Initialize lockout tracking
if "login_attempts" not in st.session_state:
    st.session_state.login_attempts = 0
    st.session_state.lockout_until = None

# Check lockout before showing login
if st.session_state.lockout_until:
    if datetime.now() < st.session_state.lockout_until:
        remaining = (st.session_state.lockout_until - datetime.now()).seconds // 60
        st.error(f"Account locked. Try again in {remaining + 1} minutes.")
        return  # Don't show login form
    else:
        # Lockout expired, reset
        st.session_state.lockout_until = None
        st.session_state.login_attempts = 0

# On failed login
if not user:
    st.session_state.login_attempts += 1

    # Enforce lockout after 5 failures
    if st.session_state.login_attempts >= 5:
        st.session_state.lockout_until = datetime.now() + timedelta(minutes=15)
        st.error("Too many failed attempts. Account locked for 15 minutes.")
```

**Best practices:**
- 5 failed attempts triggers lockout
- 15-minute lockout duration
- Display remaining lockout time
- Reset counter after successful login
- Track per-session (not persistent across browser sessions)

---

## Session Timeout

**Dual timeout pattern - Absolute + Idle:**

```python
def check_session_timeout():
    """Check and enforce session timeouts. Returns True if session is valid."""
    if not st.session_state.get("authenticated"):
        return False

    now = datetime.now()

    # Absolute timeout: 12 hours
    if st.session_state.login_timestamp:
        session_age = now - st.session_state.login_timestamp
        if session_age > timedelta(hours=12):
            st.session_state.authenticated = False
            st.warning("⏰ Session expired after 12 hours. Please log in again.")
            return False

    # Idle timeout: 30 minutes
    if st.session_state.last_activity:
        idle_time = now - st.session_state.last_activity
        if idle_time > timedelta(minutes=30):
            st.session_state.authenticated = False
            st.warning("⏰ Session expired due to inactivity. Please log in again.")
            return False

    # Update activity timestamp
    st.session_state.last_activity = now
    return True

# Call at top of main app
if not check_session_timeout():
    show_login_page()
    st.stop()
```

**Compliance considerations:**
- IRS Circular 230: Requires session timeouts for tax software
- HIPAA: 15-minute idle timeout for healthcare applications
- SOC 2: Document timeout policies and enforcement

**Implementation notes:**
- Absolute timeout: Hard limit regardless of activity
- Idle timeout: Reset on every interaction
- Store timestamps in session_state, not cookies
- Clear all session state on timeout

---

## Audit Logging

**Append-Only Pattern (IRS Circular 230 / HIPAA Compliance):**

```python
import logging
import json
from datetime import UTC, datetime

# Setup audit logger (append-only for compliance)
logs_dir = Path(__file__).parent / "logs"
logs_dir.mkdir(exist_ok=True)

audit_logger = logging.getLogger("audit")
if not audit_logger.handlers:
    handler = logging.FileHandler(logs_dir / "audit.log", mode="a")
    handler.setFormatter(logging.Formatter("%(message)s"))
    audit_logger.addHandler(handler)
    audit_logger.setLevel(logging.INFO)

def audit_log(event: str, status: str, details: dict = None):
    """Write immutable audit log entry for compliance."""
    entry = {
        "timestamp": datetime.now(UTC).isoformat(),
        "session_id": st.session_state.get("session_id"),
        "user_id": st.session_state.get("current_user", {}).get("id"),
        "event": event,
        "status": status,
        "details": details or {},
    }
    audit_logger.info(json.dumps(entry))

# Log all authentication events
audit_log("staff_login", "success", {"user_id": user["id"], "email": user["email"]})
audit_log("staff_logout", "success", {"user_id": user["id"]})
audit_log("account_lockout", "triggered", {"email": email, "duration_minutes": 15})
audit_log("session_timeout", "idle", {"idle_minutes": 30})
```

**What to log:**
- Authentication events (login, logout, failures, lockouts)
- Data access (view, create, update, delete)
- File operations (upload, download, delete)
- Administrative actions (permission changes, user management)
- Security events (suspicious activity, blocked operations)

**Compliance requirements:**
- **IRS Circular 230**: All user actions in tax software
- **HIPAA**: PHI access, modifications, deletions
- **SOC 2**: Security-relevant events
- **GDPR**: Data subject rights requests

**Log format:**
- Timestamp (UTC, ISO 8601)
- Session ID (for tracking user session)
- User ID (authenticated user)
- Event type (action performed)
- Status (success, failure, blocked)
- Details (context-specific metadata)

---

## File Upload Validation

**Multi-Layer Validation (from tax-ai-chat):**

```python
# Configuration
MAX_FILE_SIZE_BYTES = 50 * 1024 * 1024  # 50 MB
ALLOWED_EXTENSIONS = {"pdf", "png", "jpg", "jpeg"}

def validate_file(
    content: bytes,
    filename: str,
    content_type: str,
) -> tuple[bool, str]:
    """
    Validate file for upload.

    Checks:
    1. File size is within limit
    2. Extension is allowed
    3. Content type matches extension
    4. Magic bytes match file type (PDF only)

    Returns:
        Tuple of (is_valid, error_message)
    """
    # 1. Size check
    if len(content) > MAX_FILE_SIZE_BYTES:
        max_mb = MAX_FILE_SIZE_BYTES // (1024 * 1024)
        actual_mb = len(content) / (1024 * 1024)
        return False, f"File size ({actual_mb:.1f}MB) exceeds {max_mb}MB limit"

    # 2. Extension check
    ext = filename.rsplit(".", 1)[-1].lower() if "." in filename else ""
    if ext not in ALLOWED_EXTENSIONS:
        allowed_list = ", ".join(sorted(ALLOWED_EXTENSIONS))
        return False, f"File type '.{ext}' not allowed. Allowed types: {allowed_list}"

    # 3. Content type verification (lenient - warn but don't reject)
    # Some clients send incorrect content types, so we trust extension if valid
    expected_types = {"pdf": ["application/pdf"], "png": ["image/png"],
                      "jpg": ["image/jpeg"], "jpeg": ["image/jpeg"]}
    if content_type not in expected_types.get(ext, []):
        # Log warning but don't reject - extension is valid
        print(f"Warning: content type {content_type} doesn't match extension {ext}")

    # 4. Magic byte check for PDFs (catch renamed executables)
    if ext == "pdf" and not content.startswith(b"%PDF"):
        return False, "File content does not match PDF format"

    return True, ""

def compute_file_hash(content: bytes) -> str:
    """Compute SHA-256 hash for duplicate detection."""
    import hashlib
    return hashlib.sha256(content).hexdigest()

# Usage in Streamlit
uploaded_file = st.file_uploader("Upload Document", type=["pdf", "png", "jpg", "jpeg"])
if uploaded_file:
    content = uploaded_file.read()

    is_valid, error = validate_file(content, uploaded_file.name, uploaded_file.type)
    if not is_valid:
        st.error(error)
        st.stop()

    file_hash = compute_file_hash(content)
    # Check for duplicates, upload to S3, etc.
```

**Validation layers:**
1. **Size limit**: Prevent DoS via large files (50MB typical)
2. **Extension whitelist**: Only allow known file types
3. **Content type check**: Verify MIME type matches extension
4. **Magic bytes**: Verify file signature (PDF: `%PDF`, PNG: `\x89PNG`, JPEG: `\xFF\xD8\xFF`)

**Magic byte patterns:**

| File Type | Magic Bytes | Hex |
|-----------|-------------|-----|
| PDF | `%PDF` | `25 50 44 46` |
| PNG | `\x89PNG\r\n\x1a\n` | `89 50 4E 47 0D 0A 1A 0A` |
| JPEG | `\xFF\xD8\xFF` | `FF D8 FF` |
| GIF | `GIF87a` or `GIF89a` | `47 49 46 38 37 61` or `47 49 46 38 39 61` |
| ZIP | `PK\x03\x04` | `50 4B 03 04` |

**Advanced validation:**

```python
def validate_image_safety(content: bytes) -> tuple[bool, str]:
    """Additional validation for images using PIL."""
    from PIL import Image
    import io

    try:
        img = Image.open(io.BytesIO(content))
        img.verify()  # Verify it's a valid image

        # Check dimensions
        if img.width > 10000 or img.height > 10000:
            return False, "Image dimensions too large (max 10000x10000)"

        # Check format matches extension
        if img.format not in ["PNG", "JPEG"]:
            return False, f"Unexpected image format: {img.format}"

        return True, ""
    except Exception as e:
        return False, f"Image validation failed: {str(e)}"
```

**Duplicate detection:**

```python
def check_duplicate(file_hash: str, user_id: str) -> bool:
    """Check if file hash already exists for this user."""
    # Query database for existing file with same hash
    return db.execute(
        "SELECT 1 FROM uploaded_files WHERE file_hash = %s AND user_id = %s",
        (file_hash, user_id)
    ).fetchone() is not None
```

---

## Security Checklist

Use this checklist when implementing security features:

**Authentication:**
- [ ] Database-backed authentication (no hardcoded passwords)
- [ ] Bcrypt for password hashing
- [ ] Lockout protection (5 failures, 15-minute lock)
- [ ] Dual timeout (12h absolute, 30m idle)
- [ ] Session state cleared on logout
- [ ] Audit logging for all auth events

**File Uploads:**
- [ ] Size limit enforced (50MB typical)
- [ ] Extension whitelist (no executables)
- [ ] Magic byte validation (PDF, images)
- [ ] Content type verification
- [ ] Duplicate detection (hash-based)
- [ ] Virus scanning (if applicable)
- [ ] Secure storage (S3 with encryption)

**Session Management:**
- [ ] Session IDs generated securely
- [ ] Session state cleared on timeout
- [ ] Activity timestamp updated on every interaction
- [ ] No sensitive data in session state
- [ ] Session cookies with secure flag

**Audit Logging:**
- [ ] All user actions logged
- [ ] Append-only log files
- [ ] UTC timestamps (ISO 8601)
- [ ] User ID and session ID in every entry
- [ ] Log rotation configured
- [ ] Logs stored securely (encryption at rest)

---

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [IRS Circular 230](https://www.irs.gov/pub/irs-pdf/pcir230.pdf) - Tax software requirements
- [HIPAA Security Rule](https://www.hhs.gov/hipaa/for-professionals/security/index.html)
- [ali-secure-coding skill](../ali-secure-coding/SKILL.md) - General security principles
