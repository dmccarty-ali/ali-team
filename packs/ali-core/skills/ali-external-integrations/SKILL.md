---
name: ali-external-integrations
description: |
  External API integration patterns for Aliunde projects. Use when:

  PLANNING: Designing integrations with external services, planning authentication
  flows, architecting webhook handlers, evaluating third-party APIs

  IMPLEMENTATION: Implementing OAuth 2.0, webhooks, payment processing (Stripe),
  identity verification (Persona), communication (Twilio, SES), document portals

  GUIDANCE: Asking about API integration best practices, OAuth flows, webhook
  security, retry patterns, rate limiting, how to integrate with specific services

  REVIEW: Reviewing integration code for security, checking webhook signature
  verification, validating error handling, auditing retry logic

  Do NOT use for CoPilot Studio connector configuration or CoPilot-specific
  OAuth patterns (use ali-copilot-studio instead)
---

# External Integrations

## When I Speak Up

This skill activates when the conversation involves:

**Planning/Design:**
- Designing integration with external APIs
- Planning OAuth authentication flows
- Architecting webhook processing
- Evaluating third-party service options

**Implementation:**
- Implementing OAuth 2.0 (any grant type)
- Building webhook handlers
- Integrating payment processing (Stripe)
- Adding identity verification (Persona)
- Implementing SMS/voice (Twilio)
- Building document portal integrations

**Guidance/Best Practices:**
- Asking about OAuth grant types
- Asking about webhook best practices
- Asking about retry and rate limiting
- Asking how to integrate specific services

**Review/Validation:**
- Reviewing webhook signature verification
- Checking OAuth token handling
- Validating error handling and retries
- Auditing API key security

---

## Key Principles

- **Verify all webhooks**: Always verify signatures before processing
- **Idempotent handlers**: Webhooks may be delivered multiple times
- **Retry with backoff**: External APIs fail; retry with exponential backoff
- **Respect rate limits**: Track limits, back off on 429
- **Secure credentials**: Never log tokens; store in secrets manager
- **Circuit breaker**: Stop calling failing services temporarily
- **Health checks**: Monitor external service availability
- **Timeout everything**: Never wait forever for external responses

---

## OAuth 2.0 Patterns

### Authorization Code Flow (User Login)

```python
from urllib.parse import urlencode
import httpx
import secrets

class OAuthClient:
    def __init__(self, client_id: str, client_secret: str, redirect_uri: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.redirect_uri = redirect_uri

    def get_authorization_url(self, scopes: list[str]) -> tuple[str, str]:
        """Generate authorization URL and state token."""
        state = secrets.token_urlsafe(32)
        params = {
            'client_id': self.client_id,
            'redirect_uri': self.redirect_uri,
            'response_type': 'code',
            'scope': ' '.join(scopes),
            'state': state,
        }
        url = f"https://provider.com/oauth/authorize?{urlencode(params)}"
        return url, state  # Store state in session for verification

    async def exchange_code(self, code: str) -> dict:
        """Exchange authorization code for tokens."""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                'https://provider.com/oauth/token',
                data={
                    'grant_type': 'authorization_code',
                    'code': code,
                    'redirect_uri': self.redirect_uri,
                    'client_id': self.client_id,
                    'client_secret': self.client_secret,
                },
            )
            response.raise_for_status()
            return response.json()  # {access_token, refresh_token, expires_in}
```

### Client Credentials Flow (Service-to-Service)

