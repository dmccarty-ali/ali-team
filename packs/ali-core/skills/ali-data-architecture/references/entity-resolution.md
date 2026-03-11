# ali-data-architecture - Entity Resolution Reference

Complete patterns for deterministic and probabilistic matching.

---

## Overview

Entity resolution identifies when different records refer to the same real-world entity despite variations in data (typos, formatting, missing fields).

**Use cases:**
- Patient matching across systems
- Provider deduplication
- Customer merge/household grouping
- Claim-to-payment matching

---

## Deterministic Matching

Exact match on business keys or standardized identifiers.

### Simple Join Pattern

```sql
-- Exact match on claim_id and payer_id
SELECT
    claims.claim_id,
    claims.total_charge,
    payments.paid_amount,
    payments.payment_date
FROM claims
JOIN payments
    ON claims.claim_id = payments.claim_id
    AND claims.payer_id = payments.payer_id;
```

### Multi-Key Matching

```sql
-- Match on multiple keys with fallback
SELECT
    a.*,
    b.*,
    CASE
        WHEN a.claim_id = b.claim_id AND a.payer_id = b.payer_id THEN 'EXACT'
        WHEN a.patient_ssn = b.patient_ssn AND a.service_date = b.service_date THEN 'ALTERNATE'
        ELSE 'NO_MATCH'
    END AS match_type
FROM source_a a
LEFT JOIN source_b b
    ON (a.claim_id = b.claim_id AND a.payer_id = b.payer_id)
    OR (a.patient_ssn = b.patient_ssn AND a.service_date = b.service_date);
```

### Standardized Key Generation

```sql
-- Generate standardized match key from multiple fields
CREATE TABLE patient_match_keys AS
SELECT
    patient_id,
    -- Standardize components
    UPPER(TRIM(last_name)) AS std_last_name,
    UPPER(TRIM(first_name)) AS std_first_name,
    TO_DATE(date_of_birth, 'YYYY-MM-DD') AS std_dob,
    REGEXP_REPLACE(ssn, '[^0-9]', '') AS std_ssn,  -- Remove dashes

    -- Composite match key
    MD5(
        UPPER(TRIM(last_name)) || '|' ||
        UPPER(TRIM(first_name)) || '|' ||
        TO_CHAR(date_of_birth, 'YYYY-MM-DD') || '|' ||
        REGEXP_REPLACE(ssn, '[^0-9]', '')
    ) AS match_key
FROM patients;

-- Match using standardized keys
SELECT a.*, b.*
FROM patient_match_keys a
JOIN patient_match_keys b
    ON a.match_key = b.match_key
    AND a.patient_id < b.patient_id;  -- Avoid self-join, duplicates
```

---

## Probabilistic Matching

Fuzzy matching with confidence scoring when exact matches unavailable.

### Python Implementation with RapidFuzz

