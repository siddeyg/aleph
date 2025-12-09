# Collections

Complete documentation for Aleph's collection management system.

## Table of Contents

- [Overview](#overview)
- [Quick Start](#quick-start)
- [Collection Types](#collection-types)
- [Creating Collections](#creating-collections)
- [Managing Collections](#managing-collections)
- [Permission Management](#permission-management)
- [Collection Operations](#collection-operations)
- [API Reference](#api-reference)
- [Frontend Integration](#frontend-integration)
- [Advanced Topics](#advanced-topics)
- [Troubleshooting](#troubleshooting)

## Overview

Collections are the primary organizational unit in Aleph. They group related documents, entities, and investigations together, providing:

- **Access control** through role-based permissions
- **Data organization** with categories and metadata
- **Search isolation** for targeted queries
- **Processing context** for background operations
- **Collaboration** through team access

### Key Concepts

**Collections as Containers:**
- Source datasets (e.g., leaked documents, company registries)
- Investigations and case files
- Analysis workspaces with entity sets

**Collection Hierarchy:**
```
Collection
├── Documents (files uploaded or crawled)
├── Entities (structured data)
├── Mappings (CSV → entities transformation)
├── Entity Sets (lists, diagrams, timelines)
└── Cross-references (matches with other collections)
```

### Collection Categories

Aleph supports 19 collection categories:

| Category | Description | Use Case |
|----------|-------------|----------|
| `casefile` | Investigations | Internal analysis and research |
| `leak` | Leaks | Whistleblower data, leaked documents |
| `news` | News archives | Media collections |
| `court` | Court archives | Legal records and filings |
| `company` | Company registries | Business registration data |
| `sanctions` | Sanctions lists | Watchlists and restricted entities |
| `land` | Land registry | Property ownership records |
| `gazette` | Gazettes | Official government publications |
| `procurement` | Procurement | Public tender and contract data |
| `finance` | Financial records | Banking, tax, financial disclosures |
| `license` | Licenses | Permits, licenses, concessions |
| `regulatory` | Regulatory filings | Corporate compliance documents |
| `poi` | Persons of interest | Lists of notable individuals |
| `customs` | Customs declarations | Import/export records |
| `transport` | Transport registers | Air and maritime registries |
| `census` | Census data | Population statistics |
| `grey` | Grey literature | Reports, studies, misc documents |
| `library` | Document libraries | General document collections |
| `other` | Other material | Uncategorized collections |

**Code Reference:** `aleph/model/collection.py:26-46`

## Quick Start

### Creating a Collection via UI

1. Navigate to Aleph homepage
2. Click "New Investigation" or "Upload Documents"
3. Fill in metadata:
   - **Label**: Display name
   - **Summary**: Description
   - **Category**: Type of collection
   - **Countries**: Relevant jurisdictions
   - **Languages**: Content languages
4. Click "Create"
5. Grant permissions to team members

### Creating a Collection via API

```bash
# Create new collection
curl -X POST "https://aleph.example.com/api/2/collections" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "My Investigation",
    "summary": "Research on XYZ Corp",
    "category": "casefile",
    "countries": ["US", "GB"],
    "languages": ["eng"]
  }'

# Response
{
  "id": 123,
  "label": "My Investigation",
  "category": "casefile",
  "casefile": true,
  "secret": true,
  "writeable": true,
  "creator": {
    "id": 1,
    "name": "Alice"
  },
  "team": [
    {"id": 1, "name": "Alice", "type": "user"}
  ],
  "created_at": "2025-12-09T10:00:00Z"
}
```

### Viewing Collections

```bash
# List accessible collections
curl "https://aleph.example.com/api/2/collections" \
  -H "Authorization: ApiKey YOUR_KEY"

# Get collection details
curl "https://aleph.example.com/api/2/collections/123" \
  -H "Authorization: ApiKey YOUR_KEY"
```

## Collection Types

### 1. Source Collections

**Purpose:** Store original source data (documents, entities)

**Characteristics:**
- Category: `leak`, `company`, `court`, etc.
- Contains uploaded or crawled documents
- May be public or restricted
- Read-only for most users

**Example:**
```json
{
  "id": 456,
  "label": "ICIJ Panama Papers",
  "category": "leak",
  "casefile": false,
  "secret": false,
  "publisher": "ICIJ",
  "publisher_url": "https://www.icij.org",
  "data_url": "https://offshoreleaks.icij.org",
  "countries": ["PA", "BVI", "KY"],
  "frequency": "never"
}
```

### 2. Casefiles (Investigations)

**Purpose:** Private investigation workspaces

**Characteristics:**
- Category: `casefile`
- Always secret (no public access)
- User-created for analysis
- Contains entity sets (diagrams, timelines, lists)
- Writeable by creator and team

**Example:**
```json
{
  "id": 123,
  "label": "Operation Phoenix",
  "category": "casefile",
  "casefile": true,
  "secret": true,
  "writeable": true,
  "restricted": true,
  "creator": {"id": 1, "name": "Alice"},
  "team": [
    {"id": 1, "name": "Alice"},
    {"id": 5, "name": "Investigators Group"}
  ]
}
```

**Special Properties:**
- `casefile`: Always `true` for investigations
- `secret`: Always `true` (no public roles)
- `restricted`: Flag for highly sensitive material

**Code Reference:** `aleph/model/collection.py:150-154`

### 3. Public Collections

**Purpose:** Openly accessible datasets

**Characteristics:**
- Guest role has read permission
- Listed in public catalog
- `secret`: `false`
- Often official registries or published leaks

**Example:**
```bash
# Check if collection is public
GET /api/2/collections/456

# Response includes:
{
  "secret": false,  # Public collection
  "writeable": false  # Read-only for this user
}
```

**Code Reference:** `aleph/model/collection.py:142-147`

## Creating Collections

### Collection Properties

**Required:**
- `label` (string): Display name

**Optional Metadata:**
- `summary` (string): Description of contents
- `category` (string): One of 19 categories (default: `casefile`)
- `countries` (array): ISO country codes
- `languages` (array): ISO language codes

**Optional Settings:**
- `foreign_id` (string): External identifier (auto-generated if not provided)
- `restricted` (boolean): Mark as restricted/secret
- `xref` (boolean): Enable cross-reference matching (default: `false`)

**Optional Publication Info:**
- `publisher` (string): Publishing organization
- `publisher_url` (string): Publisher website
- `info_url` (string): Additional information URL
- `data_url` (string): Raw data download URL

**Optional Frequency:**
- `frequency` (string): Update schedule
  - `unknown` (default)
  - `never`: Static dataset
  - `daily`, `weekly`, `monthly`, `annual`: Regular updates

**Code Reference:** `aleph/model/collection.py:20-86`

### Creating via API

```bash
# Full example
curl -X POST "https://aleph.example.com/api/2/collections" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "foreign_id": "my-unique-id",
    "label": "UK Companies House",
    "summary": "Official UK company registry data",
    "category": "company",
    "countries": ["GB"],
    "languages": ["eng"],
    "publisher": "Companies House",
    "publisher_url": "https://www.gov.uk/government/organisations/companies-house",
    "info_url": "https://www.gov.uk/government/organisations/companies-house/about",
    "data_url": "http://download.companieshouse.gov.uk/en_output.html",
    "frequency": "daily",
    "xref": true
  }'
```

**Sync Parameter:**
```bash
# Wait for indexing to complete (default: true)
POST /api/2/collections?sync=true

# Async (faster response, indexing happens in background)
POST /api/2/collections?sync=false
```

**Code Reference:** `aleph/views/collections_api.py:47-76`, `aleph/logic/collections.py:21-32`

### Automatic Permission Grant

When you create a collection, you automatically receive:
- **Read** permission (view contents)
- **Write** permission (modify and add content)

These permissions are stored in the `Permission` table:
```python
Permission.grant(collection, creator_role, read=True, write=True)
```

**Code Reference:** `aleph/model/collection.py:258-262`

### Foreign ID Uniqueness

Collections must have unique `foreign_id`:
```python
# Auto-generated if not provided
foreign_id = make_textid()  # e.g., "01HF2G3M4K5N6P7Q8R9S0T"

# Custom foreign_id
foreign_id = "my-dataset-2024"

# Collision check
existing = Collection.by_foreign_id(foreign_id)
if existing and not existing.deleted_at:
    raise ValueError("Collection exists with foreign_id: ...")
```

**Re-creating Deleted Collections:**
Only the original creator can recreate a collection with the same `foreign_id`.

**Code Reference:** `aleph/model/collection.py:237-262`

## Managing Collections

### Updating Metadata

**Via API:**
```bash
# Update collection
curl -X POST "https://aleph.example.com/api/2/collections/123" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "Updated Name",
    "summary": "New description",
    "countries": ["US", "CA"],
    "xref": true
  }'
```

**Updateable Fields (Regular Users):**
- `label`, `summary`
- `countries`, `languages`
- `publisher`, `publisher_url`, `info_url`, `data_url`
- `frequency`
- `restricted`, `xref`

**Admin-Only Fields:**
- `category`
- `creator_id`

**Code Reference:** `aleph/model/collection.py:93-127`, `aleph/views/collections_api.py:113-150`

### Touching Collections

"Touching" updates the `data_updated_at` timestamp:

```bash
# Touch collection (admin only)
curl -X POST "https://aleph.example.com/api/2/collections/123/touch" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Use Cases:**
- Mark collection as recently updated
- Trigger re-indexing in downstream systems
- Refresh cache

**Code Reference:** `aleph/model/collection.py:88-91`, `aleph/views/collections_api.py:415-441`

### Deleting Collections

```bash
# Delete collection (removes all content)
curl -X DELETE "https://aleph.example.com/api/2/collections/123" \
  -H "Authorization: ApiKey YOUR_KEY"

# Delete content but keep metadata
curl -X DELETE "https://aleph.example.com/api/2/collections/123?keep_metadata=true" \
  -H "Authorization: ApiKey YOUR_KEY"

# Async deletion
curl -X DELETE "https://aleph.example.com/api/2/collections/123?sync=false" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Deletion Process:**
1. Cancel queued background tasks
2. Delete aggregator data (cached entities)
3. Flush notifications
4. Delete indexed entities (Elasticsearch)
5. Delete xref matches
6. Delete mappings
7. Delete entity sets (diagrams, lists, timelines)
8. Delete entities (database)
9. Delete documents (database and files)
10. Delete permissions (if not `keep_metadata`)
11. Delete collection record (if not `keep_metadata`)
12. Refresh caches

**Warning:** Deletion is permanent and cannot be undone.

**Code Reference:** `aleph/logic/collections.py:192-212`, `aleph/views/collections_api.py:377-412`

## Permission Management

### Permission Model

Collections use role-based access control (RBAC):

```
Collection → Permission → Role (User or Group)
```

**Permission Types:**
- **Read**: View collection contents
- **Write**: Modify collection and add content (implies read)

**Special Roles:**
- **Guest** (`system:guest`): Anonymous users
- **User Groups**: Shared team access
- **Individual Users**: Personal access

**Code Reference:** `aleph/model/permission.py:8-63`

### Viewing Permissions

```bash
# Get collection permissions
curl "https://aleph.example.com/api/2/collections/123/permissions" \
  -H "Authorization: ApiKey YOUR_KEY"

# Response
{
  "total": 3,
  "results": [
    {
      "id": 1,
      "role_id": 5,
      "role": {"id": 5, "name": "Investigators", "type": "group"},
      "read": true,
      "write": true,
      "collection_id": 123
    },
    {
      "id": 2,
      "role_id": 10,
      "role": {"id": 10, "name": "Bob", "type": "user"},
      "read": true,
      "write": false,
      "collection_id": 123
    },
    {
      "role_id": 2,
      "role": {"id": 2, "name": "Analysts", "type": "group"},
      "read": false,
      "write": false
    }
  ]
}
```

**Interpretation:**
- Granted permissions: `read: true` or `write: true`
- Available roles: `read: false, write: false` (can be granted)

**Code Reference:** `aleph/views/permissions_api.py:17-79`

### Granting Permissions

```bash
# Grant permissions to multiple roles
curl -X POST "https://aleph.example.com/api/2/collections/123/permissions" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '[
    {"role_id": 10, "read": true, "write": false},
    {"role_id": 5, "read": true, "write": true}
  ]'
```

**Rules:**
1. **Write implies read**: Setting `write: true` automatically grants read
2. **No public write**: Guest role cannot have write permission
3. **No public casefiles**: Guest role cannot access casefile collections
4. **Revoke**: Set `read: false` to revoke all access

**Code Reference:** `aleph/model/permission.py:33-49`, `aleph/views/permissions_api.py:82-158`

### Making Collections Public

To publish a collection:
```bash
# Grant read to guest role
curl -X POST "https://aleph.example.com/api/2/collections/123/permissions" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '[
    {"role_id": 2, "read": true, "write": false}
  ]'
```

**Note:** Role ID `2` is the system guest role.

**Event:** Triggers `PUBLISH_COLLECTION` notification to all users.

**Code Reference:** `aleph/logic/permissions.py:11-35`

### Permission Caching

Permissions are cached in Redis for performance:

**Cache Key:** `authzca:<role_id>`

**Cache Contains:**
- List of readable collection IDs
- List of writable collection IDs

**TTL:** Cached until permission change or explicit flush

**Invalidation:**
```python
# Automatic on permission update
Authz.flush(role_id)
```

**Code Reference:** `aleph/authz.py:38-61`

## Collection Operations

### Re-indexing

Re-index entities in collection (useful after schema changes or corruption):

```bash
# Re-index all entities
curl -X POST "https://aleph.example.com/api/2/collections/123/reindex" \
  -H "Authorization: ApiKey YOUR_KEY"

# Flush existing index first
curl -X POST "https://aleph.example.com/api/2/collections/123/reindex?flush=true" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Process:**
1. Re-apply all mappings to aggregator
2. Aggregate database model entities
3. Profile entity fragments (resolve relationships)
4. Optionally flush existing Elasticsearch index
5. Batch index all aggregated entities
6. Recompute collection statistics

**Duration:** Minutes to hours depending on collection size

**Code Reference:** `aleph/logic/collections.py:168-189`, `aleph/views/collections_api.py:189-219`

### Re-ingesting

Re-parse all documents (useful after ingest-file updates):

```bash
# Re-ingest all documents
curl -X POST "https://aleph.example.com/api/2/collections/123/reingest" \
  -H "Authorization: ApiKey YOUR_KEY"

# Re-ingest and index immediately
curl -X POST "https://aleph.example.com/api/2/collections/123/reingest?index=true" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Process:**
1. Queue STAGE_REINGEST task for each document
2. Documents sent to ingest-file service
3. Extract text, metadata, entities
4. Store results in database
5. Optionally index during ingestion

**Duration:** Hours to days for large document collections

**Code Reference:** `aleph/logic/collections.py:156-165`, `aleph/views/collections_api.py:153-186`

### Checking Status

Monitor background processing:

```bash
# Check collection status
curl "https://aleph.example.com/api/2/collections/123/status" \
  -H "Authorization: ApiKey YOUR_KEY"

# Response
{
  "pending": 150,
  "running": 5,
  "finished": 845
}
```

**Status Fields:**
- `pending`: Tasks waiting in queue
- `running`: Tasks currently processing
- `finished`: Completed tasks

**Code Reference:** `aleph/views/collections_api.py:314-342`

### Canceling Operations

Stop all queued tasks:

```bash
# Cancel all pending tasks
curl -X DELETE "https://aleph.example.com/api/2/collections/123/status" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Effect:** Removes tasks from RabbitMQ queue, but cannot stop running tasks.

**Code Reference:** `aleph/views/collections_api.py:345-374`

### Bulk Loading Entities

Load entities programmatically via API:

```bash
# Bulk load entities
curl -X POST "https://aleph.example.com/api/2/collections/123/_bulk" \
  -H "Authorization: ApiKey YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "id": "entity-1",
      "schema": "Person",
      "properties": {
        "name": ["John Smith"],
        "birthDate": ["1980-01-15"]
      }
    },
    {
      "id": "entity-2",
      "schema": "Company",
      "properties": {
        "name": ["ACME Corp"]
      }
    }
  ]'
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `safe` | boolean | `true` | Remove checksums from entities (admin only) |
| `clean` | boolean | `true` | Validate entity properties (admin only) |
| `mutable` | boolean | `false` | Allow UI modification of bulk entities |

**Safe Mode:**
Removes integrity checksums, allowing entities to be edited. Only admins can set `safe=false` to preserve checksums (making entities immutable).

**Clean Mode:**
Validates entities against FollowTheMoney schema. Set `clean=false` to skip validation (faster, but may store invalid data).

**Mutable Flag:**
- `mutable=true`: Entities can be edited in UI
- `mutable=false`: Entities read-only in UI (use for generated/imported data)

**Code Reference:** `aleph/views/collections_api.py:222-311`, `aleph/logic/processing.py:36-58`

## API Reference

### List Collections

```
GET /api/2/collections
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | string | Text search in collection labels |
| `filter:writeable` | boolean | Only show user-editable collections |
| `limit` | integer | Results per page (default: 50) |
| `offset` | integer | Pagination offset |

**Response:**
```json
{
  "total": 150,
  "results": [
    {
      "id": 123,
      "label": "My Investigation",
      "category": "casefile",
      "casefile": true,
      "secret": true,
      "writeable": true,
      "count": 450,
      "creator": {"id": 1, "name": "Alice"},
      "created_at": "2025-12-09T10:00:00Z",
      "links": {
        "self": "https://aleph.example.com/api/2/collections/123",
        "ui": "https://aleph.example.com/investigations/123"
      }
    }
  ]
}
```

**Fuzzy Search:**
Supports fuzzy matching (e.g., "russia" matches "Russian Federation"):
```bash
curl "https://aleph.example.com/api/2/collections?q=russia" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Code Reference:** `aleph/views/collections_api.py:23-44`, `aleph/search/__init__.py:21-60`

### Get Collection

```
GET /api/2/collections/<id>
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `refresh` | boolean | Recompute statistics (expensive) |

**Response:**
```json
{
  "id": 123,
  "label": "Panama Papers",
  "summary": "ICIJ leaked documents",
  "category": "leak",
  "casefile": false,
  "secret": false,
  "restricted": true,
  "writeable": false,
  "count": 1500000,
  "schemata": {
    "Person": 250000,
    "Company": 400000,
    "Document": 850000
  },
  "countries": ["PA", "BVI", "KY"],
  "languages": ["eng", "spa"],
  "frequency": "never",
  "publisher": "ICIJ",
  "publisher_url": "https://www.icij.org",
  "creator": {"id": 10, "name": "ICIJ Admin"},
  "team": [
    {"id": 10, "name": "ICIJ Admin"},
    {"id": 15, "name": "ICIJ Team"}
  ],
  "created_at": "2016-04-03T00:00:00Z",
  "updated_at": "2025-12-09T10:00:00Z",
  "links": {
    "self": "https://aleph.example.com/api/2/collections/123",
    "xref_export": "https://aleph.example.com/api/2/collections/123/xref.xlsx",
    "reconcile": "https://aleph.example.com/api/2/reconcile/123",
    "ui": "https://aleph.example.com/datasets/123"
  }
}
```

**Statistics:**
- `count`: Total entity count
- `schemata`: Entity type distribution

**Code Reference:** `aleph/views/collections_api.py:79-110`

### Create Collection

```
POST /api/2/collections
```

**Request Body:**
```json
{
  "label": "My Collection",
  "summary": "Description",
  "category": "casefile",
  "countries": ["US"],
  "languages": ["eng"],
  "xref": false
}
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sync` | boolean | `true` | Wait for indexing to complete |

**Response:** 200 OK with collection object

**Code Reference:** `aleph/views/collections_api.py:47-76`

### Update Collection

```
POST /api/2/collections/<id>
```

**Request Body:** Same as Create (partial updates supported)

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sync` | boolean | `true` | Wait for re-indexing |

**Response:** 200 OK with updated collection object

**Code Reference:** `aleph/views/collections_api.py:113-150`

### Delete Collection

```
DELETE /api/2/collections/<id>
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `sync` | boolean | `true` | Wait for deletion to complete |
| `keep_metadata` | boolean | `false` | Delete content but preserve collection record |

**Response:** 204 No Content

**Code Reference:** `aleph/views/collections_api.py:377-412`

### Re-index Collection

```
POST /api/2/collections/<id>/reindex
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `flush` | boolean | `false` | Clear existing index before re-indexing |

**Response:** 202 Accepted

**Code Reference:** `aleph/views/collections_api.py:189-219`

### Re-ingest Collection

```
POST /api/2/collections/<id>/reingest
```

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `index` | boolean | `false` | Index documents during ingestion |

**Response:** 202 Accepted

**Code Reference:** `aleph/views/collections_api.py:153-186`

### Bulk Load Entities

```
POST /api/2/collections/<id>/_bulk
```

**Request Body:** Array of entity objects

**Query Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `safe` | boolean | `true` | Remove checksums (admin only) |
| `clean` | boolean | `true` | Validate properties (admin only) |
| `mutable` | boolean | `false` | Allow UI editing |

**Response:** 204 No Content

**Code Reference:** `aleph/views/collections_api.py:222-311`

### Get Permissions

```
GET /api/2/collections/<id>/permissions
```

**Response:** List of permission objects with granted and available roles

**Code Reference:** `aleph/views/permissions_api.py:17-79`

### Update Permissions

```
POST /api/2/collections/<id>/permissions
```

**Request Body:** Array of permission updates
```json
[
  {"role_id": 5, "read": true, "write": true},
  {"role_id": 10, "read": true, "write": false},
  {"role_id": 15, "read": false, "write": false}
]
```

**Response:** 200 OK with updated permissions

**Code Reference:** `aleph/views/permissions_api.py:82-158`

## Frontend Integration

### Collection Components

**File:** `ui/src/components/Collection/`

#### CollectionView.jsx
Main collection page with tabs:
- Overview (metadata and statistics)
- Documents (browse files)
- Entities (structured data)
- Mappings (CSV transformations)
- Diagrams (network graphs)
- Xref (cross-reference matches)

#### CollectionIndex.jsx
Collection browser:
- Search by label
- Filter by creator
- Sort by update date, label, entity count
- Facet by category

#### CollectionManageMenu.jsx
Management dropdown:
- Edit (opens settings dialog)
- Share (permission management)
- Delete
- Re-ingest documents
- Re-index entities

#### CollectionInfo.jsx
Metadata display panel:
- Publisher information
- External URLs
- Creator and team
- Countries and languages
- Category and frequency

#### CollectionStatistics.jsx
Visual statistics:
- Entity type distribution (bar chart)
- Country distribution
- Language distribution

#### CollectionStatus.jsx
Background processing indicator:
- Shows pending/running/finished tasks
- Displays progress bar
- Cancel button

**Code Reference:** `ui/src/components/Collection/`

### Redux Actions

**File:** `ui/src/actions/collectionActions.js`

**Key Actions:**

```javascript
// Query collections
export function queryCollections(location) {
  const query = collectionsQuery(location);
  return queryEndpoint(query);
}

// Fetch collection
export async function fetchCollection(collectionId) {
  const response = await get(`/api/2/collections/${collectionId}`);
  return response.data;
}

// Create collection
export async function createCollection(data) {
  const response = await post('/api/2/collections', data);
  return response.data;
}

// Update collection
export async function updateCollection(collectionId, data) {
  const response = await post(`/api/2/collections/${collectionId}`, data);
  return response.data;
}

// Delete collection
export async function deleteCollection(collectionId) {
  await del(`/api/2/collections/${collectionId}`);
}

// Trigger re-index
export async function triggerCollectionReindex(collectionId) {
  await post(`/api/2/collections/${collectionId}/reindex`);
}

// Trigger re-ingest
export async function triggerCollectionReingest(collectionId) {
  await post(`/api/2/collections/${collectionId}/reingest`);
}

// Cancel tasks
export async function triggerCollectionCancel(collectionId) {
  await del(`/api/2/collections/${collectionId}/status`);
}
```

**Code Reference:** `ui/src/actions/collectionActions.js`

## Advanced Topics

### Collection Namespaces

Each collection has a FollowTheMoney namespace for entity ID signing:

```python
# Namespace creation
namespace = Namespace(collection.foreign_id)

# Apply namespace to entity
entity = namespace.apply(entity)
# Result: entity.id = "namespace.sign(original_id)"
```

**Purpose:**
- Prevent ID collisions across collections
- Deterministic entity IDs based on foreign_id + original ID
- Enables entity deduplication

**Code Reference:** `aleph/model/collection.py:157-160`

### Collection Aggregator

Collections use an aggregator pattern for entity storage:

```python
from aleph.logic.aggregator import get_aggregator

# Get collection aggregator
aggregator = get_aggregator(collection)

# Write entities
writer = aggregator.bulk()
writer.put(entity, origin="bulk")
writer.flush()

# Read entities
for entity in aggregator:
    process(entity)
```

**Aggregator Features:**
- In-memory entity cache
- Deduplication by entity ID
- Fragment merging (multiple partial entities → single complete entity)
- Origin tracking (bulk, model, mapping, xref)

**Code Reference:** `aleph/logic/aggregator.py`

### Collection Statistics

Statistics are computed and cached:

```python
# Compute statistics
compute_collection(collection, force=True)

# Retrieve from cache
stats = get_collection_stats(collection.id)
# Returns: {"schema": {...}, "countries": {...}, ...}
```

**Cached Facets:**
- `schema`: Entity type distribution
- `names`: Most common names
- `addresses`: Most common addresses
- `phones`: Phone number distribution
- `emails`: Email distribution
- `countries`: Geographic distribution
- `languages`: Language distribution
- `ibans`: Bank account distribution

**Cache TTL:** Until next compute operation or manual invalidation

**Code Reference:** `aleph/index/collections.py:131-161`, `aleph/logic/collections.py:92-100`

### Multi-Collection Operations

Process multiple collections in batch:

```python
from aleph.logic.collections import index_many

# Index entities from multiple collections
collections = [col1, col2, col3]
index_many(collections, sync=True)
```

**Use Cases:**
- Global re-indexing
- Bulk statistics updates
- Cross-collection operations

**Code Reference:** `aleph/logic/processing.py:18-33`

### Collection Events

Collections trigger notification events:

| Event | Trigger | Channels |
|-------|---------|----------|
| `CREATE_COLLECTION` | Collection created | Creator, Collection |
| `PUBLISH_COLLECTION` | Made public (guest access) | GLOBAL |
| `GRANT_COLLECTION` | Shared with user/group | Recipient |
| `LOAD_MAPPING` | Entities loaded from mapping | Collection |
| `DELETE_COLLECTION` | Collection deleted | Creator |

**Event Structure:**
```python
publish(
    Events.GRANT_COLLECTION,
    actor_id=granter.id,
    params={"role": recipient, "collection": collection},
    channels=[recipient]
)
```

**Code Reference:** `aleph/logic/collections.py:21-32`, `aleph/logic/permissions.py:11-35`

## Troubleshooting

### Cannot Create Collection

**Symptoms:**
- 400 Bad Request
- Error: "Collection exists with foreign_id"

**Cause:** Duplicate `foreign_id`

**Solutions:**
1. Use different `foreign_id`
2. If recreating deleted collection:
   - Must be original creator
   - Include same `foreign_id` in request

**Code Reference:** `aleph/model/collection.py:247-255`

### Collection Not Appearing

**Symptoms:**
- Collection exists but not visible in UI or API

**Possible Causes:**

1. **No Read Permission**
   ```bash
   # Check permissions
   GET /api/2/collections/<id>/permissions

   # Grant read access
   POST /api/2/collections/<id>/permissions
   {"role_id": YOUR_ROLE, "read": true}
   ```

2. **Soft-Deleted**
   ```python
   # Check if deleted
   collection = Collection.by_id(id)
   if collection.deleted_at:
       print("Collection is soft-deleted")
   ```

3. **Not Indexed**
   ```bash
   # Re-index collection
   POST /api/2/collections/<id>/reindex
   ```

### Re-index Stuck

**Symptoms:**
- Re-index never completes
- Status shows tasks pending indefinitely

**Diagnosis:**

1. **Check Worker Logs**
   ```bash
   docker-compose logs -f worker
   ```

2. **Check Queue**
   - Open RabbitMQ management UI
   - Look for stuck messages in `reindex` queue

3. **Check Elasticsearch**
   ```bash
   GET /_cluster/health
   GET /_tasks?detailed=true
   ```

**Solutions:**
- Restart worker: `docker-compose restart worker`
- Cancel tasks: `DELETE /api/2/collections/<id>/status`
- Retry: `POST /api/2/collections/<id>/reindex?flush=true`

### Statistics Not Updating

**Symptoms:**
- Entity counts don't change after adding/removing entities
- Schema distribution outdated

**Solution:**
```bash
# Force statistics recomputation
GET /api/2/collections/<id>?refresh=true
```

**Automatic Refresh:**
Statistics are recomputed after:
- Re-indexing
- Bulk entity loading
- Mapping execution

**Code Reference:** `aleph/logic/collections.py:92-100`

### Permission Changes Not Applying

**Symptoms:**
- Granted permissions don't take effect immediately
- User still cannot access collection

**Cause:** Permission cache not flushed

**Solution:**
```python
# Manual cache flush
from aleph.authz import Authz
Authz.flush(role_id)
```

**Automatic Flush:**
Permission cache is flushed automatically when:
- Permissions updated via API
- Role deleted
- User logs out

**Code Reference:** `aleph/authz.py:91-99`, `aleph/logic/permissions.py:21`

### Bulk Load Fails

**Symptoms:**
- 400 Bad Request
- Error: "No ID for entity"

**Cause:** Entity missing `id` field

**Solution:**
```json
{
  "id": "must-have-id",
  "schema": "Person",
  "properties": {"name": ["John"]}
}
```

**Other Common Errors:**

1. **Invalid Schema**
   ```
   Error: Schema 'Perzon' does not exist
   Solution: Use valid FTM schema (e.g., "Person")
   ```

2. **Invalid Properties**
   ```
   Error: Property 'age' not defined for schema 'Person'
   Solution: Use valid FTM properties
   ```

3. **Validation Errors** (with `clean=true`)
   ```bash
   # Skip validation (faster, less safe)
   POST /collections/<id>/_bulk?clean=false
   ```

**Code Reference:** `aleph/logic/processing.py:36-58`

### Deletion Hangs

**Symptoms:**
- DELETE request times out
- Collection partially deleted

**Cause:** Large collection with many entities

**Solution:**
```bash
# Async deletion
DELETE /api/2/collections/<id>?sync=false

# Monitor status
GET /api/2/collections/<id>/status
```

**Deletion Order:**
Entities deleted before documents to maintain referential integrity.

**Code Reference:** `aleph/logic/collections.py:192-212`

---

**Related Documentation:**
- [SEARCH.md](./SEARCH.md) - Searching collections
- [XREF.md](./XREF.md) - Cross-referencing collections
- [ENTITIES.md](./ENTITIES.md) - Entity management
- [INVESTIGATIONS.md](./INVESTIGATIONS.md) - Casefile features
- [API.md](./API.md) - Complete API reference

**External Resources:**
- [FollowTheMoney Documentation](https://followthemoney.tech/) - Entity schema
- [Aleph User Guide](https://docs.alephdata.org/) - User documentation
