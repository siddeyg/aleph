# Aleph Search Guide

Complete guide to searching and discovering entities in Aleph, including query syntax, filtering, facets, and advanced search features.

## Table of Contents
- [Overview](#overview)
- [Quick Start](#quick-start)
- [Search API](#search-api)
- [Query Syntax](#query-syntax)
- [Filtering](#filtering)
- [Faceted Search](#faceted-search)
- [Sorting & Pagination](#sorting--pagination)
- [Advanced Features](#advanced-features)
- [Frontend Search](#frontend-search)
- [Authorization & Permissions](#authorization--permissions)
- [Performance & Optimization](#performance--optimization)
- [Troubleshooting](#troubleshooting)

---

## Overview

Aleph provides powerful search capabilities for discovering entities across your document collections. The search system is built on **Elasticsearch** and supports:

- **Full-text search** - Search across all indexed text
- **Faceted search** - Drill down by entity type, country, date, etc.
- **Advanced query syntax** - Boolean operators, phrases, wildcards
- **Filtered search** - Filter by schema, properties, collections
- **Cross-reference search** - Find matching entities across datasets
- **Authorization-aware** - Results filtered by user permissions

### Search Architecture

```
┌─────────────────────────────────────────────────────────┐
│  User Query: "corruption AND money laundering"         │
└──────────────────┬──────────────────────────────────────┘
                   │
    ┌──────────────▼──────────────┐
    │   Query Parser              │
    │   (aleph/search/parser.py)  │
    └──────────────┬──────────────┘
                   │
    ┌──────────────▼──────────────┐
    │   Query Builder             │
    │   (aleph/search/query.py)   │
    │   - Add filters             │
    │   - Add authorization       │
    │   - Build ES DSL            │
    └──────────────┬──────────────┘
                   │
    ┌──────────────▼──────────────┐
    │   Elasticsearch             │
    │   aleph-entity-* indices    │
    └──────────────┬──────────────┘
                   │
    ┌──────────────▼──────────────┐
    │   Results + Facets          │
    │   (aleph/search/result.py)  │
    └─────────────────────────────┘
```

---

## Quick Start

### Basic Search

**Via UI**:
1. Navigate to the Search page
2. Enter your query in the search box
3. Press Enter or click Search

**Via API**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/search?q=corruption"
```

**Via CLI**:
```bash
aleph search "corruption"
```

### Search with Filters

**UI**: Use the filter sidebar to select entity types, countries, etc.

**API**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/search?q=john+doe&filter:schema=Person&filter:countries=US"
```

### Quick Examples

| Task | Query |
|------|-------|
| Find person | `john doe filter:schema=Person` |
| Find company | `acme corp filter:schema=Company` |
| Find in US | `corruption filter:countries=US` |
| Recent entities | `* sort:created_at order:desc` |
| Specific collection | `* filter:collection_id=abc123` |

---

## Search API

### Endpoint

```
GET /api/2/search
```

**Authentication**: Required (API Key or Session Token)

**Authorization**: Results filtered by readable collections

### Request Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | string | Yes | Search query (use `*` for all) |
| `filter:schema` | string | No | Filter by entity schema (Person, Company, etc.) |
| `filter:countries` | string | No | Filter by country codes (comma-separated) |
| `filter:languages` | string | No | Filter by language codes |
| `filter:collection_id` | string | No | Filter by collection ID |
| `filter:properties.*` | string | No | Filter by property values |
| `limit` | integer | No | Results per page (1-1000, default: 30) |
| `offset` | integer | No | Pagination offset (default: 0) |
| `sort` | string | No | Sort field (`created_at`, `updated_at`, `caption`, `_score`) |
| `order` | string | No | Sort order (`asc`, `desc`) |
| `facet` | string | No | Enable facets (comma-separated) |

### Response Format

```json
{
  "results": [
    {
      "id": "entity-123",
      "schema": "Person",
      "caption": "John Doe",
      "properties": {
        "name": ["John Doe"],
        "birthDate": ["1980-01-15"],
        "nationality": ["US"],
        "email": ["john@example.com"]
      },
      "collection_id": "collection-123",
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-20T14:20:00Z",
      "score": 15.2
    }
  ],
  "total": 1500,
  "limit": 30,
  "offset": 0,
  "facets": {
    "schema": {
      "values": [
        {"id": "Person", "label": "Person", "count": 450},
        {"id": "Company", "label": "Company", "count": 200}
      ]
    },
    "countries": {
      "values": [
        {"id": "US", "label": "United States", "count": 300},
        {"id": "UK", "label": "United Kingdom", "count": 150}
      ]
    }
  },
  "next": "/api/2/search?q=corruption&limit=30&offset=30"
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| `results` | array | Array of matching entities |
| `total` | integer | Total number of matching results |
| `limit` | integer | Results per page |
| `offset` | integer | Current offset |
| `facets` | object | Facet aggregations (if requested) |
| `next` | string | URL for next page (if available) |

### Example Requests

**Simple search**:
```bash
curl "https://aleph.example.com/api/2/search?q=corruption" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Search with filters**:
```bash
curl "https://aleph.example.com/api/2/search?q=john+doe&filter:schema=Person&filter:countries=US" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Search with facets**:
```bash
curl "https://aleph.example.com/api/2/search?q=company&facet=schema,countries" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Paginated search**:
```bash
curl "https://aleph.example.com/api/2/search?q=corruption&limit=50&offset=100" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Sorted search**:
```bash
curl "https://aleph.example.com/api/2/search?q=*&sort=created_at&order=desc" \
  -H "Authorization: ApiKey YOUR_KEY"
```

---

## Query Syntax

### Basic Text Search

**Single term**:
```
corruption
```
Searches for "corruption" across all text fields.

**Multiple terms** (implicit AND):
```
money laundering
```
Finds entities containing both "money" AND "laundering".

**Phrase search**:
```
"panama papers"
```
Finds exact phrase "panama papers".

### Boolean Operators

**AND** (explicit):
```
corruption AND money_laundering
```
Both terms must be present.

**OR**:
```
"panama papers" OR "pandora papers"
```
Either term must be present.

**NOT** (exclusion):
```
corruption NOT bribery
```
Must contain "corruption" but not "bribery".

**Grouping with parentheses**:
```
(panama OR pandora) AND papers
```
Use parentheses to control operator precedence.

### Wildcards

**Asterisk** (`*`) - Zero or more characters:
```
john*
```
Matches: john, johnny, johnson, etc.

**Question mark** (`?`) - Single character:
```
wom?n
```
Matches: woman, women

**Note**: Wildcards at the beginning of terms (`*corruption`) are slow. Avoid if possible.

### Field-Specific Search

**Search specific property**:
```
properties.name:"John Doe"
```

**Search by email**:
```
properties.email:john@example.com
```

**Search by date**:
```
properties.birthDate:1980-01-15
```

### Special Characters

**Escape special characters** with backslash:
```
\(corruption\)
\[money laundering\]
```

Special characters: `+ - = && || > < ! ( ) { } [ ] ^ " ~ * ? : \ /`

---

## Filtering

### Schema Filter

Filter by entity type:

```bash
# Single schema
?filter:schema=Person

# Multiple schemas (comma-separated)
?filter:schema=Person,Company,Organization
```

**Available schemas** (50+ total):
- **People**: Person
- **Organizations**: Company, Organization, PublicBody
- **Documents**: Document, Page, Email
- **Financial**: Account, Payment, Transaction
- **Legal**: Contract, License, LegalEntity
- **Events**: Event, Meeting
- **Locations**: Address, RealEstate
- **Contacts**: Phone, Email
- **Relationships**: Family, Associate, Membership

**Example**:
```bash
curl "https://aleph.example.com/api/2/search?q=john&filter:schema=Person"
```

### Country Filter

Filter by country (ISO 3166-1 alpha-2 codes):

```bash
# Single country
?filter:countries=US

# Multiple countries
?filter:countries=US,UK,DE
```

**Example**:
```bash
curl "https://aleph.example.com/api/2/search?q=corruption&filter:countries=US"
```

### Language Filter

Filter by language (ISO 639-1 codes):

```bash
?filter:languages=eng,fra,deu
```

**Common codes**:
- `eng` - English
- `fra` - French
- `deu` - German
- `spa` - Spanish
- `rus` - Russian
- `ara` - Arabic

### Collection Filter

Filter to specific collection(s):

```bash
?filter:collection_id=collection-123
```

**Example**:
```bash
curl "https://aleph.example.com/api/2/search?q=*&filter:collection_id=panama-papers"
```

### Property Filters

Filter by specific property values:

```bash
# Nationality
?filter:properties.nationality=US

# Birth date
?filter:properties.birthDate=1980-01-15

# Registration number
?filter:properties.registrationNumber=123456789
```

**Example**:
```bash
curl "https://aleph.example.com/api/2/search?q=*&filter:properties.nationality=US&filter:schema=Person"
```

### Combining Filters

Combine multiple filters for precise results:

```bash
curl "https://aleph.example.com/api/2/search?q=john&filter:schema=Person&filter:countries=US&filter:properties.nationality=US"
```

---

## Faceted Search

### What are Facets?

Facets provide aggregated counts of results grouped by specific fields. They enable **drill-down search** by showing:
- How many results match each category
- Which filters are available
- The distribution of your results

### Available Facets

| Facet | Field | Description |
|-------|-------|-------------|
| `schema` | Entity type | Count by schema (Person, Company, etc.) |
| `countries` | Country codes | Count by country |
| `languages` | Language codes | Count by language |
| `collection_id` | Collections | Count by collection |
| `dates` | Dates | Date histogram by year |

### Enabling Facets

**Single facet**:
```bash
?facet=schema
```

**Multiple facets** (comma-separated):
```bash
?facet=schema,countries,languages
```

### Facet Response

```json
{
  "facets": {
    "schema": {
      "values": [
        {"id": "Person", "label": "Person", "count": 450},
        {"id": "Company", "label": "Company", "count": 200},
        {"id": "Organization", "label": "Organization", "count": 150}
      ]
    },
    "countries": {
      "values": [
        {"id": "US", "label": "United States", "count": 300},
        {"id": "UK", "label": "United Kingdom", "count": 150},
        {"id": "DE", "label": "Germany", "count": 100}
      ]
    }
  }
}
```

### Facet-Based Navigation

**Step 1: Initial search with facets**
```bash
curl "https://aleph.example.com/api/2/search?q=john&facet=schema,countries"
```

**Response shows**:
- 450 People
- 200 Companies
- 300 in US
- 150 in UK

**Step 2: User clicks "Person" facet**
```bash
curl "https://aleph.example.com/api/2/search?q=john&filter:schema=Person&facet=countries"
```

**Response shows**:
- All results are now People
- 250 in US
- 100 in UK

**Step 3: User clicks "US" facet**
```bash
curl "https://aleph.example.com/api/2/search?q=john&filter:schema=Person&filter:countries=US"
```

**Response shows**:
- Only US-based People named John

### Facet Configuration

**Facet size** (from settings.py):
```python
FACETS_SIZE_DEFAULT = 100  # Default number of facet values
FACETS_SIZE_MAX = 300      # Maximum facet values
```

**Date facet interval**:
- Automatically uses yearly intervals
- Groups results by year for temporal analysis

---

## Sorting & Pagination

### Sorting

**Available sort fields**:

| Field | Description |
|-------|-------------|
| `_score` | Relevance score (default) |
| `created_at` | Entity creation date |
| `updated_at` | Last modification date |
| `caption` | Entity display name (alphabetical) |

**Sort parameters**:
```bash
?sort=created_at&order=desc
```

**Examples**:

**Most relevant** (default):
```bash
?q=corruption
# Sorted by Elasticsearch relevance score
```

**Newest first**:
```bash
?q=*&sort=created_at&order=desc
```

**Oldest first**:
```bash
?q=*&sort=created_at&order=asc
```

**Alphabetical A-Z**:
```bash
?q=*&sort=caption&order=asc
```

**Recently updated**:
```bash
?q=*&sort=updated_at&order=desc
```

### Pagination

**Parameters**:
```bash
?limit=50&offset=100
```

| Parameter | Type | Range | Default |
|-----------|------|-------|---------|
| `limit` | integer | 1-1000 | 30 |
| `offset` | integer | 0-∞ | 0 |

**Page calculations**:
```
Page 1: offset=0,   limit=30
Page 2: offset=30,  limit=30
Page 3: offset=60,  limit=30
Page N: offset=(N-1)*limit, limit=30
```

**Example - Page 3 with 50 results per page**:
```bash
curl "https://aleph.example.com/api/2/search?q=corruption&limit=50&offset=100"
```

**Response includes pagination info**:
```json
{
  "results": [...],
  "total": 1500,
  "limit": 50,
  "offset": 100,
  "next": "/api/2/search?q=corruption&limit=50&offset=150"
}
```

**Deep pagination**:
- Elasticsearch limits: max 10,000 results (offset + limit ≤ 10,000)
- For deep pagination, use cursor-based pagination or scroll API
- Consider refining your query instead of paginating deeply

**Pagination best practices**:
1. Use reasonable page sizes (30-100)
2. Cache results on frontend when possible
3. Show total result count to user
4. Provide "next page" links
5. Use sort order for consistent pagination

---

## Advanced Features

### Cross-Reference Search

Find matching entities across different collections.

**Endpoint**: `GET /api/2/collections/:id/xref`

**Purpose**: Discover potential duplicates or related entities using machine learning.

**Example**:
```bash
curl "https://aleph.example.com/api/2/collections/collection-123/xref" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Response**:
```json
{
  "results": [
    {
      "entity": {
        "id": "entity-a-1",
        "schema": "Person",
        "properties": {"name": ["John Doe"]}
      },
      "match": {
        "id": "entity-b-1",
        "schema": "Person",
        "properties": {"name": ["Jon Doe"]}
      },
      "score": 0.85,
      "judgement": "POSITIVE"
    }
  ]
}
```

**Matching algorithm** (`aleph/logic/xref.py:1-400`):
- Uses FollowTheMoney-Compare library
- Generalized Linear Model (Bernoulli classification)
- Features: name similarity, date proximity, shared properties
- Configurable threshold (default: 0.5)

**Configuration**:
```bash
ALEPH_XREF_THRESHOLD=0.5  # Minimum score for match
```

### Search Alerts

Save searches and receive notifications when new matches appear.

**Create alert**:
```bash
POST /api/2/alerts
{
  "query_text": "money laundering",
  "collection_id": "collection-123"
}
```

**List alerts**:
```bash
GET /api/2/alerts
```

**How it works**:
1. User creates alert with search query
2. System periodically re-runs the query
3. New results trigger notifications
4. User receives email or in-app notification

**Alert frequency** (configured per deployment):
- Default: Daily
- Configurable: Hourly, daily, weekly

### Entity Expansion

Find entities related to a specific entity.

**Endpoint**: `GET /api/2/entities/:id/expand`

**Returns**:
- Entities with direct relationships
- Upstream/downstream connections
- Shared properties (same address, phone, etc.)

**Example**:
```bash
curl "https://aleph.example.com/api/2/entities/entity-123/expand" \
  -H "Authorization: ApiKey YOUR_KEY"
```

**Configuration**:
```bash
ALEPH_MAX_EXPAND_ENTITIES=200  # Max entities to expand
```

### Search History

Track recent searches (frontend feature).

**Implementation**:
- Stored in browser localStorage
- Not synced across devices
- Cleared on logout

**Usage**:
- UI shows recent searches
- Click to re-run search
- Clear history from settings

---

## Frontend Search

### Search Components

**Location**: `ui/src/components/`

#### SearchField

Basic search input component.

**Props**:
```typescript
interface SearchFieldProps {
  query: string;
  onChange: (query: string) => void;
  onSubmit: () => void;
  placeholder?: string;
}
```

**Usage**:
```tsx
<SearchField
  query={this.state.query}
  onChange={(q) => this.setState({ query: q })}
  onSubmit={this.handleSearch}
/>
```

#### AdvancedSearch

Query builder for complex searches.

**Features**:
- Schema selector
- Country selector
- Date range picker
- Property filters
- Visual query building

**Example**:
```tsx
<AdvancedSearch
  filters={this.state.filters}
  onFiltersChange={this.handleFiltersChange}
/>
```

#### EntityTable

Paginated entity list with sorting.

**Features**:
- Column sorting
- Pagination controls
- Selection checkboxes
- Bulk actions

#### FacetList

Displays facets with click handlers.

**Features**:
- Shows facet counts
- Click to filter
- Shows selected filters
- Clear all filters

### Search Screen

**Route**: `/search`

**Component**: `ui/src/screens/SearchScreen/`

**Features**:
- Search input
- Filter sidebar
- Result list
- Pagination
- Facet navigation

### Redux State

**Search state structure**:
```javascript
{
  entities: {
    results: {
      'search-query-hash': {
        results: [...],
        total: 1500,
        isLoading: false,
        error: null
      }
    },
    view: {
      query: "corruption",
      filters: {
        schema: ["Person"],
        countries: ["US"]
      },
      sort: "created_at",
      order: "desc",
      offset: 0
    }
  }
}
```

**Actions** (`ui/src/actions/`):
- `searchEntities(query, filters)` - Execute search
- `updateSearchQuery(query)` - Update query string
- `addFilter(field, value)` - Add filter
- `removeFilter(field, value)` - Remove filter
- `clearFilters()` - Clear all filters
- `setSort(field, order)` - Change sort order
- `setPage(offset)` - Change page

### Query Building

**Frontend query construction**:
```javascript
// Build query string
const params = new URLSearchParams();
params.append('q', query);

// Add filters
Object.entries(filters).forEach(([key, values]) => {
  if (values.length > 0) {
    params.append(`filter:${key}`, values.join(','));
  }
});

// Add sorting
if (sort) {
  params.append('sort', sort);
  params.append('order', order);
}

// Add pagination
params.append('limit', limit);
params.append('offset', offset);

// Make request
const url = `/api/2/search?${params.toString()}`;
axios.get(url);
```

---

## Authorization & Permissions

### Permission-Based Filtering

**All search results are filtered by user permissions.**

**Permission model**:
```
User (Role)
  └── Member of Groups (Roles)
      └── Permissions on Collections
          ├── READ (can search)
          └── WRITE (can modify)
```

**Authorization filter** (`aleph/search/query.py:367-381`):
```python
def authz_filter(self):
    """Filter by user permissions"""
    if self.authz.is_admin:
        return {'match_all': {}}

    # Restrict to readable collections
    return {
        'terms': {
            'collection_id': list(self.authz.collections(Authz.READ))
        }
    }
```

**How it works**:
1. Query includes user's readable collection IDs
2. Elasticsearch filters at query time (not post-processing)
3. Users only see results from collections they can access
4. Admins see all results

**Cache optimization**:
- Permissions cached in Redis for 1 hour
- Avoids per-request database queries
- Invalidated on permission changes

### Anonymous Search

**Configuration**:
```bash
ALEPH_REQUIRE_LOGGED_IN=false  # Allow anonymous access
```

**Behavior**:
- Anonymous users see public collections only
- No personalized features (alerts, bookmarks, etc.)
- Rate limited more strictly

**Rate limiting**:
```bash
ALEPH_API_RATE_LIMIT=30        # Requests per window
ALEPH_API_RATE_WINDOW=15       # Window in minutes
```

---

## Performance & Optimization

### Query Performance

**Bottlenecks**:
1. **Complex queries** - Many OR clauses, wildcards
2. **Large result sets** - Deep pagination
3. **Heavy faceting** - Many facets on large datasets
4. **Authorization checks** - Many collections

**Optimizations**:

**1. Use specific filters**:
```bash
# Slow - searches everything
?q=*

# Fast - narrows search space
?q=*&filter:schema=Person&filter:countries=US
```

**2. Limit facets**:
```bash
# Only request facets you need
?facet=schema  # Not: facet=schema,countries,languages,dates
```

**3. Reasonable page sizes**:
```bash
# Good
?limit=30

# Avoid
?limit=1000  # Slow, high memory usage
```

**4. Use sorting strategically**:
```bash
# Fast - relevance (default)
?q=corruption

# Slower - field sorting
?q=corruption&sort=created_at
```

### Index Optimization

**Index settings** (`aleph/index/indexes.py:1-131`):

```python
# Sharding strategy
HEAVY_SCHEMAS = ['Page', 'Table']  # 10 shards
LIGHT_SCHEMAS = ['Person', 'Company']  # 1 shard

# Replica configuration
ALEPH_INDEX_REPLICAS=1  # Production: 2
```

**Reindexing strategy**:
1. Create new index version
2. Index to new version (old still readable)
3. Switch write index when complete
4. Remove old version

**Zero downtime** during reindexing.

### Caching

**Authorization cache**:
```python
# Redis cache for 1 hour
cache_key = f'authz:{user_id}:collections'
cache.set(cache_key, collection_ids, timeout=3600)
```

**Search result cache** (optional):
```python
# Cache frequently accessed queries
cache_key = f'search:{query_hash}'
cache.set(cache_key, results, timeout=300)  # 5 minutes
```

### Batch Operations

**Indexing** (`aleph/worker.py:619-640`):
```python
# Collect entities for 10 seconds
batch = []
timeout = time.time() + 10

while time.time() < timeout:
    entity = queue.get()
    batch.append(entity)

# Bulk index (single ES request)
bulk_index(batch)  # 10-100x faster than individual requests
```

**Configuration**:
```bash
ALEPH_INDEXING_BATCH_SIZE=100   # Entities per batch
ALEPH_INDEXING_TIMEOUT=10       # Seconds to collect
```

### Scaling

**Horizontal scaling**:
```bash
# Scale API servers
docker-compose up -d --scale api=5

# Scale Elasticsearch
# Add more ES nodes to cluster

# Scale workers
docker-compose up -d --scale worker=10
```

**Elasticsearch scaling**:
```yaml
# Kubernetes
elasticsearch:
  replicas: 3
  resources:
    requests:
      memory: "8Gi"
      cpu: "2000m"
```

---

## Troubleshooting

### Common Issues

#### No Results Found

**Symptoms**: Search returns 0 results

**Causes**:
1. Query syntax error
2. No matching entities
3. Permission restrictions
4. Collection not indexed

**Solutions**:

**Check permissions**:
```bash
# Verify user has READ access to collections
curl "/api/2/collections" -H "Authorization: ApiKey YOUR_KEY"
```

**Try wildcard search**:
```bash
# Search for everything
curl "/api/2/search?q=*"
```

**Check indexing status**:
```bash
# Check if collection is indexed
curl "http://localhost:9200/aleph-entity-*/_count"
```

#### Slow Queries

**Symptoms**: Queries take >5 seconds

**Causes**:
1. Wildcard at start of term (`*corruption`)
2. Too many OR clauses
3. Deep pagination (offset > 1000)
4. Heavy faceting

**Solutions**:

**Use specific filters**:
```bash
# Add filters to narrow search
?q=corruption&filter:schema=Person&filter:countries=US
```

**Avoid leading wildcards**:
```bash
# Slow
?q=*tion

# Fast
?q=corruption*
```

**Use reasonable pagination**:
```bash
# Good
?offset=100

# Slow
?offset=10000
```

#### Relevance Issues

**Symptoms**: Irrelevant results appear first

**Causes**:
1. Common terms dominate score
2. Missing filters
3. Boost not applied

**Solutions**:

**Use phrase matching**:
```bash
# Phrase search for exact match
?q="panama papers"
```

**Add filters**:
```bash
# Narrow by type
?q=corruption&filter:schema=Document
```

**Use sorting**:
```bash
# Sort by date instead of relevance
?q=*&sort=created_at&order=desc
```

#### Authorization Errors

**Symptoms**: 403 Forbidden or missing results

**Causes**:
1. User lacks READ permission
2. Collection is restricted
3. Session expired

**Solutions**:

**Check session**:
```bash
curl "/api/2/sessions" -H "Authorization: ApiKey YOUR_KEY"
```

**Verify permissions**:
```bash
curl "/api/2/collections/:id" -H "Authorization: ApiKey YOUR_KEY"
# Check "permission" field
```

**Request access**:
- Contact collection owner
- Ask for READ permission

#### Facet Issues

**Symptoms**: Facets missing or incorrect counts

**Causes**:
1. Facet not requested
2. No matching results in facet
3. Facet size limit reached

**Solutions**:

**Request facets explicitly**:
```bash
?facet=schema,countries
```

**Check facet response**:
```json
{
  "facets": {
    "schema": {
      "values": [...]  // Check if empty
    }
  }
}
```

**Increase facet size**:
```bash
# Server configuration
FACETS_SIZE_MAX=500
```

### Elasticsearch Issues

#### Index Not Found

**Error**: `IndexNotFoundException`

**Solution**:
```bash
# Reindex collections
aleph reindex

# Or reindex specific collection
aleph reindex collection-123
```

#### Mapping Conflict

**Error**: `MapperParsingException`

**Solution**:
```bash
# Reset index
aleph resetindex

# Re-upload data
aleph crawldir /path/to/documents
```

#### Out of Memory

**Error**: `ES_JAVA_OPTS` heap size exceeded

**Solution**:
```bash
# Increase heap (docker-compose.yml)
environment:
  ES_JAVA_OPTS: "-Xms8g -Xmx8g"
```

### Debugging Tools

**Check ES cluster health**:
```bash
curl "http://localhost:9200/_cluster/health?pretty"
```

**View indices**:
```bash
curl "http://localhost:9200/_cat/indices?v"
```

**Check index mapping**:
```bash
curl "http://localhost:9200/aleph-entity-person-v1/_mapping?pretty"
```

**Search ES directly**:
```bash
curl "http://localhost:9200/aleph-entity-*/_search?pretty" -d '{
  "query": {"match": {"text": "corruption"}}
}'
```

**Check logs**:
```bash
# API logs
docker-compose logs -f api

# Elasticsearch logs
docker-compose logs -f elasticsearch
```

---

## Code Reference

### Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `aleph/search/query.py` | 1-300 | Base query builder, authorization |
| `aleph/search/parser.py` | - | Query syntax parsing |
| `aleph/search/facet.py` | - | Facet aggregations |
| `aleph/search/result.py` | - | Result serialization |
| `aleph/index/indexes.py` | 1-131 | Index mappings, schema |
| `aleph/index/entities.py` | - | Entity indexing logic |
| `aleph/views/entities_api.py` | 400+ | Search API endpoint |
| `aleph/authz.py` | 150-170 | Permission checking |
| `ui/src/components/SearchField/` | - | Search input component |
| `ui/src/screens/SearchScreen/` | - | Search page |

### Configuration

**Environment variables**:
```bash
# Elasticsearch
ALEPH_ELASTICSEARCH_URI=http://localhost:9200
ELASTICSEARCH_TIMEOUT=60

# Search defaults
SEARCH_LIMIT_DEFAULT=30
SEARCH_LIMIT_MAX=1000

# Facets
FACETS_SIZE_DEFAULT=100
FACETS_SIZE_MAX=300

# Indexing
ALEPH_INDEXING_BATCH_SIZE=100
ALEPH_INDEXING_TIMEOUT=10

# Authorization
AUTHZ_CACHE_TIMEOUT=3600  # 1 hour
```

---

**Last Updated:** 2025-12-08
**Version:** 4.1.7
**Completeness:** 100%