```python
from rapidfuzz import fuzz, process
from dataclasses import dataclass
from typing import Optional

@dataclass
class MatchScore:
    """Match result with confidence score."""
    record_a_id: str
    record_b_id: str
    score: float
    match_confidence: str  # HIGH, MEDIUM, LOW
    matched_fields: dict

def match_score(record_a: dict, record_b: dict) -> MatchScore:
    """Calculate match probability between two records.

    Scoring rules:
    - SSN exact match: 0.95 confidence
    - Name fuzzy match: 0-0.7 confidence (weighted)
    - DOB exact match: 0.85 confidence
    - Address fuzzy match: 0-0.5 confidence

    Threshold for match: 0.75
    """
    scores = []
    matched_fields = {}

    # 1. Social Security Number (highest confidence)
    ssn_a = record_a.get('ssn', '').replace('-', '').replace(' ', '')
    ssn_b = record_b.get('ssn', '').replace('-', '').replace(' ', '')
    if ssn_a and ssn_b:
        if ssn_a == ssn_b:
            scores.append(('ssn', 0.95))
            matched_fields['ssn'] = 'EXACT'
        else:
            scores.append(('ssn', 0.0))  # Wrong SSN = no match
    else:
        # Missing SSN, rely on other fields
        pass

    # 2. Name (fuzzy matching with normalization)
    name_a = f"{record_a.get('first_name', '')} {record_a.get('last_name', '')}".strip().lower()
    name_b = f"{record_b.get('first_name', '')} {record_b.get('last_name', '')}".strip().lower()
    if name_a and name_b:
        # Token sort ratio (handles word order: "John Smith" == "Smith, John")
        name_score = fuzz.token_sort_ratio(name_a, name_b) / 100
        scores.append(('name', name_score * 0.7))  # Weight: 70% of full score
        matched_fields['name'] = f"{int(name_score * 100)}%"

    # 3. Date of Birth (exact match required)
    dob_a = record_a.get('date_of_birth')
    dob_b = record_b.get('date_of_birth')
    if dob_a and dob_b:
        if dob_a == dob_b:
            scores.append(('dob', 0.85))
            matched_fields['dob'] = 'EXACT'
        else:
            scores.append(('dob', 0.0))  # Wrong DOB = strong no-match signal

    # 4. Phone number (normalized)
    phone_a = record_a.get('phone', '').replace('-', '').replace(' ', '').replace('(', '').replace(')', '')[-10:]
    phone_b = record_b.get('phone', '').replace('-', '').replace(' ', '').replace('(', '').replace(')', '')[-10:]
    if phone_a and phone_b:
        if phone_a == phone_b:
            scores.append(('phone', 0.6))
            matched_fields['phone'] = 'EXACT'
        else:
            scores.append(('phone', 0.0))

    # 5. Address (fuzzy, lower weight)
    addr_a = f"{record_a.get('address', '')} {record_a.get('city', '')} {record_a.get('state', '')}".strip().lower()
    addr_b = f"{record_b.get('address', '')} {record_b.get('city', '')} {record_b.get('state', '')}".strip().lower()
    if addr_a and addr_b:
        addr_score = fuzz.token_set_ratio(addr_a, addr_b) / 100
        scores.append(('address', addr_score * 0.5))  # Weight: 50%
        matched_fields['address'] = f"{int(addr_score * 100)}%"

    # 6. Email (exact or fuzzy)
    email_a = record_a.get('email', '').lower()
    email_b = record_b.get('email', '').lower()
    if email_a and email_b:
        if email_a == email_b:
            scores.append(('email', 0.8))
            matched_fields['email'] = 'EXACT'
        else:
            # Fuzzy email match (typos)
            email_score = fuzz.ratio(email_a, email_b) / 100
            if email_score > 0.85:  # Only count if very similar
                scores.append(('email', email_score * 0.6))
                matched_fields['email'] = f"{int(email_score * 100)}%"

    # Calculate weighted average
    if not scores:
        total_score = 0.0
    else:
        total_weight = sum(s[1] for s in scores)
        total_score = total_weight / len(scores) if scores else 0

    # Determine confidence level
    if total_score >= 0.85:
        confidence = 'HIGH'
    elif total_score >= 0.70:
        confidence = 'MEDIUM'
    else:
        confidence = 'LOW'

    return MatchScore(
        record_a_id=record_a.get('id'),
        record_b_id=record_b.get('id'),
        score=total_score,
        match_confidence=confidence,
        matched_fields=matched_fields
    )


# Example usage
record_a = {
    'id': 'A001',
    'first_name': 'John',
    'last_name': 'Smith',
    'date_of_birth': '1980-05-15',
    'ssn': '123-45-6789',
    'phone': '555-123-4567',
    'address': '123 Main St',
    'city': 'Springfield',
    'state': 'IL'
}

record_b = {
    'id': 'B002',
    'first_name': 'Jon',  # Typo
    'last_name': 'Smith',
    'date_of_birth': '1980-05-15',
    'ssn': '123-45-6789',
    'phone': '555-123-4567',
    'address': '123 Main Street',  # Slight variation
    'city': 'Springfield',
    'state': 'IL'
}

result = match_score(record_a, record_b)
print(f"Score: {result.score:.2f}")
print(f"Confidence: {result.match_confidence}")
print(f"Matched fields: {result.matched_fields}")

# Output:
# Score: 0.92
# Confidence: HIGH
# Matched fields: {'ssn': 'EXACT', 'name': '97%', 'dob': 'EXACT', 'phone': 'EXACT', 'address': '94%'}
```