```python
import httpx
from datetime import datetime, timedelta
from functools import lru_cache

class ServiceClient:
    def __init__(self, client_id: str, client_secret: str, token_url: str):
        self.client_id = client_id
        self.client_secret = client_secret
        self.token_url = token_url
        self._token = None
        self._token_expires = None

    async def get_token(self) -> str:
        """Get access token, refreshing if needed."""
        if self._token and self._token_expires > datetime.utcnow():
            return self._token

        async with httpx.AsyncClient() as client:
            response = await client.post(
                self.token_url,
                data={
                    'grant_type': 'client_credentials',
                    'client_id': self.client_id,
                    'client_secret': self.client_secret,
                },
            )
            response.raise_for_status()
            data = response.json()

            self._token = data['access_token']
            # Refresh 5 minutes before expiry
            self._token_expires = datetime.utcnow() + timedelta(
                seconds=data['expires_in'] - 300
            )
            return self._token

    async def api_call(self, method: str, url: str, **kwargs) -> dict:
        """Make authenticated API call."""
        token = await self.get_token()
        async with httpx.AsyncClient() as client:
            response = await client.request(
                method,
                url,
                headers={'Authorization': f'Bearer {token}'},
                **kwargs,
            )
            response.raise_for_status()
            return response.json()
```

### Token Refresh

```python
async def refresh_token(refresh_token: str) -> dict:
    """Refresh an expired access token."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            'https://provider.com/oauth/token',
            data={
                'grant_type': 'refresh_token',
                'refresh_token': refresh_token,
                'client_id': CLIENT_ID,
                'client_secret': CLIENT_SECRET,
            },
        )
        if response.status_code == 400:
            # Refresh token expired - user must re-authenticate
            raise TokenExpiredError("Refresh token expired")
        response.raise_for_status()
        return response.json()
```

---

## Webhook Patterns

### Signature Verification

```python
import hmac
import hashlib
from fastapi import Request, HTTPException

async def verify_webhook_signature(
    request: Request,
    secret: str,
    header_name: str = 'X-Signature'
) -> bytes:
    """Verify webhook signature and return raw body."""
    body = await request.body()
    signature = request.headers.get(header_name)

    if not signature:
        raise HTTPException(401, "Missing signature header")

    expected = hmac.new(
        secret.encode(),
        body,
        hashlib.sha256
    ).hexdigest()

    # Use compare_digest to prevent timing attacks
    if not hmac.compare_digest(f"sha256={expected}", signature):
        raise HTTPException(401, "Invalid signature")

    return body
```

### Idempotent Processing

```python
from fastapi import Request, HTTPException
import json

# Track processed webhook IDs (use Redis in production)
processed_webhooks: set[str] = set()

async def handle_webhook(request: Request):
    """Process webhook idempotently."""
    body = await verify_webhook_signature(request, WEBHOOK_SECRET)
    event = json.loads(body)

    event_id = event.get('id')
    if not event_id:
        raise HTTPException(400, "Missing event ID")

    # Check if already processed
    if event_id in processed_webhooks:
        return {"status": "already_processed"}

    try:
        # Process the event
        await process_event(event)

        # Mark as processed AFTER successful processing
        processed_webhooks.add(event_id)

        return {"status": "processed"}
    except Exception as e:
        # Don't mark as processed - allow retry
        raise
```

### Webhook Handler Structure

```python
from fastapi import APIRouter, Request, BackgroundTasks

router = APIRouter()

@router.post("/webhooks/stripe")
async def stripe_webhook(request: Request, background_tasks: BackgroundTasks):
    """Handle Stripe webhooks."""
    body = await verify_stripe_signature(request)
    event = json.loads(body)

    # Acknowledge receipt quickly (within 5 seconds)
    # Process in background
    background_tasks.add_task(process_stripe_event, event)

    return {"received": True}

async def process_stripe_event(event: dict):
    """Process Stripe event in background."""
    event_type = event['type']

    handlers = {
        'payment_intent.succeeded': handle_payment_success,
        'payment_intent.payment_failed': handle_payment_failure,
        'customer.subscription.updated': handle_subscription_update,
    }

    handler = handlers.get(event_type)
    if handler:
        await handler(event['data']['object'])
```

---

## Retry Patterns

### Exponential Backoff with Jitter

