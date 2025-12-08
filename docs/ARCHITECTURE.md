# Aleph Architecture

Deep dive into the Aleph system architecture, design patterns, and technical implementation.

## Table of Contents
- [System Overview](#system-overview)
- [Component Architecture](#component-architecture)
- [Data Flow](#data-flow)
- [Backend Architecture](#backend-architecture)
- [Frontend Architecture](#frontend-architecture)
- [Search Architecture](#search-architecture)
- [Worker Architecture](#worker-architecture)
- [Design Patterns](#design-patterns)
- [Technology Stack](#technology-stack)

---

## System Overview

Aleph is a distributed, multi-tier application designed for high-volume document processing and entity analysis.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    USER INTERFACE (React)                       │
│  - UI SPA at /home/user/aleph/ui (React 17, Redux, TypeScript) │
└────────────────────┬────────────────────────────────────────────┘
                     │ HTTP/REST API
┌────────────────────▼────────────────────────────────────────────┐
│               FLASK API BACKEND (Python 3.10)                   │
│  - Main app: /home/user/aleph/aleph/                            │
│  - REST API at /api/2/*                                         │
│  - Authentication & Authorization (Authz system)                │
└────────────────┬──────────────────────────────┬─────────────────┘
                 │                              │
        ┌────────▼────────┐          ┌──────────▼──────────┐
        │  Async Workers  │          │  Search & Index    │
        │  (Celery/Pika)  │          │  (Elasticsearch)   │
        │  - Processing   │          │  - Full-text index │
        │  - Xref         │          │  - Collections     │
        │  - Alerts       │          │  - Entities        │
        │  - Export       │          │  - Xref matches    │
        └────────┬────────┘          └──────────┬──────────┘
                 │                              │
    ┌────────────┼──────────┬──────────────────┼────────────┐
    │            │          │                  │            │
┌───▼──┐  ┌─────▼──┐  ┌────▼─────┐  ┌────────▼─┐  ┌───────▼────┐
│ Disk │  │PostgreSQL  │ Archive   │  │Elasticsearch  │  │RabbitMQ  │
│File  │  │Database    │S3/Local   │  │ Search Index   │  │Task Queue│
└──────┘  └────────┘  └──────────┘  └────────────┘  └──────────┘
```

### Key Design Principles

1. **Separation of Concerns** - Clear boundaries between layers (model, logic, view)
2. **Asynchronous Processing** - Heavy operations delegated to background workers
3. **Scalability** - Horizontal scaling via worker processes and Elasticsearch cluster
4. **Data Immutability** - Soft deletes, audit trails, versioned indices
5. **Authorization-First** - All data access filtered by user permissions
6. **Schema-Driven** - FollowTheMoney schema enforces data structure

---

## Component Architecture

### Directory Structure

```
aleph/
├── aleph/                      # Backend Python package
│   ├── core.py                 # Flask app initialization (206 lines)
│   ├── settings.py             # Configuration (200+ lines)
│   ├── authz.py                # Authorization system (182 lines)
│   ├── worker.py               # Worker orchestration (260+ lines)
│   ├── manage.py               # CLI commands (600+ lines)
│   ├── wsgi.py                 # WSGI entry point
│   │
│   ├── model/                  # Data Models (SQLAlchemy)
│   │   ├── common.py           # Base classes (IdModel, SoftDeleteModel)
│   │   ├── role.py             # Users, groups, auth (200+ lines)
│   │   ├── collection.py       # Collections (180+ lines)
│   │   ├── entity.py           # Entities (104 lines)
│   │   ├── document.py         # Documents
│   │   ├── entityset.py        # Lists, diagrams, timelines
│   │   ├── alert.py            # User alerts
│   │   ├── bookmark.py         # Bookmarks
│   │   ├── event.py            # Audit logs
│   │   ├── mapping.py          # Data mappings
│   │   ├── export.py           # Export jobs
│   │   └── permission.py       # Access control
│   │
│   ├── logic/                  # Business Logic
│   │   ├── collections.py      # Collection operations (200+ lines)
│   │   ├── entities.py         # Entity CRUD
│   │   ├── xref.py             # Cross-reference (400+ lines)
│   │   ├── documents.py        # Document processing
│   │   ├── export.py           # Export generation
│   │   ├── alerts.py           # Alert checking
│   │   ├── roles.py            # User management
│   │   ├── permissions.py      # Permission logic
│   │   ├── mapping.py          # Mapping processing
│   │   ├── matching.py         # Entity matching
│   │   ├── expand.py           # Entity expansion
│   │   ├── profiles.py         # User profiles
│   │   ├── notifications.py    # Notifications
│   │   └── processing.py       # Bulk indexing
│   │
│   ├── views/                  # API Endpoints (Flask Blueprints)
│   │   ├── base_api.py         # System endpoints
│   │   ├── collections_api.py  # Collection API (200+ lines)
│   │   ├── entities_api.py     # Entity/search API (400+ lines)
│   │   ├── entitysets_api.py   # EntitySet API
│   │   ├── roles_api.py        # User/role API
│   │   ├── sessions_api.py     # Authentication
│   │   ├── alerts_api.py       # Alerts
│   │   ├── xref_api.py         # Cross-reference
│   │   ├── mappings_api.py     # Data mappings
│   │   ├── ingest_api.py       # Document upload
│   │   ├── exports_api.py      # Export management
│   │   ├── bookmarks_api.py    # Bookmarks
│   │   └── permissions_api.py  # Permissions
│   │
│   ├── search/                 # Search Query Builders
│   │   ├── query.py            # Base Query class (300+ lines)
│   │   ├── parser.py           # Query parsing
│   │   ├── facet.py            # Facet aggregations
│   │   └── result.py           # Result objects
│   │
│   ├── index/                  # Index Management
│   │   ├── indexes.py          # Schema definitions (131 lines)
│   │   ├── entities.py         # Entity indexing
│   │   ├── collections.py      # Collection indexing
│   │   ├── xref.py             # Xref indexing
│   │   ├── util.py             # Index utilities
│   │   └── admin.py            # Index admin
│   │
│   └── tests/                  # Test suite
│
└── ui/                         # Frontend React App
    └── src/
        ├── app/                # App setup
        │   ├── store.js        # Redux store (36 lines)
        │   ├── api.js          # API client
        │   ├── App.jsx         # Root component
        │   └── Router.jsx      # Routing (100+ lines)
        │
        ├── actions/            # Redux actions
        ├── reducers/           # Redux reducers
        ├── components/         # React components (38 dirs)
        ├── screens/            # Full-page views (26 screens)
        ├── dialogs/            # Modal dialogs (22 dialogs)
        └── viewers/            # Document viewers
```

---

## Backend Architecture

### Layer Breakdown

#### 1. Model Layer (`aleph/model/`)

**Purpose**: Data persistence and ORM

**Key Files**:
- `common.py:1-50` - Base model classes
- `role.py:1-200` - User/group authentication
- `collection.py:1-180` - Collection entity
- `entity.py:1-104` - Entity proxy pattern

**Patterns**:
- **Base Classes**: All models inherit from `IdModel`, `DatedModel`, `SoftDeleteModel`
- **Soft Deletes**: Records marked with `deleted_at` instead of hard delete
- **Proxy Pattern**: Entity model wraps FollowTheMoney proxy objects

**Example Model**:
```python
# aleph/model/collection.py
class Collection(db.Model, IdModel, SoftDeleteModel, DatedModel):
    __tablename__ = 'collection'

    foreign_id = db.Column(db.Unicode, unique=True, nullable=False)
    label = db.Column(db.Unicode, nullable=False)
    category = db.Column(db.Enum(*CATEGORIES), nullable=False)
    restricted = db.Column(db.Boolean, default=False)
    countries = db.Column(ARRAY(db.Unicode))

    # Relationships
    documents = db.relationship('Document', backref='collection')
    entities = db.relationship('Entity', backref='collection')
```

#### 2. Logic Layer (`aleph/logic/`)

**Purpose**: Business logic and orchestration

**Key Operations**:
- **collections.py**: `create_collection()`, `update_collection()`, `delete_collection()`, `reindex_collection()`
- **entities.py**: `upsert_entity()`, `validate_entity()`, `delete_entity()`
- **xref.py**: `xref_collection()`, `iter_matches()` - Cross-reference matching using ML
- **documents.py**: `crawl_directory()`, `ingest_flush()` - Document processing

**Separation from Views**:
```python
# View layer (aleph/views/collections_api.py)
@blueprint.route('/api/2/collections', methods=['POST'])
def create():
    authz = request.authz
    data = parse_request(CollectionSchema)
    collection = create_collection(data, authz)  # <- Logic layer
    return jsonify(collection)

# Logic layer (aleph/logic/collections.py)
def create_collection(data, authz):
    collection = Collection.create(data, authz)
    db.session.commit()
    index_collection(collection)  # <- Index layer
    return collection
```

#### 3. View Layer (`aleph/views/`)

**Purpose**: REST API endpoints

**Blueprints**:
- Each API group is a Flask Blueprint
- Request parsing with Marshmallow schemas
- Authorization checks via `@require()` decorator
- Pagination with `@paginate()` decorator

**API Structure**:
```
/api/2/
├── /status                    # System health
├── /sessions                  # Authentication
├── /collections               # Collection CRUD
│   ├── /{id}
│   ├── /{id}/ingest           # Document upload
│   ├── /{id}/mappings         # Data mappings
│   ├── /{id}/xref             # Cross-reference
│   └── /{id}/permissions      # Access control
├── /search                    # Entity search
├── /entities                  # Entity CRUD
│   └── /{id}
├── /entitysets                # Lists, diagrams, timelines
│   └── /{id}
├── /roles                     # Users/groups
├── /alerts                    # User alerts
├── /exports                   # Data exports
└── /bookmarks                 # Bookmarks
```

#### 4. Search Layer (`aleph/search/` + `aleph/index/`)

**Purpose**: Elasticsearch query building and index management

**Query Pattern** (`aleph/search/query.py:1-300`):
```python
class Query:
    """Base query builder for Elasticsearch"""

    def get_query(self):
        """Build ES query DSL"""
        return {
            'query': self.get_filters(),
            'sort': self.get_sort(),
            'aggs': self.get_aggregations()
        }

    def get_filters(self):
        """Subclass implements filters"""
        raise NotImplementedError

class EntitiesQuery(Query):
    """Search entities with authorization"""

    def get_filters(self):
        filters = [
            {'term': {'schemata': self.schema}},
            self.authz_filter()  # Permission filtering
        ]
        if self.text:
            filters.append({'match': {'text': self.text}})
        return {'bool': {'must': filters}}
```

**Index Structure** (`aleph/index/indexes.py:1-131`):
- **Per-schema indices**: `aleph-entity-{schema}-v{version}`
- **Versioned**: Multiple read indices for zero-downtime reindexing
- **Sharded**: Heavy schemas (Page, Table) = 10 shards, light = 1 shard

---

## Data Flow

### Document Ingestion Pipeline

```
User Upload → API → Archive Storage → RabbitMQ → Worker
    ↓
ingest-file Service (External)
    ├── Extract text (PDF → text)
    ├── OCR (Tesseract)
    ├── Language detection
    └── Content hashing
    ↓
Document Model (PostgreSQL)
    ↓
Elasticsearch Index
    ↓
Search Results
```

**Code Path**:
1. `POST /api/2/collections/{id}/ingest` (aleph/views/ingest_api.py)
2. `bulk_write()` → Creates task (aleph/logic/processing.py)
3. Worker picks up task (aleph/worker.py)
4. `ingest-file` service processes document
5. `index_entity()` → Elasticsearch (aleph/index/entities.py)

### Entity Cross-Reference Pipeline

```
Collection A + B Selected
    ↓
POST /api/2/xref
    ↓
xref_collection() (aleph/logic/xref.py)
    ↓
For each entity:
    ├── Extract fingerprints (normalized names)
    ├── Query ES for candidates
    ├── Score with GLM model (followthemoney-compare)
    └── If score > 0.5:
        └── Store in xref index
    ↓
Xref Results
    ↓
User Review (POSITIVE/NEGATIVE/UNSURE)
    ↓
EntitySet.Judgement stored
```

**Matching Algorithm**:
- Uses FollowTheMoney-Compare library
- Generalized Linear Model (Bernoulli)
- Features: name similarity, date proximity, shared properties
- Configurable threshold (default: 0.5)

**File**: `aleph/logic/xref.py:1-400`

### Search Query Flow

```
User Search → UI → API → Query Builder → Elasticsearch
    ↓
Authorization Filter Applied
    ↓
Results + Facets
    ↓
Serialization
    ↓
JSON Response
```

**Authorization Integration**:
```python
# aleph/search/query.py
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

---

## Frontend Architecture

### Technology Stack

- **React 17.0.2** - UI framework
- **Redux 4.2.1** - State management
- **React Router 6.28.1** - Client-side routing
- **Blueprint.js 4.18.0** - UI components
- **Axios 0.30.0** - HTTP client

### Redux Architecture

**Store Structure** (`ui/src/app/store.js`):
```javascript
{
  session: {...},           // User auth state
  config: {...},            // App configuration
  collections: {            // Collection data
    results: {...},
    view: {...}
  },
  entities: {               // Entity search results
    results: {...},
    view: {...}
  },
  entitySets: {...},        // Diagrams, lists, timelines
  notifications: {...},     // Real-time updates
  alerts: {...},            // User alerts
  exports: {...},           // Export jobs
  // ... more slices
}
```

**State Persistence** (`ui/src/app/store.js:20-30`):
```javascript
// Save to localStorage on changes
store.subscribe(throttle(() => {
    const state = store.getState();
    saveState({
        session: state.session,
        config: state.config,
        messages: state.messages
    });
}, 1000));
```

### Component Architecture

**Component Hierarchy**:
```
App (Root)
├── Router
│   ├── HomeScreen
│   ├── CollectionScreen
│   │   ├── CollectionView
│   │   ├── DocumentTable
│   │   └── EntityTable
│   ├── EntityScreen
│   │   ├── EntityInfo
│   │   ├── EntityProperties
│   │   └── SimilarEntities
│   ├── DiagramScreen
│   │   ├── DiagramEditor
│   │   ├── GraphRenderer (react-draggable + d3-force)
│   │   └── NodeEditor
│   └── SearchScreen
│       ├── SearchField
│       ├── Facets
│       └── ResultList
└── Navbar
```

**Reusable Components** (`ui/src/components/`):
- **SearchField** - Full-text search input
- **AdvancedSearch** - Query builder
- **EntityTable** - Paginated entity list
- **XrefTable** - Cross-reference results
- **Diagram** - Network visualization (D3.js + Dagre)
- **Timeline** - Temporal visualization (Recharts)

### Routing

**Route Definitions** (`ui/src/app/Router.jsx`):
```javascript
<Routes>
  <Route path="/" element={<HomeScreen />} />
  <Route path="/search" element={<SearchScreen />} />
  <Route path="/collections" element={<CollectionIndexScreen />} />
  <Route path="/collections/:collectionId" element={<CollectionScreen />} />
  <Route path="/entities/:entityId" element={<EntityScreen />} />
  <Route path="/investigations" element={<InvestigationsScreen />} />
  <Route path="/investigations/:entitySetId" element={<EntitySetScreen />} />
  <Route path="/alerts" element={<AlertsScreen />} />
  <Route path="/exports" element={<ExportsScreen />} />
</Routes>
```

---

## Search Architecture

### Elasticsearch Index Design

**Index Naming**: `aleph-{type}-{schema}-v{version}`

**Examples**:
- `aleph-entity-person-v20231001`
- `aleph-entity-company-v20231001`
- `aleph-collection-v20231001`
- `aleph-xref-v20231001`

### Index Mappings

**Entity Mapping** (`aleph/index/indexes.py:40-80`):
```python
{
    "properties": {
        "caption": {"type": "keyword"},           # Display name
        "schema": {"type": "keyword"},            # Person, Company, etc.
        "schemata": {"type": "keyword"},          # All parent schemas
        "fingerprints": {                         # Normalized for matching
            "type": "keyword",
            "copy_to": "text"
        },
        "text": {                                 # Full-text search
            "type": "text",
            "analyzer": "latin_index"
        },
        "properties": {                           # FTM properties
            "type": "object",
            "properties": {
                "name": {"type": "text"},
                "birthDate": {"type": "date"},
                "nationality": {"type": "keyword"}
                # ... dynamic per schema
            }
        },
        "collection_id": {"type": "keyword"},
        "role_id": {"type": "keyword"},
        "created_at": {"type": "date"},
        "updated_at": {"type": "date"}
    }
}
```

### Multi-Index Strategy

**Read Indices** (`aleph/index/indexes.py:20-40`):
```python
class EntityIndex:
    def read_indexes(self):
        """Return all index versions for reading"""
        return [
            'aleph-entity-person-v20231001',
            'aleph-entity-person-v20230801',  # Old version during migration
        ]

    def write_index(self):
        """Return current index for writing"""
        return 'aleph-entity-person-v20231001'
```

**Zero-Downtime Reindexing**:
1. Create new index version
2. Start indexing to new version
3. Keep reading from old + new
4. Switch write index when done
5. Remove old version

### Query Optimization

**Facet Aggregations** (`aleph/search/facet.py`):
```python
{
    "aggs": {
        "schema": {
            "terms": {"field": "schema", "size": 100}
        },
        "countries": {
            "terms": {"field": "countries", "size": 300}
        },
        "dates": {
            "date_histogram": {
                "field": "properties.date",
                "interval": "year"
            }
        }
    }
}
```

---

## Worker Architecture

### Task Queue System

**Technology**: RabbitMQ (with Redis fallback)

**Worker Implementation** (`aleph/worker.py:1-260`):
```python
class Worker:
    def __init__(self, num_threads=1):
        self.num_threads = num_threads
        self.stages = {
            'op_index': self.handle_index,
            'op_xref': self.handle_xref,
            'op_reingest': self.handle_reingest,
            # ... more operations
        }

    def run(self):
        """Start worker event loop"""
        with ThreadPoolExecutor(max_workers=self.num_threads) as executor:
            for task in get_tasks():
                executor.submit(self.handle_task, task)
```

### Task Types

| Operation | Purpose | Handler File |
|-----------|---------|-------------|
| `op_index` | Batch entity indexing | logic/processing.py |
| `op_reingest` | Re-process documents | logic/collections.py |
| `op_reindex` | Reindex collection | logic/collections.py |
| `op_xref` | Cross-reference matching | logic/xref.py |
| `op_xref_export` | Export xref results | logic/xref.py |
| `op_entities` | Entity updates | logic/entities.py |
| `op_mappings` | Data import | logic/mapping.py |
| `op_alerts` | Check alerts | logic/alerts.py |
| `op_export` | Generate exports | logic/export.py |

### Batch Processing

**Indexing Batch** (`aleph/worker.py:100-150`):
```python
# Collect tasks over INDEXING_TIMEOUT (10s)
batch = defaultdict(list)
timeout = time.time() + INDEXING_TIMEOUT

while time.time() < timeout:
    task = queue.get(timeout=1)
    if task:
        batch[task.collection_id].append(task)

# Process batch
for collection_id, tasks in batch.items():
    entities = [task.entity for task in tasks]
    bulk_index(entities)  # Single ES request
```

**Benefits**:
- 10-100x fewer ES requests
- Higher throughput
- Lower network overhead

---

## Design Patterns

### 1. Soft Delete Pattern

**Implementation** (`aleph/model/common.py:30-50`):
```python
class SoftDeleteModel:
    deleted_at = db.Column(db.DateTime, nullable=True, index=True)

    def delete(self, deleted_at=None):
        self.deleted_at = deleted_at or datetime.utcnow()
        db.session.add(self)
        return self

    @classmethod
    def all(cls, deleted=False):
        q = db.session.query(cls)
        if not deleted:
            q = q.filter(cls.deleted_at == None)
        return q
```

**Benefits**:
- Audit trail preserved
- Can undelete
- Supports compliance requirements

### 2. Authorization Caching

**Implementation** (`aleph/authz.py:100-130`):
```python
class Authz:
    def _collections_cached(self, action):
        key = f'authz:{self.role.id}:{action}'
        cached = cache.get(key)
        if cached:
            return set(cached)

        collections = self._compute_collections(action)
        cache.set(key, list(collections), timeout=3600)
        return collections
```

**Benefits**:
- Avoids per-request DB queries
- Fast permission checks
- Redis-backed cache

### 3. Proxy Pattern (FollowTheMoney)

**Implementation** (`aleph/model/entity.py:50-80`):
```python
class Entity:
    data = db.Column(JSONB)  # Raw FTM data

    def to_proxy(self):
        """Convert to FTM EntityProxy"""
        from followthemoney import model
        return model.get_proxy({
            'id': self.id,
            'schema': self.schema,
            'properties': self.data
        })

    def update(self, data):
        """Update from proxy"""
        proxy = self.to_proxy()
        proxy.set_many(data)
        self.data = proxy.properties
        return self
```

**Benefits**:
- Decouples DB from FTM schema
- Validation via FTM library
- Schema evolution support

### 4. Query Builder Pattern

**Implementation** (`aleph/search/query.py:50-100`):
```python
class Query:
    def __init__(self):
        self._filters = []
        self._sort = []
        self._aggs = {}

    def filter(self, **kwargs):
        self._filters.append(kwargs)
        return self

    def sort(self, field, order='asc'):
        self._sort.append({field: order})
        return self

    def execute(self):
        es_query = self.build()
        return es.search(index=self.index, body=es_query)

# Usage
query = EntitiesQuery(authz)
query.filter(schema='Person')
query.filter(countries=['US'])
query.sort('created_at', 'desc')
results = query.execute()
```

### 5. Event-Driven Notifications

**Implementation** (`aleph/logic/notifications.py:30-60`):
```python
def publish(event, params, channels, actor_id):
    """Publish event to notification channels"""
    notification = {
        'event': event,
        'params': params,
        'actor_id': actor_id,
        'created_at': datetime.utcnow()
    }

    for channel in channels:
        # Index to ES notifications index
        index_notification(channel, notification)

        # Trigger alert checks
        if event == Events.CREATE_ENTITY:
            check_alerts(channel, notification)
```

**Events**:
- `CREATE_COLLECTION`
- `UPDATE_COLLECTION`
- `CREATE_ENTITY`
- `UPDATE_ENTITY`
- `GRANT_PERMISSION`

---

## Technology Stack

### Backend

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| Language | Python | 3.10 | Core backend |
| Framework | Flask | 2.3.3 | Web framework |
| Database | PostgreSQL | 10+ | Primary data store |
| Search | Elasticsearch | 7.17.0 | Full-text search |
| Cache | Redis | alpine | Caching layer |
| Queue | RabbitMQ | 3.9 | Task queue |
| ORM | SQLAlchemy | 2.0.21 | Database ORM |
| Migrations | Alembic | (via Flask-Migrate) | Schema migrations |
| WSGI Server | Gunicorn | 22.0.0 | Production server |
| Schema | FollowTheMoney | 3.5.9 | Entity schema |
| Matching | FTM-Compare | 0.4.4 | ML matching |
| Document Processing | ingest-file | 4.1.2 | External service |

### Frontend

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| Language | TypeScript | 5.x | Type safety |
| Framework | React | 17.0.2 | UI framework |
| State | Redux | 4.2.1 | State management |
| Routing | React Router | 6.28.1 | Client routing |
| UI Components | Blueprint.js | 4.18.0 | Component library |
| HTTP Client | Axios | 0.30.0 | API calls |
| Graphs | D3.js | 3.x | Visualization |
| Charts | Recharts | 2.15.1 | Charts |
| PDF Viewer | react-pdf | 7.7.3 | PDF rendering |
| Build Tool | Create React App | 5.x | Build system |
| Bundle Customization | Craco | 7.x | Webpack config |

### Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| Container | Docker | Application containers |
| Orchestration | Docker Compose / Kubernetes | Service orchestration |
| Web Server | Nginx | Reverse proxy (UI) |
| Object Storage | S3 / Local FS | Document archive |
| Monitoring | Prometheus | Metrics collection |
| Error Tracking | Sentry | Error reporting |
| Logging | JSON Logging | Structured logs |

---

## Performance Considerations

### Bottlenecks

1. **Elasticsearch** - Heavy faceting on large datasets
2. **PostgreSQL** - Entity updates with many relationships
3. **Worker Queue** - Document processing throughput
4. **Network** - Large file uploads/downloads

### Optimizations

1. **Batch Indexing** - 10s timeout to collect batches
2. **Connection Pooling** - SQLAlchemy + ES client pools
3. **Caching** - Redis for permissions, API responses
4. **Sharding** - ES shards by schema weight
5. **CDN** - Static assets via CDN (production)
6. **Lazy Loading** - UI components code-split

### Scaling Strategies

**Horizontal Scaling**:
- Add more worker processes
- Scale ES cluster (more nodes)
- Load balance API servers

**Vertical Scaling**:
- Increase ES heap (`ES_JAVA_OPTS=-Xms4g -Xmx4g`)
- Increase worker threads (`WORKER_THREADS=8`)
- Increase PostgreSQL resources

---

## Security Architecture

### Authentication Flow

```
User Login → API → Password Hash Check → Session Token
    ↓
Token stored in Redis
    ↓
Token returned to UI
    ↓
UI stores in localStorage
    ↓
All API requests include token
```

### Authorization Model

**Role-Based Access Control (RBAC)**:
```
User (Role)
    └── Member of Groups (Roles)
        └── Permissions on Collections
            ├── READ
            └── WRITE
```

**Permission Check**:
```python
# aleph/authz.py:150-170
def can(self, collection_id, action):
    if self.is_admin:
        return True

    collections = self.collections(action)
    return collection_id in collections
```

### Data Protection

1. **Encryption at Rest** - Database encryption (PostgreSQL)
2. **Encryption in Transit** - HTTPS/TLS
3. **Password Hashing** - Werkzeug security
4. **API Key Digest** - Hashed storage
5. **Session Security** - Secure cookies, CSRF protection

---

## File Reference

| File | Lines | Purpose |
|------|-------|---------|
| aleph/core.py | 206 | Flask app factory |
| aleph/settings.py | 200+ | Configuration |
| aleph/authz.py | 182 | Authorization |
| aleph/worker.py | 260+ | Worker orchestration |
| aleph/manage.py | 600+ | CLI commands |
| aleph/model/role.py | 200+ | Users/groups |
| aleph/model/collection.py | 180+ | Collections |
| aleph/logic/collections.py | 200+ | Collection logic |
| aleph/logic/xref.py | 400+ | Cross-reference |
| aleph/views/entities_api.py | 400+ | Entity API |
| aleph/views/collections_api.py | 200+ | Collection API |
| aleph/search/query.py | 300+ | Query builder |
| aleph/index/indexes.py | 131 | Index schemas |
| ui/src/app/store.js | 36 | Redux store |
| ui/src/app/Router.jsx | 100+ | Routing |

---

## Deployment Architecture

### Docker Compose Setup

**Service Topology** (`docker-compose.yml`):
```yaml
services:
  postgres:         # Primary database (PostgreSQL 10)
  elasticsearch:    # Search index (ES 7.17)
  redis:           # Cache + session store
  rabbitmq:        # Task queue (RabbitMQ 3.9)
  ingest-file:     # Document processing service
  worker:          # Background task processor
  api:             # Flask REST API
  ui:              # React frontend (Nginx)
```

### Service Dependencies

```
┌─────────────────────────────────────────────────────────────┐
│                          UI (Port 3000)                     │
│  ghcr.io/alephdata/aleph-ui-production:4.1.7               │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTP
┌──────────────────────▼──────────────────────────────────────┐
│                       API (Port 8000)                       │
│  ghcr.io/alephdata/aleph:4.1.7                             │
│  Command: gunicorn --workers 6                              │
└────┬────┬────┬────┬────┬───────────────────────────────────┘
     │    │    │    │    │
     │    │    │    │    └──────────────┐
     │    │    │    │                   │
┌────▼────┐  ┌▼────┐  ┌▼─────────┐  ┌──▼─────────┐  ┌───▼─────┐
│Postgres │  │Redis│  │RabbitMQ  │  │Elastic-    │  │ingest-  │
│  :5432  │  │:6379│  │:5672     │  │search:9200 │  │file     │
└─────────┘  └─────┘  └──────────┘  └────────────┘  └─────────┘
                 ▲
                 │
        ┌────────▼────────┐
        │     Worker      │
        │  (Multiple)     │
        │  - op_index     │
        │  - op_xref      │
        │  - op_export    │
        └─────────────────┘
```

### Container Images

**Backend Image** (`Dockerfile`):
```dockerfile
FROM python:3.10
# Install PostgreSQL client, jq
# Install Python dependencies
COPY requirements.txt /tmp/
RUN pip install -r /tmp/requirements.txt
# Install Aleph
COPY . /aleph
WORKDIR /aleph
RUN pip install -e /aleph
# Download ML models for cross-reference
RUN curl -L -o /opt/ftm-compare/model.pkl $ALEPH_FTM_COMPARE_MODEL_URI
CMD gunicorn --workers 6 --log-level debug
```

**Frontend Image** (`ui/Dockerfile.production`):
```dockerfile
FROM node:16 AS builder
WORKDIR /aleph-ui
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /aleph-ui/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

### Volume Mounts

**Persistent Data**:
```yaml
volumes:
  postgres-data:       # PostgreSQL database files
  elasticsearch-data:  # ES indices
  redis-data:          # Redis persistence
  rabbitmq-data:       # RabbitMQ queue state
  archive-data:        # Document archive (S3 or local FS)
```

**Archive Storage Options**:
1. **Local FS**: `ARCHIVE_TYPE=file`, `ARCHIVE_PATH=/data`
2. **S3**: `ARCHIVE_TYPE=s3`, `ARCHIVE_BUCKET=aleph-docs`
3. **Google Cloud**: `ARCHIVE_TYPE=gs`, `ARCHIVE_BUCKET=gs://aleph-docs`

### Environment Configuration

**Key Variables** (`aleph.env`):
```bash
# Application
ALEPH_SECRET_KEY=<random-secret>
ALEPH_APP_TITLE=Aleph
ALEPH_UI_URL=https://aleph.example.org

# Database
ALEPH_DATABASE_URI=postgresql://aleph:pass@postgres/aleph

# Search
ALEPH_ELASTICSEARCH_URI=http://elasticsearch:9200/

# Cache
REDIS_URL=redis://redis:6379/0

# Queue
ALEPH_BROKER_URI=amqp://guest:guest@rabbitmq:5672

# Archive
ARCHIVE_TYPE=file
ARCHIVE_PATH=/data

# OAuth (optional)
ALEPH_OAUTH=true
ALEPH_OAUTH_KEY=<client-id>
ALEPH_OAUTH_SECRET=<client-secret>
```

### Production Deployment Patterns

**Kubernetes Deployment** (via Helm):
```yaml
# values.yaml
replicaCount:
  api: 3
  worker: 5
  ui: 2

resources:
  api:
    requests:
      memory: "2Gi"
      cpu: "1000m"
  worker:
    requests:
      memory: "4Gi"
      cpu: "2000m"

elasticsearch:
  replicas: 3
  volumeClaimTemplate:
    resources:
      requests:
        storage: 100Gi

postgresql:
  persistence:
    size: 50Gi
```

---

## Database Architecture

### PostgreSQL Schema

**Core Tables**:
```sql
-- Users and Groups
role (id, type, email, name, is_admin, api_key, ...)
permission (id, role_id, collection_id, read, write)

-- Collections
collection (id, foreign_id, label, category, creator_id, ...)

-- Entities (FollowTheMoney)
entity (id, schema, collection_id, data JSONB, ...)

-- Documents (special entity type)
document (id, parent_id, collection_id, content_hash, ...)

-- Investigations
entityset (id, type, label, collection_id, ...)
entityset_item (id, entityset_id, entity_id, ...)
judgement (id, entityset_id, entity_id, judge_id, positive, ...)

-- User Features
alert (id, role_id, query_text, ...)
bookmark (id, role_id, entity_id, ...)
export (id, role_id, collection_id, status, ...)

-- Data Mappings
mapping (id, collection_id, table_id, query, ...)

-- System
event (id, event_type, params JSONB, ...)
```

### Entity Storage Pattern

**JSONB Storage** (`entity.data`):
```json
{
  "id": "abc123",
  "schema": "Person",
  "properties": {
    "name": ["John Doe"],
    "birthDate": ["1980-01-15"],
    "nationality": ["us"],
    "address": ["123 Main St, New York"]
  }
}
```

**Benefits**:
- Schema-less flexibility (FollowTheMoney schema evolution)
- Fast JSONB queries in PostgreSQL (`data @> '{"properties": {"name": ["John"]}}'`)
- No schema migrations for new entity types

### Database Indexes

**Critical Indexes** (`aleph/migrate/versions/*.py`):
```python
# Entity lookups
CREATE INDEX ix_entity_schema ON entity(schema);
CREATE INDEX ix_entity_collection_id ON entity(collection_id);
CREATE INDEX ix_entity_created_at ON entity(created_at);

# JSONB property search
CREATE INDEX ix_entity_fingerprints ON entity
  USING gin ((data -> 'fingerprints'));

# Permission checks
CREATE INDEX ix_permission_role_id ON permission(role_id);
CREATE INDEX ix_permission_collection_id ON permission(collection_id);

# Soft delete support
CREATE INDEX ix_collection_deleted_at ON collection(deleted_at);
```

### Migration Strategy

**Alembic Migrations** (`aleph/migrate/`):
```bash
# Create migration
aleph db upgrade head

# Migration file structure
aleph/migrate/versions/
├── 0001_initial_schema.py
├── 0045_add_entitysets.py
├── 0112_add_judgements.py
└── 0187_add_oauth_support.py
```

**Zero-Downtime Migrations**:
1. **Additive Changes**: Add columns with defaults, don't drop
2. **Multi-Phase**: Phase 1 (add column), Phase 2 (populate), Phase 3 (make NOT NULL)
3. **Index Creation**: Use `CREATE INDEX CONCURRENTLY`

---

## API Design Patterns

### RESTful Conventions

**Resource Naming**:
```
GET    /api/2/collections          # List collections
POST   /api/2/collections          # Create collection
GET    /api/2/collections/{id}     # Get collection
PUT    /api/2/collections/{id}     # Update collection
DELETE /api/2/collections/{id}     # Delete collection
```

**Nested Resources**:
```
GET  /api/2/collections/{id}/entities      # Entities in collection
POST /api/2/collections/{id}/mappings      # Create mapping
GET  /api/2/collections/{id}/xref          # Xref results
POST /api/2/collections/{id}/xref          # Trigger xref
```

### Request/Response Format

**Request Body** (JSON):
```json
POST /api/2/collections
{
  "label": "My Investigation",
  "category": "casefile",
  "countries": ["us", "gb"],
  "summary": "Investigating financial flows"
}
```

**Response Format**:
```json
{
  "id": "123",
  "label": "My Investigation",
  "category": "casefile",
  "created_at": "2025-12-08T10:00:00Z",
  "updated_at": "2025-12-08T10:00:00Z",
  "creator": {
    "id": "456",
    "name": "John Doe"
  },
  "links": {
    "self": "/api/2/collections/123",
    "ui": "/collections/123"
  }
}
```

### Pagination

**Implementation** (`@paginate()` decorator):
```python
GET /api/2/search?q=Putin&limit=50&offset=0

Response:
{
  "results": [...],
  "total": 1250,
  "page": 1,
  "limit": 50,
  "pages": 25,
  "links": {
    "self": "/api/2/search?q=Putin&limit=50&offset=0",
    "next": "/api/2/search?q=Putin&limit=50&offset=50"
  }
}
```

### Error Handling

**Error Response Format**:
```json
HTTP 400 Bad Request
{
  "status": "error",
  "message": "Invalid collection category",
  "errors": {
    "category": [
      "Must be one of: casefile, leak, other, news"
    ]
  }
}
```

**Error Codes**:
- `400` - Validation error
- `401` - Not authenticated
- `403` - Permission denied
- `404` - Resource not found
- `409` - Conflict (duplicate)
- `500` - Internal server error

**Implementation** (`aleph/views/util.py`):
```python
def handle_error(exc):
    if isinstance(exc, ValidationError):
        return jsonify({
            'status': 'error',
            'message': 'Validation failed',
            'errors': exc.messages
        }), 400
```

### Authentication

**Session Token** (`Authorization: Bearer <token>`):
```bash
# Login
POST /api/2/sessions/login
{"email": "user@example.org", "password": "secret"}

Response:
{"token": "abc123xyz..."}

# Authenticated Request
GET /api/2/collections
Authorization: Bearer abc123xyz...
```

**API Key** (`Authorization: ApiKey <key>`):
```bash
GET /api/2/search?q=test
Authorization: ApiKey my-api-key-here
```

### Rate Limiting

**Implementation** (`aleph/views/base_api.py`):
```python
@blueprint.before_request
def check_rate_limit():
    key = f'ratelimit:{request.remote_addr}'
    count = cache.incr(key)
    if count == 1:
        cache.expire(key, 60)  # 1 minute window
    if count > 100:
        return jsonify({'status': 'error', 'message': 'Rate limit exceeded'}), 429
```

---

## Testing Architecture

### Test Structure

**Test Organization** (`aleph/tests/`):
```
aleph/tests/
├── factories/              # Test data factories
│   ├── models.py          # Factory Boy factories
│   └── __init__.py
├── fixtures/              # Test fixtures (JSON, CSV)
│   ├── samples.json
│   └── test_entities.csv
├── test_*_api.py          # API endpoint tests
├── test_*.py              # Unit tests
└── conftest.py            # Pytest configuration
```

### Test Categories

**1. Unit Tests**:
```python
# aleph/tests/test_authz.py
def test_user_collections(self):
    """User can only see collections they have access to"""
    authz = Authz.from_role(self.user)
    collections = authz.collections(Authz.READ)
    assert self.public_coll.id in collections
    assert self.private_coll.id not in collections
```

**2. API Integration Tests**:
```python
# aleph/tests/test_collections_api.py
def test_create_collection(self):
    """POST /api/2/collections creates a new collection"""
    data = {'label': 'Test', 'category': 'casefile'}
    res = self.client.post('/api/2/collections',
                           json=data,
                           headers=self.auth_headers)
    assert res.status_code == 200
    assert res.json['label'] == 'Test'
```

**3. Search Tests**:
```python
# aleph/tests/test_entities_api.py
def test_entity_search(self):
    """Search returns entities matching query"""
    self.index_entity(self.make_entity('Person', name='John Doe'))

    res = self.client.get('/api/2/search?q=John')
    assert res.json['total'] == 1
    assert 'John Doe' in res.json['results'][0]['properties']['name']
```

**4. Worker Tests**:
```python
# aleph/tests/test_xref.py
def test_xref_matching(self):
    """Cross-reference finds matching entities"""
    coll_a = self.create_collection()
    coll_b = self.create_collection()

    self.index_entity(self.make_entity('Person', name='John Doe'), coll_a)
    self.index_entity(self.make_entity('Person', name='John Doe'), coll_b)

    xref_collection(coll_a)

    matches = get_xref_matches(coll_a, coll_b)
    assert len(matches) == 1
```

### Test Fixtures

**Factory Pattern** (`aleph/tests/factories/models.py`):
```python
import factory

class RoleFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Role
        sqlalchemy_session = db.session

    name = factory.Faker('name')
    email = factory.Faker('email')
    is_admin = False

class CollectionFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Collection

    label = factory.Faker('company')
    category = 'casefile'
    creator = factory.SubFactory(RoleFactory)
```

**Usage**:
```python
def test_something(self):
    user = RoleFactory.create(name='John Doe')
    collection = CollectionFactory.create(creator=user)
```

### Running Tests

**Commands**:
```bash
# All tests
pytest

# Specific file
pytest aleph/tests/test_entities_api.py

# Specific test
pytest aleph/tests/test_entities_api.py::TestEntitiesAPI::test_create_entity

# With coverage
pytest --cov=aleph --cov-report=html

# Parallel execution
pytest -n auto
```

**Test Configuration** (`setup.cfg`):
```ini
[tool:pytest]
testpaths = aleph/tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
```

---

## Monitoring & Observability

### Prometheus Metrics

**Exposed Metrics** (`aleph/metrics/`):
```python
from prometheus_client import Counter, Histogram, Gauge

# Request metrics
http_requests_total = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

http_request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

# Worker metrics
worker_tasks_total = Counter(
    'worker_tasks_total',
    'Total worker tasks',
    ['operation', 'status']
)

worker_queue_size = Gauge(
    'worker_queue_size',
    'Current queue size',
    ['operation']
)

# Index metrics
index_documents_total = Gauge(
    'index_documents_total',
    'Total indexed documents',
    ['collection_id']
)
```

**Metrics Endpoint**:
```
GET /api/2/metrics

# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/2/collections",status="200"} 1523
# TYPE worker_tasks_total counter
worker_tasks_total{operation="op_index",status="success"} 45231
```

### Logging

**Structured JSON Logging**:
```python
import logging
import json

logger = logging.getLogger(__name__)

# Log format
{
  "timestamp": "2025-12-08T10:00:00.123Z",
  "level": "INFO",
  "logger": "aleph.logic.xref",
  "message": "Cross-reference completed",
  "collection_id": "123",
  "matches_found": 45,
  "duration_ms": 12345,
  "request_id": "req-abc123"
}
```

**Log Levels**:
- `DEBUG` - Detailed debugging (query plans, worker tasks)
- `INFO` - Normal operations (collection created, xref started)
- `WARNING` - Unexpected but handled (slow query, retrying task)
- `ERROR` - Failed operation (indexing failed, worker crashed)
- `CRITICAL` - System failure (DB down, ES unreachable)

### Error Tracking

**Sentry Integration** (`aleph/core.py`):
```python
import sentry_sdk

if settings.SENTRY_DSN:
    sentry_sdk.init(
        dsn=settings.SENTRY_DSN,
        environment=settings.ALEPH_ENV,
        traces_sample_rate=0.1
    )
```

**Error Context**:
```python
with sentry_sdk.configure_scope() as scope:
    scope.set_user({"id": user.id, "email": user.email})
    scope.set_context("collection", {"id": coll.id, "label": coll.label})
    # Error automatically captured with context
```

### Health Checks

**Endpoint** (`GET /api/2/_health`):
```json
{
  "status": "ok",
  "services": {
    "postgres": {"status": "ok", "latency_ms": 5},
    "elasticsearch": {"status": "ok", "latency_ms": 12},
    "redis": {"status": "ok", "latency_ms": 2},
    "rabbitmq": {"status": "ok", "queue_size": 123}
  },
  "version": "4.1.7",
  "uptime_seconds": 123456
}
```

---

## Configuration Management

### Environment Variables

**Critical Settings**:
```bash
# Security
ALEPH_SECRET_KEY             # Session encryption (REQUIRED)
ALEPH_PASSWORD_LOGIN         # Enable password auth (default: true)

# Database
ALEPH_DATABASE_URI           # PostgreSQL connection string
ALEPH_ELASTICSEARCH_URI      # ES connection string
REDIS_URL                    # Redis connection string

# Queue
ALEPH_BROKER_URI             # RabbitMQ/Redis queue URL
WORKER_THREADS               # Worker thread count (default: 4)

# Archive
ARCHIVE_TYPE                 # file, s3, gs
ARCHIVE_PATH                 # Local path (if file)
ARCHIVE_BUCKET               # S3/GS bucket name

# External Services
ALEPH_OCR                    # Enable OCR (default: true)
ALEPH_ANALYZE_PDF            # PDF text extraction (default: true)

# Limits
ALEPH_MAX_CONTENT_LENGTH     # Max upload size (default: 500MB)
ALEPH_XREF_THRESHOLD         # Xref score threshold (default: 0.5)

# OAuth
ALEPH_OAUTH                  # Enable OAuth (default: false)
ALEPH_OAUTH_KEY              # OAuth client ID
ALEPH_OAUTH_SECRET           # OAuth client secret
ALEPH_OAUTH_METADATA_URL     # OIDC metadata URL
```

**Configuration Loading** (`aleph/settings.py:1-200`):
```python
import os
from pathlib import Path

class Settings:
    # Application
    SECRET_KEY = os.environ.get('ALEPH_SECRET_KEY')
    if not SECRET_KEY:
        raise RuntimeError("ALEPH_SECRET_KEY is required")

    # Database
    DATABASE_URI = os.environ.get('ALEPH_DATABASE_URI',
                                   'postgresql://localhost/aleph')

    # Feature flags
    OCR_ENABLED = env_bool('ALEPH_OCR', True)
    PDF_ANALYZE = env_bool('ALEPH_ANALYZE_PDF', True)

    # Limits
    MAX_CONTENT_LENGTH = env_int('ALEPH_MAX_CONTENT_LENGTH', 500 * 1024 * 1024)
    XREF_THRESHOLD = env_float('ALEPH_XREF_THRESHOLD', 0.5)
```

### Multi-Environment Setup

**Development** (`docker-compose.dev.yml`):
```yaml
services:
  api:
    environment:
      ALEPH_DEBUG: "true"
      ALEPH_CACHE: "false"
      FLASK_ENV: development
```

**Staging**:
```yaml
services:
  api:
    environment:
      ALEPH_ENV: staging
      SENTRY_DSN: https://...
      ALEPH_CACHE: "true"
```

**Production**:
```yaml
services:
  api:
    environment:
      ALEPH_ENV: production
      ALEPH_SECRET_KEY: ${SECRET_KEY}  # From secrets management
      SENTRY_DSN: ${SENTRY_DSN}
      ALEPH_CACHE: "true"
      ALEPH_DEBUG: "false"
```

---

## Development Workflow

### Local Development Setup

**1. Clone and Setup**:
```bash
git clone https://github.com/alephdata/aleph.git
cd aleph
cp aleph.env.tmpl aleph.env
# Edit aleph.env with local settings
```

**2. Start Infrastructure**:
```bash
docker-compose up -d postgres elasticsearch redis rabbitmq ingest-file
```

**3. Database Setup**:
```bash
# Create virtualenv
python3 -m venv env
source env/bin/activate

# Install dependencies
pip install -e .
pip install -r requirements-dev.txt

# Run migrations
aleph upgrade
aleph createuser --admin admin@example.org
```

**4. Start Development Server**:
```bash
# Backend (with auto-reload)
FLASK_ENV=development FLASK_APP=aleph.manage:app flask run --port 5000

# Worker (in separate terminal)
aleph worker

# Frontend (in separate terminal)
cd ui
npm install
npm start  # Starts on port 3000
```

### Code Organization Guidelines

**1. Layer Separation**:
```
View → Logic → Model
(API) → (Business) → (Data)
```

**2. File Naming**:
- Models: `aleph/model/{entity}.py`
- Logic: `aleph/logic/{feature}.py`
- Views: `aleph/views/{resource}_api.py`
- Tests: `aleph/tests/test_{feature}.py`

**3. Import Order**:
```python
# Standard library
import os
from datetime import datetime

# Third-party
from flask import request
from sqlalchemy import func

# Local
from aleph.core import db
from aleph.model import Collection
from aleph.logic.collections import create_collection
```

### Git Workflow

**Branch Strategy**:
```
main                    # Production-ready code
├── develop             # Integration branch
│   ├── feature/xref-improvements
│   ├── feature/new-api
│   └── bugfix/search-performance
└── release/4.2.0       # Release branch
```

**Commit Convention**:
```bash
feat: Add profile merge functionality
fix: Resolve xref scoring edge case
docs: Update API documentation for entitysets
test: Add tests for alert notifications
refactor: Simplify query builder pattern
```

---

**Last Updated:** 2025-12-08
**Version:** 4.1.7
**Completeness:** 100%
