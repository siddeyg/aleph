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

---

## Roles & Users

### GET /api/2/roles/_suggest

Suggest users matching a search prefix (autocomplete).

**Auth**: Required (logged in)

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `prefix` | string | Email prefix (min 6 chars, required) |
| `exclude:id` | string | Role IDs to exclude |

**Response**:
```json
{
  "total": 5,
  "results": [
    {
      "id": "42",
      "type": "user",
      "email": "john@example.com",
      "name": "John Doe"
    }
  ]
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/roles/_suggest?prefix=john@ex"
```

---

### POST /api/2/roles/code

Begin account registration by sending a verification code to an email.

**Auth**: Public (registration must be enabled)

**Request**:
```json
{
  "email": "newuser@example.com"
}
```

**Response**: 200 OK
```json
{
  "status": "ok",
  "message": "To proceed, please check your email."
}
```

**Example**:
```bash
curl -X POST https://aleph.example.com/api/2/roles/code \
  -H "Content-Type: application/json" \
  -d '{"email": "newuser@example.com"}'
```

---

### POST /api/2/roles

Create a new user account after email verification.

**Auth**: Public (registration must be enabled)

**Request**:
```json
{
  "code": "verification-code-from-email",
  "name": "John Doe",
  "password": "secure-password"
}
```

**Response**: 201 Created
```json
{
  "id": "123",
  "email": "newuser@example.com",
  "name": "John Doe",
  "is_admin": false,
  "type": "user"
}
```

**Example**:
```bash
curl -X POST https://aleph.example.com/api/2/roles \
  -H "Content-Type: application/json" \
  -d '{
    "code": "abc123...",
    "name": "John Doe",
    "password": "mypassword"
  }'
```

---

### GET /api/2/roles/:id

Retrieve role details (user or group).

**Auth**: Required (must be able to read role)

**Response**:
```json
{
  "id": "42",
  "type": "user",
  "email": "user@example.com",
  "name": "User Name",
  "is_admin": false,
  "created_at": "2023-01-15T10:00:00Z",
  "has_password": true,
  "has_api_key": true,
  "locale": "en"
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/roles/42
```

---

### PUT /api/2/roles/:id

Update role settings (name, password, locale).

**Auth**: Required (must be able to write role - usually own role)

**Request**:
```json
{
  "name": "Updated Name",
  "password": "new-password",
  "current_password": "old-password",
  "locale": "de"
}
```

**Response**: 200 OK

**Example**:
```bash
curl -X PUT https://aleph.example.com/api/2/roles/42 \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "new-secure-password",
    "current_password": "old-password"
  }'
```

---

### POST /api/2/roles/:id/generate_api_key

Generate a new API key for a role (invalidates old key).

**Auth**: Required (must be able to write role)

**Response**: 200 OK
```json
{
  "id": "42",
  "email": "user@example.com",
  "api_key": "aleph-abc123def456...",
  "api_key_expires_at": "2024-03-15T10:00:00Z"
}
```

**Important**: The `api_key` field is only returned once during generation. Store it securely.

**Example**:
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_OLD_API_KEY" \
  https://aleph.example.com/api/2/roles/42/generate_api_key