```python
import asyncio
import random
from functools import wraps

def retry_with_backoff(
    max_attempts: int = 3,
    base_delay: float = 1.0,
    max_delay: float = 60.0,
    retryable_exceptions: tuple = (httpx.HTTPStatusError, httpx.ConnectError),
):
    """Decorator for retry with exponential backoff and jitter."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_attempts):
                try:
                    return await func(*args, **kwargs)
                except retryable_exceptions as e:
                    last_exception = e
                    if attempt == max_attempts - 1:
                        raise

                    # Exponential backoff with jitter
                    delay = min(base_delay * (2 ** attempt), max_delay)
                    delay *= (0.5 + random.random())  # Add jitter
                    await asyncio.sleep(delay)

            raise last_exception
        return wrapper
    return decorator

@retry_with_backoff(max_attempts=3, base_delay=1.0)
async def call_external_api(url: str, data: dict) -> dict:
    async with httpx.AsyncClient(timeout=30.0) as client:
        response = await client.post(url, json=data)
        response.raise_for_status()
        return response.json()
```

### Circuit Breaker

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
import asyncio

class CircuitState(Enum):
    CLOSED = "closed"      # Normal operation
    OPEN = "open"          # Failing, reject requests
    HALF_OPEN = "half_open"  # Testing recovery

@dataclass
class CircuitBreaker:
    failure_threshold: int = 5
    recovery_timeout: float = 30.0
    _state: CircuitState = field(default=CircuitState.CLOSED, init=False)
    _failures: int = field(default=0, init=False)
    _last_failure: datetime = field(default=None, init=False)

    async def call(self, func, *args, **kwargs):
        if self._state == CircuitState.OPEN:
            if datetime.utcnow() - self._last_failure > timedelta(seconds=self.recovery_timeout):
                self._state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenError("Circuit breaker is open")

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        self._failures = 0
        self._state = CircuitState.CLOSED

    def _on_failure(self):
        self._failures += 1
        self._last_failure = datetime.utcnow()
        if self._failures >= self.failure_threshold:
            self._state = CircuitState.OPEN

# Usage
stripe_circuit = CircuitBreaker(failure_threshold=3, recovery_timeout=60)

async def charge_card(amount: int) -> dict:
    return await stripe_circuit.call(stripe_api.create_payment_intent, amount=amount)
```

---

## Rate Limiting

### Handling 429 Responses

```python
import asyncio
import httpx

async def api_call_with_rate_limit(url: str) -> dict:
    """Handle rate limiting with automatic retry."""
    async with httpx.AsyncClient() as client:
        for attempt in range(5):
            response = await client.get(url)

            if response.status_code == 429:
                # Check for Retry-After header
                retry_after = response.headers.get('Retry-After')
                if retry_after:
                    wait_time = int(retry_after)
                else:
                    wait_time = 2 ** attempt  # Exponential backoff

                await asyncio.sleep(wait_time)
                continue

            response.raise_for_status()
            return response.json()

    raise RateLimitExceededError("Rate limit exceeded after retries")
```

### Token Bucket Rate Limiter

```python
import asyncio
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class TokenBucket:
    """Client-side rate limiter."""
    rate: float  # Tokens per second
    capacity: int  # Max tokens
    _tokens: float = field(init=False)
    _last_update: datetime = field(init=False)

    def __post_init__(self):
        self._tokens = self.capacity
        self._last_update = datetime.utcnow()

    async def acquire(self, tokens: int = 1):
        """Wait until tokens available."""
        while True:
            self._refill()
            if self._tokens >= tokens:
                self._tokens -= tokens
                return
            # Wait for tokens to refill
            wait_time = (tokens - self._tokens) / self.rate
            await asyncio.sleep(wait_time)

    def _refill(self):
        now = datetime.utcnow()
        elapsed = (now - self._last_update).total_seconds()
        self._tokens = min(self.capacity, self._tokens + elapsed * self.rate)
        self._last_update = now

