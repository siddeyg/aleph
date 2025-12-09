# Entities

Complete documentation for Aleph's entity management system and FollowTheMoney schema.

## Table of Contents

- [Overview](#overview)
- [FollowTheMoney Schema](#followthemoney-schema)
- [Quick Start](#quick-start)
- [Entity Model](#entity-model)
- [Creating Entities](#creating-entities)
- [Managing Entities](#managing-entities)
- [Entity Search](#entity-search)
- [Entity Relationships](#entity-relationships)
- [Entity Profiles](#entity-profiles)
- [API Reference](#api-reference)
- [Frontend Integration](#frontend-integration)
- [Advanced Topics](#advanced-topics)
- [Troubleshooting](#troubleshooting)

## Overview

Entities are structured data objects in Aleph that represent real-world things: people, companies, documents, relationships, and more. They follow the **FollowTheMoney** (FTM) schema, a data model designed for investigative journalism and anti-corruption work.

### Key Concepts

**Entity = Schema + Properties**
```json
{
  "id": "entity-abc123",
  "schema": "Person",
  "properties": {
    "name": ["John Smith"],
    "birthDate": ["1980-01-15"],
    "nationality": ["US"]
  }
}
```

**Why Entities?**
- **Structured data**: Consistent format across collections
- **Relationships**: Link entities together (e.g., Person → Company)
- **Validation**: Type-safe properties
- **Search**: Full-text and faceted search
- **Cross-reference**: Automatic matching across datasets

### Architecture

```
User Input (UI/API/Bulk)
         ↓
  Entity Validation (FTM Schema)
         ↓
  Namespace Signing (Collection Context)
         ↓
  Storage Layer
    ├── PostgreSQL (Primary Store)
    ├── FTM Aggregator (Merging)
    └── Elasticsearch (Search Index)
         ↓
  Post-Processing
    ├── Cross-reference (Xref)
    ├── Profile Merging
    └── Entity Expansion
```

## FollowTheMoney Schema

FollowTheMoney (FTM) is a data model for anti-corruption investigations. It defines entity types (schemas) and their properties.

### Core Concepts

**Schema Hierarchy:**
```
Thing (abstract)
├── LegalEntity
│   ├── Person
│   ├── Company
│   ├── Organization
│   └── PublicBody
├── Asset
│   ├── BankAccount
│   ├── RealEstate
│   └── Vehicle
├── Document
│   ├── Email
│   ├── Folder
│   └── Table
└── Value (abstract)
    ├── Address
    ├── Identification
    └── Payment
```

**Schema Types:**
- **Thing**: Physical or abstract entities
- **Interval**: Relationships between entities (e.g., Ownership, Directorship)
- **Value**: Property values (addresses, IDs)

### Common Entity Types

#### People & Organizations

**Person**
```json
{
  "schema": "Person",
  "properties": {
    "name": ["John Smith", "J. Smith"],
    "birthDate": ["1980-01-15"],
    "nationality": ["US"],
    "idNumber": ["123-45-6789"],
    "address": ["123 Main St, New York, NY"],
    "phone": ["+1-555-0100"],
    "email": ["john@example.com"]
  }
}
```

**Company**
```json
{
  "schema": "Company",
  "properties": {
    "name": ["ACME Corp", "ACME Corporation"],
    "incorporationDate": ["2010-05-20"],
    "jurisdiction": ["US"],
    "registrationNumber": ["12-3456789"],
    "address": ["456 Business Ave, NY"],
    "taxNumber": ["98-7654321"]
  }
}
```

**Organization** (Non-profit, NGO)
```json
{
  "schema": "Organization",
  "properties": {
    "name": ["Red Cross"],
    "country": ["US"]
  }
}
```

#### Relationships

**Ownership**
```json
{
  "schema": "Ownership",
  "properties": {
    "owner": ["person-id-123"],
    "asset": ["company-id-456"],
    "startDate": ["2015-01-01"],
    "percentage": ["51"],
    "role": ["Shareholder"]
  }
}
```

**Directorship**
```json
{
  "schema": "Directorship",
  "properties": {
    "director": ["person-id-123"],
    "organization": ["company-id-456"],
    "role": ["CEO"],
    "startDate": ["2018-03-01"]
  }
}
```

**Family** (Relationship)
```json
{
  "schema": "Family",
  "properties": {
    "person": ["person-id-123"],
    "relative": ["person-id-789"],
    "relationship": ["spouse"]
  }
}
```

#### Assets

**BankAccount**
```json
{
  "schema": "BankAccount",
  "properties": {
    "holder": ["person-id-123"],
    "iban": ["GB82 WEST 1234 5698 7654 32"],
    "balance": ["1000000"],
    "currency": ["USD"]
  }
}
```

**RealEstate**
```json
{
  "schema": "RealEstate",
  "properties": {
    "owner": ["person-id-123"],
    "address": ["789 Park Ave, NY"],
    "country": ["US"],
    "area": ["250"],
    "summary": ["Luxury apartment"]
  }
}
```

#### Documents

**Document**
```json
{
  "schema": "Document",
  "properties": {
    "fileName": ["contract.pdf"],
    "title": ["Service Agreement"],
    "date": ["2024-01-15"],
    "author": ["John Smith"],
    "contentHash": ["abc123..."],
    "mimeType": ["application/pdf"]
  }
}
```

### Property Types

FTM defines property types with validation:

| Type | Description | Example |
|------|-------------|---------|
| `name` | Person/organization names | "John Smith" |
| `identifier` | Tax IDs, registration numbers | "12-3456789" |
| `iban` | Bank account numbers | "GB82 WEST..." |
| `date` | Dates (YYYY, YYYY-MM, YYYY-MM-DD) | "2024-01-15" |
| `country` | ISO country codes | "US", "GB" |
| `address` | Physical addresses | "123 Main St" |
| `phone` | Phone numbers | "+1-555-0100" |
| `email` | Email addresses | "user@example.com" |
| `url` | URLs | "https://example.com" |
| `number` | Numeric values | "1000000" |
| `entity` | References to other entities | "entity-id-123" |
| `text` | Free text | "Description..." |

**Multi-Value Properties:**
Most properties accept arrays:
```json
{
  "name": ["John Smith", "J. Smith", "Johnny S."],
  "nationality": ["US", "CA"]
}
```

**Code Reference:** [FollowTheMoney Documentation](https://followthemoney.tech/)

## Quick Start

### Viewing Entities via UI

1. Navigate to a collection
2. Click "Entities" tab
3. Browse or search entities
4. Click entity to view details

### Viewing Entities via API

```bash
# Get single entity
curl "https://aleph.example.com/api/2/entities/entity-abc123" \
  -H "Authorization: ApiKey YOUR_KEY"

# Search entities
curl "https://aleph.example.com/api/2/entities?filter:schema=Person&q=john" \
  -H "Authorization: ApiKey YOUR_KEY"
```

### Creating Entity via API

```bash
curl -X POST "https://aleph.example.com/api/2/entities" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "schema": "Person",
    "properties": {
      "name": ["John Smith"],
      "birthDate": ["1980-01-15"],
      "nationality": ["US"]
    },
    "collection_id": 123
  }'
```

### Bulk Loading Entities

```bash
curl -X POST "https://aleph.example.com/api/2/collections/123/_bulk" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "id": "person-1",
      "schema": "Person",
      "properties": {"name": ["Alice"]}
    },
    {
      "id": "person-2",
      "schema": "Person",
      "properties": {"name": ["Bob"]}
    }
  ]'
```

## Entity Model

### Database Schema

**Table:** `entity`

| Column | Type | Description |
|--------|------|-------------|
| `id` | String (128) | Unique entity ID |
| `schema` | String | FTM schema name |
| `data` | JSONB | Entity properties |
| `collection_id` | Integer | Parent collection |
| `role_id` | Integer | Creator user ID |
| `created_at` | DateTime | Creation timestamp |
| `updated_at` | DateTime | Last modification |

**Code Reference:** `aleph/model/entity.py:1-104`

### Entity Structure

**Complete Entity Object:**
```json
{
  "id": "entity-abc123",
  "schema": "Person",
  "properties": {
    "name": ["John Smith"],
    "birthDate": ["1980-01-15"]
  },
  "collection_id": 123,
  "role_id": 1,
  "created_at": "2025-12-09T10:00:00Z",
  "updated_at": "2025-12-09T11:00:00Z"
}
```

**Indexed Fields (Elasticsearch):**
```json
{
  "id": "entity-abc123",
  "schema": "Person",
  "schemata": ["Thing", "LegalEntity", "Person"],
  "caption": "John Smith",
  "fingerprints": ["johnsmith", "jsmith"],
  "text": "John Smith 1980-01-15 US",
  "properties": {...},
  "collection_id": 123,
  "role_id": 1,
  "mutable": true,
  "origin": ["model"]
}
```

**Code Reference:** `aleph/index/entities.py:183-236`

### Entity Origins

Entities can come from different sources:

| Origin | Description | Editable |
|--------|-------------|----------|
| `model` | Created interactively (UI/API) | Yes |
| `bulk` | Bulk import via API | Configurable |
| `xref` | Cross-reference generated | No |
| `profile` | Profile fragment | No |
| `aleph` | System-generated | Varies |

**Code Reference:** `aleph/logic/aggregator.py`

## Creating Entities

### Via API (Interactive)

**Create New Entity:**
```bash
POST /api/2/entities

# Request
{
  "schema": "Company",
  "properties": {
    "name": ["ACME Corp"],
    "jurisdiction": ["US"],
    "registrationNumber": ["12-3456789"]
  },
  "collection_id": 123
}

# Response
{
  "id": "entity-generated-id",
  "schema": "Company",
  "properties": {...},
  "collection_id": 123,
  "writeable": true,
  "created_at": "2025-12-09T10:00:00Z",
  "links": {
    "self": "https://aleph.example.com/api/2/entities/entity-generated-id",
    "ui": "https://aleph.example.com/entities/entity-generated-id"
  }
}
```

**With Custom ID:**
```json
{
  "id": "my-custom-id",
  "schema": "Person",
  "properties": {
    "name": ["Alice"]
  },
  "collection_id": 123
}
```

**Note:** Custom IDs will be signed with collection namespace:
```python
signed_id = collection.ns.sign("my-custom-id")
# Result: "namespace-prefix.my-custom-id"
```

**Code Reference:** `aleph/views/entities_api.py:226-277`, `aleph/logic/entities.py:24-53`

### Via Bulk Import

**Bulk Loading:**
```bash
POST /api/2/collections/123/_bulk

# Request body: Array of entities
[
  {
    "id": "person-1",
    "schema": "Person",
    "properties": {"name": ["Alice"], "birthDate": ["1990"]}
  },
  {
    "id": "company-1",
    "schema": "Company",
    "properties": {"name": ["ACME"], "jurisdiction": ["US"]}
  },
  {
    "id": "ownership-1",
    "schema": "Ownership",
    "properties": {
      "owner": ["person-1"],
      "asset": ["company-1"],
      "percentage": ["100"]
    }
  }
]
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `safe` | boolean | `true` | Remove checksums (admin only) |
| `clean` | boolean | `true` | Validate properties |
| `mutable` | boolean | `false` | Allow UI editing |

**Safe Mode:**
- `safe=true`: Removes integrity checksums, allows editing
- `safe=false`: Preserves checksums, prevents tampering (admin only)

**Clean Mode:**
- `clean=true`: Validates against FTM schema
- `clean=false`: Skips validation (faster, risky)

**Mutable Flag:**
- `mutable=true`: Users can edit in UI
- `mutable=false`: Read-only (recommended for imported data)

**Code Reference:** `aleph/views/collections_api.py:222-311`, `aleph/logic/processing.py:36-58`

### Via Mapping (CSV Import)

**Mapping Workflow:**
1. Upload CSV to collection
2. Create mapping (define column → property mappings)
3. Execute mapping to generate entities

**Example Mapping:**
```json
{
  "table": "table-entity-id",
  "query": {
    "schema": "Person",
    "keys": ["id"],
    "properties": {
      "name": {"column": "full_name"},
      "birthDate": {"column": "dob"},
      "nationality": {"column": "country"}
    }
  }
}
```

**API Endpoint:**
```bash
POST /api/2/collections/123/mappings/456/trigger
```

**Code Reference:** `aleph/model/mapping.py`, `aleph/views/mappings_api.py`

### Via Document Ingestion

Documents are automatically created as entities when uploaded:

```bash
POST /api/2/collections/123/ingest

# Multipart form data
file: contract.pdf
metadata: {"title": "Service Agreement", "date": "2024-01-15"}
```

**Generated Entity:**
```json
{
  "schema": "Document",
  "properties": {
    "fileName": ["contract.pdf"],
    "title": ["Service Agreement"],
    "date": ["2024-01-15"],
    "contentHash": ["sha1:abc123..."],
    "mimeType": ["application/pdf"],
    "fileSize": ["1234567"]
  }
}
```

**Code Reference:** `aleph/views/ingest_api.py:77-152`

## Managing Entities

### Updating Entities

```bash
POST /api/2/entities/{entity_id}

# Request (partial update)
{
  "properties": {
    "name": ["John Smith", "J. Smith"],
    "email": ["john.smith@example.com"]
  }
}
```

**Update Behavior:**
- Properties are **replaced**, not merged
- Omitted properties remain unchanged
- Checksums are preserved (protected from tampering)

**Protected Properties:**
- `id`: Cannot be changed
- `schema`: Cannot be changed
- Checksums: Preserved during updates (unless admin overrides)

**Code Reference:** `aleph/model/entity.py:42-56`, `aleph/logic/entities.py:24-53`

### Deleting Entities

```bash
DELETE /api/2/entities/{entity_id}
```

**Deletion Process:**
1. Remove from Elasticsearch index
2. Clear Redis cache
3. Queue prune operation (async)

**Prune Operation:**
- Recursively deletes adjacent entities (if orphaned)
- Removes from entity sets
- Clears bookmarks
- Deletes xref matches
- Removes from aggregator

**Warning:** Deletion is permanent and cannot be undone.

**Code Reference:** `aleph/logic/entities.py:142-179`

### Validating Entities

**Automatic Validation:**
Entities are validated on creation/update if `clean=true` (default).

**Validation Checks:**
1. Schema exists in FTM model
2. All properties are valid for schema
3. Property values match type (date, country, etc.)
4. Required properties present (schema-dependent)

**Validation Errors:**
```json
{
  "status": "error",
  "errors": {
    "properties": {
      "birthDate": ["Invalid date format: '1980'"]
    }
  }
}
```

**Manual Validation:**
```python
from aleph.logic.entities import validate_entity

try:
    validate_entity(data)
except InvalidData as e:
    print(e.errors)
```

**Code Reference:** `aleph/logic/entities.py:103-112`

## Entity Search

### Basic Search

```bash
# Search by text
GET /api/2/entities?q=john+smith

# Filter by schema
GET /api/2/entities?filter:schema=Person

# Filter by collection
GET /api/2/entities?filter:collection_id=123

# Combine filters
GET /api/2/entities?q=smith&filter:schema=Person&filter:countries=US
```

### Advanced Filtering

**Property Filters:**
```bash
# Filter by specific property
GET /api/2/entities?filter:properties.nationality=US

# Multiple values (OR)
GET /api/2/entities?filter:countries=US&filter:countries=GB
```

**Range Queries:**
```bash
# Date range
GET /api/2/entities?filter:properties.birthDate>gte:1980-01-01&filter:properties.birthDate<lt:1990-01-01

# Numeric range
GET /api/2/entities?filter:properties.amount>gte:1000000
```

**Schema Filtering:**
```bash
# Single schema
GET /api/2/entities?filter:schema=Person

# Multiple schemas (OR)
GET /api/2/entities?filter:schemata=Person&filter:schemata=Company
```

**Code Reference:** `aleph/search/__init__.py:63-79`, `aleph/search/query.py`

### Faceted Search

```bash
# Get facets
GET /api/2/entities?facet=schema,countries,languages

# Response
{
  "total": 1500,
  "facets": {
    "schema": {"Person": 800, "Company": 500, "Document": 200},
    "countries": {"US": 600, "GB": 400, "FR": 300},
    "languages": {"eng": 1000, "fra": 300, "spa": 200}
  },
  "results": [...]
}
```

### Sorting and Pagination

```bash
# Sort by date (descending)
GET /api/2/entities?sort=properties.date:desc

# Sort by name (ascending)
GET /api/2/entities?sort=caption:asc

# Pagination
GET /api/2/entities?limit=50&offset=100
```

**Default Sort:** By relevance score (`_score`)

**Code Reference:** `aleph/search/query.py`

## Entity Relationships

### Viewing Related Entities

**Entity Expansion:**
```bash
GET /api/2/entities/{entity_id}/expand

# Response
{
  "results": [
    {
      "property": "director",
      "count": 5,
      "entities": [
        {"id": "person-1", "schema": "Person", "properties": {...}},
        {"id": "person-2", "schema": "Person", "properties": {...}}
      ]
    },
    {
      "property": "shareholder",
      "count": 3,
      "entities": [...]
    }
  ]
}
```

**Filter by Property:**
```bash
GET /api/2/entities/{entity_id}/expand?filter:property=director
```

**Limit Results:**
```bash
GET /api/2/entities/{entity_id}/expand?limit=10
```

**Code Reference:** `aleph/views/entities_api.py:510-572`, `aleph/logic/expand.py:50-100`

### Creating Relationships

**Example: Ownership**
```bash
POST /api/2/entities

{
  "schema": "Ownership",
  "properties": {
    "owner": ["person-id-123"],
    "asset": ["company-id-456"],
    "percentage": ["51"],
    "startDate": ["2015-01-01"]
  },
  "collection_id": 123
}
```

**Example: Directorship**
```bash
POST /api/2/entities

{
  "schema": "Directorship",
  "properties": {
    "director": ["person-id-123"],
    "organization": ["company-id-456"],
    "role": ["CEO"]
  },
  "collection_id": 123
}
```

### Network Diagrams

Relationships can be visualized as network graphs in the UI:
- Nodes: Entities
- Edges: Relationships
- Interactive exploration

**Code Reference:** Frontend components in `ui/src/components/NetworkDiagram/`

## Entity Profiles

Profiles are merged representations of entities that represent the same real-world entity.

### Creating Profiles

**Via Cross-Reference Decision:**
```bash
POST /api/2/profiles/_pairwise

{
  "entity_id": "person-1",
  "match_id": "person-2",
  "judgement": "positive"
}
```

**Effect:**
- Creates profile containing both entities
- Merged entity shown in search results
- Properties combined from both sources

**Code Reference:** `aleph/views/profiles_api.py:207-261`, `aleph/logic/profiles.py:144-186`

### Viewing Profiles

```bash
GET /api/2/profiles/{profile_id}

# Response
{
  "id": "profile-123",
  "type": "profile",
  "label": "John Smith",
  "collection_id": 123,
  "entities": [
    {"id": "person-1", "judgement": "positive"},
    {"id": "person-2", "judgement": "positive"}
  ],
  "merged": {
    "id": "profile-123",
    "schema": "Person",
    "properties": {
      "name": ["John Smith", "J. Smith", "Johnny"],
      "birthDate": ["1980-01-15"],
      "nationality": ["US", "CA"]
    }
  }
}
```

**Merged Properties:**
- Union of all properties from constituent entities
- Deduplicated values
- Most specific values preferred

**Code Reference:** `aleph/logic/profiles.py:34-88`

### Profile Judgements

| Judgement | Meaning | Effect |
|-----------|---------|--------|
| `positive` | Confirmed match | Included in merged profile |
| `negative` | Not a match | Excluded from profile |
| `unsure` | Uncertain | Marked for review |
| `no_judgement` | Not reviewed | Default state |

**Code Reference:** `aleph/model/entityset.py:17-31`

## API Reference

### Search Entities

```
GET /api/2/entities
GET /api/2/search
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | string | Text search query |
| `filter:schema` | string | Filter by entity type |
| `filter:schemata` | string | Filter by schema (including subtypes) |
| `filter:collection_id` | string | Filter by collection |
| `filter:countries` | string | Filter by country |
| `filter:properties.{name}` | string | Filter by property value |
| `facet` | string | Comma-separated facet fields |
| `sort` | string | Sort field and direction |
| `limit` | integer | Results per page (max: 10000) |
| `offset` | integer | Pagination offset |

**Response:** Standard query response with entities

**Code Reference:** `aleph/views/entities_api.py:75-145`

### Get Entity

```
GET /api/2/entities/{entity_id}
```

**Response:**
```json
{
  "id": "entity-abc123",
  "schema": "Person",
  "properties": {...},
  "collection": {...},
  "collection_id": 123,
  "writeable": true,
  "bookmarked": false,
  "created_at": "2025-12-09T10:00:00Z",
  "links": {
    "self": "https://aleph.example.com/api/2/entities/entity-abc123",
    "expand": "https://aleph.example.com/api/2/entities/entity-abc123/expand",
    "similar": "https://aleph.example.com/api/2/entities/entity-abc123/similar",
    "tags": "https://aleph.example.com/api/2/entities/entity-abc123/tags",
    "ui": "https://aleph.example.com/entities/entity-abc123"
  }
}
```

**Code Reference:** `aleph/views/entities_api.py:280-324`

### Create Entity

```
POST /api/2/entities
```

**Request Body:**
```json
{
  "id": "optional-custom-id",
  "schema": "Person",
  "properties": {
    "name": ["John Smith"],
    "birthDate": ["1980-01-15"]
  },
  "collection_id": 123
}
```

**Response:** 200 OK with created entity

**Code Reference:** `aleph/views/entities_api.py:226-277`

### Update Entity

```
POST /api/2/entities/{entity_id}
```

**Request Body:** Same as Create (properties replaced)

**Response:** 200 OK with updated entity

**Code Reference:** `aleph/views/entities_api.py:226-277`

### Delete Entity

```
DELETE /api/2/entities/{entity_id}
```

**Response:** 204 No Content

**Code Reference:** `aleph/views/entities_api.py:327-352`

### Expand Entity

```
GET /api/2/entities/{entity_id}/expand
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `filter:property` | string | Limit to specific property |
| `limit` | integer | Results per property |

**Response:**
```json
{
  "results": [
    {
      "property": "director",
      "count": 5,
      "entities": [...]
    }
  ]
}
```

**Code Reference:** `aleph/views/entities_api.py:510-572`

### Similar Entities

```
GET /api/2/entities/{entity_id}/similar
```

**Response:** Scored matches from cross-reference

**Code Reference:** `aleph/views/entities_api.py:327-381`

### Entity Tags (Mentions)

```
GET /api/2/entities/{entity_id}/tags
```

**Response:** Documents mentioning the entity

**Code Reference:** `aleph/views/entities_api.py:383-419`

### Match Query

```
POST /api/2/match
```

**Request Body:**
```json
{
  "schema": "Person",
  "properties": {
    "name": ["John Smith"],
    "birthDate": ["1980"]
  }
}
```

**Query Parameters:**
- `?collection_ids=123,456` - Limit search to collections

**Response:** Scored similar entities

**Code Reference:** `aleph/views/entities_api.py:186-224`

### Bulk Operations

```
POST /api/2/collections/{collection_id}/_bulk
```

**Request Body:** Array of entities

**Parameters:** `safe`, `clean`, `mutable` (see [Creating Entities](#creating-entities))

**Response:** 204 No Content

**Code Reference:** `aleph/views/collections_api.py:222-311`

## Frontend Integration

### Entity Display Components

**EntityScreen:**
Main entity view page with tabs:
- Overview (properties)
- Similar (xref matches)
- Tags (document mentions)
- Network (related entities)

**Location:** `ui/src/screens/EntityScreen/EntityScreen.jsx`

**EntityViewer:**
Property display component:
- Grouped by property type
- Transliterated names
- Clickable entity references

**Location:** `ui/src/components/Timeline/EntityViewer2.tsx`

**EntityTable:**
Table view of entities:
- Sortable columns
- Schema icons
- Bulk selection

**Location:** `ui/src/react-ftm/components/EntityTable/EntityTable.tsx`

### Entity Editing

**EntityCreateDialog:**
Modal for creating new entities:
- Schema selection
- Property editor
- Validation feedback

**Location:** `ui/src/react-ftm/components/common/EntityCreateDialog.tsx`

**EntitySelect:**
Autocomplete for entity references:
- Search existing entities
- Create new entity inline
- Used in relationship properties

**Code Reference:** `ui/src/react-ftm/components/`

### Redux Actions

```javascript
// Fetch entity
export async function fetchEntity(entityId) {
  const response = await get(`/api/2/entities/${entityId}`);
  return response.data;
}

// Create entity
export async function createEntity(data) {
  const response = await post('/api/2/entities', data);
  return response.data;
}

// Update entity
export async function updateEntity(entityId, data) {
  const response = await post(`/api/2/entities/${entityId}`, data);
  return response.data;
}

// Delete entity
export async function deleteEntity(entityId) {
  await del(`/api/2/entities/${entityId}`);
}

// Expand entity
export async function expandEntity(entityId, params) {
  const response = await get(`/api/2/entities/${entityId}/expand`, {params});
  return response.data;
}
```

**Code Reference:** `ui/src/actions/entityActions.js`

## Advanced Topics

### Entity Namespacing

Each collection has a namespace for entity IDs:

```python
# Namespace signing
entity_id = collection.ns.sign("my-id")
# Result: "collection-foreign-id.my-id"

# Apply namespace to entity
entity = collection.ns.apply(entity)
```

**Purpose:**
- Prevent ID collisions across collections
- Deterministic ID generation
- Enable entity deduplication

**Code Reference:** `aleph/model/collection.py:157-160`

### Entity Aggregation

Entities are aggregated using FTM store:

```python
from aleph.logic.aggregator import get_aggregator

# Get collection aggregator
aggregator = get_aggregator(collection)

# Write entity
writer = aggregator.bulk()
writer.put(entity, origin="bulk")
writer.flush()

# Read entities
for entity in aggregator:
    process(entity)
```

**Aggregation Features:**
- Fragment merging (partial entities → complete entity)
- Deduplication by ID
- Origin tracking
- Profile generation

**Code Reference:** `aleph/logic/aggregator.py`

### Entity Fingerprinting

Names are fingerprinted for fuzzy matching:

```python
from fingerprints import generate

fingerprint = generate("John Smith")
# Result: "johnsmith"

fingerprint = generate("Société Générale")
# Result: "societegenerale"
```

**Indexed:**
- `fingerprints.text`: Searchable fingerprints
- Weighted 3x in search queries

**Code Reference:** `aleph/logic/matching.py:26`, `aleph/index/entities.py:195-198`

### Entity Caching

**Redis Cache:**
- Key: `cache.object_key(Entity, entity_id)`
- TTL: 2 hours (EXPIRE setting)
- Invalidated on update/delete

**Bulk Retrieval:**
```python
from aleph.index.entities import entities_by_ids

entities = entities_by_ids(["id1", "id2", "id3"], cached=True)
```

**Code Reference:** `aleph/index/entities.py:112-150`

### Mutable vs Immutable Entities

**Mutable** (`mutable=true`):
- Created interactively
- Users can edit in UI
- Checksums removed

**Immutable** (`mutable=false`):
- Bulk imported
- Read-only in UI
- Checksums preserved

**Check Write Permission:**
```python
from aleph.logic.entities import check_write_entity

writeable = check_write_entity(entity, authz)
# Admin: always True
# User: mutable and has collection write permission
```

**Code Reference:** `aleph/logic/entities.py:115-124`

### Entity Context

Entities carry metadata context:

```json
{
  "id": "entity-id",
  "schema": "Person",
  "properties": {...},
  "context": {
    "created_at": "2025-12-09T10:00:00Z",
    "updated_at": "2025-12-09T11:00:00Z",
    "role_id": 1,
    "mutable": true,
    "profile_id": "profile-123",
    "collection_id": 123
  }
}
```

**Code Reference:** FTM EntityProxy context

## Troubleshooting

### Entity Not Found

**Symptoms:**
- 404 Not Found when accessing entity
- Entity missing from search results

**Possible Causes:**

1. **No Read Permission**
   ```bash
   # Check collection access
   GET /api/2/collections/{collection_id}

   # Grant permission if needed
   POST /api/2/collections/{collection_id}/permissions
   ```

2. **Not Indexed**
   ```bash
   # Re-index collection
   POST /api/2/collections/{collection_id}/reindex
   ```

3. **Deleted**
   ```sql
   -- Check database
   SELECT * FROM entity WHERE id = 'entity-id';
   ```

### Validation Errors

**Symptoms:**
- 400 Bad Request on create/update
- Error: "Invalid property 'xyz' for schema 'Person'"

**Solutions:**

1. **Check Schema Definition**
   ```bash
   # View FTM schema
   curl "https://alephdata.github.io/followthemoney/schema/Person.json"
   ```

2. **Fix Property Names**
   ```json
   {
     "properties": {
       "name": ["John"],  // ✓ Correct
       "full_name": ["John Smith"]  // ✗ Invalid for Person
     }
   }
   ```

3. **Skip Validation** (if absolutely necessary)
   ```bash
   POST /api/2/collections/123/_bulk?clean=false
   ```

### Cannot Edit Entity

**Symptoms:**
- UI shows read-only
- API returns: "Cannot write to entity"

**Causes:**

1. **No Write Permission on Collection**
   ```bash
   GET /api/2/collections/{collection_id}
   # Check: "writeable": true
   ```

2. **Entity Not Mutable**
   ```bash
   GET /api/2/entities/{entity_id}
   # Check: "writeable": false

   # Bulk imported entities are immutable by default
   ```

**Solutions:**
- Grant write permission to collection
- Re-import with `mutable=true` flag
- Admins can always edit

### Relationship Not Showing

**Symptoms:**
- Created relationship entity but not visible in network diagram
- Expand doesn't return related entities

**Causes:**

1. **Invalid Entity Reference**
   ```json
   {
     "schema": "Ownership",
     "properties": {
       "owner": ["non-existent-id"],  // ✗ Invalid
       "asset": ["company-id"]
     }
   }
   ```

2. **Different Collections**
   - Entity A in Collection 1
   - Entity B in Collection 2
   - Relationship in Collection 3
   - User may not have access to all collections

**Solutions:**
- Verify entity IDs exist
- Grant read permissions to all relevant collections
- Re-index collection

### Duplicate Entities

**Symptoms:**
- Same entity appearing multiple times in results
- Xref not detecting duplicates

**Solutions:**

1. **Use Cross-Reference**
   ```bash
   POST /api/2/collections/{collection_id}/xref
   ```

2. **Make Pairwise Decision**
   ```bash
   POST /api/2/profiles/_pairwise
   {
     "entity_id": "duplicate-1",
     "match_id": "duplicate-2",
     "judgement": "positive"
   }
   ```

3. **Delete Duplicates**
   ```bash
   DELETE /api/2/entities/{entity_id}
   ```

### Search Not Finding Entity

**Symptoms:**
- Entity exists but doesn't appear in search results
- Filter returns 0 results

**Diagnosis:**

1. **Check Index**
   ```bash
   GET /api/2/entities/{entity_id}
   # Verify entity is indexed
   ```

2. **Test Direct Access**
   ```bash
   # Can access directly?
   GET /api/2/entities/{entity_id}

   # Appears in collection search?
   GET /api/2/entities?filter:collection_id={collection_id}
   ```

**Solutions:**
- Re-index collection
- Check Elasticsearch cluster health
- Verify search query syntax

### Bulk Import Fails

**Symptoms:**
- 400 Bad Request on bulk import
- Error: "Invalid entity data"

**Common Issues:**

1. **Missing ID**
   ```json
   // ✗ Missing id
   {"schema": "Person", "properties": {"name": ["John"]}}

   // ✓ With id
   {"id": "person-1", "schema": "Person", "properties": {"name": ["John"]}}
   ```

2. **Invalid Schema**
   ```json
   // ✗ Typo
   {"id": "p1", "schema": "Perzon", "properties": {...}}

   // ✓ Correct
   {"id": "p1", "schema": "Person", "properties": {...}}
   ```

3. **Malformed JSON**
   ```bash
   # Validate JSON
   cat entities.json | jq .
   ```

**Solutions:**
- Validate entity structure
- Use `clean=false` to skip validation (if needed)
- Check logs for detailed error messages

---

**Related Documentation:**
- [COLLECTIONS.md](./COLLECTIONS.md) - Collection management
- [XREF.md](./XREF.md) - Cross-referencing entities
- [SEARCH.md](./SEARCH.md) - Searching entities
- [INVESTIGATIONS.md](./INVESTIGATIONS.md) - Entity sets and diagrams
- [API.md](./API.md) - Complete API reference

**External Resources:**
- [FollowTheMoney Documentation](https://followthemoney.tech/) - Complete FTM schema reference
- [FollowTheMoney GitHub](https://github.com/alephdata/followthemoney) - FTM library
- [Aleph User Guide](https://docs.alephdata.org/) - User documentation