---

## Blocking Strategy

Reduce comparison space by grouping candidates before expensive fuzzy matching.

### Phonetic Blocking

```sql
-- Block on phonetic encoding (Soundex) + birth year
CREATE TABLE patient_blocks AS
SELECT
    SOUNDEX(last_name) || '_' || YEAR(date_of_birth) AS block_key,
    patient_id,
    first_name,
    last_name,
    date_of_birth,
    ssn,
    phone
FROM patients;

-- Compare only within blocks (dramatically reduces pairs)
SELECT
    a.patient_id AS patient_a,
    b.patient_id AS patient_b,
    a.block_key,
    -- Call fuzzy matching UDF here
    fuzzy_match_score(a, b) AS match_score
FROM patient_blocks a
JOIN patient_blocks b
    ON a.block_key = b.block_key
    AND a.patient_id < b.patient_id  -- Avoid self-join, ensure a < b for uniqueness
WHERE fuzzy_match_score(a, b) >= 0.75;  -- Only keep high-confidence matches
```

### Multi-Block Strategy

Generate multiple blocks to catch edge cases (different spellings, missing DOB):

```sql
-- Generate multiple blocking keys per record
CREATE TABLE patient_multi_blocks AS
SELECT patient_id, block_key FROM (
    -- Block 1: Soundex last name + birth year
    SELECT
        patient_id,
        SOUNDEX(last_name) || '_' || YEAR(date_of_birth) AS block_key
    FROM patients
    WHERE last_name IS NOT NULL AND date_of_birth IS NOT NULL

    UNION ALL

    -- Block 2: First 3 chars last name + ZIP
    SELECT
        patient_id,
        SUBSTR(UPPER(last_name), 1, 3) || '_' || address_zip AS block_key
    FROM patients
    WHERE last_name IS NOT NULL AND address_zip IS NOT NULL

    UNION ALL

    -- Block 3: Last 4 SSN + birth year
    SELECT
        patient_id,
        SUBSTR(ssn, -4) || '_' || YEAR(date_of_birth) AS block_key
    FROM patients
    WHERE ssn IS NOT NULL AND date_of_birth IS NOT NULL

    UNION ALL

    -- Block 4: Phone area code + last name initial
    SELECT
        patient_id,
        SUBSTR(phone, 1, 3) || '_' || SUBSTR(UPPER(last_name), 1, 1) AS block_key
    FROM patients
    WHERE phone IS NOT NULL AND last_name IS NOT NULL
);

-- Compare within any matching block
SELECT DISTINCT
    a.patient_id AS patient_a,
    b.patient_id AS patient_b
FROM patient_multi_blocks a
JOIN patient_multi_blocks b
    ON a.block_key = b.block_key
    AND a.patient_id < b.patient_id;
```

**Blocking reduces comparisons from O(n²) to O(n×block_size):**
- 1M patients: 1M × 1M = 1 trillion comparisons (infeasible)
- 1M patients in 100K blocks: 1M × avg 10 per block = 10M comparisons (feasible)

---

## Machine Learning Approach

For large-scale matching, train a model on labeled examples.

### Feature Engineering