# Usage: 10 requests per second
rate_limiter = TokenBucket(rate=10, capacity=10)

async def call_api():
    await rate_limiter.acquire()
    return await make_request()
```

---

## Stripe Patterns

### Payment Intent

```python
import stripe

stripe.api_key = get_secret('STRIPE_SECRET_KEY')

async def create_payment(
    amount: int,  # In cents
    customer_id: str,
    idempotency_key: str,
) -> stripe.PaymentIntent:
    """Create a payment intent with idempotency."""
    return stripe.PaymentIntent.create(
        amount=amount,
        currency='usd',
        customer=customer_id,
        automatic_payment_methods={'enabled': True},
        idempotency_key=idempotency_key,  # Prevents duplicate charges
        metadata={
            'client_id': customer_id,
            'created_by': 'api',
        },
    )
```

### Stripe Webhook Verification

```python
import stripe
from fastapi import Request, HTTPException

async def verify_stripe_signature(request: Request) -> bytes:
    """Verify Stripe webhook signature."""
    body = await request.body()
    sig_header = request.headers.get('Stripe-Signature')

    try:
        stripe.Webhook.construct_event(
            body,
            sig_header,
            STRIPE_WEBHOOK_SECRET,
        )
        return body
    except stripe.error.SignatureVerificationError:
        raise HTTPException(401, "Invalid signature")
```

### Handling Stripe Events

```python
async def handle_payment_success(payment_intent: dict):
    """Handle successful payment."""
    client_id = payment_intent['metadata']['client_id']
    amount = payment_intent['amount']

    # Update your database
    await update_payment_status(
        client_id=client_id,
        stripe_payment_id=payment_intent['id'],
        amount=amount / 100,  # Convert cents to dollars
        status='completed',
    )

    # Send confirmation
    await send_payment_confirmation(client_id, amount / 100)

async def handle_payment_failure(payment_intent: dict):
    """Handle failed payment."""
    client_id = payment_intent['metadata']['client_id']
    error = payment_intent.get('last_payment_error', {})

    await update_payment_status(
        client_id=client_id,
        stripe_payment_id=payment_intent['id'],
        status='failed',
        error_message=error.get('message'),
    )

    await notify_payment_failure(client_id, error.get('message'))
```

---

## Persona Patterns

### Creating an Inquiry

```python
import httpx

PERSONA_API_KEY = get_secret('PERSONA_API_KEY')

async def create_inquiry(
    reference_id: str,
    template_id: str,
) -> dict:
    """Create Persona identity verification inquiry."""
    async with httpx.AsyncClient() as client:
        response = await client.post(
            'https://withpersona.com/api/v1/inquiries',
            headers={
                'Authorization': f'Bearer {PERSONA_API_KEY}',
                'Persona-Version': '2023-01-05',
            },
            json={
                'data': {
                    'attributes': {
                        'inquiry-template-id': template_id,
                        'reference-id': reference_id,
                    }
                }
            },
        )
        response.raise_for_status()
        return response.json()
```

### Handling Persona Webhooks

```python
async def handle_persona_webhook(event: dict):
    """Process Persona verification events."""
    event_type = event['data']['attributes']['name']

    if event_type == 'inquiry.completed':
        inquiry = event['data']['attributes']['payload']['data']
        status = inquiry['attributes']['status']
        reference_id = inquiry['attributes']['reference-id']

        if status == 'approved':
            await mark_client_verified(reference_id)
        elif status == 'declined':
            await handle_verification_declined(reference_id)
        elif status == 'needs_review':
            await queue_for_manual_review(reference_id)
```

---

## Twilio Patterns

### Sending SMS

```python
from twilio.rest import Client

twilio_client = Client(
    get_secret('TWILIO_ACCOUNT_SID'),
    get_secret('TWILIO_AUTH_TOKEN'),
)

