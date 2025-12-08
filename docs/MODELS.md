# Aleph Database Models Documentation

**Last Updated:** 2025-12-08
**Project:** Aleph - OCCRP Investigative Data Platform
**Version:** 4.x

## Overview

Aleph uses PostgreSQL for persistent data storage with SQLAlchemy as the ORM layer. The database architecture separates concerns between:

1. **Application Database** - User management, permissions, metadata
2. **FollowTheMoney Store** - Entity data using the FtM schema

This document details all database models in the Aleph application layer.

## Model Hierarchy

```
Base Models (Abstract)
├── IdModel           # Provides integer primary key and basic queries
├── DatedModel        # Adds created_at and updated_at timestamps
└── SoftDeleteModel   # Adds deleted_at for soft deletion (extends IdModel + DatedModel)

Concrete Models
├── Role              # Users, groups, and system actors
├── Collection        # Datasets and investigations
├── Entity            # FollowTheMoney entities (lightweight metadata)
├── Document          # File metadata and hierarchies
├── Permission        # Access control grants
├── EntitySet         # Lists, diagrams, timelines, profiles
├── EntitySetItem     # Entity membership in sets with judgements
├── Alert             # Saved search notifications
├── Export            # Export job tracking
├── Mapping           # Data import mapping configurations
├── Bookmark          # User-saved entities
└── Event             # Audit log entries
```

---

## Core Models

### 1. Role (`aleph/model/role.py`)

Represents users, groups, and system identities. Central to authentication and authorization.

**Location:** `aleph/model/role.py:32`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | Integer (PK) | Auto-increment primary key |
| `foreign_id` | Unicode(2048) | External identifier (unique) |
| `name` | Unicode | Display name |
| `email` | Unicode | Email address (nullable) |
| `type` | Enum | 'user', 'group', or 'system' |
| `api_key_digest` | Unicode | Hashed API key for programmatic access |
| `api_key_expires_at` | DateTime | API key expiration timestamp |
| `is_admin` | Boolean | Global admin privileges (default: False) |
| `is_muted` | Boolean | Disable notifications (default: False) |
| `is_tester` | Boolean | Beta feature access (default: False) |
| `is_blocked` | Boolean | Account disabled (default: False) |
| `password_digest` | Unicode | Hashed password (werkzeug.security) |
| `reset_token` | Unicode | Password reset token |
| `locale` | Unicode | Preferred language code |
| `last_login_at` | DateTime | Most recent login timestamp |
| `created_at` | DateTime | Account creation time |
| `updated_at` | DateTime | Last modification time |
| `deleted_at` | DateTime | Soft deletion timestamp |

#### Types

- **USER**: Individual user accounts
- **GROUP**: Collections of users for permission grants
- **SYSTEM**: Special system roles (e.g., 'guest', 'user')

#### Special System Roles

- `SYSTEM_GUEST`: Unauthenticated public access
- `SYSTEM_USER`: All authenticated users

#### Relationships

```python
# Many-to-many self-referential for group membership
Role.members → [Role]  # Members of this group
Role.roles → [Role]    # Groups this role belongs to

# One-to-many
Role.permissions → [Permission]
Role.alerts → [Alert]
```

#### Key Properties

- **has_password**: Boolean - Whether password authentication is set
- **has_api_key**: Boolean - Whether API key is configured
- **is_public**: Boolean - Whether role is publicly accessible
- **is_actor**: Boolean - Can perform actions (user, not blocked, not deleted)
- **is_alertable**: Boolean - Can receive notifications (active email, not muted)
- **label**: String - Anonymized display name

#### Authentication Methods

```python
# Password login
Role.set_password(secret)           # Hash and store password
Role.check_password(secret)         # Verify password
Role.login(email, password)         # Authenticate user

# API key authentication
Role.by_api_key(api_key)           # Lookup by API key (with expiration check)

# External identity
Role.load_or_create(foreign_id, type, name, email, is_admin)
```

#### Security Features