```

---

## Groups

### GET /api/2/groups

List all groups the authenticated user belongs to.

**Auth**: Required (logged in)

**Response**:
```json
{
  "total": 3,
  "results": [
    {
      "id": "100",
      "type": "group",
      "name": "Editors",
      "created_at": "2023-01-01T00:00:00Z"
    },
    {
      "id": "101",
      "type": "group",
      "name": "Analysts",
      "created_at": "2023-02-01T00:00:00Z"
    }
  ]
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/groups
```

---

## Permissions

### GET /api/2/collections/:id/permissions

Get all permissions for a collection.

**Auth**: Required (must have WRITE access to collection)

**Response**:
```json
{
  "total": 4,
  "results": [
    {
      "id": "1",
      "role_id": "42",
      "collection_id": "123",
      "read": true,
      "write": false,
      "role": {
        "id": "42",
        "name": "John Doe",
        "type": "user"
      }
    },
    {
      "id": "2",
      "role_id": "100",
      "collection_id": "123",
      "read": true,
      "write": true,
      "role": {
        "id": "100",
        "name": "Editors",
        "type": "group"
      }
    }
  ]
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections/123/permissions
```

---

### PUT /api/2/collections/:id/permissions

Update permissions for a collection (grant/revoke access).

**Auth**: Required (must have WRITE access to collection)

**Request**:
```json
[
  {
    "role_id": "42",
    "read": true,
    "write": false
  },
  {
    "role_id": "100",
    "read": true,
    "write": true
  },
  {
    "role_id": "50",
    "read": false,
    "write": false
  }
]
```

**Response**: 200 OK (returns updated permissions list)

**Example**:
```bash
curl -X PUT \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '[
    {"role_id": "42", "read": true, "write": false},
    {"role_id": "100", "read": true, "write": true}
  ]' \
  https://aleph.example.com/api/2/collections/123/permissions
```

**Notes**:
- Setting `read=false` and `write=false` revokes all access
- `write=true` automatically implies `read=true`
- Public roles cannot have write access
- Casefiles cannot be made public

---

## Alerts

Alerts are saved search queries that notify users when new matching content appears.

### GET /api/2/alerts

List all alerts created by the authenticated user.

**Auth**: Required (logged in)

**Query Parameters**:
- `limit` (integer) - Number of results to return (pagination)
- `offset` (integer) - Number of results to skip (pagination)

**Response**: 200 OK
```json
{
  "results": [
    {
      "id": 42,
      "query": "Putin AND offshore",
      "query_text": "Putin AND offshore",
      "created_at": "2023-08-15T10:30:00Z",
      "updated_at": "2023-08-15T10:30:00Z",
      "role_id": "123",
      "notified_at": null
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/alerts
```

### POST /api/2/alerts

Create a new alert for a search query. The system will notify the user when new entities match this query.

**Auth**: Required (session write)

**Request Body**:
```json
{
  "query": "Putin AND offshore",
  "query_text": "Putin AND offshore"
}
```

**Response**: 200 OK (returns created Alert object)
```json
{
  "id": 42,
  "query": "Putin AND offshore",
  "query_text": "Putin AND offshore",
  "created_at": "2023-08-15T10:30:00Z",
  "updated_at": "2023-08-15T10:30:00Z",
  "role_id": "123",
  "notified_at": null
}
```

**Example**:
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "Putin AND offshore"}' \
  https://aleph.example.com/api/2/alerts
```

### GET /api/2/alerts/:id

Retrieve a specific alert by ID. User can only access their own alerts.

**Auth**: Required (logged in)

**Path Parameters**:
- `id` (integer) - Alert ID

**Response**: 200 OK
```json
{
  "id": 42,
  "query": "Putin AND offshore",
  "query_text": "Putin AND offshore",
  "created_at": "2023-08-15T10:30:00Z",
  "updated_at": "2023-08-15T10:30:00Z",
  "role_id": "123",
  "notified_at": "2023-08-16T09:00:00Z"
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/alerts/42
```

### DELETE /api/2/alerts/:id

Delete a specific alert. User can only delete their own alerts.

**Auth**: Required (session write)

**Path Parameters**:
- `id` (integer) - Alert ID

**Response**: 204 No Content

**Example**:
```bash
curl -X DELETE \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/alerts/42
```

---

## Bookmarks

Bookmarks allow users to save entities for quick access later.

### GET /api/2/bookmarks

Get all bookmarks created by the current user. Only returns bookmarks for entities in collections the user has read access to.

**Auth**: Required (logged in)

**Query Parameters**:
- `limit` (integer) - Number of results to return (pagination)
- `offset` (integer) - Number of results to skip (pagination)

**Response**: 200 OK
```json
{
  "results": [
    {
      "id": 789,
      "entity_id": "a1b2c3d4e5f6",
      "collection_id": 10,
      "role_id": "123",
      "created_at": "2023-08-15T14:20:00Z"
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/bookmarks
```

**Notes**:
- Bookmarks are automatically filtered by collection access permissions
- Results ordered by creation date (newest first)

### POST /api/2/bookmarks

Bookmark an entity. If the entity is already bookmarked by the user, returns the existing bookmark.

**Auth**: Required (session write)

**Request Body**:
```json
{
  "entity_id": "a1b2c3d4e5f6"
}
```

**Response**: 201 Created
```json
{
  "id": 789,
  "entity_id": "a1b2c3d4e5f6",
  "collection_id": 10,
  "role_id": "123",
  "created_at": "2023-08-15T14:20:00Z"
}
```

**Errors**:
- 400 Bad Request - Entity does not exist or user lacks read access

**Example**:
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"entity_id": "a1b2c3d4e5f6"}' \
  https://aleph.example.com/api/2/bookmarks
```

### DELETE /api/2/bookmarks/:entity_id

Remove a bookmark for the specified entity.

**Auth**: Required (session write)

**Path Parameters**:
- `entity_id` (string) - ID of the bookmarked entity

**Response**: 204 No Content

**Example**:
```bash
curl -X DELETE \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/bookmarks/a1b2c3d4e5f6
```

**Notes**:
- Silently succeeds even if bookmark doesn't exist
- Only removes bookmarks owned by the current user

---

## Mappings

Mappings define how to transform structured data (CSV/Excel tables) into Follow the Money entities. They are used for bulk data imports into collections.

### GET /api/2/collections/:id/mappings

List all mappings for a collection, optionally filtered by table.

**Auth**: Required (can browse)

**Path Parameters**:
- `id` (integer) - Collection ID

**Query Parameters**:
- `table` (string) - Filter by table entity ID
- `limit` (integer) - Number of results to return
- `offset` (integer) - Number of results to skip

**Response**: 200 OK
```json
{
  "results": [
    {
      "id": 123,
      "collection_id": 10,
      "table_id": "abc123",
      "query": {
        "entities": {
          "person": {
            "schema": "Person",
            "keys": ["full_name"],
            "properties": {
              "name": {"column": "full_name"},
              "birthDate": {"column": "dob"}
            }
          }
        }
      },
      "entityset_id": null,
      "disabled": false,
      "last_run_status": "success",
      "last_run_err_msg": null,
      "created_at": "2023-08-15T10:00:00Z",
      "updated_at": "2023-08-16T12:30:00Z"
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.com/api/2/collections/10/mappings?table=abc123"
```

### POST /api/2/collections/:id/mappings

Create a new mapping to transform table data into entities.

**Auth**: Required (collection write)

**Path Parameters**:
- `id` (integer) - Collection ID

**Request Body**:
```json
{
  "table_id": "abc123",
  "mapping_query": {
    "entities": {
      "person": {
        "schema": "Person",
        "keys": ["full_name"],
        "properties": {
          "name": {"column": "full_name"},
          "birthDate": {"column": "date_of_birth"},
          "nationality": {"column": "country"}
        }
      },
      "company": {
        "schema": "Company",
        "keys": ["company_name"],
        "properties": {
          "name": {"column": "company_name"},
          "jurisdiction": {"column": "country"}
        }
      }
    }
  },
  "entityset": {
    "entityset_id": 456
  }
}
```

**Response**: 200 OK (returns created Mapping object)

**Example**:
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d @mapping-config.json \
  https://aleph.example.com/api/2/collections/10/mappings
```

**Notes**:
- `mapping_query` follows FollowTheMoney mapping format
- `table_id` must be an entity in the same collection
- Optional `entityset_id` to load entities into a specific investigation

### GET /api/2/collections/:id/mappings/:mapping_id

Retrieve a specific mapping configuration.

**Auth**: Required (collection write)

**Path Parameters**:
- `id` (integer) - Collection ID
- `mapping_id` (integer) - Mapping ID

**Response**: 200 OK (returns Mapping object)

**Example**:
```bash
curl -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections/10/mappings/123
```

### PUT /api/2/collections/:id/mappings/:mapping_id

Update an existing mapping configuration.

**Auth**: Required (collection write)

**Path Parameters**:
- `id` (integer) - Collection ID
- `mapping_id` (integer) - Mapping ID

**Request Body**: Same format as POST `/mappings`

**Response**: 200 OK (returns updated Mapping object)

**Example**:
```bash
curl -X PUT \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d @updated-mapping.json \
  https://aleph.example.com/api/2/collections/10/mappings/123
```

### POST /api/2/collections/:id/mappings/:mapping_id/trigger

Execute the mapping to load entities from the table. Flushes previously loaded entities before loading new ones.

**Auth**: Required (collection write)

**Path Parameters**:
- `id` (integer) - Collection ID
- `mapping_id` (integer) - Mapping ID

**Response**: 202 Accepted

**Example**:
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections/10/mappings/123/trigger
```

**Notes**:
- Queues a background job to process the mapping
- Sets mapping status to PENDING
- Enables the mapping if previously disabled
- Previous entities from this mapping are deleted before loading new ones

### POST /api/2/collections/:id/mappings/:mapping_id/flush

Remove all entities that were loaded by this mapping.

**Auth**: Required (collection write)

**Path Parameters**:
- `id` (integer) - Collection ID
- `mapping_id` (integer) - Mapping ID

**Response**: 202 Accepted

**Example**:
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections/10/mappings/123/flush
```

**Notes**:
- Queues a background job to delete entities
- Disables the mapping
- Clears last run status and error messages

### DELETE /api/2/collections/:id/mappings/:mapping_id

Delete a mapping and flush all entities loaded by it.

**Auth**: Required (collection write)

**Path Parameters**:
- `id` (integer) - Collection ID
- `mapping_id` (integer) - Mapping ID

**Response**: 204 No Content

**Example**:
```bash
curl -X DELETE \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  https://aleph.example.com/api/2/collections/10/mappings/123
```

**Notes**:
- Permanently deletes the mapping configuration
- Queues a job to flush all entities created by this mapping
- Cannot be undone