def send_sms(to: str, message: str) -> str:
    """Send SMS and return message SID."""
    msg = twilio_client.messages.create(
        body=message,
        from_=TWILIO_PHONE_NUMBER,
        to=to,
    )
    return msg.sid

def send_verification_code(phone: str) -> str:
    """Send verification code via Twilio Verify."""
    verification = twilio_client.verify.v2.services(
        TWILIO_VERIFY_SERVICE_SID
    ).verifications.create(
        to=phone,
        channel='sms',
    )
    return verification.sid

def check_verification_code(phone: str, code: str) -> bool:
    """Verify the code entered by user."""
    try:
        verification_check = twilio_client.verify.v2.services(
            TWILIO_VERIFY_SERVICE_SID
        ).verification_checks.create(
            to=phone,
            code=code,
        )
        return verification_check.status == 'approved'
    except Exception:
        return False
```

---

## Google Workspace Patterns

### Service Account Authentication

```python
from google.oauth2 import service_account
from googleapiclient.discovery import build

SCOPES = [
    'https://www.googleapis.com/auth/calendar',
    'https://www.googleapis.com/auth/drive',
]

def get_google_service(service_name: str, version: str):
    """Get authenticated Google API service."""
    credentials = service_account.Credentials.from_service_account_file(
        'service-account.json',
        scopes=SCOPES,
    )
    # For domain-wide delegation
    delegated = credentials.with_subject('admin@company.com')

    return build(service_name, version, credentials=delegated)

# Usage
calendar = get_google_service('calendar', 'v3')
drive = get_google_service('drive', 'v3')
```

### Calendar Event

```python
def create_calendar_event(
    summary: str,
    start: datetime,
    end: datetime,
    attendees: list[str],
) -> dict:
    """Create Google Calendar event."""
    calendar = get_google_service('calendar', 'v3')

    event = {
        'summary': summary,
        'start': {'dateTime': start.isoformat(), 'timeZone': 'America/New_York'},
        'end': {'dateTime': end.isoformat(), 'timeZone': 'America/New_York'},
        'attendees': [{'email': email} for email in attendees],
        'conferenceData': {
            'createRequest': {'requestId': str(uuid.uuid4())},
        },
    }

    return calendar.events().insert(
        calendarId='primary',
        body=event,
        conferenceDataVersion=1,  # Create Google Meet link
        sendUpdates='all',
    ).execute()
```

---

## Anthropic API / Claude Integration

Direct integration with Claude API:

```python
import anthropic
import json

class AnthropicService:
    """Service for Claude API integration."""

    def __init__(self, api_key: str):
        self.client = anthropic.Anthropic(api_key=api_key)

    def classify_document(
        self,
        document_text: str,
        categories: list[str],
        max_tokens: int = 500
    ) -> dict:
        """Classify document into predefined categories."""
        prompt = f"""Classify the following document into one of these categories: {', '.join(categories)}

Document:
{document_text}

Respond with JSON: {{"category": "...", "confidence": "high/medium/low", "reasoning": "..."}}"""

        message = self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=max_tokens,
            messages=[{"role": "user", "content": prompt}]
        )
        return json.loads(message.content[0].text)

    def extract_structured_data(
        self,
        document_text: str,
        schema: dict
    ) -> dict:
        """Extract structured data from document using schema."""
        message = self.client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=2000,
            system="Extract data according to the schema. Return valid JSON only.",
            messages=[{
                "role": "user",
                "content": f"Schema: {json.dumps(schema)}\n\nDocument:\n{document_text}"
            }]
        )
        return json.loads(message.content[0].text)

    async def stream_response(self, prompt: str, on_chunk: callable):
        """Stream responses for real-time display."""
        with self.client.messages.stream(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}]
        ) as stream:
            for text in stream.text_stream:
                await on_chunk(text)