- Password hashing using `werkzeug.security.generate_password_hash`
- API keys stored as hashed digests (SHA256)
- API key expiration enforcement
- Account blocking capability
- Signature tokens for email invitations (URLSafeTimedSerializer)

---

### 2. Collection (`aleph/model/collection.py`)

Primary data container enforcing access control. Can be either a **dataset** (source material) or **casefile** (investigation).

**Location:** `aleph/model/collection.py:20`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | Integer (PK) | Auto-increment primary key |
| `foreign_id` | Unicode (unique) | External identifier |
| `label` | Unicode | Collection name |
| `summary` | Unicode | Description |
| `category` | Unicode | Classification (see categories below) |
| `frequency` | Unicode | Update schedule |
| `countries` | ARRAY(Unicode) | Related countries (ISO codes) |
| `languages` | ARRAY(Unicode) | Content languages (ISO codes) |
| `publisher` | Unicode | Source organization |
| `publisher_url` | Unicode | Publisher website |
| `info_url` | Unicode | Collection homepage |
| `data_url` | Unicode | Data download URL |
| `restricted` | Boolean | Highly sensitive (default: False) |
| `xref` | Boolean | Auto cross-reference on updates (default: False) |
| `creator_id` | Integer (FK→Role) | Creating user |
| `data_updated_at` | DateTime | Last content modification |
| `created_at` | DateTime | Collection creation |
| `updated_at` | DateTime | Metadata last modified |
| `deleted_at` | DateTime | Soft deletion timestamp |

#### Categories

19 predefined collection types:

| Category | Description |
|----------|-------------|
| `news` | News archives |
| `leak` | Leaks |
| `land` | Land registry |
| `gazette` | Gazettes |
| `court` | Court archives |
| `company` | Company registries |
| `sanctions` | Sanctions lists |
| `procurement` | Procurement data |
| `finance` | Financial records |
| `grey` | Grey literature |
| `library` | Document libraries |
| `license` | Licenses and concessions |
| `regulatory` | Regulatory filings |
| `poi` | Persons of interest |
| `customs` | Customs declarations |
| `census` | Population census |
| `transport` | Air and maritime registers |
| `casefile` | Investigations (special type) |
| `other` | Uncategorized material |

#### Update Frequencies

- `unknown`: Not known
- `never`: Not updated
- `daily`: Daily updates
- `weekly`: Weekly updates
- `monthly`: Monthly updates
- `annual`: Annual updates

#### Special Properties

- **casefile**: Boolean - Returns True if category == 'casefile'
- **secret**: Boolean - Returns True if no public permissions exist
- **team_id**: List[str] - IDs of all roles with read access
- **ns**: Namespace - FollowTheMoney namespace for entity ID signing

#### Relationships

```python
Collection.creator → Role
Collection.entities → [Entity]
Collection.documents → [Document]
Collection.permissions → [Permission]  # via backref
Collection.entitysets → [EntitySet]    # via backref
```

#### Access Control

Collections are the **granularity boundary** for permissions. You cannot grant access to individual entities or documents - only entire collections.

#### Methods

```python
Collection.create(data, authz, created_at=None)
Collection.by_foreign_id(foreign_id, deleted=False)
Collection.all_authz(authz, deleted=False)        # Filter by user permissions
Collection.all_casefiles(authz=None)              # Only investigations
Collection.all_by_secret(secret, authz=None)      # Filter by privacy
Collection.update(data, authz)                    # Update metadata
Collection.touch()                                # Update data_updated_at
```

---

### 3. Entity (`aleph/model/entity.py`)

Lightweight metadata record for FollowTheMoney entities. The full entity data is stored in the FtM Store (PostgreSQL JSONB), while this model provides collection association and indexing metadata.

