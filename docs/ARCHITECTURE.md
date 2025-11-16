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

**Last Updated:** 2025-11-16
**Version:** 4.1.7