```

### Claude with Tool Use

```python
def process_with_tools(self, user_request: str) -> dict:
    """Use Claude's tool use for structured actions."""
    tools = [
        {
            "name": "lookup_client",
            "description": "Look up client by email or ID",
            "input_schema": {
                "type": "object",
                "properties": {
                    "identifier": {"type": "string"},
                    "identifier_type": {"type": "string", "enum": ["email", "id"]}
                },
                "required": ["identifier", "identifier_type"]
            }
        }
    ]

    response = self.client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        tools=tools,
        messages=[{"role": "user", "content": user_request}]
    )

    for block in response.content:
        if block.type == "tool_use":
            result = self._execute_tool(block.name, block.input)
            return {"tool": block.name, "input": block.input, "result": result}

    return {"response": response.content[0].text}
```

---

## Email Service Integration

```python
import boto3
from dataclasses import dataclass
from typing import Optional

@dataclass
class EmailMessage:
    to: str
    subject: str
    body_html: str
    body_text: str
    reply_to: Optional[str] = None

class EmailService:
    """Email service using AWS SES."""

    def __init__(self, sender_email: str, configuration_set: str = None):
        self.ses = boto3.client('ses')
        self.sender = sender_email
        self.config_set = configuration_set

    def send(self, message: EmailMessage) -> dict:
        """Send a single email."""
        params = {
            'Source': self.sender,
            'Destination': {'ToAddresses': [message.to]},
            'Message': {
                'Subject': {'Data': message.subject},
                'Body': {
                    'Html': {'Data': message.body_html},
                    'Text': {'Data': message.body_text}
                }
            }
        }
        if message.reply_to:
            params['ReplyToAddresses'] = [message.reply_to]
        if self.config_set:
            params['ConfigurationSetName'] = self.config_set

        response = self.ses.send_email(**params)
        return {'message_id': response['MessageId']}

    def send_templated(self, to: str, template: str, data: dict) -> dict:
        """Send using SES template."""
        import json
        response = self.ses.send_templated_email(
            Source=self.sender,
            Destination={'ToAddresses': [to]},
            Template=template,
            TemplateData=json.dumps(data)
        )
        return {'message_id': response['MessageId']}
```

### Email Intake Processing

```python
class EmailIntakeService:
    """Process incoming emails from SES/S3."""

    def process_incoming_email(self, s3_key: str) -> dict:
        """Process email stored in S3 by SES."""
        import email
        from email import policy

        response = self.s3.get_object(Bucket=self.bucket, Key=s3_key)
        msg = email.message_from_bytes(response['Body'].read(), policy=policy.default)

        result = {
            'from': msg['From'],
            'to': msg['To'],
            'subject': msg['Subject'],
            'attachments': []
        }

        for part in msg.walk():
            if part.get_content_disposition() == 'attachment':
                result['attachments'].append({
                    'filename': part.get_filename(),
                    'content': part.get_payload(decode=True)
                })
        return result
```

---

## Batch API Requests

```python
import asyncio
from dataclasses import dataclass

@dataclass
class BatchResult:
    successful: list
    failed: list
    total: int

class BatchProcessor:
    """Process API requests with concurrency control."""

    def __init__(self, max_concurrent: int = 10, batch_size: int = 100):
        self.max_concurrent = max_concurrent
        self.batch_size = batch_size
        self.semaphore = asyncio.Semaphore(max_concurrent)

    async def process_batch(self, items: list, processor: callable) -> BatchResult:
        """Process items with controlled concurrency."""
        successful, failed = [], []

        async def process_one(item):
            async with self.semaphore:
                try:
                    result = await processor(item)
                    successful.append({'item': item, 'result': result})
                except Exception as e:
                    failed.append({'item': item, 'error': str(e)})

        for i in range(0, len(items), self.batch_size):
            batch = items[i:i + self.batch_size]
            await asyncio.gather(*[process_one(item) for item in batch])

        return BatchResult(successful=successful, failed=failed, total=len(items))
```

---

## Idempotency Patterns

```python
import hashlib
import json

