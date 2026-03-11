# ali-sap-mdg - Integration Patterns Reference

This reference contains detailed integration configuration examples for MDI, ALE/IDoc, REST/OData, and event-driven patterns.

---

## SAP Master Data Integration (MDI) Details

**Cloud-native integration service (recommended for new implementations):**

### Capabilities

1. **Real-time or near-real-time replication**
   - Change Data Capture (CDC) from MDG Hub
   - Push changes to subscribers within seconds
   - Configurable replication frequency (real-time, hourly, daily)

2. **REST/OData APIs for non-SAP systems**
   - Standard REST endpoints for consuming applications
   - OAuth 2.0 authentication
   - JSON and XML payload support

3. **Pre-built connectors**
   - SAP S/4HANA (on-premise and cloud)
   - SAP ECC
   - SAP CRM
   - Third-party systems via generic connectors

4. **Transformation and Mapping**
   - Field-level mapping (MDG fields → target system fields)
   - Value transformation (e.g., currency conversion)
   - Data enrichment (add calculated fields)

5. **Part of SAP Integration Suite**
   - Integrated with Cloud Integration (CPI)
   - Monitoring and alerting via SAP BTP Cockpit

### Example MDI Flow

```
MDG Hub (Change Request Activated)
    ↓
MDI Service (Capture change)
    ↓
Transformation Layer
  - Map MDG Business Partner → Target Customer
  - Convert currency EUR → USD
  - Add derived fields (e.g., customer segment)
    ↓
Target Systems (parallel distribution)
  - S/4HANA Production
  - Salesforce CRM
  - Snowflake Data Warehouse
```

### MDI Configuration Example

**Scenario:** Distribute customer master from MDG Hub to Salesforce

```yaml
# MDI Configuration (SAP BTP Integration Suite)
integration_flow:
  name: MDG_Customer_to_Salesforce
  trigger: MDG change request activated (entity: Customer)

  source:
    system: MDG_HUB_PROD
    entity: Business Partner (BUT000)
    filter: BP_TYPE = '1' (Customer only)

  transformation:
    mappings:
      - MDG.BP_ID → Salesforce.AccountId
      - MDG.NAME → Salesforce.Name
      - MDG.TAX_ID → Salesforce.TaxNumber__c
      - MDG.STREET → Salesforce.BillingStreet
      - MDG.CITY → Salesforce.BillingCity
      - MDG.POSTAL_CODE → Salesforce.BillingPostalCode
      - MDG.COUNTRY → Salesforce.BillingCountry

    enrichment:
      - Add: Salesforce.RecordTypeId = 'Customer'
      - Add: Salesforce.Source__c = 'MDG'
      - Add: Salesforce.LastModifiedDate = NOW()

  destination:
    system: Salesforce_Production
    api: REST (Salesforce Composite API)
    auth: OAuth 2.0 (Connected App)
    endpoint: /services/data/v52.0/composite/sobjects

  error_handling:
    on_failure: Retry 3 times (exponential backoff)
    notification: Email to data-team@example.com
    log: SAP BTP Monitoring
```

---

## ALE/IDoc Details

**Traditional SAP-to-SAP distribution (legacy but stable):**

### Common IDoc Types

| IDoc Type | Purpose | Direction | Frequency |
|-----------|---------|-----------|-----------|
| **DEBMAS** | Customer master distribution | MDG → ECC/S/4 | Near-real-time or batch |
| **MATMAS** | Material master distribution | MDG → ECC/S/4 | Near-real-time or batch |
| **CREMAS** | Vendor master distribution | MDG → ECC/S/4 | Near-real-time or batch |
| **ADRMAS** | Address distribution | MDG → Multiple | Batch (nightly) |

### IDoc Processing Flow

```
1. MDG Hub (Change Request Activated)
   ↓
2. USMD_IDOC_CREATE (Background job creates IDoc)
   ↓
3. ALE Layer (Routes IDoc based on distribution model)
   ↓
4. Target System (Receives IDoc, processes via standard inbound function)
   ↓
5. Application Layer (Updates database tables)
```

### ALE Configuration Example

**Scenario:** Distribute customer master from MDG Hub to multiple S/4HANA systems

**Step 1: Define Logical Systems (Transaction: SALE)**
```
MDG_HUB_PROD  → MDG Hub (sender)
S4H_PROD_US   → S/4HANA US (receiver)
S4H_PROD_EU   → S/4HANA EU (receiver)
```

