# Cross-Reference (Xref) System

Complete documentation for Aleph's cross-reference matching system.

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [How Cross-Reference Works](#how-cross-reference-works)
- [API Reference](#api-reference)
- [Matching Algorithm](#matching-algorithm)
- [User Decisions](#user-decisions)
- [Profiles and Merging](#profiles-and-merging)
- [Frontend Integration](#frontend-integration)
- [Configuration](#configuration)
- [Performance](#performance)
- [Troubleshooting](#troubleshooting)

## Overview

The cross-reference (xref) system is Aleph's powerful entity matching feature that automatically identifies potential duplicate or related entities across different collections. It helps investigators discover connections between datasets that might otherwise remain hidden.

### Key Features

- **Automatic matching** across collections using multiple algorithms
- **Smart scoring** with confidence metrics (0-1 score)
- **User decisions** to accept, reject, or mark matches as unsure
- **Profile merging** to create unified entity representations
- **Export capabilities** for analysis and reporting
- **Configurable thresholds** to balance precision and recall

### Use Cases

1. **Deduplication**: Find the same person or company mentioned in multiple datasets
2. **Network analysis**: Discover connections between entities across sources
3. **Data quality**: Identify inconsistencies and variants
4. **Investigation**: Link subjects across different document collections

### Architecture

```
┌─────────────────┐
│ User triggers   │
│ xref on         │
│ collection      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Queue xref task │
│ (RabbitMQ)      │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│ Worker processes entities:          │
│ 1. Generate match queries           │
│ 2. Search Elasticsearch (50 cands)  │
│ 3. Score with FTM or ML model       │
│ 4. Store matches > 0                │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────┐
│ Index results   │
│ in xref index   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ User reviews    │
│ matches in UI   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Accept/Reject   │
│ creates Profile │
└─────────────────┘
```

## Quick Start

### Triggering Cross-Reference

**Via UI:**
1. Navigate to a collection
2. Click "Cross-reference" in the menu
3. Click "Compute" or "Re-compute"
4. Wait for processing to complete

**Via API:**
```bash
# Trigger xref computation
curl -X POST "https://aleph.example.com/api/2/collections/123/xref" \
  -H "Authorization: ApiKey YOUR_KEY"

# Response: 202 Accepted
{
  "status": "accepted"
}
```

### Viewing Results

```bash
# Get xref matches
curl "https://aleph.example.com/api/2/collections/123/xref?limit=50" \
  -H "Authorization: ApiKey YOUR_KEY"

# Response
{
  "total": 150,
  "results": [
    {
      "score": 0.95,
      "entity_id": "entity-abc",
      "match_id": "entity-xyz",
      "match_collection_id": "456",
      "judgement": "no_judgement"
    }
  ]
}
```

### Making Decisions

```bash
# Accept a match
curl -X POST "https://aleph.example.com/api/2/profiles/_pairwise" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "entity_id": "entity-abc",
    "match_id": "entity-xyz",
    "judgement": "positive"
  }'

# Response
{
  "status": "ok",
  "profile_id": "profile-123"
}
```

## How Cross-Reference Works

### Processing Pipeline

**1. Entity Extraction (aleph/logic/xref.py:268-279)**

When you trigger xref on a collection:
- All existing xref matches are cleared
- Entities with `origin="xref"` are deleted (from previous runs)
- All matchable schema entities are queried from Elasticsearch
- Mentions are aggregated into pseudo-entities

**2. Match Query Generation (aleph/logic/matching.py:43-89)**

For each entity:
```python
# Extract properties with specificity > 0
# Example: Name (high specificity), Country (low specificity)

# Build required clauses (at least 1 must match)
REQUIRED = [name, iban, identifier]

# Build scoring clauses (boost score if match)
# All other properties with specificity > 0

# Create Elasticsearch query
{
  "bool": {
    "must": [
      {"bool": {"should": required_clauses, "minimum_should_match": 1}}
    ],
    "should": scoring_clauses,
    "must_not": [{"term": {"_id": entity.id}}]  # Exclude self
  }
}
```

**3. Candidate Search (aleph/logic/xref.py:127-192)**

- Query Elasticsearch for up to **50 candidates**
- Track metrics: query duration, roundtrip time
- Return entities matching required filters

**4. Scoring (_bulk_compare) (aleph/logic/xref.py:99-115)**

Two scoring modes:

**Mode A: FollowTheMoney Comparison (default)**
```python
from followthemoney.compare import compare

score = compare(model, left_entity, right_entity)
# Returns: 0.0 to 1.0
# Uses property-specific comparison functions
# Considers: names, dates, identifiers, addresses, etc.
```

**Mode B: Machine Learning Model (optional)**
```python
from rigour.ml.evaluate import GLMBernoulli2EEvaluator

evaluator = GLMBernoulli2EEvaluator.load(MODEL_PATH)
score, doubt = evaluator.predict_proba_std(left, right)
# Returns: score (0-1) and confidence/doubt measure
```

**5. Result Storage (aleph/logic/xref.py:171-192)**

- All matches with score > 0 are stored
- Matches with score > **SCORE_CUTOFF (0.5)** counted in metrics
- Each match indexed with:
  - Score and doubt (if available)
  - Source entity and collection
  - Match entity and collection
  - Method version (e.g., "ftm-3.5.9")
  - Random field for unbiased review
  - Searchable text from entity names

**6. Indexing (aleph/index/xref.py:80-82)**

Matches indexed to dedicated `xref` index:
```python
{
  "score": 0.95,
  "doubt": 0.02,
  "method": "ftm-3.5.9",
  "entity_id": "abc123",
  "collection_id": "col-1",
  "match_id": "xyz789",
  "match_collection_id": "col-2",
  "schema": "Person",
  "country": ["US", "UK"],
  "text": "John Smith; ABC Corp; ...",
  "created_at": "2025-12-09T10:00:00Z",
  "random": 42
}
```

### Metrics Tracking

Prometheus metrics collected during xref:

```python
# Total entities processed
XREF_ENTITIES.inc()

# Matches per entity (histogram)
XREF_MATCHES.observe(match_count)
# Buckets: 0, 5, 10, 25, 50

# Query performance
XREF_CANDIDATES_QUERY_DURATION.observe(query_time)
XREF_CANDIDATES_QUERY_ROUNDTRIP_DURATION.observe(total_time)
```

## API Reference

### 1. Fetch Cross-Reference Results

```
GET /api/2/collections/<collection_id>/xref
```

**Authentication:** Requires READ permission on collection

**Query Parameters:**

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `q` | string | Text search in match names | `q=john smith` |
| `filter:schema` | string | Filter by entity schema | `filter:schema=Person` |
| `filter:countries` | string | Filter by country | `filter:countries=US` |
| `filter:match_collection_id` | string | Filter by target collection | `filter:match_collection_id=456` |
| `sort` | string | Sort field: `score`, `random`, `doubt` | `sort=score` |
| `limit` | integer | Results per page (default: 50) | `limit=100` |
| `offset` | integer | Pagination offset | `offset=50` |

**Default Behavior:**
- Shows only matches with score > 0.5
- When sorting by `random` or `doubt`, shows ALL matches (including < 0.5)
- Results include user `judgement` field

**Response:**
```json
{
  "total": 150,
  "results": [
    {
      "id": "match-hash-123",
      "score": 0.95,
      "doubt": 0.02,
      "method": "ftm-3.5.9",
      "entity_id": "entity-abc",
      "entity": {
        "id": "entity-abc",
        "schema": "Person",
        "properties": {"name": ["John Smith"]},
        "collection_id": "col-1"
      },
      "collection_id": "col-1",
      "match_id": "entity-xyz",
      "match": {
        "id": "entity-xyz",
        "schema": "Person",
        "properties": {"name": ["Jon Smith"]},
        "collection_id": "col-2"
      },
      "match_collection_id": "col-2",
      "judgement": "positive",
      "created_at": "2025-12-09T10:00:00Z"
    }
  ],
  "facets": {
    "schema": {"Person": 120, "Company": 30},
    "countries": {"US": 80, "UK": 45, "FR": 25},
    "match_collection_id": {"456": 100, "789": 50}
  }
}
```

**Example: Filter High-Confidence Person Matches**
```bash
curl "https://aleph.example.com/api/2/collections/123/xref?filter:schema=Person&sort=score&limit=20" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Example: Review Low-Confidence Matches**
```bash
# Use random sort to see ALL matches, including < 0.5
curl "https://aleph.example.com/api/2/collections/123/xref?sort=random&limit=50" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Code Reference:** `aleph/views/xref_api.py:22-64`

### 2. Trigger Cross-Reference Computation

```
POST /api/2/collections/<collection_id>/xref
```

**Authentication:** Requires WRITE permission on collection

**Request Body:** None

**Response:** 202 Accepted
```json
{
  "status": "accepted",
  "job_id": "xref-job-123"
}
```

**Processing:**
- Queues xref task to RabbitMQ
- Worker processes in background (QoS: 1 concurrent task)
- Can take several minutes to hours for large collections
- Check collection status or xref results to see progress

**Code Reference:** `aleph/views/xref_api.py:67-98`

### 3. Export Cross-Reference Results

```
POST /api/2/collections/<collection_id>/xref.xlsx
```

**Authentication:** Requires READ permission on collection

**Request Body:** None (inherits query params from current xref search)

**Response:** 202 Accepted
```json
{
  "status": "accepted",
  "export_id": "export-123"
}
```

**Excel Format:**

| Column | Description |
|--------|-------------|
| Score | Match score (0-1) |
| Doubt | Confidence measure (if available) |
| Entity Name | Source entity name |
| Entity Date | Source entity dates |
| Entity Countries | Source entity countries |
| Candidate Collection | Target collection name |
| Candidate Name | Match entity name |
| Candidate Date | Match entity dates |
| Candidate Countries | Match entity countries |
| Entity Link | URL to source entity |
| Candidate Link | URL to match entity |

**Download:**
- File available at: `GET /api/2/exports/<export_id>`
- Check export status: `GET /api/2/exports/<export_id>/status`

**Code Reference:** `aleph/views/xref_api.py:101-132`, `aleph/logic/xref.py:332-380`

### 4. Make Pairwise Decision

```
POST /api/2/profiles/_pairwise
```

**Authentication:** Requires WRITE permission on both collections

**Request Body:**
```json
{
  "entity_id": "entity-abc",
  "match_id": "entity-xyz",
  "judgement": "positive"  // or "negative" or "unsure"
}
```

**Judgement Values:**

| Value | Meaning | Action |
|-------|---------|--------|
| `positive` | User confirms match | Creates/updates profile, merges entities |
| `negative` | User rejects match | Stores rejection, won't suggest again |
| `unsure` | User uncertain | Marks for later review |

**Response:**
```json
{
  "status": "ok",
  "profile_id": "profile-123"
}
```

**What Happens:**
1. Validates entity schemas are compatible
2. Finds or creates PROFILE entity set for source entity
3. Adds source entity with POSITIVE judgement
4. Adds match entity with provided judgement
5. Stores `compared_to_entity_id` to track comparison lineage
6. If POSITIVE: Merges into existing profile or creates new one
7. Queues UPDATE_ENTITY task for profile reification

**Code Reference:** `aleph/views/profiles_api.py`, `aleph/logic/profiles.py:144-186`

## Matching Algorithm

### Property Specificity

Aleph uses **property specificity** to determine which fields are most discriminative for matching:

```python
# High specificity (very unique)
registry.identifier  # Tax IDs, registration numbers
registry.iban        # Bank account numbers
registry.name        # Names of people/organizations

# Medium specificity
registry.phone
registry.email
registry.address

# Low specificity (common values)
registry.country
registry.date
```

Properties with specificity > 0 are used for matching. Higher specificity properties get more weight.

**Code Reference:** `aleph/logic/matching.py:43-89`

### Match Query Structure

For an entity like:
```json
{
  "schema": "Person",
  "properties": {
    "name": ["John Smith"],
    "birthDate": ["1980-01-15"],
    "country": ["US"],
    "idNumber": ["123-45-6789"]
  }
}
```

Generated query:
```json
{
  "bool": {
    "must": [
      {
        "bool": {
          "should": [
            {
              "match": {
                "names": {
                  "query": "john smith",
                  "operator": "and",
                  "minimum_should_match": "60%"
                }
              }
            },
            {
              "term": {"fingerprints": "johnsmith"}
            },
            {
              "term": {"identifier": "123-45-6789"}}
            }
          ],
          "minimum_should_match": 1
        }
      }
    ],
    "should": [
      {"match": {"dates": "1980-01-15"}},
      {"term": {"countries": "US"}}
    ],
    "must_not": [
      {"term": {"_id": "entity-abc"}}
    ]
  }
}
```

**Key Principles:**
1. At least 1 REQUIRED property must match (name, iban, or identifier)
2. SCORING properties boost the score if they match
3. Query capped at 500 clauses (MAX_CLAUSES)
4. Fingerprints used for fuzzy name matching

### Scoring Methods

#### FollowTheMoney Comparison (Default)

Uses structural comparison from the `followthemoney` library:

```python
from followthemoney.compare import compare

score = compare(model, entity1, entity2)
```

**Scoring Logic:**
- Each property type has custom comparison function
- Names: String similarity + phonetic matching
- Dates: Temporal overlap and proximity
- Identifiers: Exact match or normalized comparison
- Addresses: Component-wise matching
- Overall score: Weighted average based on specificity

**Version Tracking:**
```python
method = f"ftm-{followthemoney.__version__}"
# Example: "ftm-3.5.9"
```

**Code Reference:** `aleph/logic/xref.py:99-115`

#### Machine Learning Model (Optional)

If `XREF_MODEL` is configured:

```python
from rigour.ml.evaluate import GLMBernoulli2EEvaluator

evaluator = GLMBernoulli2EEvaluator.load(MODEL_PATH)
score, doubt = evaluator.predict_proba_std(entity1, entity2)
```

**Features:**
- Trained on historical matching decisions
- Returns score (0-1) and confidence measure
- Lower doubt = higher confidence
- Version tracked from model metadata

**Configuration:**
```bash
# In aleph settings
XREF_MODEL=/path/to/model.pkl
```

**Code Reference:** `aleph/logic/xref.py:109-115`

### Thresholds

```python
SCORE_CUTOFF = 0.5  # Default display threshold
```

**Behavior:**
- All matches with score > 0 are **stored**
- Only matches with score > 0.5 are **shown by default**
- Use `sort=random` or `sort=doubt` to review ALL matches
- This allows reviewers to find false negatives

### Entity Fingerprinting

Names are fingerprinted for fuzzy matching:

```python
from fingerprints import generate

fingerprint = generate("John Smith")
# Returns: "johnsmith"

fingerprint = generate("Smith, John Q.")
# Returns: "smithjohnq"
```

**Benefits:**
- Handles punctuation and whitespace variations
- Catches reordering (e.g., "Smith, John" vs "John Smith")
- Case-insensitive matching

**Code Reference:** `aleph/index/entities.py:195-198`

### Mention-Based Matching

In addition to entities, xref can match **mentions** extracted from documents:

```python
# Aggregate mentions by resolved entity ID
# Create synthetic LEGAL_ENTITY with:
# - Combined names from all mentions
# - Merged schemas (e.g., Person + LegalEntity)
# - Aggregated countries
```

This allows matching against entities mentioned in text even if they're not fully structured entities.

**Code Reference:** `aleph/logic/xref.py:194-243`

## User Decisions

### Judgement Types

Users can make four types of decisions on matches:

```python
class Judgement(Enum):
    POSITIVE = "positive"       # Confirmed match
    NEGATIVE = "negative"       # Not a match
    UNSURE = "unsure"          # Need more information
    NO_JUDGEMENT = "no_judgement"  # Default, not reviewed
```

**Code Reference:** `aleph/model/entityset.py:17-31`

### Making Decisions via UI

**Keyboard Shortcuts:**
- `Y` or `Enter`: Accept match (POSITIVE)
- `N`: Reject match (NEGATIVE)
- `U`: Mark as unsure
- `↓` or `J`: Next match
- `↑` or `K`: Previous match

**Mouse Actions:**
- Click ✓ button: Accept
- Click ✗ button: Reject
- Click ? button: Unsure

### Decision Storage

When a user makes a decision:

1. **Find or create PROFILE** entity set for source entity
2. **Add source entity** to profile with POSITIVE judgement
3. **Add match entity** to profile with user's judgement
4. **Store comparison lineage** via `compared_to_entity_id`
5. **Queue reification task** to update entity representations

**Database Structure:**
```sql
-- EntitySet table (PROFILE type)
id: UUID
label: "John Smith Profile"
type: "profile"
collection_id: 123
role_id: 1

-- EntitySetItem table
entityset_id: profile-uuid
entity_id: "entity-abc"
collection_id: 123
judgement: "positive"
compared_to_entity_id: "entity-xyz"
added_by_id: 1
```

**Code Reference:** `aleph/model/entityset.py:278-381`, `aleph/logic/profiles.py:144-186`

### Judgement Combination Logic

When multiple decisions exist for the same entity pair:

```python
def __add__(self, other):
    # NEGATIVE > UNSURE > POSITIVE
    if self == Judgement.NEGATIVE or other == Judgement.NEGATIVE:
        return Judgement.NEGATIVE
    if self == Judgement.UNSURE or other == Judgement.UNSURE:
        return Judgement.UNSURE
    return Judgement.POSITIVE
```

**Behavior:**
- A single NEGATIVE overrides any POSITIVE
- UNSURE overrides POSITIVE but not NEGATIVE
- This prevents conflicting decisions from creating incorrect profiles

**Code Reference:** `aleph/model/entityset.py:26-31`

## Profiles and Merging

### What is a Profile?

A **Profile** is a special entity set (type: PROFILE) that groups multiple entities representing the same real-world entity. It's Aleph's way of implementing entity resolution.

**Example:**
```
Profile: "John Smith"
├── Entity 1 (Collection A): John Smith, CEO, born 1980
├── Entity 2 (Collection B): J. Smith, Director, US citizen
└── Entity 3 (Collection C): John Q. Smith, born 1980-01-15
```

### Profile Creation Flow

**1. User accepts first match:**
```
POST /api/2/profiles/_pairwise
{
  "entity_id": "A",
  "match_id": "B",
  "judgement": "positive"
}
```

**Action:**
- Create new PROFILE entity set
- Add entity A with POSITIVE judgement
- Add entity B with POSITIVE judgement
- Store: A `compared_to_entity_id` = B

**2. User accepts second match for entity A:**
```
POST /api/2/profiles/_pairwise
{
  "entity_id": "A",
  "match_id": "C",
  "judgement": "positive"
}
```

**Action:**
- Find existing PROFILE containing A
- Add entity C with POSITIVE judgement
- Store: A `compared_to_entity_id` = C
- Now profile contains: {A, B, C}

### Profile Merging

**Scenario:** User accepts match between entities already in separate profiles.

**Example:**
- Profile 1 (older): {A, B}
- Profile 2 (newer): {C, D}
- User accepts: A ↔ C

**Result:**
- Profile 1 becomes: {A, B, C, D}
- Profile 2 is deleted
- All judgements preserved
- Older profile always retained

**Code Reference:** `aleph/model/entityset.py:305-346`

### Profile Reification

After profile changes, entities are **reified** (merged) to create a unified representation:

```python
# Queued task
queue_task(collection, STAGE.UPDATE_ENTITY, payload={
    'entity_id': entity.id
})
```

**Reification Process:**
1. Load all entities in profile
2. Merge properties (union of all values)
3. Resolve conflicts (most specific value wins)
4. Create composite entity with combined data
5. Re-index for search

**Result:** Searching for any entity in the profile returns the merged representation.

### Viewing Profiles

**API Endpoint:**
```
GET /api/2/entitysets?filter:type=profile&filter:collection_id=123
```

**Response:**
```json
{
  "results": [
    {
      "id": "profile-123",
      "type": "profile",
      "label": "John Smith",
      "collection_id": 123,
      "entities": ["entity-abc", "entity-xyz"],
      "created_at": "2025-12-09T10:00:00Z"
    }
  ]
}
```

**Code Reference:** `aleph/model/entityset.py:34-275`

## Frontend Integration

### Main Components

**1. CollectionXrefMode (Main View)**

Location: `ui/src/components/Collection/CollectionXrefMode.jsx`

**Features:**
- Displays xref results in table format
- Faceted filtering by:
  - `match_collection_id`: Which collections contain matches
  - `schema`: Entity types (Person, Company, etc.)
  - `countries`: Geographic distribution
- Sorting options:
  - Default: By score (highest first)
  - Random: Unbiased review
  - Doubt: By confidence (requires ML model)
- Infinite scroll pagination
- Export to Excel button

**2. XrefTable & XrefTableRow**

Location: `ui/src/components/Xref/XrefTable.jsx`

**Columns:**
- Decision buttons (✓, ✗, ?)
- Reference entity (source)
- Possible match (target)
- Score (with visual indicator)
- Dataset (match collection)

**Features:**
- Keyboard navigation (↑/↓, J/K)
- Hotkeys for decisions (Y/N/U)
- Side-by-side comparison on click
- Skeleton loaders during fetch

**3. EntityCompare**

Location: `ui/src/components/Entity/EntityCompare.jsx`

**Purpose:** Side-by-side entity comparison

**Features:**
- Shows only properties that differ
- Highlights matching vs. non-matching values
- Links to full entity views
- Embedded in xref review workflow

**4. CollectionXrefManageMenu**

Location: `ui/src/components/Collection/CollectionXrefManageMenu.jsx`

**Features:**
- "Compute" or "Re-compute" button
- Shows xref status
- Disabled if user lacks WRITE permission
- Opens confirmation dialog

### Redux Actions

**1. Query Xref Results**
```javascript
// ui/src/actions/collectionActions.js
export function queryCollectionXref(location, collectionId) {
  const query = collectionXrefFacetsQuery(location, collectionId);
  return queryEndpoint(query);
}
```

**2. Trigger Xref**
```javascript
export async function triggerCollectionXref(collectionId) {
  const response = await post(`/api/2/collections/${collectionId}/xref`);
  return response;
}
```

**3. Make Decision**
```javascript
// ui/src/actions/profileActions.js
export async function pairwiseJudgement(entityId, matchId, judgement) {
  const response = await post('/api/2/profiles/_pairwise', {
    entity_id: entityId,
    match_id: matchId,
    judgement: judgement  // "positive", "negative", or "unsure"
  });
  return response;
}
```

**Code Reference:** `ui/src/actions/collectionActions.js:75-94`, `ui/src/actions/profileActions.js`

### Search Query Structure

```javascript
const query = new Query('xref', {
  'filter:match_collection_id': '456',
  'filter:schema': 'Person',
  'filter:countries': 'US',
  'sort': 'score',
  'limit': 50,
  'offset': 0
});
```

**Backend Query Class:**
```python
class XrefQuery(Query):
    TEXT_FIELDS = ["text"]
    SORT_DEFAULT = [{"score": "desc"}]
    SORT_FIELDS = {
        "random": "random",
        "doubt": "doubt",
        "score": "_score"
    }
    AUTHZ_FIELD = "match_collection_id"
    SCORE_CUTOFF = 0.5  # Filtered unless sort=random/doubt
```

**Code Reference:** `aleph/search/__init__.py:111-137`

## Configuration

### Environment Variables

```bash
# Elasticsearch scroll settings for large collections
XREF_SCROLL=5m                  # Scroll timeout (default: 5 minutes)
XREF_SCROLL_SIZE=1000          # Documents per scroll page (default: 1000)

# Machine learning model (optional)
XREF_MODEL=/path/to/model.pkl  # Path to trained GLM model

# Queue configuration
RABBITMQ_QOS_XREF_QUEUE=1      # Max concurrent xref tasks (default: 1)
RABBITMQ_QOS_EXPORT_XREF_QUEUE=1  # Max concurrent exports (default: 1)

# Stage names
STAGE_XREF=xref                 # Xref queue name
STAGE_EXPORT_XREF=exportxref    # Export queue name
```

**Code Reference:** `aleph/settings.py`

### Queue Settings

**Why QoS = 1?**

Cross-reference is computationally expensive. Only 1 xref task runs at a time to:
- Prevent Elasticsearch overload
- Avoid memory exhaustion
- Ensure fair scheduling across collections

**For large deployments**, consider:
- Dedicated xref workers
- Separate Elasticsearch cluster for xref
- Batch processing during off-peak hours

### Tuning Parameters

**1. Candidate Limit**

Hardcoded to 50 candidates per entity:
```python
# aleph/logic/xref.py:158
result = es.search(index=index, body=query, size=50)
```

**Trade-off:**
- Higher = more matches found, slower processing
- Lower = faster, may miss matches

**2. Score Cutoff**

```python
SCORE_CUTOFF = 0.5  # aleph/logic/xref.py:25
```

**Trade-off:**
- Higher = fewer false positives, more false negatives
- Lower = more matches to review, higher recall

**3. MAX_CLAUSES**

```python
MAX_CLAUSES = 500  # aleph/logic/matching.py:11
```

**Trade-off:**
- Higher = more properties used, slower queries
- Lower = faster queries, may miss matches on rare properties

### Monitoring

**Prometheus Metrics:**

```python
# Track in your monitoring system
aleph_xref_entities_total          # Total entities processed
aleph_xref_matches_bucket          # Distribution of matches per entity
aleph_xref_candidates_query_duration_seconds  # Query performance
```

**Example Queries:**
```promql
# Average matches per entity
rate(aleph_xref_matches_sum[5m]) / rate(aleph_xref_matches_count[5m])

# 95th percentile query time
histogram_quantile(0.95, rate(aleph_xref_candidates_query_duration_seconds_bucket[5m]))

# Entities processed per second
rate(aleph_xref_entities_total[5m])
```

**Code Reference:** `aleph/logic/xref.py:33-52`

## Performance

### Processing Speed

**Factors:**
- Collection size (number of entities)
- Entity complexity (number of properties)
- Number of candidate collections
- Elasticsearch cluster performance
- Scoring method (FTM vs. ML model)

**Typical Performance:**
- Small collection (1,000 entities): 1-5 minutes
- Medium collection (10,000 entities): 10-30 minutes
- Large collection (100,000 entities): 2-6 hours

### Optimization Strategies

**1. Pre-filter Collections**

Only xref against relevant collections:
```python
# In match_query(), specify collection_ids
match_query(entity, collection_ids=["col-1", "col-2"])
```

**2. Schema-Specific Matching**

Match only compatible schemas:
```python
# Only match Person → Person, Company → Company
# Reduces false positives and computation
```

**3. Batch Processing**

For very large collections:
```bash
# Run xref during off-peak hours
# Use dedicated worker pool
# Consider incremental xref (new entities only)
```

**4. Index Optimization**

```bash
# Elasticsearch settings for xref index
PUT /aleph-xref-v1/_settings
{
  "index": {
    "number_of_replicas": 0,     # Reduce during bulk operations
    "refresh_interval": "30s"     # Less frequent refreshes
  }
}
```

**5. Parallel Workers**

For very large deployments:
```bash
# Increase QoS (carefully!)
RABBITMQ_QOS_XREF_QUEUE=2

# Run multiple worker containers
docker-compose up -d --scale worker=4
```

**Warning:** Monitor Elasticsearch cluster health when increasing parallelism.

### Memory Usage

**Worker Memory:**
- Base: ~500 MB
- Per entity batch: ~50-100 MB
- Peak during scoring: ~1-2 GB

**Recommendations:**
- Worker containers: 2-4 GB RAM
- For large collections: 8 GB RAM
- Monitor with: `docker stats`

### Database Load

**Writes During Xref:**
- Delete existing matches: ~1-10K deletes
- Insert new matches: ~1-100K inserts (depends on match rate)
- Index updates: Continuous

**Optimization:**
```sql
-- Indexes on xref-related tables
CREATE INDEX idx_entityset_item_entity ON entityset_item(entity_id);
CREATE INDEX idx_entityset_item_entityset ON entityset_item(entityset_id);
CREATE INDEX idx_entityset_item_judgement ON entityset_item(judgement);
```

## Troubleshooting

### No Matches Found

**Symptoms:**
- Xref completes but returns 0 matches
- Expected matches not appearing

**Possible Causes:**

1. **Permissions Issue**
   ```bash
   # Check if user has READ permission on target collections
   GET /api/2/collections/<collection_id>/xref
   # Should not return 403 Forbidden
   ```

2. **Schema Incompatibility**
   ```python
   # Only compatible schemas match
   # Person ↔ Person, Company ↔ Company
   # Not: Person ↔ Document
   ```

3. **No Shared Properties**
   ```bash
   # Check entity properties
   GET /api/2/entities/<entity_id>

   # Ensure entities have matchable properties:
   # - Names, identifiers, IBAN, etc.
   ```

4. **Threshold Too High**
   ```bash
   # Review low-confidence matches
   GET /api/2/collections/123/xref?sort=random
   # Shows ALL matches, including < 0.5 score
   ```

**Solutions:**
- Grant READ permissions to relevant collections
- Enrich entities with more identifying properties
- Lower SCORE_CUTOFF if needed
- Check Elasticsearch index health

### Xref Task Stuck

**Symptoms:**
- Xref status shows "processing" indefinitely
- No progress in logs

**Diagnosis:**

1. **Check Worker Logs**
   ```bash
   docker-compose logs -f worker
   # Look for errors or exceptions
   ```

2. **Check Queue**
   ```bash
   # RabbitMQ management UI
   # Look for messages in "xref" queue
   ```

3. **Check Elasticsearch**
   ```bash
   # Index health
   GET /_cluster/health

   # Long-running queries
   GET /_tasks?detailed=true&actions=*search*
   ```

**Solutions:**
- Restart worker: `docker-compose restart worker`
- Clear stuck queue: Use RabbitMQ management UI
- Increase timeouts: `XREF_SCROLL=15m`
- Check Elasticsearch logs for errors

### Out of Memory Errors

**Symptoms:**
```
MemoryError: Unable to allocate array
# or
Killed (OOM)
```

**Solutions:**

1. **Increase Worker Memory**
   ```yaml
   # docker-compose.yml
   worker:
     deploy:
       resources:
         limits:
           memory: 4G
   ```

2. **Reduce Batch Size**
   ```bash
   XREF_SCROLL_SIZE=500  # Lower from 1000
   ```

3. **Process in Stages**
   ```bash
   # Xref subset of entities at a time
   # Manual batching required
   ```

### Slow Performance

**Symptoms:**
- Xref takes hours for small collections
- Query timeouts

**Diagnosis:**

1. **Check Metrics**
   ```python
   # View Prometheus metrics
   aleph_xref_candidates_query_duration_seconds
   # High values indicate slow queries
   ```

2. **Check Elasticsearch**
   ```bash
   GET /_nodes/stats/indices/search
   # Look for slow query times
   ```

**Solutions:**

1. **Optimize Elasticsearch**
   ```bash
   # Increase heap size
   ES_JAVA_OPTS="-Xms2g -Xmx2g"

   # Tune thread pools
   PUT /_cluster/settings
   {
     "persistent": {
       "thread_pool.search.size": 30
     }
   }
   ```

2. **Reduce Candidate Limit**
   ```python
   # Edit aleph/logic/xref.py:158
   result = es.search(..., size=25)  # Lower from 50
   ```

3. **Use ML Model**
   ```bash
   # Faster than FTM comparison if well-tuned
   XREF_MODEL=/path/to/model.pkl
   ```

### Incorrect Matches

**Symptoms:**
- Many false positives
- Unrelated entities matched

**Solutions:**

1. **Raise Threshold**
   ```python
   # Edit aleph/logic/xref.py:25
   SCORE_CUTOFF = 0.7  # Higher threshold
   ```

2. **Improve Entity Data**
   - Add more disambiguating properties
   - Clean up noisy data
   - Add specific identifiers (tax IDs, etc.)

3. **Review Matching Logic**
   ```python
   # Check property specificity
   # Ensure high-specificity properties are populated
   ```

4. **Use Judgements**
   - Mark false positives as NEGATIVE
   - System learns from decisions
   - Train ML model on historical judgements

### Export Fails

**Symptoms:**
- Export never completes
- Download link returns 404

**Diagnosis:**

1. **Check Export Status**
   ```bash
   GET /api/2/exports/<export_id>/status
   # Shows: pending, running, complete, failed
   ```

2. **Check Worker Logs**
   ```bash
   docker-compose logs -f worker | grep export
   ```

**Solutions:**
- Check worker has access to result store
- Ensure sufficient disk space
- Check S3/storage permissions if using external storage
- Retry export with smaller result set (add filters)

---

**Related Documentation:**
- [SEARCH.md](./SEARCH.md) - Search functionality
- [ENTITIES.md](./ENTITIES.md) - Entity types and schema
- [INVESTIGATIONS.md](./INVESTIGATIONS.md) - Entity sets and profiles
- [WORKERS.md](./WORKERS.md) - Background task processing
- [PERFORMANCE.md](./PERFORMANCE.md) - System optimization

**External Resources:**
- [FollowTheMoney Documentation](https://followthemoney.tech/) - Entity schema
- [Elasticsearch Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/query-dsl.html)
- [RabbitMQ Management](https://www.rabbitmq.com/management.html)