**Location:** `aleph/model/entity.py:17`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | String(ENTITY_ID_LEN) | Entity ID (signed by collection namespace) |
| `schema` | String(255) | FtM schema name (e.g., 'Person', 'Company') |
| `data` | JSONB | FtM properties dictionary |
| `collection_id` | Integer (FK→Collection) | Parent collection |
| `role_id` | Integer (FK→Role) | Creating user (nullable) |
| `created_at` | DateTime | Creation timestamp |
| `updated_at` | DateTime | Last modification |

#### Key Concepts

- **ID Signing**: Entity IDs are namespace-signed to ensure uniqueness across collections
- **Schema**: References FollowTheMoney entity types (Person, Company, Asset, Document, etc.)
- **Data**: JSONB column stores the entity's properties as key-value pairs
- **Immutable Checksums**: Hash properties (contentHash, pdfHash) cannot be overwritten by users

#### Common Schemas

- **Thing**: Base type for all entities
- **LegalEntity**: Person, Company, Organization
- **Analyzable**: Documents, Pages, Email

#### Entity Structure (JSONB data format)

```json
{
  "schema": "Person",
  "properties": {
    "name": ["John Doe"],
    "nationality": ["us"],
    "birthDate": ["1980-01-15"],
    "idNumber": ["123-45-6789"]
  }
}
```

#### Relationships

```python
Entity.collection → Collection
```

#### Methods

```python
Entity.create(data, collection, sign=True, role_id=None)
Entity.by_id(entity_id, collection=None)
Entity.by_collection(collection_id)
Entity.update(data, collection, sign=True)
Entity.to_proxy()                              # Convert to FtM proxy object
Entity.delete_by_collection(collection_id)     # Bulk deletion
```

#### Integration with FollowTheMoney

```python
# Get FtM model for this entity
entity.model  # Returns followthemoney.model.Schema

# Convert to FtM proxy for manipulation
proxy = entity.to_proxy()
proxy.set('name', 'Jane Doe')
entity.update(proxy.to_dict(), collection)
```

---

### 4. Document (`aleph/model/document.py`)

File metadata for uploaded documents. Supports hierarchical folder structures. The actual file binary is stored in the archive (S3/filesystem), not the database.

**Location:** `aleph/model/document.py:19`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | BigInteger (PK) | Auto-increment primary key |
| `content_hash` | Unicode(65) | SHA1 hash of file contents |
| `foreign_id` | Unicode | External identifier (not unique) |
| `schema` | String(255) | FtM schema ('Document', 'Folder', 'Table') |
| `meta` | JSONB | File metadata dictionary |
| `parent_id` | BigInteger | Parent folder ID (nullable) |
| `collection_id` | Integer (FK→Collection) | Parent collection |
| `role_id` | Integer (FK→Role) | Uploader |
| `created_at` | DateTime | Upload timestamp |
| `updated_at` | DateTime | Last modification |

#### Schemas

- **Document**: Generic file (PDF, Word, etc.)
- **Folder**: Directory container
- **Table**: Spreadsheet/CSV data

#### Metadata Fields (JSONB)

Stored in the `meta` column:

```python
{
    "title": "Document title",
    "summary": "Brief description",
    "author": "Document author",
    "publisher": "Publishing organization",
    "crawler": "Data source/scraper name",
    "source_url": "Original URL",
    "file_name": "example.pdf",
    "mime_type": "application/pdf",
    "headers": {"content-type": "application/pdf"},
    "date": "2023-01-15",
    "authored_at": "2023-01-10T12:00:00",
    "modified_at": "2023-01-12T15:30:00",
    "published_at": "2023-01-15T09:00:00",
    "retrieved_at": "2023-01-16T14:20:00",
    "languages": ["eng", "deu"],
    "countries": ["us", "de"],
    "keywords": ["investigation", "corruption"]
}
```

#### Content Hashing

Files are stored by SHA1 content hash:
- **Path format**: `{hash[0:2]}/{hash[2:4]}/{hash[4:6]}/{full_hash}`
- **Example**: `34/d4/e3/34d4e388b7994b3846504e89f54e10a6fd869eb8`
- **Deduplication**: Identical files stored once, referenced multiple times

#### Hierarchical Structure