**Step 2: Configure Distribution Model (Transaction: BD64)**
```
Model: CUSTOMER_MASTER_DIST
  Sender: MDG_HUB_PROD
  Message Type: DEBMAS
  Receivers:
    - S4H_PROD_US (filter: COUNTRY = 'US')
    - S4H_PROD_EU (filter: COUNTRY IN ('DE', 'FR', 'GB'))
```

**Step 3: Create Output Control (Transaction: USMD_IDOC_CREATE)**
```
Change Request Type: Customer Change
IDoc Type: DEBMAS06
Output Control: Immediate (near-real-time) or Batch (nightly job)
Filter: Only send if status = 'ACTIVATED'
```

**Step 4: Monitor IDoc Processing (Transaction: WE02, WE05)**
```
WE02: IDoc display (individual IDocs)
WE05: IDoc list (bulk monitoring)

Status Codes:
  03 = Data passed to port OK (success)
  51 = Application document not posted (error)
  64 = Error during IDoc processing (error)
```

---

## REST/OData API Examples

**For non-SAP system integration:**

### OData Service Configuration

**MDG exposes standard OData services for master data:**

**Available Services (Transaction: /IWFND/MAINT_SERVICE):**
- USMD_CUSTOMER_SRV - Customer master OData service
- USMD_MATERIAL_SRV - Material master OData service
- USMD_SUPPLIER_SRV - Supplier master OData service

### Example 1: Query Customers via OData

```http
GET /sap/opu/odata/sap/USMD_CUSTOMER_SRV/CustomerSet
Host: mdg-hub.example.com:8000
Authorization: Bearer <OAuth_token>
Accept: application/json

Response:
{
  "d": {
    "results": [
      {
        "CustomerId": "0000100001",
        "Name": "Acme Corporation",
        "TaxId": "12-3456789",
        "Street": "123 Main Street",
        "City": "Anytown",
        "PostalCode": "90210",
        "Country": "US",
        "CreditLimit": 1000000.00,
        "Currency": "USD",
        "LastModified": "/Date(1705881600000)/"
      },
      ...
    ]
  }
}
```

### Example 2: Filter Customers by Country

```http
GET /sap/opu/odata/sap/USMD_CUSTOMER_SRV/CustomerSet?$filter=Country eq 'US'
Authorization: Bearer <OAuth_token>
Accept: application/json
```

### Example 3: Create Customer via OData (POST)

```http
POST /sap/opu/odata/sap/USMD_CUSTOMER_SRV/CustomerSet
Host: mdg-hub.example.com:8000
Authorization: Bearer <OAuth_token>
Content-Type: application/json

{
  "Name": "New Customer Inc",
  "TaxId": "98-7654321",
  "Street": "456 Oak Avenue",
  "City": "Springfield",
  "PostalCode": "62701",
  "Country": "US",
  "CreditLimit": 500000.00,
  "Currency": "USD"
}

Response:
{
  "d": {
    "CustomerId": "0000100042",
    "Name": "New Customer Inc",
    ...
    "__metadata": {
      "uri": ".../CustomerSet('0000100042')"
    }
  }
}
```

### Example 4: Update Customer via OData (PATCH)

```http
PATCH /sap/opu/odata/sap/USMD_CUSTOMER_SRV/CustomerSet('0000100042')
Authorization: Bearer <OAuth_token>
Content-Type: application/json
If-Match: *

{
  "CreditLimit": 750000.00
}

Response: 204 No Content (success)
```

### Authentication Configuration

**OAuth 2.0 Setup:**

1. **Create OAuth Client (Transaction: /IWFND/MAINT_SERVICE)**
   ```
   Client ID: mdg_api_client
   Client Secret: <generated>
   Grant Type: Client Credentials
   Scope: USMD_CUSTOMER_SRV
   ```

2. **Obtain Access Token:**
   ```http
   POST /sap/bc/sec/oauth2/token
   Content-Type: application/x-www-form-urlencoded
   Authorization: Basic <base64(client_id:client_secret)>

   grant_type=client_credentials&scope=USMD_CUSTOMER_SRV

   Response:
   {
     "access_token": "eyJhbGciOiJSUzI1NiIs...",
     "token_type": "Bearer",
     "expires_in": 3600
   }
   ```

3. **Use Token in API Calls:**
   ```http
   GET /sap/opu/odata/sap/USMD_CUSTOMER_SRV/CustomerSet
   Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
   ```

---

## Kafka Integration (Event-Driven)

**For real-time event streaming:**

### Architecture

```
MDG Hub
  ↓ (Change Request Activated)
SAP Event Mesh / Custom Integration
  ↓ (Publish event to Kafka topic)
Kafka Topic: mdg.customer.changes
  ↓
Consumers (subscribe):
  - Microservice 1 (Customer 360 app)
  - Microservice 2 (Marketing automation)
  - Microservice 3 (Data warehouse CDC)
  - Microservice 4 (Analytics platform)
```

