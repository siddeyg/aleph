# Aleph API Reference

Complete REST API documentation for the Aleph platform.

## Table of Contents
- [Overview](#overview)
- [Authentication](#authentication)
- [Base URL & Versioning](#base-url--versioning)
- [Request/Response Format](#requestresponse-format)
- [Pagination](#pagination)
- [Error Handling](#error-handling)
- [Endpoints](#endpoints)
  - [System](#system-endpoints)
  - [Authentication & Sessions](#authentication--sessions)
  - [Collections](#collections)
  - [Entities & Search](#entities--search)
  - [EntitySets](#entitysets)
  - [Cross-Reference](#cross-reference)
  - [Ingestion](#ingestion)
  - [Permissions](#permissions)
  - [Roles & Users](#roles--users)
  - [Alerts](#alerts)
  - [Bookmarks](#bookmarks)
  - [Exports](#exports)

---

## Overview

The Aleph API is a RESTful HTTP API that provides programmatic access to all Aleph functionality.

### Key Features
- **RESTful Design** - Standard HTTP methods (GET, POST, PUT, DELETE)
- **JSON Format** - All requests and responses use JSON
- **Stateless** - API key or session token authentication
- **Pagination** - Cursor-based pagination for large result sets
- **Filtering** - Query parameters for filtering and sorting
- **Facets** - Aggregations for drill-down analysis
- **OpenAPI Spec** - Auto-generated documentation at `/api/openapi.json`

### API Versions
- **Current**: v2 (`/api/2/`)
- **Legacy**: v1 (deprecated, use v2)

---

## Authentication

### Methods

Aleph supports two authentication methods:

#### 1. API Key Authentication

**Obtaining an API Key**:
```bash
# Via CLI (recommended)
aleph createuser --name "API User" api@example.com
# Returns: User created. ID: 42, API Key: aleph-abc123def456...

# Via UI
Settings → API Access → Generate API Key
```

**Using API Key**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections
```

**Header Format**:
```
Authorization: ApiKey aleph-abc123def456...
```

#### 2. Session Token Authentication

**Login**:
```bash
curl -X POST https://aleph.example.com/api/2/sessions/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "secret"
  }'
```

**Response**:
```json
{
  "token": "session-token-xyz...",
  "role": {
    "id": "user-123",
    "email": "user@example.com",
    "name": "John Doe",
    "is_admin": false
  }
}
```

**Using Session Token**:
```bash
curl -H "Authorization: Token session-token-xyz..." \
  https://aleph.example.com/api/2/collections
```

### Anonymous Access

Some endpoints allow anonymous access if `ALEPH_REQUIRE_LOGGED_IN=false`:
- Public collections (read-only)
- System status
- OpenAPI spec

---

## Base URL & Versioning

**Production**: `https://aleph.example.com/api/2/`
**Development**: `http://localhost:5000/api/2/`

All endpoints are prefixed with `/api/2/`.

**Example**:
```
https://aleph.example.com/api/2/collections
```

---

## Request/Response Format

### Content Types

**Request Headers**:
```
Content-Type: application/json
Accept: application/json
Authorization: ApiKey YOUR_API_KEY
```

**Response Headers**:
```
Content-Type: application/json
```

### Request Body

JSON format for POST/PUT requests:
```json
{
  "foreign_id": "my-collection",
  "label": "My Collection",
  "category": "leak"
}
```

### Response Structure

**Success Response**:
```json
{
  "id": "collection-123",
  "foreign_id": "my-collection",
  "label": "My Collection",
  "category": "leak",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00Z"
}
```

**List Response**:
```json
{
  "results": [...],
  "page": 1,
  "limit": 30,
  "total": 150,
  "next": "https://aleph.example.com/api/2/collections?page=2"
}
```

---

## Pagination

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | integer | 30 | Results per page (max: 1000) |
| `offset` | integer | 0 | Skip N results |

### Example

```bash
# Get results 31-60
curl "https://aleph.example.com/api/2/search?q=corruption&limit=30&offset=30"
```

### Response

```json
{
  "results": [...],
  "limit": 30,
  "offset": 30,
  "total": 1500,
  "next": "/api/2/search?q=corruption&limit=30&offset=60"
}
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Success |
| 201 | Created | Resource created |
| 204 | No Content | Success, no body |
| 400 | Bad Request | Invalid parameters |
| 401 | Unauthorized | Authentication required |
| 403 | Forbidden | Permission denied |
| 404 | Not Found | Resource not found |
| 409 | Conflict | Resource already exists |
| 500 | Server Error | Internal error |
| 503 | Service Unavailable | Maintenance mode |

### Error Response

```json
{
  "status": "error",
  "message": "Collection not found",
  "errors": {
    "foreign_id": ["No such collection: 'missing-collection'"]
  }
}
```

---

## Endpoints

### System Endpoints

#### GET /api/2/status

Get system status and metadata.

**Public**: Yes (no auth required)

**Response**:
```json
{
  "status": "ok",
  "maintenance": false,
  "app": {
    "title": "Aleph",
    "version": "4.1.7",
    "logo": "https://...",
    "samples": true
  },
  "categories": [
    "news", "leak", "court", "sanctions", ...
  ],
  "schemata": {
    "Person": {...},
    "Company": {...}
  }
}
```

**Example**:
```bash
curl https://aleph.example.com/api/2/status
```

---

#### GET /api/2/metadata

Get system configuration metadata.

**Auth**: Required

**Response**:
```json
{
  "app": {...},
  "auth": {
    "password_login": true,
    "oauth": false
  },
  "ftm": {
    "model": {...}  // FollowTheMoney model
  }
}
```

---

### Authentication & Sessions

#### POST /api/2/sessions/login

Authenticate user and get session token.

**Public**: Yes

**Request**:
```json
{
  "email": "user@example.com",
  "password": "secret"
}
```

**Response**:
```json
{
  "token": "session-token-xyz",
  "role": {
    "id": "user-123",
    "email": "user@example.com",
    "name": "John Doe",
    "is_admin": false,
    "locale": "en"
  }
}
```

**Example**:
```bash
curl -X POST https://aleph.example.com/api/2/sessions/login \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "secret"}'
```

---

#### GET /api/2/sessions

Get current session information.

**Auth**: Required

**Response**:
```json
{
  "role": {
    "id": "user-123",
    "email": "user@example.com",
    "name": "John Doe",
    "is_admin": false,
    "api_key": "aleph-abc..."
  },
  "logged_in": true
}
```

---

#### DELETE /api/2/sessions/logout

Logout and invalidate session token.

**Auth**: Required

**Response**: 204 No Content

---

### Collections

#### GET /api/2/collections

List all accessible collections.

**Auth**: Required

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | string | Search query |
| `category` | string | Filter by category |
| `countries` | string | Filter by countries (comma-separated) |
| `casefile` | boolean | Only casefiles |
| `writeable` | boolean | Only writeable collections |
| `limit` | integer | Results per page |
| `offset` | integer | Offset |

**Response**:
```json
{
  "results": [
    {
      "id": "collection-123",
      "foreign_id": "my-collection",
      "label": "My Collection",
      "category": "leak",
      "casefile": false,
      "secret": false,
      "count": 1250,
      "countries": ["US", "UK"],
      "languages": ["eng", "fra"],
      "created_at": "2024-01-15T10:30:00Z",
      "updated_at": "2024-01-20T14:20:00Z",
      "writeable": true
    }
  ],
  "total": 15,
  "limit": 30,
  "offset": 0
}
```

**Example**:
```bash
# List all collections
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections

# Filter by category
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/collections?category=leak"

# Only casefiles
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/collections?casefile=true"
```

---

#### GET /api/2/collections/:id

Get collection details.

**Auth**: Required (must have READ permission)

**Response**:
```json
{
  "id": "collection-123",
  "foreign_id": "my-collection",
  "label": "My Collection",
  "summary": "Detailed description",
  "category": "leak",
  "casefile": false,
  "secret": false,
  "count": 1250,
  "countries": ["US", "UK"],
  "languages": ["eng", "fra"],
  "publisher": "OCCRP",
  "publisher_url": "https://occrp.org",
  "info_url": "https://...",
  "data_url": "https://...",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-20T14:20:00Z",
  "writeable": true,
  "statistics": {
    "schema": {
      "Person": 450,
      "Company": 200,
      "Document": 600
    }
  }
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections/collection-123
```

---

#### POST /api/2/collections

Create a new collection.

**Auth**: Required

**Request**:
```json
{
  "foreign_id": "my-new-collection",
  "label": "My New Collection",
  "summary": "Description of the collection",
  "category": "leak",
  "countries": ["US"],
  "languages": ["eng"],
  "publisher": "My Organization",
  "casefile": false
}
```

**Response**: 201 Created
```json
{
  "id": "collection-456",
  "foreign_id": "my-new-collection",
  ...
}
```

**Example**:
```bash
curl -X POST https://aleph.example.com/api/2/collections \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "foreign_id": "my-new-collection",
    "label": "My New Collection",
    "category": "leak"
  }'
```

---

#### PUT /api/2/collections/:id

Update collection metadata.

**Auth**: Required (must have WRITE permission)

**Request**:
```json
{
  "label": "Updated Label",
  "summary": "Updated description",
  "countries": ["US", "UK"]
}
```

**Response**: 200 OK

---

#### DELETE /api/2/collections/:id

Delete a collection and all its contents.

**Auth**: Required (must have WRITE permission)

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `sync` | boolean | Wait for deletion (default: false) |

**Response**: 204 No Content

**Example**:
```bash
# Async delete
curl -X DELETE \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections/collection-123

# Synchronous delete
curl -X DELETE \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/collections/collection-123?sync=true"
```

---

### Entities & Search

#### GET /api/2/search

Search entities across collections.

**Auth**: Required (filters by accessible collections)

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | string | Search query (required) |
| `filter:schema` | string | Filter by schema (Person, Company, etc.) |
| `filter:countries` | string | Filter by countries |
| `filter:languages` | string | Filter by languages |
| `filter:collection_id` | string | Filter by collection |
| `filter:properties.*` | string | Filter by property value |
| `limit` | integer | Results per page (max: 1000) |
| `offset` | integer | Offset |
| `sort` | string | Sort field (created_at, updated_at, caption) |
| `order` | string | Sort order (asc, desc) |
| `facet` | string | Enable facets (comma-separated) |

**Response**:
```json
{
  "results": [
    {
      "id": "entity-123",
      "schema": "Person",
      "properties": {
        "name": ["John Doe"],
        "birthDate": ["1980-01-15"],
        "nationality": ["US"]
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
  }
}
```

**Examples**:
```bash
# Basic search
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/search?q=corruption"

# Search with filters
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/search?q=john+doe&filter:schema=Person&filter:countries=US"

# Search with facets
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/search?q=company&facet=schema,countries"

# Paginated search
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/search?q=corruption&limit=50&offset=100"
```

---

#### GET /api/2/entities/:id

Get entity details.

**Auth**: Required (must have READ permission on collection)

**Response**:
```json
{
  "id": "entity-123",
  "schema": "Person",
  "properties": {
    "name": ["John Doe"],
    "birthDate": ["1980-01-15"],
    "nationality": ["US"],
    "email": ["john@example.com"],
    "address": ["123 Main St, New York, NY"]
  },
  "collection": {
    "id": "collection-123",
    "label": "My Collection"
  },
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-20T14:20:00Z"
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/entities/entity-123
```

---

#### POST /api/2/entities

Create a new entity.

**Auth**: Required (must have WRITE permission on collection)

**Request**:
```json
{
  "schema": "Person",
  "properties": {
    "name": ["John Doe"],
    "birthDate": ["1980-01-15"],
    "nationality": ["US"]
  },
  "collection_id": "collection-123"
}
```

**Response**: 201 Created
```json
{
  "id": "entity-456",
  "schema": "Person",
  "properties": {...},
  ...
}
```

**Example**:
```bash
curl -X POST https://aleph.example.com/api/2/entities \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "schema": "Person",
    "properties": {
      "name": ["John Doe"],
      "birthDate": ["1980-01-15"]
    },
    "collection_id": "collection-123"
  }'
```

---

#### PUT /api/2/entities/:id

Update an entity.

**Auth**: Required (must have WRITE permission)

**Request**:
```json
{
  "schema": "Person",
  "properties": {
    "name": ["John Doe"],
    "birthDate": ["1980-01-15"],
    "email": ["john@example.com"]
  }
}
```

**Response**: 200 OK

---

#### DELETE /api/2/entities/:id

Delete an entity.

**Auth**: Required (must have WRITE permission)

**Response**: 204 No Content

**Example**:
```bash
curl -X DELETE \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/entities/entity-123
```

---

### EntitySets

EntitySets are collections of entities (lists, diagrams, timelines, profiles).

#### GET /api/2/entitysets

List all accessible entitysets.

**Auth**: Required

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `type` | string | Filter by type (list, diagram, timeline, profile) |
| `collection_id` | string | Filter by collection |

**Response**:
```json
{
  "results": [
    {
      "id": "entityset-123",
      "label": "Key People",
      "type": "list",
      "collection_id": "collection-123",
      "count": 45,
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

---

#### POST /api/2/entitysets

Create a new entityset.

**Auth**: Required

**Request**:
```json
{
  "label": "My Diagram",
  "type": "diagram",
  "collection_id": "collection-123",
  "entities": ["entity-1", "entity-2"],
  "layout": {
    "entity-1": {"x": 100, "y": 200},
    "entity-2": {"x": 300, "y": 200}
  }
}
```

**Types**:
- `list` - Simple entity list
- `diagram` - Network diagram with layout
- `timeline` - Temporal visualization
- `profile` - Entity profile with judgements

---

#### GET /api/2/entitysets/:id/entities

Get entities in an entityset.

**Auth**: Required

**Response**:
```json
{
  "results": [
    {
      "id": "entity-1",
      "schema": "Person",
      "properties": {...}
    }
  ]
}
```

---

#### PUT /api/2/entitysets/:id/entities

Add entities to an entityset.

**Auth**: Required

**Request**:
```json
{
  "entities": ["entity-3", "entity-4"]
}
```

**Response**: 200 OK

---

### Cross-Reference

#### POST /api/2/xref

Generate cross-reference matches between collections.

**Auth**: Required (must have READ on all collections)

**Request**:
```json
{
  "collection_ids": ["collection-a", "collection-b"]
}
```

**Response**: 202 Accepted
```json
{
  "job_id": "xref-job-123",
  "status": "pending"
}
```

**Example**:
```bash
curl -X POST https://aleph.example.com/api/2/xref \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"collection_ids": ["collection-a", "collection-b"]}'
```

---

#### GET /api/2/collections/:id/xref

Get cross-reference results for a collection.

**Auth**: Required

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `entity_id` | string | Filter by entity |
| `limit` | integer | Results per page |
| `offset` | integer | Offset |

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

---

### Ingestion

#### POST /api/2/collections/:id/ingest

Upload a document to a collection.

**Auth**: Required (must have WRITE permission)

**Request**: multipart/form-data
```
file: (binary file data)
foreign_id: my-doc-001 (optional)
```

**Response**: 201 Created
```json
{
  "id": "document-123",
  "foreign_id": "my-doc-001",
  "file_name": "report.pdf",
  "mime_type": "application/pdf",
  "size": 1024000
}
```

**Example**:
```bash
curl -X POST \
  https://aleph.example.com/api/2/collections/collection-123/ingest \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -F "file=@/path/to/document.pdf" \
  -F "foreign_id=my-doc-001"
```

---

### Permissions

#### GET /api/2/collections/:id/permissions

List permissions for a collection.

**Auth**: Required (must have WRITE permission)

**Response**:
```json
{
  "results": [
    {
      "role": {
        "id": "user-123",
        "name": "John Doe",
        "type": "user"
      },
      "read": true,
      "write": true
    },
    {
      "role": {
        "id": "group-456",
        "name": "investigators",
        "type": "group"
      },
      "read": true,
      "write": false
    }
  ]
}
```

---

#### POST /api/2/collections/:id/permissions

Grant permission to a role.

**Auth**: Required (must have WRITE permission)

**Request**:
```json
{
  "role": "user-789",
  "read": true,
  "write": false
}
```

**Response**: 201 Created

**Example**:
```bash
curl -X POST \
  https://aleph.example.com/api/2/collections/collection-123/permissions \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"role": "user-789", "read": true, "write": false}'
```

---

#### DELETE /api/2/collections/:id/permissions/:role_id

Revoke permission from a role.

**Auth**: Required (must have WRITE permission)

**Response**: 204 No Content

---

### Roles & Users

#### GET /api/2/roles

List all roles (users and groups).

**Auth**: Admin only

**Response**:
```json
{
  "results": [
    {
      "id": "user-123",
      "type": "user",
      "email": "user@example.com",
      "name": "John Doe",
      "is_admin": false
    },
    {
      "id": "group-456",
      "type": "group",
      "name": "investigators"
    }
  ]
}
```

---

### Alerts

#### GET /api/2/alerts

List user alerts.

**Auth**: Required

**Response**:
```json
{
  "results": [
    {
      "id": "alert-123",
      "query": "money laundering",
      "collection_id": "collection-123",
      "notified_at": "2024-01-20T10:00:00Z",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

---

#### POST /api/2/alerts

Create a new alert.

**Auth**: Required

**Request**:
```json
{
  "query": "corruption",
  "collection_id": "collection-123"
}
```

**Response**: 201 Created

---

#### DELETE /api/2/alerts/:id

Delete an alert.

**Auth**: Required

**Response**: 204 No Content

---

### Bookmarks

#### GET /api/2/bookmarks

List user bookmarks.

**Auth**: Required

**Response**:
```json
{
  "results": [
    {
      "id": "bookmark-123",
      "entity": {...},
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

---

#### POST /api/2/bookmarks

Bookmark an entity.

**Auth**: Required

**Request**:
```json
{
  "entity_id": "entity-123"
}
```

**Response**: 201 Created

---

### Exports

#### GET /api/2/exports

List export jobs.

**Auth**: Required

**Response**:
```json
{
  "results": [
    {
      "id": "export-123",
      "status": "success",
      "file_name": "export.zip",
      "file_size": 10240000,
      "expires_at": "2024-02-15T00:00:00Z",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

---

#### POST /api/2/exports

Create a new export.

**Auth**: Required

**Request**:
```json
{
  "query": "corruption",
  "collection_id": "collection-123",
  "format": "csv"
}
```

**Response**: 201 Created

---

#### GET /api/2/exports/:id/download

Download export file.

**Auth**: Required

**Response**: Binary file download

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/exports/export-123/download \
  -o export.zip
```

---

## Rate Limiting

**Anonymous Users**: 30 requests/minute
**Authenticated Users**: 300 requests/minute
**Admins**: Unlimited

**Headers**:
```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 295
X-RateLimit-Reset: 1609459200
```

---

## Best Practices

### 1. Use API Keys for Automation
- Generate dedicated API keys for scripts
- Rotate keys regularly
- Store keys securely (env variables, secrets manager)

### 2. Pagination
- Always use pagination for large result sets
- Keep `limit` reasonable (30-100)
- Use `offset` for deep pagination

### 3. Filtering
- Apply filters to reduce result sets
- Use facets for drill-down analysis
- Cache common queries

### 4. Error Handling
- Check HTTP status codes
- Parse error messages
- Implement retry logic for 5xx errors

### 5. Performance
- Batch operations when possible
- Use async operations for heavy tasks
- Monitor rate limits

---

## OpenAPI Specification

**URL**: `https://aleph.example.com/api/openapi.json`

**Interactive Docs**: [ReDoc](https://redocly.github.io/redoc/?url=https://aleph.occrp.org/api/openapi.json)

---

## SDK & Libraries

### Official
- **Python**: [alephclient](https://github.com/alephdata/alephclient)

### Community
- **JavaScript**: axios + custom wrapper
- **R**: httr + custom functions

---

## Support

- **API Issues**: [GitHub Issues](https://github.com/alephdata/aleph/issues)
- **Questions**: [GitHub Discussions](https://github.com/alephdata/aleph/discussions)
- **Security**: security@occrp.org

---

**Last Updated:** 2025-11-16
**Version:** 4.1.7