```
Collection
└── Folder (parent_id=NULL)
    ├── Folder (parent_id=parent.id)
    │   └── Document (parent_id=parent.id)
    └── Document (parent_id=parent.id)
```

#### Key Properties

- **model**: FtM schema object
- **ancestors**: List[int] - All parent folder IDs up to root (cached)

#### Relationships

```python
Document.collection → Collection
```

#### Methods

```python
Document.save(collection, parent, foreign_id, content_hash, meta, role_id)
Document.by_id(document_id, collection=None)
Document.by_content_hash(content_hash)
Document.by_collection(collection_id)
Document.to_proxy(ns=None)                    # Convert to FtM entity
Document.update(data)                         # Update metadata
Document.delete_by_collection(collection_id)
```

---

### 5. Permission (`aleph/model/permission.py`)

Access control grants linking roles to collections with read/write privileges.

**Location:** `aleph/model/permission.py:8`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | Integer (PK) | Auto-increment primary key |
| `role_id` | Integer (FK→Role) | Granted role (user or group) |
| `collection_id` | Integer (FK→Collection) | Target collection |
| `read` | Boolean | View access (default: False) |
| `write` | Boolean | Edit access (default: False) |
| `created_at` | DateTime | Grant timestamp |
| `updated_at` | DateTime | Last modification |

#### Permission Levels

- **read=True, write=False**: View-only access
  - View all collection contents
  - See metadata and entities
  - View files and network diagrams
  - See other users' permissions

- **read=True, write=True**: Full access
  - All read permissions, plus:
  - Upload and edit documents
  - Create and modify entities
  - Edit collection settings
  - Grant/revoke permissions for other users

#### Important Rules

1. **write implies read**: Setting write=True automatically grants read=True
2. **Admins bypass permissions**: Admin users have implicit read+write to all collections
3. **Collection-level only**: Cannot grant permissions to individual entities/documents
4. **Group expansion**: Permissions granted to groups apply to all members

#### Relationships

```python
Permission.role → Role
Permission.collection → Collection  # via collection_id (no explicit relationship)
```

#### Methods

```python
Permission.grant(collection, role, read, write)
Permission.by_collection_role(collection, role)
Permission.delete_by_collection(collection_id)
```

#### Usage Example

```python
# Grant read-only access
Permission.grant(collection, user_role, read=True, write=False)

# Grant full access
Permission.grant(collection, editor_role, read=True, write=True)

# Revoke all access
Permission.grant(collection, former_member, read=False, write=False)
# This deletes the permission record

# Check permissions
perm = Permission.by_collection_role(collection, role)
if perm and perm.write:
    # User can edit
```

---

### 6. EntitySet (`aleph/model/entityset.py`)

Collections of entities for analysis and organization. Four distinct types: lists, diagrams, timelines, and profiles.

**Location:** `aleph/model/entityset.py:34`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | String(ENTITY_ID_LEN) (PK) | Random text ID |
| `label` | Unicode | EntitySet name |
| `type` | String(10) | 'list', 'diagram', 'timeline', or 'profile' |
| `summary` | Unicode | Description (nullable) |
| `layout` | JSONB | UI layout configuration |
| `role_id` | Integer (FK→Role) | Creator |
| `collection_id` | Integer (FK→Collection) | Parent collection |
| `parent_id` | String(ENTITY_ID_LEN) (FK→EntitySet) | Parent set (for hierarchies) |
| `created_at` | DateTime | Creation timestamp |
| `updated_at` | DateTime | Last modification |
| `deleted_at` | DateTime | Soft deletion timestamp |

#### Types

| Type | Description | Use Case |
|------|-------------|----------|
| `list` | Simple entity collection | Watchlists, saved searches |
| `diagram` | Network visualization | Relationship mapping |
| `timeline` | Chronological view | Event sequencing |
| `profile` | Detailed entity dossier | Person/company profiles |

#### Layout Structure (JSONB)

Stores UI-specific configuration:

```json
{
  "viewport": {"x": 0, "y": 0, "zoom": 1},
  "nodes": {
    "entity-id-1": {"x": 100, "y": 200},
    "entity-id-2": {"x": 300, "y": 400}
  },
  "selection": ["entity-id-1"]
}
```

#### Relationships

```python
EntitySet.role → Role
EntitySet.collection → Collection
EntitySet.parent → EntitySet
EntitySet.children → [EntitySet]
EntitySet.items → [EntitySetItem]  # Many-to-many through EntitySetItem
EntitySet.mappings → [Mapping]     # Data import configs (via backref)
```

#### Key Properties

- **entities**: List[str] - IDs of all positively judged entities in this set

#### Profile Uniqueness Constraint

**Special behavior for profiles**: An entity can only belong to ONE profile per collection. Attempting to add an entity to a second profile triggers automatic profile merging.

```python
# If entity E1 is in Profile P1
# Adding E1 to Profile P2 will merge P2 into P1
```

#### Methods

```python
EntitySet.create(data, collection, authz)
EntitySet.by_id(entityset_id, types=None, deleted=False)
EntitySet.by_authz(authz, types=None, prefix=None)
EntitySet.by_type(types, deleted=False)
EntitySet.by_collection_id(collection_id, types=None)
EntitySet.by_entity_id(entity_id, collection_ids, judgements, types, labels)
EntitySet.entity_entitysets(entity_id, collection_id=None)
EntitySet.all_profiles(collection_id, entity_id=None)
EntitySet.type_counts(authz=None, collection_id=None)

# Instance methods
entityset.items(authz=None, deleted=False)
entityset.profile(judgements=None, deleted=False)
entityset.merge(other, merged_by_id)          # Merge two entity sets
entityset.update(data)
entityset.delete(deleted_at=None)
```

---

### 7. EntitySetItem (`aleph/model/entityset.py:278`)

Join table linking entities to EntitySets with judgement metadata. Supports cross-reference decision-making.

**Location:** `aleph/model/entityset.py:278`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | Integer (PK) | Auto-increment primary key |
| `entityset_id` | String(FK→EntitySet) | Parent EntitySet |
| `entity_id` | String(ENTITY_ID_LEN) | Target entity ID |
| `collection_id` | Integer (FK→Collection) | Entity's collection |
| `compared_to_entity_id` | String | Reference entity for xref |
| `added_by_id` | Integer (FK→Role) | User who added/judged |
| `judgement` | Enum(Judgement) | Decision status |
| `created_at` | DateTime | Addition timestamp |
| `updated_at` | DateTime | Last modification |
| `deleted_at` | DateTime | Soft deletion timestamp |

#### Judgement Enum

```python
class Judgement(Enum):
    POSITIVE = "positive"      # Confirmed match/inclusion
    NEGATIVE = "negative"      # Confirmed non-match/exclusion
    UNSURE = "unsure"         # Undecided
    NO_JUDGEMENT = "no_judgement"  # No decision made (not stored in DB)
```

#### Judgement Algebra

When merging entity sets, judgements combine:

| J1 | J2 | Result |
|----|----|----|
| positive | positive | positive |
| negative | * | negative |
| unsure | positive | unsure |
| unsure | unsure | unsure |

#### Relationships

```python
EntitySetItem.entityset → EntitySet
EntitySetItem.collection → Collection
EntitySetItem.added_by → Role
```

#### Methods

```python
EntitySetItem.save(entityset, entity_id, judgement, collection_id, **data)
EntitySetItem.by_entity_id(entityset, entity_id)
EntitySetItem.delete_by_collection(collection_id)
EntitySetItem.delete_by_entity(entity_id)
```

#### Cross-Reference Workflow