### Event Payload Example

**Topic:** `mdg.customer.changes`

**Message:**
```json
{
  "event_id": "evt_20260216_123456",
  "event_type": "CUSTOMER_UPDATED",
  "timestamp": "2026-02-16T10:30:00Z",
  "source": "MDG_HUB_PROD",
  "entity": "CUSTOMER",
  "entity_id": "0000100001",
  "change_request_id": "CR_12345",
  "changed_by": "USER_JOE",
  "changes": [
    {
      "field": "CREDIT_LIMIT",
      "old_value": "1000000.00",
      "new_value": "1500000.00"
    },
    {
      "field": "ADDRESS.STREET",
      "old_value": "123 Main Street",
      "new_value": "456 Oak Avenue"
    }
  ],
  "full_record": {
    "customer_id": "0000100001",
    "name": "Acme Corporation",
    "tax_id": "12-3456789",
    "credit_limit": 1500000.00,
    "currency": "USD",
    "address": {
      "street": "456 Oak Avenue",
      "city": "Anytown",
      "postal_code": "90210",
      "country": "US"
    }
  }
}
```

### SAP Event Mesh Configuration

**Option 1: SAP Event Mesh (BTP service)**
```yaml
service: SAP Event Mesh
plan: default (standard)

queue: mdg-customer-changes
topic: mdg/customer/changes

webhook:
  url: https://mdg-hub.example.com/events/webhook
  auth: OAuth 2.0
  events:
    - CHANGE_REQUEST_ACTIVATED
  filter: entity_type = 'CUSTOMER'
```

**Option 2: Custom Kafka Integration**
```python
# Python example: Publish MDG change events to Kafka
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka.example.com:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def publish_customer_change(change_request):
    event = {
        'event_id': f"evt_{change_request.id}",
        'event_type': 'CUSTOMER_UPDATED',
        'timestamp': change_request.activated_at.isoformat(),
        'entity_id': change_request.entity_id,
        'changes': change_request.changes,
        'full_record': change_request.to_dict()
    }

    producer.send('mdg.customer.changes', value=event)
    producer.flush()
```

### Consumer Example

**Microservice subscribing to customer changes:**

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'mdg.customer.changes',
    bootstrap_servers=['kafka.example.com:9092'],
    group_id='customer-360-service',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    event = message.value
    print(f"Received event: {event['event_type']}")

    if event['event_type'] == 'CUSTOMER_UPDATED':
        update_customer_360(event['entity_id'], event['full_record'])
    elif event['event_type'] == 'CUSTOMER_CREATED':
        create_customer_360(event['full_record'])
```

---

## Integration Pattern Comparison

| Pattern | Latency | Complexity | Cost | Use Case |
|---------|---------|------------|------|----------|
| **MDI (Cloud)** | <1 min | Medium | Medium | Modern cloud integrations, real-time requirements |
| **ALE/IDoc** | 5-30 min | Low | Low | SAP-to-SAP, legacy systems, batch acceptable |
| **REST/OData** | Real-time | Medium | Low | Custom applications, API-first architecture |
| **Kafka/Events** | <1 sec | High | Medium-High | Microservices, event-driven architecture, real-time analytics |

---

## Integration Monitoring

### Key Metrics to Track

| Metric | Target | Monitoring Tool |
|--------|--------|-----------------|
| **MDI Replication Lag** | <5 minutes | SAP BTP Cockpit |
| **IDoc Error Rate** | <1% | Transaction WE05, custom reports |
| **API Response Time** | <500ms (p95) | APM tools (Dynatrace, New Relic) |
| **Kafka Consumer Lag** | <10 seconds | Kafka Manager, Prometheus |
| **Failed Messages** | 0 per day | Alert on first failure |

### Alerting Rules

```yaml
alerts:
  - name: MDI_Replication_Lag_High
    condition: replication_lag > 10 minutes
    severity: WARNING
    notify: data-team@example.com

  - name: IDoc_Error_Spike
    condition: error_rate > 5% over 1 hour
    severity: CRITICAL
    notify: sap-team@example.com, data-team@example.com

  - name: API_Latency_High
    condition: p95_latency > 1000ms
    severity: WARNING
    notify: platform-team@example.com

  - name: Kafka_Consumer_Lag_Critical
    condition: consumer_lag > 60 seconds
    severity: CRITICAL
    notify: data-engineering@example.com
```