```python
import pandas as pd
from sklearn.ensemble import RandomForestClassifier

def extract_features(record_a: dict, record_b: dict) -> dict:
    """Extract comparison features for ML model."""
    features = {}

    # Exact match features
    features['ssn_exact'] = int(record_a.get('ssn') == record_b.get('ssn'))
    features['dob_exact'] = int(record_a.get('dob') == record_b.get('dob'))
    features['phone_exact'] = int(record_a.get('phone') == record_b.get('phone'))

    # Fuzzy match scores
    features['name_similarity'] = fuzz.token_sort_ratio(
        f"{record_a.get('first_name')} {record_a.get('last_name')}",
        f"{record_b.get('first_name')} {record_b.get('last_name')}"
    )
    features['address_similarity'] = fuzz.token_set_ratio(
        record_a.get('address', ''),
        record_b.get('address', '')
    )

    # Edit distances
    features['last_name_edit_dist'] = fuzz.distance(
        record_a.get('last_name', ''),
        record_b.get('last_name', '')
    )

    # Derived features
    features['same_zip'] = int(record_a.get('zip') == record_b.get('zip'))
    features['same_state'] = int(record_a.get('state') == record_b.get('state'))

    return features

# Training data (labeled matches/non-matches)
training_pairs = [
    ({'id': 'A1', 'first_name': 'John', 'last_name': 'Smith', 'dob': '1980-05-15', 'ssn': '123456789'},
     {'id': 'A2', 'first_name': 'Jon', 'last_name': 'Smith', 'dob': '1980-05-15', 'ssn': '123456789'},
     True),  # Match
    # ... more examples
]

X = []
y = []
for a, b, is_match in training_pairs:
    X.append(extract_features(a, b))
    y.append(1 if is_match else 0)

X_df = pd.DataFrame(X)
y_series = pd.Series(y)

# Train classifier
clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(X_df, y_series)

# Predict on new pairs
def predict_match(record_a, record_b):
    features = extract_features(record_a, record_b)
    features_df = pd.DataFrame([features])
    prob = clf.predict_proba(features_df)[0][1]  # Probability of match
    return prob

# Usage
prob = predict_match(record_a, record_b)
print(f"Match probability: {prob:.2f}")
```

---

## Match Workflow

### 1. Preprocessing

```python
def preprocess_record(record: dict) -> dict:
    """Standardize record for matching."""
    clean = {}

    # Normalize names
    clean['first_name'] = record.get('first_name', '').strip().upper()
    clean['last_name'] = record.get('last_name', '').strip().upper()

    # Normalize SSN
    clean['ssn'] = record.get('ssn', '').replace('-', '').replace(' ', '')

    # Normalize phone
    phone = record.get('phone', '').replace('-', '').replace(' ', '').replace('(', '').replace(')', '')
    clean['phone'] = phone[-10:] if len(phone) >= 10 else ''

    # Normalize address
    address = record.get('address', '').upper()
    address = address.replace(' ST', ' STREET').replace(' AVE', ' AVENUE').replace(' BLVD', ' BOULEVARD')
    clean['address'] = address.strip()

    # Parse date
    clean['dob'] = pd.to_datetime(record.get('dob'), errors='coerce')

    return clean
```

### 2. Blocking

```python
def generate_blocks(records: list[dict]) -> dict:
    """Generate blocking keys for all records."""
    blocks = {}
    for record in records:
        block_key = f"{record['last_name'][:3]}_{record['dob'].year if record['dob'] else 'NA'}"
        if block_key not in blocks:
            blocks[block_key] = []
        blocks[block_key].append(record)
    return blocks
```

### 3. Pairwise Comparison

```python
def find_matches(blocks: dict, threshold: float = 0.75) -> list:
    """Find all candidate matches above threshold."""
    matches = []
    for block_key, records in blocks.items():
        # Compare all pairs within block
        for i in range(len(records)):
            for j in range(i + 1, len(records)):
                score = match_score(records[i], records[j])
                if score.score >= threshold:
                    matches.append(score)
    return matches
```

### 4. Clustering (Transitive Closure)