```python
# Initial xref match found
item = EntitySetItem.save(
    entityset=investigation,
    entity_id="person-123",
    judgement=Judgement.UNSURE,
    compared_to_entity_id="person-456",  # Matched against this
    collection_id=collection.id,
    added_by_id=user.id
)

# User reviews and confirms match
item = EntitySetItem.save(
    entityset=investigation,
    entity_id="person-123",
    judgement=Judgement.POSITIVE  # Overwrites previous UNSURE
)

# User rejects match
item = EntitySetItem.save(
    entityset=investigation,
    entity_id="person-123",
    judgement=Judgement.NEGATIVE
)
```

---

### 8. Alert (`aleph/model/alert.py`)

Saved search queries that notify users when new matching content is added.

**Location:** `aleph/model/alert.py:9`

#### Schema

| Column | Type | Description |
|--------|------|-------------|
| `id` | Integer (PK) | Auto-increment primary key |
| `query` | Unicode | Elasticsearch query string |
| `role_id` | Integer (FK→Role) | Subscribing user |
| `notified_at` | DateTime | Last notification sent |
| `created_at` | DateTime | Alert creation |
| `updated_at` | DateTime | Last modification |

#### Relationships

```python
Alert.role → Role
```

#### Workflow

1. User saves a search as an alert
2. Background worker periodically checks for new results
3. If matches found, sends email notification
4. Updates `notified_at` timestamp
5. Future checks only return results newer than `notified_at`

#### Methods

```python
Alert.create(data, role_id)
Alert.by_id(id, role_id=None)
Alert.by_role_id(role_id)
alert.update()  # Updates notified_at to now
```

---

## Supporting Models

### 9. Export (`aleph/model/export.py`)

Tracks background export jobs (search results, entity data, etc.).

**Typical Fields:**
- Export format (CSV, JSON, Excel)
- Status (pending, running, completed, failed)
- Download URL
- Expiration timestamp

### 10. Mapping (`aleph/model/mapping.py`)

Configuration for importing tabular data into FollowTheMoney entities.

**Key Concepts:**
- Column-to-property mappings
- Entity schema templates
- Transformation rules

### 11. Bookmark (`aleph/model/bookmark.py`)

User-saved entities for quick access.

**Schema:**
- User ID
- Entity ID
- Collection ID
- Creation timestamp

### 12. Event (`aleph/model/event.py`)

Audit log for system actions.

**Event Types:**
- User login/logout
- Collection creation/deletion
- Permission changes
- Entity modifications
- Export requests

**Fields:**
- Actor (role_id)
- Action type
- Target resource
- Timestamp
- Metadata (JSONB)

---

## Database Inheritance Hierarchy

### Base Classes

#### IdModel (`aleph/model/common.py`)

Provides basic ID-based queries:

```python
id = db.Column(db.Integer, primary_key=True)

@classmethod
def all()                    # Base query
def all_by_ids(ids)         # Filter by ID list
def all_ids()               # Query returning only IDs
def by_id(id)               # Lookup single record
```

#### DatedModel (`aleph/model/common.py`)

Adds timestamp tracking:

```python
created_at = db.Column(db.DateTime, default=datetime.utcnow)
updated_at = db.Column(db.DateTime, default=datetime.utcnow)

def to_dict_dates()  # Serialize timestamps to ISO format
```

#### SoftDeleteModel (`aleph/model/common.py`)

Combines IdModel + DatedModel + soft deletion:

```python
deleted_at = db.Column(db.DateTime, nullable=True)

@classmethod
def all(deleted=False)      # Optionally include deleted records
def delete(deleted_at=None) # Soft delete (sets timestamp)
```

---

## FollowTheMoney Integration

### Entity Storage Architecture

Aleph uses a **dual-storage** approach:

1. **Application DB (this document)**: Lightweight metadata
   - `Entity` table: Schema name, collection association, role
   - `Document` table: File metadata, folder hierarchy

2. **FtM Store**: Full entity data
   - **followthemoney-store** library
   - Stores entity **fragments** in PostgreSQL JSONB
   - Fragments preserve provenance and can be selectively updated

### Fragment System

Entities are stored as multiple fragments:

```sql
-- ftm_store.fragments table (conceptual)
entity_id | origin   | fragment  | data
----------|----------|-----------|------------------
97e1f...  | ingest   | default   | {"schema": "Pages", "properties": {...}}
97e1f...  | ingest   | eae30...  | {"properties": {"indexText": ["..."]}}
97e1f...  | analyze  | default   | {"properties": {"peopleMentioned": ["John Doe"]}}
```

When indexing, fragments are **merged** by entity_id into a single searchable entity.

### Namespace Signing

Collections use **FollowTheMoney namespaces** to sign entity IDs:

```python
collection.foreign_id = "my-dataset"
ns = Namespace("my-dataset")

# Sign an ID
signed_id = ns.sign("entity-123")
# Result: "my-dataset.entity-123" (simplified)

# Verify and strip
assert ns.verify(signed_id)
original = ns.strip(signed_id)  # Returns "entity-123"
```

**Purpose:** Prevents ID collisions across collections while allowing references.

---

## Database Migrations

Aleph uses **Alembic** for database migrations:

```bash
# Create new migration
aleph db revision -m "Add new field to collection"

# Apply migrations
aleph db upgrade

# Rollback
aleph db downgrade
```

**Migration files:** `aleph/migrate/versions/`

---

## Indexing and Caching

### Elasticsearch Indexes

Entities are indexed in **per-schema** Elasticsearch indexes:

```
aleph-entity-company-v1
aleph-entity-person-v1
aleph-entity-document-v1
...
```

See `aleph/index/` for index management.

### Redis Caching

Cached data:
- Collection statistics
- Document ancestry chains
- User sessions
- Background job metadata

Cache keys use pattern: `aleph:{type}:{id}`

---

## Query Patterns

### Authorization Filtering

Most queries apply authorization filters:

```python
# Collections accessible to user
q = Collection.all_authz(authz)

# Expands to:
q = q.join(Permission)
q = q.filter(Permission.read == True)
q = q.filter(Permission.role_id.in_(authz.roles))
```

### Soft Deletion

Models with `SoftDeleteModel` exclude deleted records by default:

```python
# Excludes deleted
all_active = Collection.all()

# Includes deleted
all_including_deleted = Collection.all(deleted=True)

# Deleted only
deleted_only = Collection.all(deleted='only')
```

### Batch Queries

For large result sets, use `yield_per`:

```python
q = Entity.by_collection(collection_id)
for entity in q.yield_per(5000):
    process(entity)
```

---

## Performance Considerations

### Database Indexes

Key indexes for performance:

```sql
-- Collections
CREATE INDEX ix_collection_foreign_id ON collection(foreign_id);
CREATE INDEX ix_collection_deleted_at ON collection(deleted_at);

-- Entities
CREATE INDEX ix_entity_collection_id ON entity(collection_id);
CREATE INDEX ix_entity_schema ON entity(schema);

-- Documents
CREATE INDEX ix_document_collection_id ON document(collection_id);
CREATE INDEX ix_document_content_hash ON document(content_hash);
CREATE INDEX ix_document_parent_id ON document(parent_id);

-- Permissions
CREATE INDEX ix_permission_role_id ON permission(role_id);
CREATE INDEX ix_permission_collection_id ON permission(collection_id);

-- EntitySets
CREATE INDEX ix_entityset_type ON entityset(type);
CREATE INDEX ix_entityset_collection_id ON entityset(collection_id);

-- EntitySetItems
CREATE INDEX ix_entityset_item_entityset_id ON entityset_item(entityset_id);
CREATE INDEX ix_entityset_item_entity_id ON entityset_item(entity_id);
CREATE INDEX ix_entityset_item_collection_id ON entityset_item(collection_id);
```

### JSONB Query Optimization

For querying JSONB columns:

```sql
-- GIN index for JSONB containment
CREATE INDEX idx_entity_data ON entity USING GIN(data);
CREATE INDEX idx_document_meta ON document USING GIN(meta);

-- Query examples
SELECT * FROM entity WHERE data @> '{"properties": {"name": ["John Doe"]}}';
SELECT * FROM document WHERE meta->>'file_name' LIKE '%.pdf';
```

