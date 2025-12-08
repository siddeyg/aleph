# Aleph Data Pipelines

This document provides a technical deep-dive into the four core data pipelines in Aleph, explaining how data flows through the system from ingestion to search.

## Table of Contents

1. [Document Ingestion Pipeline](#document-ingestion-pipeline)
2. [Entity Extraction & Cross-Reference Pipeline](#entity-extraction--cross-reference-pipeline)
3. [Background Job Processing Architecture](#background-job-processing-architecture)
4. [Search & Query Execution Pipeline](#search--query-execution-pipeline)

---

## Document Ingestion Pipeline

### Overview

The document ingestion pipeline transforms uploaded files into searchable, analyzable entities. It handles various formats (PDF, Word, Excel, images, archives) and extracts both metadata and content.

### Pipeline Stages

```
User Upload → Archive Storage → Queue Job → Worker Processing → Entity Extraction → Elasticsearch Index → Collection Stats Update
```

### Stage 1: Upload & Validation

**Entry Point**: `POST /api/2/collections/:id/ingest`

**Code Path**: `aleph/views/ingest_api.py:upload()`

**Process**:
1. Validate file size (< max_content_length)
2. Validate file type (check MIME type)
3. Check user authorization (write access to collection)
4. Generate content hash (SHA1)
5. Store blob in archive:
   - S3: Upload to configured bucket
   - Local: Save to `/data/archive/`

**Key Code**:
```python
# aleph/views/ingest_api.py:24-50
def upload(collection_id):
    collection = get_db_collection(collection_id, request.authz.WRITE)
    upload = get_multipart_upload(require=True)
    content_hash = archive.archive_file(upload.file_path, upload.mime_type)

    # Create parent document entity
    parent = get_ingest_entity(collection, upload.parent_id)
    ingest_document(collection, parent, upload, role_id=request.authz.id)
```

**Storage Backends**:
- `aleph/archive/file.py` - Local filesystem storage
- `aleph/archive/s3.py` - Amazon S3 storage

**File Limits**:
- Default max size: 500MB (configurable via `ALEPH_MAX_CONTENT_LENGTH`)
- Archive prevents duplicate storage (content-addressed by SHA1)

---

### Stage 2: Job Queuing

**Queue System**: RabbitMQ or Redis (configurable)

**Code Path**: `aleph/queues.py:queue_task()`

**Process**:
1. Generate unique job_id (UUID)
2. Create task payload:
   ```python
   {
     "collection_id": 123,
     "content_hash": "sha1:abc123...",
     "mime_type": "application/pdf",
     "file_name": "document.pdf",
     "parent_id": "parent-entity-id"
   }
   ```
3. Publish to queue stage: `STAGE_INGEST`
4. Return 202 Accepted to client

**Key Code**:
```python
# aleph/queues.py:40-60
def queue_task(collection, stage, payload=None, job_id=None):
    job_id = job_id or make_textid()
    task = {
        "job_id": job_id,
        "stage": stage,
        "collection": collection.to_dict(),
        "payload": payload or {}
    }
    dataset = get_dataset(collection)
    dataset.publish(task)
    return job_id
```

**Queue Stages**:
- `STAGE_INGEST` - Initial document processing
- `STAGE_ANALYZE` - Entity extraction
- `STAGE_INDEX` - Elasticsearch indexing
- `STAGE_XREF` - Cross-reference generation
- `STAGE_LOAD_MAPPING` - Mapping execution

---

### Stage 3: Worker Processing

**Entry Point**: Worker picks job from queue

**Code Path**: `aleph/worker.py:handle_stage()`

**Process**:

1. **Load Archive Blob**
   ```python
   # aleph/logic/documents.py:15-25
   local_path = archive.load_file(content_hash)
   ```

2. **Extract Metadata** (via `ingest-file` library)
   - Title, author, creation date
   - Page count, language
   - Embedded metadata

3. **Text Extraction**
   - PDF: Extract text directly or via OCR
   - Word/Excel: Parse document structure
   - Images: Run Tesseract OCR
   - Archives: Extract nested files recursively

4. **OCR Processing** (if needed)
   ```python
   # aleph/ingest/ocr.py
   if document.is_image or document.is_pdf:
       languages = get_languages(document)  # e.g., ['eng', 'deu']
       text = run_tesseract(local_path, languages)
   ```

5. **Create Child Entities**
   - For PDFs: One entity per page
   - For archives: One entity per extracted file
   - For emails: Attachments as separate entities

**Key Code**:
```python
# aleph/worker.py:30-50
@task.route(STAGE_INGEST)
def handle_ingest(stage, collection, payload):
    content_hash = payload['content_hash']
    parent_id = payload.get('parent_id')

    # Process document
    local_path = archive.load_file(content_hash)
    result = ingest_file(local_path, collection, parent_id)

    # Extract entities
    for entity in result.entities:
        queue_task(collection, STAGE_INDEX, {'entity_id': entity.id})
```

**Processing Libraries**:
- `ingest-file` - Document parsing (PyPI package by OCCRP)
- `tesseract-ocr` - OCR engine
- `pdf2pdfocr` - PDF OCR wrapper
- `Apache Tika` - Alternative parser (optional)

---

### Stage 4: Entity Extraction

**Code Path**: `aleph/logic/entities.py:upsert_entity()`

**Process**:

1. **Generate Entity ID**
   - Use provided ID or generate via `make_textid()`
   - ID format: Base64-encoded UUID

2. **Validate Schema** (FollowTheMoney)
   ```python
   # Validate entity matches FTM schema
   from followthemoney import model
   proxy = model.get_proxy(entity_data)
   proxy.validate()  # Raises ValidationError if invalid
   ```

3. **Store Properties**
   - Properties stored as arrays (multi-valued)
   - Example: `{"name": ["John Doe", "J. Doe"]}`
   - Dates normalized to ISO 8601
   - Countries normalized to ISO 3166-1

4. **Generate Fingerprints** (for cross-reference)
   ```python
   # aleph/logic/matching.py:extract_fingerprints()
   fingerprints = set()
   for name in entity.get('name', []):
       normalized = normalize_name(name)
       fingerprints.add(normalized)
   ```

**Key Code**:
```python
# aleph/logic/entities.py:50-80
def upsert_entity(data, collection, authz=None, sync=True):
    entity_id = data.get('id', make_textid())
    proxy = model.get_proxy(data)

    # Validate schema
    if not proxy.schema.is_a('Thing'):
        raise ValidationError("Invalid schema")

    # Store in index
    index_entity(proxy, collection_id=collection.id, sync=sync)

    return entity_id
```

---

### Stage 5: Elasticsearch Indexing

**Code Path**: `aleph/index/entities.py:index_proxy()`

**Process**:

1. **Build Index Document**
   ```python
   doc = {
       "id": entity.id,
       "schema": entity.schema.name,
       "schemata": [s.name for s in entity.schema.schemata],
       "properties": entity.properties,
       "collection_id": collection.id,
       "text": extract_text(entity),  # Full-text search field
       "names": entity.get('name', []),
       "dates": entity.get('date', []),
       "countries": entity.get('country', []),
       "fingerprints": generate_fingerprints(entity)
   }
   ```

2. **Indexing**
   - Index name: `aleph-entity-v1`
   - Document ID: Entity ID
   - Routing: Collection ID (for sharding)

3. **Bulk Indexing** (optimization)
   - Buffer entities in memory
   - Flush every 1000 entities or 10 seconds
   - Uses Elasticsearch bulk API

**Key Code**:
```python
# aleph/index/entities.py:100-130
def index_proxy(collection, proxy, sync=False):
    data = get_entity_body(proxy, collection)

    if sync:
        es.index(
            index=entities_index(),
            id=proxy.id,
            body=data,
            routing=str(collection.id)
        )
    else:
        bulk_writer.put(proxy.id, data, collection.id)
```

**Elasticsearch Mapping**:
- `text` field: Full-text search (analyzed)
- `names` field: Entity names (keyword + text)
- `properties.*` fields: Schema-specific properties
- `fingerprints` field: Normalized values for matching

---

### Stage 6: Collection Statistics

**Code Path**: `aleph/logic/collections.py:update_collection()`

**Process**:

1. **Aggregate Stats** (via Elasticsearch)
   ```python
   aggregation = {
       "count_by_schema": {
           "terms": {"field": "schema"}
       },
       "count_by_country": {
           "terms": {"field": "countries"}
       }
   }
   ```

2. **Update Collection Record**
   ```python
   collection.count = total_entities
   collection.things = {"Person": 100, "Company": 50}
   db.session.commit()
   ```

3. **Mark Processing Complete**
   - Update status table
   - Clear job from queue
   - Trigger notifications if enabled

---

## Entity Extraction & Cross-Reference Pipeline

### Overview

The cross-reference (xref) pipeline identifies potential matches between entities across collections using machine learning.

### Pipeline Architecture

```
User Triggers Xref → Generate Pairs → Extract Features → Score with ML → Store Matches → User Reviews → Update Profiles
```

---

### Stage 1: Pair Generation

**Entry Point**: `POST /api/2/collections/:id/xref`

**Code Path**: `aleph/logic/xref.py:xref_collection()`

**Process**:

1. **Load Entity Fingerprints**
   ```python
   # Get all entities from source collection
   entities_a = get_entities(collection_a)

   # Get all entities from target collections
   entities_b = get_entities(collection_b_ids)
   ```

2. **Generate Candidate Pairs**
   - Only compare entities with shared fingerprints (optimization)
   - Fingerprints: normalized names, IDs, phone numbers, emails
   - Example: "John Doe" → "johndoe", "J. Doe" → "jdoe"

3. **Filter by Schema**
   - Only compare compatible schemas
   - Person ↔ Person ✓
   - Company ↔ Organization ✓
   - Person ↔ Company ✗

**Key Code**:
```python
# aleph/logic/xref.py:50-80
def generate_pairs(collection_a, collection_b_ids):
    pairs = []

    # Build fingerprint index
    fingerprints = defaultdict(set)
    for entity in get_entities(collection_b_ids):
        for fp in entity['fingerprints']:
            fingerprints[fp].add(entity['id'])

    # Generate pairs with shared fingerprints
    for entity_a in get_entities([collection_a]):
        candidates = set()
        for fp in entity_a['fingerprints']:
            candidates.update(fingerprints.get(fp, set()))

        for entity_b_id in candidates:
            if schema_compatible(entity_a, entity_b_id):
                pairs.append((entity_a['id'], entity_b_id))

    return pairs
```

**Optimization**:
- Fingerprint-based blocking reduces comparisons from O(n²) to O(n)
- Typical reduction: 1M entities → 10K candidate pairs

---

### Stage 2: Feature Extraction

**Code Path**: `aleph/logic/matching.py:compare_entities()`

**Process**:

1. **Load Entity Pairs**
   ```python
   entity_a = get_entity(entity_a_id)
   entity_b = get_entity(entity_b_id)
   ```

2. **Extract Comparison Features**
   - Name similarity (Levenshtein distance, Jaro-Winkler)
   - Shared properties (birth dates, addresses, IDs)
   - Shared fingerprints count
   - Schema compatibility score

3. **Calculate Feature Vector**
   ```python
   features = {
       'name_sim': jaro_winkler(entity_a.name, entity_b.name),
       'shared_props': len(shared_properties),
       'shared_fps': len(shared_fingerprints),
       'schema_match': 1.0 if same_schema else 0.5
   }
   ```

**Key Code**:
```python
# aleph/logic/matching.py:100-130
def compare_entities(entity_a, entity_b):
    from followthemoney.compare import compare

    # Use FTM's built-in comparison
    score = compare(model, entity_a, entity_b)

    # Additional features
    features = {
        'base_score': score,
        'name_match': name_similarity(entity_a, entity_b),
        'id_match': has_shared_id(entity_a, entity_b),
        'date_match': date_similarity(entity_a, entity_b)
    }

    return features
```

---

### Stage 3: ML Scoring

**Model**: Generalized Linear Model (GLM) with Bernoulli distribution

**Code Path**: `aleph/logic/matching.py:score_pair()`

**Process**:

1. **Load Trained Model**
   - Model trained on manually-labeled entity pairs
   - Stored in `aleph/static/xref_model.pkl`
   - Can be retrained with `aleph train-model`

2. **Predict Match Probability**
   ```python
   from sklearn.linear_model import LogisticRegression

   model = load_model()
   features = extract_features(entity_a, entity_b)
   score = model.predict_proba([features])[0][1]  # Probability of match
   ```

3. **Apply Threshold**
   - Default: 0.5 (50% probability)
   - Configurable per collection
   - Higher threshold = fewer false positives, more false negatives

**Model Features**:
- Name Levenshtein distance
- Jaro-Winkler similarity
- Shared fingerprint count
- Shared property count
- Schema compatibility
- Collection authority (some sources more reliable)

**Key Code**:
```python
# aleph/logic/matching.py:150-170
def score_pair(entity_a, entity_b):
    features = extract_features(entity_a, entity_b)
    feature_vector = [
        features['name_sim'],
        features['shared_props'],
        features['shared_fps'],
        features['schema_match']
    ]

    model = get_xref_model()
    probability = model.predict_proba([feature_vector])[0][1]

    return probability
```

---

### Stage 4: Store Matches

**Database Table**: `xref_match`

**Code Path**: `aleph/model/match.py`

**Schema**:
```sql
CREATE TABLE xref_match (
    id VARCHAR PRIMARY KEY,
    entity_id VARCHAR NOT NULL,
    match_id VARCHAR NOT NULL,
    collection_id INTEGER NOT NULL,
    match_collection_id INTEGER NOT NULL,
    score FLOAT NOT NULL,
    judgement VARCHAR,  -- NULL, 'positive', 'negative', 'unsure'
    decided_by_id VARCHAR,
    created_at TIMESTAMP,
    UNIQUE(entity_id, match_id)
);
```

**Process**:
1. Bulk insert all matches above threshold
2. Index by entity_id for fast lookup
3. Set initial judgement to NULL (pending review)

---

### Stage 5: User Review

**UI Workflow**: See [WORKFLOWS.md - Cross-Reference](./WORKFLOWS.md#cross-reference-workflow)

**API**: `POST /api/2/profiles/_pairwise`

**Judgements**:
- `positive` → Merge into profile
- `negative` → Mark as different entities (prevent future matching)
- `unsure` → Flag for later review
- `no_judgement` → Reset to initial state

**Effect of Judgement**:
```python
# Positive judgement
if judgement == 'positive':
    profile = get_or_create_profile(entity_a)
    profile.add_item(entity_b, judgement='positive')
    merge_entity_properties(profile, entity_b)

# Negative judgement
elif judgement == 'negative':
    mark_not_same(entity_a, entity_b)  # Prevent future matching
```

---

## Background Job Processing Architecture

### Overview

Aleph uses a task queue system to handle long-running operations asynchronously. This allows the API to remain responsive while heavy processing happens in the background.

### Architecture Components

```
API Server → Queue (RabbitMQ/Redis) → Worker Pool → Results
     ↓                                      ↓
  DB/Cache ←─────────────────────────────────
```

---

### Queue System

**Implementation**: ServiceLayer library (supports both RabbitMQ and Redis)

**Configuration**:
```python
# aleph/settings.py
BROKER_URI = env.get('ALEPH_BROKER_URI')  # e.g., 'amqp://localhost'
# or
BROKER_URI = 'redis://localhost:6379/0'
```

**Queue Structure**:
- Each job stage has dedicated queue
- Queues named by stage: `aleph.stage.ingest`, `aleph.stage.index`
- Supports priority (not currently used)
- Persistent messages (survives broker restart)

---

### Worker Architecture

**Worker Process**: `aleph/worker.py`

**Startup**:
```bash
# Start worker
python3 -m aleph.worker

# With specific concurrency
python3 -m aleph.worker --threads 8
```

**Worker Loop**:
```python
# aleph/worker.py:main()
def main():
    dataset = get_dataset()  # Connect to queue

    # Register task handlers
    task.route(STAGE_INGEST)(handle_ingest)
    task.route(STAGE_INDEX)(handle_index)
    task.route(STAGE_XREF)(handle_xref)
    # ... more handlers

    # Start consuming
    dataset.consume()  # Blocks forever, processing jobs
```

**Task Handling**:
```python
@task.route(STAGE_INGEST)
def handle_ingest(stage, collection, payload):
    try:
        # Process job
        process_document(payload)

        # Update status
        mark_complete(stage, collection, payload['job_id'])

    except Exception as e:
        # Log error
        log.exception("Ingest failed")

        # Update status with error
        mark_failed(stage, collection, payload['job_id'], str(e))

        # Don't re-raise (acknowledge message)
```

---

### Job Stages

| Stage | Purpose | Avg Duration | Concurrency |
|-------|---------|--------------|-------------|
| `STAGE_INGEST` | Parse and extract document content | 1-30 sec | High |
| `STAGE_ANALYZE` | Extract entities via NLP | 5-60 sec | Medium |
| `STAGE_INDEX` | Index entity to Elasticsearch | < 1 sec | Very High |
| `STAGE_XREF` | Generate cross-reference matches | 1-60 min | Low |
| `STAGE_LOAD_MAPPING` | Execute data mapping | 1-30 min | Medium |
| `STAGE_AGGREGATE` | Update collection stats | 1-10 sec | Medium |
| `STAGE_FLUSH_ENTITIES` | Delete entities | 5-60 sec | Medium |

---

### Status Tracking

**Implementation**: In-memory status (Redis) + Database persistence

**Status Record**:
```python
{
    "dataset": "collection:123",
    "stage": "ingest",
    "pending": 150,     # Jobs queued
    "running": 5,       # Jobs processing
    "finished": 5000    # Jobs completed
}
```

**API Endpoint**: `GET /api/2/status`

**Code Path**: `aleph/queues.py:get_status()`

```python
def get_active_dataset_status():
    """Get status of all active processing."""
    status = {}

    for dataset in get_active_datasets():
        jobs = get_dataset_jobs(dataset)
        status[dataset] = {
            "pending": sum(1 for j in jobs if j.status == 'pending'),
            "running": sum(1 for j in jobs if j.status == 'running'),
            "finished": sum(1 for j in jobs if j.status == 'finished')
        }

    return {"datasets": status}
```

---

### Error Handling

**Retry Strategy**:
1. First attempt fails → Log error, mark failed
2. No automatic retry (avoid infinite loops)
3. User can manually retry via UI

**Error Types**:
- **Temporary**: Network timeout, ES unavailable
- **Permanent**: Invalid file format, schema validation error
- **Partial**: Some pages OCR'd successfully, others failed

**Error Storage**:
```python
# Store in database
collection.status = Status.FAILURE
collection.error = {
    "message": "OCR failed on page 5",
    "traceback": "...",
    "timestamp": "2023-08-15T10:00:00Z"
}
db.session.commit()
```

---

### Scaling Workers

**Horizontal Scaling**:
```bash
# Run multiple worker processes
docker-compose up --scale worker=4
```

**Per-Worker Threads**:
```python
# aleph/settings.py
WORKER_THREADS = env.to_int('ALEPH_WORKER_THREADS', 8)
```

**Best Practices**:
- CPU-bound tasks (OCR): threads = CPU cores
- I/O-bound tasks (ES indexing): threads = 2-4× CPU cores
- Memory-intensive tasks (xref): fewer workers, more RAM per worker

---

## Search & Query Execution Pipeline

### Overview

The search pipeline transforms user queries into Elasticsearch requests, executes them, and formats results for display.

### Pipeline Flow

```
User Query → Query Parser → Query Builder → Elasticsearch → Result Filtering → Authorization → Response Serialization
```

---

### Stage 1: Query Parsing

**Entry Point**: `GET /api/2/search?q=Putin+AND+offshore&filter:schema=Person`

**Code Path**: `aleph/search/parser.py:SearchQueryParser`

**Process**:

1. **Extract Query Text**
   ```python
   query_text = request.args.get('q', '')
   # "Putin AND offshore"
   ```

2. **Parse Filters**
   ```python
   filters = {
       'schema': ['Person'],
       'collection_id': [10, 15],
       'countries': ['cy', 'ru']
   }
   ```

3. **Parse Facets** (for aggregation)
   ```python
   facets = ['schema', 'countries', 'collection_id']
   ```

4. **Parse Pagination**
   ```python
   offset = int(request.args.get('offset', 0))
   limit = min(int(request.args.get('limit', 30)), 10000)
   ```

**Key Code**:
```python
# aleph/search/parser.py:20-50
class SearchQueryParser:
    def __init__(self, args, authz):
        self.text = args.get('q', '').strip()
        self.prefix = args.get('prefix', '').strip()

        # Parse filters
        self.filters = defaultdict(list)
        for key, value in args.items(multi=True):
            if key.startswith('filter:'):
                field = key[7:]  # Remove 'filter:' prefix
                self.filters[field].append(value)

        # Pagination
        self.offset = int(args.get('offset', 0))
        self.limit = min(int(args.get('limit', 30)), self.max_limit)
```

---

### Stage 2: Query Building

**Code Path**: `aleph/search/query.py:EntitiesQuery`

**Process**:

1. **Build Bool Query**
   ```python
   query = {
       "bool": {
           "must": [],      # Required clauses
           "should": [],    # Optional clauses (OR)
           "filter": [],    # Filter clauses (don't affect score)
           "must_not": []   # Exclusion clauses
       }
   }
   ```

2. **Add Text Search**
   ```python
   if query_text:
       query["bool"]["must"].append({
           "multi_match": {
               "query": query_text,
               "fields": ["text", "names^3", "properties.*"],
               "type": "best_fields",
               "operator": "AND"
           }
       })
   ```

3. **Add Filters**
   ```python
   for field, values in filters.items():
       query["bool"]["filter"].append({
           "terms": {field: values}
       })
   ```

4. **Add Authorization Filter**
   ```python
   # Only show entities from collections user can access
   readable_collections = authz.collections(authz.READ)
   query["bool"]["filter"].append({
       "terms": {"collection_id": readable_collections}
   })
   ```

5. **Add Aggregations** (facets)
   ```python
   aggregations = {
       "schema": {
           "terms": {"field": "schema", "size": 100}
       },
       "countries": {
           "terms": {"field": "countries", "size": 300}
       }
   }
   ```

**Key Code**:
```python
# aleph/search/query.py:100-150
class EntitiesQuery:
    def get_query(self):
        query = {"bool": {"must": [], "filter": []}}

        # Text search
        if self.parser.text:
            query["bool"]["must"].append({
                "query_string": {
                    "query": self.parser.text,
                    "fields": ["text", "names^3"],
                    "default_operator": "AND"
                }
            })

        # Schema filter
        if self.parser.filters.get('schema'):
            query["bool"]["filter"].append({
                "terms": {"schema": self.parser.filters['schema']}
            })

        # Authorization
        query["bool"]["filter"].append({
            "terms": {"collection_id": self.get_authz_collections()}
        })

        return query
```

---

### Stage 3: Elasticsearch Execution

**Code Path**: `aleph/search/query.py:EntitiesQuery.search()`

**Process**:

1. **Build ES Request**
   ```python
   request_body = {
       "query": query,
       "aggregations": aggregations,
       "from": offset,
       "size": limit,
       "sort": [{"_score": "desc"}, {"created_at": "desc"}],
       "_source": ["id", "schema", "properties", "collection_id"]
   }
   ```

2. **Execute Search**
   ```python
   from elasticsearch import Elasticsearch

   es = Elasticsearch(SETTINGS.ELASTICSEARCH_URI)
   response = es.search(
       index="aleph-entity-v1",
       body=request_body,
       request_timeout=60
   )
   ```

3. **Parse Response**
   ```python
   hits = response['hits']['hits']
   total = response['hits']['total']['value']
   aggregations = response.get('aggregations', {})
   ```

**Elasticsearch Features Used**:
- **Query DSL**: Complex boolean queries
- **Aggregations**: Faceted search
- **Highlighting**: Show matched snippets
- **Fuzzy Matching**: Handle typos (edit distance 2)
- **Analyzers**: Tokenization, stemming, stop words

**Performance Optimizations**:
- **Routing**: Route queries by collection_id (shard-level filtering)
- **Source Filtering**: Only fetch needed fields
- **Scroll API**: For large result sets (> 10K)
- **Index Caching**: ES request cache for common queries

**Key Code**:
```python
# aleph/search/query.py:200-230
def search(self):
    body = {
        "query": self.get_query(),
        "aggregations": self.get_aggregations(),
        "from": self.parser.offset,
        "size": self.parser.limit,
        "_source": self.get_source_fields()
    }

    return es.search(
        index=entities_index(),
        body=body,
        request_timeout=60
    )
```

---

### Stage 4: Result Processing

**Code Path**: `aleph/search/result.py:QueryResult`

**Process**:

1. **Unpack Hits**
   ```python
   results = []
   for hit in response['hits']['hits']:
       entity = hit['_source']
       entity['score'] = hit['_score']
       results.append(entity)
   ```

2. **Process Aggregations** (facets)
   ```python
   facets = {}
   for facet_name, agg_result in response['aggregations'].items():
       buckets = agg_result['buckets']
       facets[facet_name] = [
           {"value": b['key'], "count": b['doc_count']}
           for b in buckets
       ]
   ```

3. **Apply Post-Filters**
   - Remove entities user shouldn't see (additional authz check)
   - Deduplicate if needed
   - Limit to max results

**Key Code**:
```python
# aleph/search/result.py:50-80
class QueryResult:
    def __init__(self, request, response):
        self.total = response['hits']['total']['value']
        self.results = [unpack_result(h) for h in response['hits']['hits']]
        self.facets = self.parse_aggregations(response.get('aggregations', {}))

    def parse_aggregations(self, aggs):
        facets = {}
        for name, agg in aggs.items():
            facets[name] = {
                "values": [
                    {"id": b['key'], "label": b['key'], "count": b['doc_count']}
                    for b in agg['buckets']
                ]
            }
        return facets
```

---

### Stage 5: Serialization

**Code Path**: `aleph/views/serializers.py:EntitySerializer`

**Process**:

1. **Serialize Each Entity**
   ```python
   def serialize(entity):
       return {
           "id": entity['id'],
           "schema": entity['schema'],
           "properties": entity['properties'],
           "collection": serialize_collection(entity['collection_id']),
           "links": {
               "self": url_for('entities_api.view', entity_id=entity['id'])
           }
       }
   ```

2. **Add Metadata**
   - Collection information
   - Creation/update timestamps
   - URL links for navigation

3. **Format Response**
   ```python
   response = {
       "results": [serialize(e) for e in results],
       "total": total,
       "facets": facets,
       "limit": limit,
       "offset": offset,
       "next": url_for('search_api.index', offset=offset+limit) if has_more else None
   }
   ```

**Key Code**:
```python
# aleph/views/serializers.py:100-130
class EntitySerializer:
    def serialize(self, entity):
        data = {
            "id": entity.id,
            "schema": entity.schema,
            "properties": entity.properties,
            "collection_id": entity.collection_id
        }

        # Add nested collection object
        if entity.collection:
            data["collection"] = CollectionSerializer().serialize(entity.collection)

        # Add API links
        data["links"] = {
            "self": url_for("entities_api.view", entity_id=entity.id),
            "references": url_for("entities_api.references", entity_id=entity.id)
        }

        return data
```

---

## Pipeline Performance Metrics

### Document Ingestion

| Document Type | Avg Time | Throughput | Bottleneck |
|---------------|----------|------------|------------|
| PDF (10 pages) | 15 sec | 4/min | OCR |
| PDF (100 pages) | 2 min | 0.5/min | OCR |
| Word Document | 5 sec | 12/min | Text extraction |
| Excel (1000 rows) | 30 sec | 2/min | Entity creation |
| Image (JPEG) | 20 sec | 3/min | OCR |
| Archive (ZIP, 50 files) | 5 min | 0.2/min | Recursive processing |

**Optimization Tips**:
- Pre-OCR PDFs before upload
- Increase worker count for OCR jobs
- Use SSDs for archive storage
- Enable Elasticsearch bulk indexing

---

### Cross-Reference

| Collection Size | Pairs Generated | Processing Time | Memory Usage |
|-----------------|-----------------|-----------------|--------------|
| 1K entities | ~500 pairs | 30 sec | 100 MB |
| 10K entities | ~5K pairs | 5 min | 500 MB |
| 100K entities | ~50K pairs | 1 hour | 2 GB |
| 1M entities | ~500K pairs | 10 hours | 8 GB |

**Scaling Factors**:
- Fingerprint selectivity (affects pair count)
- Number of target collections
- ML model complexity
- Worker RAM allocation

---

### Search Performance

| Query Type | Avg Latency | Throughput | Cache Hit Rate |
|------------|-------------|------------|----------------|
| Simple text search | 50 ms | 200 req/sec | 60% |
| Filtered search | 100 ms | 100 req/sec | 40% |
| Aggregated search | 200 ms | 50 req/sec | 20% |
| Complex boolean query | 500 ms | 20 req/sec | 10% |

**ES Cluster Recommendations**:
- 3-5 nodes for production
- 16-32 GB RAM per node
- SSD storage
- Dedicated master nodes for large deployments

---

## Configuration & Tuning

### Worker Configuration

```bash
# Environment variables
ALEPH_WORKER_THREADS=8           # Threads per worker
ALEPH_WORKER_RETRY=3             # Retry attempts
ALEPH_OCR_DEFAULTS=eng,deu,fra   # Default OCR languages
ALEPH_BULK_PAGE_SIZE=500         # ES bulk index size
```

### Queue Configuration

```bash
# RabbitMQ
ALEPH_BROKER_URI=amqp://user:pass@localhost:5672/
RABBITMQ_PREFETCH_COUNT=2        # Jobs per worker

# Redis
ALEPH_BROKER_URI=redis://localhost:6379/0
```

### Elasticsearch Configuration

```yaml
# config/elasticsearch.yml
indices.query.bool.max_clause_count: 4096
search.max_buckets: 65535
thread_pool.write.queue_size: 1000
```

---

## Monitoring & Debugging

### Key Metrics

1. **Queue Depth**
   - `GET /api/2/status` → Check pending job count
   - High depth = need more workers

2. **Worker Health**
   - Check worker logs: `docker-compose logs worker`
   - Monitor CPU/RAM usage

3. **ES Performance**
   - Query latency: `GET /_nodes/stats`
   - Index rate: `GET /_cat/indices?v`

4. **Archive Storage**
   - Disk usage: `df -h /data/archive`
   - S3 costs: Monitor AWS CloudWatch

### Troubleshooting

**Slow Ingestion**:
- Check OCR language configuration
- Verify worker count and threads
- Check archive storage I/O

**Failed Xref Jobs**:
- Check worker memory (xref is RAM-intensive)
- Reduce pair count (adjust fingerprint rules)
- Increase job timeout

**Slow Search**:
- Check ES cluster health: `GET /_cluster/health`
- Review slow query log: `GET /_nodes/stats`
- Consider adding more ES nodes

---

## Related Documentation

- [Architecture Overview](./ARCHITECTURE.md) - System design
- [API Reference](./API.md) - Endpoint documentation
- [Workflows](./WORKFLOWS.md) - User-facing workflows
- [Data Models](./MODELS.md) - Database schema

---

**Last Updated**: 2025-12-08