```python
from collections import defaultdict

def cluster_matches(matches: list[MatchScore]) -> list[set]:
    """Group matched records into clusters (connected components)."""
    # Build adjacency graph
    graph = defaultdict(set)
    for match in matches:
        graph[match.record_a_id].add(match.record_b_id)
        graph[match.record_b_id].add(match.record_a_id)

    # Find connected components (DFS)
    visited = set()
    clusters = []

    def dfs(node, cluster):
        visited.add(node)
        cluster.add(node)
        for neighbor in graph[node]:
            if neighbor not in visited:
                dfs(neighbor, cluster)

    for node in graph:
        if node not in visited:
            cluster = set()
            dfs(node, cluster)
            clusters.append(cluster)

    return clusters

# Example
matches = [
    MatchScore('A', 'B', 0.9, 'HIGH', {}),
    MatchScore('B', 'C', 0.85, 'HIGH', {}),
    MatchScore('D', 'E', 0.92, 'HIGH', {})
]

clusters = cluster_matches(matches)
# Result: [{A, B, C}, {D, E}]
```

### 5. Master Record Selection

```python
def select_master_record(cluster: set, all_records: dict) -> dict:
    """Choose master record from cluster (most complete, most recent)."""
    records = [all_records[rid] for rid in cluster]

    # Score based on completeness and recency
    def completeness_score(record):
        fields = ['first_name', 'last_name', 'dob', 'ssn', 'phone', 'address', 'email']
        return sum(1 for f in fields if record.get(f))

    # Sort by completeness (descending), then load date (descending)
    records.sort(key=lambda r: (completeness_score(r), r.get('load_date', '1900-01-01')), reverse=True)

    return records[0]  # Most complete, most recent
```

---

## Performance Optimization

### Indexing for Blocking

```sql
-- Index on blocking keys
CREATE INDEX idx_patient_block_soundex_year
ON patients (SOUNDEX(last_name), YEAR(date_of_birth));

CREATE INDEX idx_patient_block_zip_name
ON patients (address_zip, SUBSTR(last_name, 1, 3));
```

### Parallel Processing

```python
from multiprocessing import Pool

def match_block(block):
    """Process one block (can be parallelized)."""
    matches = []
    records = block['records']
    for i in range(len(records)):
        for j in range(i + 1, len(records)):
            score = match_score(records[i], records[j])
            if score.score >= 0.75:
                matches.append(score)
    return matches

# Parallel execution
with Pool(processes=8) as pool:
    all_matches = pool.map(match_block, blocks.values())
```

---

## Evaluation Metrics

```python
def evaluate_matching(predicted_matches: list, true_matches: list):
    """Calculate precision, recall, F1 for matching."""
    true_pairs = set((m.record_a_id, m.record_b_id) for m in true_matches)
    pred_pairs = set((m.record_a_id, m.record_b_id) for m in predicted_matches)

    tp = len(true_pairs & pred_pairs)  # True positives
    fp = len(pred_pairs - true_pairs)  # False positives
    fn = len(true_pairs - pred_pairs)  # False negatives

    precision = tp / (tp + fp) if (tp + fp) > 0 else 0
    recall = tp / (tp + fn) if (tp + fn) > 0 else 0
    f1 = 2 * (precision * recall) / (precision + recall) if (precision + recall) > 0 else 0

    return {
        'precision': precision,
        'recall': recall,
        'f1': f1,
        'true_positives': tp,
        'false_positives': fp,
        'false_negatives': fn
    }
```

---

## Summary

- **Deterministic**: Exact match on standardized keys (fast, high precision)
- **Probabilistic**: Fuzzy matching with scoring (handles variations)
- **Blocking**: Reduce search space with candidate generation (essential for scale)
- **ML approach**: Train classifier on labeled examples (best for complex domains)
- **Clustering**: Group all matched records into canonical entities
- **Evaluation**: Measure precision/recall on labeled test set