---

## Common Operations

### Create a New Collection

```python
from aleph.model import Collection, Permission

collection = Collection.create(
    data={
        "label": "Investigation X",
        "category": "casefile",
        "summary": "Corruption case",
        "countries": ["us", "ru"],
        "languages": ["eng"]
    },
    authz=authz
)

# Creator automatically gets read+write permissions
db.session.commit()
```

### Grant User Access

```python
Permission.grant(
    collection=collection,
    role=user_role,
    read=True,
    write=False
)
db.session.commit()
```

### Add Entity to EntitySet

```python
from aleph.model import EntitySetItem, Judgement

EntitySetItem.save(
    entityset=investigation,
    entity_id="person-abc123",
    judgement=Judgement.POSITIVE,
    collection_id=collection.id,
    added_by_id=user.id
)
db.session.commit()
```

### Create Cross-Reference Decision

```python
# Cross-reference found potential match
EntitySetItem.save(
    entityset=profile,
    entity_id="company-xyz",
    judgement=Judgement.UNSURE,
    compared_to_entity_id="company-abc",  # Compared against
    collection_id=collection.id,
    added_by_id=user.id
)

# User confirms match
EntitySetItem.save(
    entityset=profile,
    entity_id="company-xyz",
    judgement=Judgement.POSITIVE  # Updates previous record
)
db.session.commit()
```

### Search for Documents by Hash

```python
documents = Document.by_content_hash(
    "34d4e388b7994b3846504e89f54e10a6fd869eb8"
)
for doc in documents:
    print(f"{doc.id}: {doc.meta.get('file_name')}")
```

---

## Security Considerations

### SQL Injection Prevention

SQLAlchemy ORM provides automatic parameterization:

```python
# SAFE - uses parameterized query
Collection.by_foreign_id(foreign_id)

# UNSAFE - never construct raw SQL from user input
db.session.execute(f"SELECT * FROM collection WHERE foreign_id = '{foreign_id}'")
```

### Password Security

```python
# GOOD - uses werkzeug password hashing
role.set_password("user_password")

# NEVER store plaintext passwords
role.password_digest = "user_password"  # DON'T DO THIS
```

### API Key Security

API keys are stored as **digests** (hashed), not plaintext:

```python
# Storage
api_key = generate_random_key()  # Only shown once to user
role.api_key_digest = hash_api_key(api_key)
role.save()

# Verification
role = Role.by_api_key(submitted_key)  # Hashes and compares digest
```

### Permission Checks

**Always** filter queries by authorization:

```python
# GOOD
collections = Collection.all_authz(authz)

# DANGEROUS - exposes all collections
collections = Collection.all()
```

---

## Troubleshooting

### Common Issues

**Problem:** Entity not appearing in search
**Solution:** Check indexing status, reindex collection:
```bash
aleph reindex --foreign-id collection-name
```

**Problem:** Permission denied errors
**Solution:** Verify Permission records:
```python
Permission.by_collection_role(collection, role)
```

**Problem:** Orphaned documents after deletion
**Solution:** Soft-deletion should cascade. If not:
```python
Document.delete_by_collection(collection_id)
Entity.delete_by_collection(collection_id)
```

**Problem:** API key expired
**Solution:** Check expiration:
```python
role.api_key_expires_at  # Must be in future
role.api_key_expiration_notification_sent  # Notification count
```

---

## Further Reading

- **FollowTheMoney Spec:** https://followthemoney.tech
- **Aleph API Docs:** https://docs.aleph.occrp.org/developers/
- **SQLAlchemy ORM:** https://docs.sqlalchemy.org/
- **Alembic Migrations:** https://alembic.sqlalchemy.org/

---

**Generated:** 2025-12-08 by Claude Code
**Repository:** `/media/Daten1/projects/aleph`
**Maintainer:** OCCRP Development Team