class IdempotencyService:
    """Manage idempotency for external API calls."""

    def __init__(self, cache_backend):
        self.cache = cache_backend

    def generate_key(self, operation: str, params: dict) -> str:
        """Generate deterministic idempotency key."""
        param_str = json.dumps(params, sort_keys=True)
        return hashlib.sha256(f"{operation}:{param_str}".encode()).hexdigest()[:32]

    async def execute_idempotent(
        self,
        idempotency_key: str,
        operation: callable
    ) -> dict:
        """Execute operation with idempotency guarantee."""
        cached = await self.cache.get(idempotency_key)
        if cached:
            return {'status': 'cached', 'result': cached}

        result = await operation()
        await self.cache.set(idempotency_key, result, ttl=86400)
        return {'status': 'executed', 'result': result}

# Usage with Stripe
async def create_payment_idempotent(client_id: str, amount: int):
    key = idempotency_service.generate_key(
        'stripe_payment',
        {'client_id': client_id, 'amount': amount, 'date': str(date.today())}
    )
    # Stripe also accepts idempotency_key in API call
    return stripe.PaymentIntent.create(
        amount=amount,
        customer=client_id,
        idempotency_key=key
    )
```

---

## Health Checks

```python
import asyncio
import httpx
from dataclasses import dataclass

@dataclass
class ServiceHealth:
    name: str
    healthy: bool
    latency_ms: float
    error: str = None

async def check_service(name: str, url: str, timeout: float = 5.0) -> ServiceHealth:
    """Check if external service is healthy."""
    try:
        start = asyncio.get_event_loop().time()
        async with httpx.AsyncClient(timeout=timeout) as client:
            response = await client.get(url)
            latency = (asyncio.get_event_loop().time() - start) * 1000

            return ServiceHealth(
                name=name,
                healthy=response.status_code < 500,
                latency_ms=latency,
            )
    except Exception as e:
        return ServiceHealth(
            name=name,
            healthy=False,
            latency_ms=0,
            error=str(e),
        )

async def check_all_services() -> list[ServiceHealth]:
    """Check all external service dependencies."""
    checks = [
        check_service('stripe', 'https://api.stripe.com/v1/'),
        check_service('persona', 'https://withpersona.com/api/v1/'),
        check_service('twilio', 'https://api.twilio.com/'),
    ]
    return await asyncio.gather(*checks)
```

---

## Anti-Patterns

| Anti-Pattern | Why It's Bad | Do This Instead |
|--------------|--------------|-----------------|
| No webhook signature verification | Accepts spoofed requests | Always verify signatures |
| Logging tokens/secrets | Security breach via logs | Log IDs only, mask secrets |
| No idempotency keys | Duplicate operations | Use idempotency keys |
| Sync webhook processing | Timeouts, retries | Acknowledge fast, process async |
| No retry logic | Single failures break flow | Retry with backoff |
| Ignoring rate limits | Getting blocked | Respect limits, back off on 429 |
| Hardcoded credentials | Security risk | Use secrets manager |
| No circuit breaker | Cascade failures | Break circuit on repeated fails |

---

## Your Codebase

| Purpose | Location |
|---------|----------|
| OAuth implementations | `ali-ai-acctg/app/auth/oauth.py` |
| Webhook handlers | `ali-ai-acctg/app/webhooks/` |
| Stripe integration | `ali-ai-acctg/app/services/stripe.py` |
| External service clients | `ali-ai-acctg/app/services/` |
| Integration configs | `ali-ai-acctg/app/config/integrations.py` |

---

## References

- [OAuth 2.0 RFC 6749](https://tools.ietf.org/html/rfc6749)
- [Stripe API Documentation](https://stripe.com/docs/api)
- [Persona API Documentation](https://docs.withpersona.com/)
- [Twilio API Documentation](https://www.twilio.com/docs/usage/api)
- [Google Workspace APIs](https://developers.google.com/workspace)
