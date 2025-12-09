# Background Workers

**Last Updated:** 2025-12-09

## Overview

Aleph uses background workers to process asynchronous tasks such as document indexing, cross-reference matching, data imports, and exports. Workers consume tasks from RabbitMQ queues, execute the appropriate handlers, and coordinate complex multi-stage processing pipelines.

### Key Features

- **Multi-threaded processing** - Configurable thread pool for concurrent task execution
- **10 processing stages** - Indexing, xref, reingest, reindex, mappings, exports, entity operations
- **Intelligent batching** - Automatic batch accumulation for Elasticsearch indexing
- **Priority queues** - High-priority tasks processed first
- **Horizontal scaling** - Run multiple worker instances
- **Stage specialization** - Workers can handle specific stages only
- **Automatic retry** - Built-in retry logic for transient failures
- **Progress tracking** - Real-time status monitoring via Redis

## Table of Contents

- [Architecture](#architecture)
- [Processing Stages](#processing-stages)
- [Task Processing](#task-processing)
- [Indexing Pipeline](#indexing-pipeline)
- [Monitoring](#monitoring)
- [Configuration](#configuration)
- [Scaling](#scaling)
- [Troubleshooting](#troubleshooting)
- [CLI Commands](#cli-commands)

---

## Architecture

### System Components

```
┌─────────────────┐
│   RabbitMQ      │ Task queues (10 stages)
│   Queues        │ - index, xref, reingest, etc.
└────────┬────────┘
         │
         │ Pull tasks (based on QoS)
         ▼
┌─────────────────┐
│ Aleph Worker    │ Multi-threaded task processor
│  (Python)       │ - Thread pool (configurable)
│                 │ - Task dispatcher
│                 │ - Batch accumulator (index)
└────────┬────────┘
         │
         ├─────────► Elasticsearch (bulk indexing)
         ├─────────► PostgreSQL (entity ops, status)
         ├─────────► Redis (status tracking)
         └─────────► Archive (S3/filesystem)
```

### Worker Class Structure

**File:** `aleph/worker.py:109-261`

**Class:** `AlephWorker` extends `servicelayer.taskqueue.Worker`

**Key Components:**
- **Thread Pool:** Managed by servicelayer base class
- **Task Dispatcher:** Routes tasks to appropriate handlers (`dispatch_task()`)
- **Batch Accumulator:** Special logic for indexing performance
- **Rate Limiters:** Periodic task scheduling (5 min, 24 hour)
- **Database Sessions:** Thread-local Flask app contexts

**Initialization:**
```python
def __init__(self, queues, conn=None, num_threads=WORKER_THREADS,
             version=None, prefetch_count_mapping=None):
    self.often = get_rate_limit("often", unit=300, interval=1, limit=1)
    self.daily = get_rate_limit("daily", unit=3600, interval=24, limit=1)
    self.indexing_batch_last_updated = 0.0
    self.indexing_batches = defaultdict(list)
    self.prefetch_count_mapping = prefetch_count_mapping or {}
```

**Parameters:**
- `queues` - List of stage names to process (e.g., `["index", "xref"]`)
- `conn` - Redis connection for task state tracking
- `num_threads` - Number of concurrent worker threads (default: 1)
- `version` - Aleph version string
- `prefetch_count_mapping` - QoS settings per stage (prefetch counts)

### Queue System

**Backend:** RabbitMQ via servicelayer

**Queue Names (Stages):**
```python
# Handled by Aleph workers
STAGE_INDEX = "index"              # Elasticsearch indexing
STAGE_XREF = "xref"                # Cross-reference matching
STAGE_REINGEST = "reingest"        # Re-process documents
STAGE_REINDEX = "reindex"          # Rebuild index from cache
STAGE_LOAD_MAPPING = "loadmapping" # Import via table mappings
STAGE_FLUSH_MAPPING = "flushmapping" # Clear mapping data
STAGE_EXPORT_SEARCH = "exportsearch" # Export search results
STAGE_EXPORT_XREF = "exportxref"   # Export xref matches
STAGE_UPDATE_ENTITY = "updateentity" # Post-process entity changes
STAGE_PRUNE_ENTITY = "pruneentity" # Delete entity completely

# Handled by ingest-file service
STAGE_INGEST = "ingest"            # Document text extraction
STAGE_ANALYZE = "analyze"          # NER, entity extraction
```

**Queue Properties:**
- **Durable:** Queues persist across RabbitMQ restarts
- **Priority:** Max priority 10 (higher processed first)
- **Acknowledgment:** Manual ACK after task completion
- **Persistence:** Messages persist to disk

### Task Structure

**Task Object:**
```python
class Task:
    task_id: str           # Unique identifier (UUID)
    operation: str         # Stage name (queue name)
    collection_id: int     # Collection ID (0 for global tasks)
    job_id: str           # Job ID for grouping related tasks
    priority: int         # Priority 0-10 (10 = highest)
    payload: dict         # Stage-specific data
    context: dict         # Pipeline context (languages, namespace, etc.)
```

**Example Task:**
```json
{
  "task_id": "abc-123-def-456",
  "operation": "index",
  "collection_id": 1,
  "job_id": "upload-789",
  "priority": 5,
  "payload": {
    "entity_ids": ["entity-1", "entity-2", "entity-3"]
  },
  "context": {
    "sync": false
  }
}
```

### Task Dispatcher

**File:** `aleph/worker.py:169-216`

**Process:**
1. **Database Session Management** - Commit pending changes, rollback errors
2. **Collection Lookup** - Load collection from database if needed
3. **Special Indexing Logic** - Batch accumulation for STAGE_INDEX
4. **Handler Execution** - Call appropriate handler function
5. **After-Task Hook** - Refresh collection if job complete

**Dispatch Logic:**
```python
def dispatch_task(self, task: Task) -> Task:
    # Session cleanup
    try:
        db.session.commit()
    except sqlalchemy.exc.PendingRollbackError:
        db.session.rollback()

    # Load collection
    with db.session.begin():
        if task.collection_id:
            collection = Collection.by_id(task.collection_id, deleted=True)

    # Special batching for index operations
    if task.operation == STAGE_INDEX:
        with indexing_lock:
            self.indexing_batches[task.collection_id].append(task)
            self.indexing_batch_last_updated = time.time()

            total_size = sum(len(batch) for batch in self.indexing_batches.values())
            if total_size >= INDEXING_BATCH_SIZE:
                op_index(self.indexing_batches, worker=self)
                self.indexing_batches = defaultdict(list)
                self.indexing_batch_last_updated = 0.0
            else:
                task.context["skip_ack"] = True  # Batch will ACK later
                return task

    # Execute handler
    handler = OPERATIONS[task.operation]
    handler(collection, task)
    return task
```

---

## Processing Stages

### STAGE_INDEX - Elasticsearch Indexing

**Purpose:** Index entities to Elasticsearch for full-text search

**Handler:** `op_index` (aleph/worker.py:42-73)

**Payload:**
```python
{"entity_ids": ["entity-1", "entity-2", ...]}
```

**Process:**
1. **Batch Accumulation:** Tasks collected until size threshold or timeout
2. **Bulk Indexing:** Entities indexed in single Elasticsearch bulk request
3. **Error Handling:** `skip_errors=True` continues on individual failures
4. **Thread-Safe ACK:** Tasks acknowledged via connection callback

**Batch Triggers:**
- **Size:** When batch reaches `INDEXING_BATCH_SIZE` (default: 100)
- **Timeout:** After `INDEXING_TIMEOUT` seconds (default: 10)

**Performance:**
```python
# Batch processing significantly reduces ES request overhead
# Single task: 1 entity = 1 ES request (~10ms each)
# Batched: 100 entities = 1 ES bulk request (~50ms total)
# Speedup: ~20x improvement
```

**Context:**
- `sync` (boolean) - Force immediate Elasticsearch refresh

**Files:**
- `aleph/worker.py:42-73` - Batch handler
- `aleph/logic/processing.py:18-33` - `index_many()`
- `aleph/logic/collections.py:133-154` - `index_aggregator_bulk()`
- `aleph/index/entities.py:168-176` - `index_bulk()`

---

### STAGE_XREF - Cross-Reference Matching

**Purpose:** Find similar entities across collections using fuzzy matching

**Handler:** `xref_collection` (aleph/logic/xref.py:268-280)

**Payload:** `{}` (operates on entire collection)

**Process:**
1. **Clear Old State:** Delete previous xref matches and generated entities
2. **Entity-Based Xref:** Match all matchable entities in collection
3. **Mention-Based Xref:** Aggregate mentions, match, and reify
4. **Index Matches:** Store results in xref index
5. **Reindex Collection:** Add generated entities to main index

**Matching Algorithm:**
```python
def match_entity(entity):
    # 1. Build Elasticsearch query (names, addresses, dates)
    query = match_query(entity)

    # 2. Search for candidates (up to 50)
    candidates = es.search(index=entities_index, body=query)

    # 3. Score candidates using FTM or ML model
    for match in candidates:
        score, doubt, method = compare(entity, match)
        if score > 0.5:  # Cutoff threshold
            yield Match(score, doubt, method, entity, match)
```

**Matching Methods:**
- **FTM Default:** `compare.compare()` from followthemoney library
- **ML Model:** GLMBernoulli2EEvaluator if `XREF_MODEL` configured

**Score Interpretation:**
- **0.0-0.5:** Not a match (not shown to users)
- **0.5-0.7:** Possible match (needs review)
- **0.7-0.9:** Likely match
- **0.9-1.0:** Very strong match

**Configuration:**
```bash
ALEPH_XREF_SCROLL=5m          # Elasticsearch scroll timeout
ALEPH_XREF_SCROLL_SIZE=1000   # Scroll batch size
FTM_COMPARE_MODEL=/path/to/model.pkl  # Optional ML model
```

**Performance:**
- **Memory-intensive:** Loads entities into memory for comparison
- **CPU-intensive:** Scoring algorithm for each candidate pair
- **Time:** Varies by collection size (minutes to hours)

**QoS:** Prefetch = 1 (sequential processing to limit memory usage)

**Files:**
- `aleph/logic/xref.py:268-280` - Collection xref
- `aleph/logic/xref.py:127-192` - Entity matching
- `aleph/logic/matching.py` - Match query building
- `aleph/index/xref.py` - Xref index operations

---

### STAGE_REINGEST - Re-process Documents

**Purpose:** Send all documents through ingest pipeline again (e.g., after OCR config change)

**Handler:** `reingest_collection` (aleph/logic/collections.py:156-166)

**Payload:**
```python
{
    "index": bool,           # Queue indexing after ingest
    "flush": bool,           # Clear cached fragments first
    "include_ingest": bool   # Also clear ingest-stage fragments
}
```

**Process:**
```python
def reingest_collection(collection, job_id=None, index=False,
                        flush=True, include_ingest=False):
    # 1. Clear aggregator cache (optional)
    if flush:
        ingest_flush(collection, include_ingest=include_ingest)

    # 2. Queue each document to STAGE_INGEST
    for document in Document.by_collection(collection.id):
        proxy = document.to_proxy(ns=collection.ns)
        ingest_entity(collection, proxy, job_id=job_id, index=index)
```

**When to Use:**
- OCR languages changed
- ingest-file service updated with better extraction
- Documents failed initial processing
- Need to refresh extracted entities

**Pipeline:** Documents flow through `STAGE_INGEST` → `STAGE_ANALYZE` → `STAGE_INDEX`

**Files:**
- `aleph/logic/collections.py:156-166` - Reingest handler
- `aleph/logic/documents.py:14-22` - `ingest_flush()`
- `aleph/queues.py:70-79` - `ingest_entity()`

---

### STAGE_REINDEX - Rebuild Index

**Purpose:** Rebuild Elasticsearch index from aggregator cache and mappings

**Handler:** `reindex_collection` (aleph/logic/collections.py:168-190)

**Payload:**
```python
{
    "skip_errors": bool,  # Continue on mapping errors
    "sync": bool,         # Force ES refresh
    "flush": bool         # Delete existing index first
}
```

**Process:**
```python
def reindex_collection(collection, skip_errors=True, sync=False, flush=False):
    aggregator = get_aggregator(collection)

    # 1. Regenerate mapping entities
    for mapping in collection.mappings:
        if not mapping.disabled:
            map_to_aggregator(collection, mapping, aggregator)

    # 2. Aggregate database models
    aggregate_model(collection, aggregator)

    # 3. Add profile fragments (entityset membership)
    profile_fragments(collection, aggregator)

    # 4. Delete existing index (optional)
    if flush:
        delete_entities(collection.id, sync=True)

    # 5. Bulk index from aggregator
    index_aggregator(collection, aggregator, skip_errors=skip_errors, sync=sync)

    # 6. Update statistics
    compute_collection(collection, force=True)
```

**When to Use:**
- Elasticsearch index corrupted
- Changed index settings
- Fixed mapping configuration errors
- Regenerate entities from database

**Difference from Reingest:**
- **Reindex:** Uses cached aggregator data (fast, no re-processing)
- **Reingest:** Re-processes files through ingest-file (slow, full re-extraction)

**Files:**
- `aleph/logic/collections.py:168-190` - Reindex handler
- `aleph/logic/mapping.py` - Mapping aggregation
- `aleph/logic/profiles.py` - Profile fragments
- `aleph/logic/aggregator.py` - Aggregator operations

---

### STAGE_LOAD_MAPPING - Import via Mappings

**Purpose:** Transform tabular data to entities using FollowTheMoney mapping configuration

**Handler:** `load_mapping` (aleph/logic/mapping.py:74-103)

**Payload:**
```python
{
    "mapping_id": str,  # Database ID of mapping
    "sync": bool        # Force ES refresh
}
```

**Process:**
```python
def load_mapping(collection, mapping_id, sync=False):
    mapping = Mapping.by_id(mapping_id)
    aggregator = get_aggregator(collection)

    # 1. Clear old mapping data
    origin = f"mapping:{mapping_id}"
    aggregator.delete(origin=origin)
    delete_entities(collection.id, origin=origin, sync=True)

    if mapping.disabled:
        return

    # 2. Get CSV data
    table = get_entity(mapping.table_id)
    csv_url = get_table_csv_link(table)

    # 3. Create FTM mapper
    config = {"csv_url": csv_url, "entities": mapping.query}
    mapper = model.make_mapping(config, key_prefix=collection.foreign_id)

    # 4. Process rows
    writer = aggregator.bulk()
    for idx, record in enumerate(mapper.source.records, 1):
        for entity in mapper.map(record).values():
            entity.context = mapping.get_proxy_context()
            entity.add("proof", mapping.table_id)
            entity = collection.ns.apply(entity)
            writer.put(entity, fragment=idx, origin=origin)
    writer.flush()

    # 5. Index results
    index_aggregator(collection, aggregator, sync=sync)
    mapping.set_status(status=Status.SUCCESS)
```

**Mapping Query Example:**
```yaml
entities:
  person:
    schema: Person
    keys:
      - id
    properties:
      name:
        column: full_name
      birthDate:
        column: dob
      nationality:
        column: country
```

**Files:**
- `aleph/logic/mapping.py:74-103` - Load handler
- `aleph/model/mapping.py` - Mapping model
- `followthemoney.mapping` - FTM mapper

---

### STAGE_FLUSH_MAPPING - Clear Mapping Data

**Purpose:** Remove all entities generated by a specific mapping

**Handler:** `flush_mapping` (aleph/logic/mapping.py:105-115)

**Payload:**
```python
{
    "mapping_id": str,
    "sync": bool
}
```

**Process:**
```python
def flush_mapping(collection, mapping_id, sync=True):
    origin = f"mapping:{mapping_id}"
    aggregator = get_aggregator(collection)

    # Delete from aggregator cache
    aggregator.delete(origin=origin)

    # Delete from Elasticsearch
    delete_entities(collection.id, origin=origin, sync=sync)

    # Update collection statistics
    update_collection(collection, sync=sync)
    collection.touch()
```

**When to Use:**
- Mapping configuration changed
- Need to re-import with different settings
- Mapping disabled

**Files:**
- `aleph/logic/mapping.py:105-115` - Flush handler
- `aleph/index/collections.py` - `delete_entities()`

---

### STAGE_EXPORT_SEARCH - Export Search Results

**Purpose:** Export search results to downloadable Excel/ZIP file

**Handler:** `export_entities` (aleph/logic/export.py:58-103)

**Payload:**
```python
{"export_id": str}  # Database ID of export record
```

**Process:**
```python
def export_entities(export_id):
    export = Export.by_id(export_id)
    export_dir = mkdtemp(prefix="aleph.export.")

    # Extract query from export metadata
    query = export.meta.get("query", {"match_none": {}})
    schemata = export.meta.get("schemata", [Entity.THING])

    # Create ZIP with Excel file
    with ZipFile(file_path, mode="w") as zf:
        exporter = ExcelExporter(excel_path, extra=["url", "collection"])

        # Iterate matching entities
        for idx, entity in enumerate(iter_proxies(schemata=schemata, filters=[query])):
            collection = get_collection(entity.context["collection_id"])
            exporter.write(entity, extra=[entity_url, collection.label])

            # Include document files
            write_document(export_dir, zf, collection, entity)

            # Check limits
            if file_size >= EXPORT_MAX_SIZE: break
            if idx >= EXPORT_MAX_RESULTS: break

        # Finalize Excel
        zf.write(excel_path, arcname="export.xlsx")

    # Upload to archive
    complete_export(export_id, file_path, file_name)
```

**Export Limits:**
- **Max File Size:** `EXPORT_MAX_SIZE` (default: 1 GB)
- **Max Results:** `EXPORT_MAX_RESULTS` (default: 100,000)

**Export Format:**
- **Excel:** Entity properties as columns
- **ZIP:** Excel + referenced document files

**Files:**
- `aleph/logic/export.py:58-103` - Export handler
- `followthemoney.export.excel` - Excel exporter

---

### STAGE_EXPORT_XREF - Export Xref Matches

**Purpose:** Export cross-reference matches to Excel file

**Handler:** `export_matches` (aleph/logic/xref.py:332-381)

**Payload:**
```python
{"export_id": str}
```

**Process:**
```python
def export_matches(export_id):
    export = Export.by_id(export_id)
    collection = Collection.by_id(export.collection_id)

    # Create Excel
    excel = ExcelWriter()
    headers = ["Score", "Doubt", "Entity Name", "Entity Date",
               "Entity Countries", "Candidate Collection", ...]
    sheet = excel.make_sheet("Cross-reference", headers)

    # Batch process matches
    for match in iter_matches(collection, authz):
        sheet.append([
            match["score"], match["doubt"],
            match["entity"]["properties"]["name"],
            match["entity"]["properties"]["date"],
            ...
        ])

    # Write to archive
    with open(file_path, "wb") as fp:
        fp.write(excel.get_bytesio())

    complete_export(export_id, file_path, file_name)
```

**Excel Columns:**
- Score, Doubt
- Entity Name, Date, Countries
- Candidate Name, Collection, Countries
- Match Method

**Files:**
- `aleph/logic/xref.py:332-381` - Export handler
- `aleph/index/xref.py` - `iter_matches()`

---

### STAGE_UPDATE_ENTITY - Post-Process Entity

**Purpose:** Update xref, profiles, and trigger NER pipeline after entity changes

**Handler:** `update_entity` (aleph/logic/entities.py:56-75)

**Payload:**
```python
{"entity_id": str}
```

**Process:**
```python
def update_entity(collection, entity_id=None, job_id=None):
    # Get entity from index
    entity = index.get_entity(entity_id)
    proxy = model.get_proxy(entity)

    # Update xref for casefiles
    if collection.casefile:
        xref_entity(collection, proxy)

    # Update profile fragments
    aggregator = get_aggregator(collection, origin=MODEL_ORIGIN)
    profile_fragments(collection, aggregator, entity_id=entity_id)

    # Inline related entity names
    inline_names(aggregator, proxy)

    # Send through pipeline for NER
    pipeline_entity(collection, proxy, job_id=job_id)
```

**When Triggered:**
- Entity created via API
- Entity properties updated
- Entity relationships changed

**Pipeline:** `[STAGE_ANALYZE, STAGE_INDEX]` for analyzable entities

**Files:**
- `aleph/logic/entities.py:56-75` - Update handler
- `aleph/logic/xref.py` - `xref_entity()`
- `aleph/logic/profiles.py` - `profile_fragments()`

---

### STAGE_PRUNE_ENTITY - Delete Entity

**Purpose:** Complete deletion of entity and all references

**Handler:** `prune_entity` (aleph/logic/entities.py:152-180)

**Payload:**
```python
{"entity_id": str}
```

**Process:**
```python
def prune_entity(collection, entity_id=None, job_id=None):
    # 1. Recursive delete of adjacent entities
    for adjacent in index.iter_adjacent(collection.id, entity_id):
        delete_entity(collection, adjacent, job_id=job_id)

    # 2. Delete notifications
    flush_notifications(entity_id, clazz=Entity)

    # 3. Delete database records
    obj = Entity.by_id(entity_id, collection=collection)
    if obj: obj.delete()

    doc = Document.by_id(entity_id, collection=collection)
    if doc: doc.delete()

    # 4. Delete references
    EntitySetItem.delete_by_entity(entity_id)
    Bookmark.delete_by_entity(entity_id)
    Mapping.delete_by_table(entity_id)

    # 5. Delete xref matches
    xref_index.delete_xref(collection, entity_id=entity_id)

    # 6. Delete from aggregator
    aggregator = get_aggregator(collection)
    aggregator.delete(entity_id=entity_id)

    # 7. Refresh index
    refresh_entity(collection, entity_id)
    collection.touch()
```

**Cascading Deletes:**
- Adjacent entities (children, relationships)
- Notifications, bookmarks
- EntitySet memberships
- Xref matches
- Aggregator fragments
- Mapping references

**Files:**
- `aleph/logic/entities.py:152-180` - Prune handler
- `aleph/index/entities.py` - `iter_adjacent()`

---

## Task Processing

### Task Lifecycle

```
┌──────────┐
│  Queued  │ Task added to RabbitMQ queue
└────┬─────┘
     │
     ▼
┌──────────┐
│ Fetched  │ Worker prefetches based on QoS
└────┬─────┘
     │
     ▼
┌──────────┐
│Dispatched│ dispatch_task() routes to handler
└────┬─────┘
     │
     ▼
┌──────────┐
│ Running  │ Handler executes business logic
└────┬─────┘
     │
     ▼
┌──────────┐
│Completed │ Task acknowledged, removed from queue
└────┬─────┘
     │
     ▼
┌──────────┐
│After-Task│ Collection refreshed if job done
└──────────┘
```

**After-Task Hook** (aleph/worker.py:218-222):
```python
def after_task(self, task):
    if task.collection_id and task.get_dataset(conn=kv).is_done():
        refresh_collection(task.collection_id)
```

### Priority Handling

**Priority Range:** 0-10 (10 = highest)

**Queue Configuration:**
```python
RABBITMQ_MAX_PRIORITY = 10  # Max priority for all queues
```

**Processing Order:**
1. Higher priority tasks processed first
2. Within same priority: FIFO order
3. All workers respect priority

**Use Cases:**
- **Priority 10:** User-initiated actions (immediate feedback needed)
- **Priority 5:** Background processing (default)
- **Priority 0:** Maintenance tasks (can wait)

### QoS (Quality of Service)

**Prefetch Counts per Stage:**

| Stage | Prefetch | Rationale |
|-------|----------|-----------|
| index | 100 | Enable batching for performance |
| xref | 1 | Memory-intensive, sequential processing |
| reingest | 1 | Queues many sub-tasks |
| reindex | 1 | Memory-intensive |
| loadmapping | 1 | Memory-intensive |
| flushmapping | 1 | Sequential delete operations |
| exportsearch | 1 | I/O-intensive |
| exportxref | 1 | I/O-intensive |
| updateentity | 1 | Sequential processing |
| pruneentity | 1 | Cascading deletes |

**Configuration:**
```bash
# aleph/settings.py
ALEPH_RABBITMQ_QOS_INDEX_QUEUE=100
ALEPH_RABBITMQ_QOS_XREF_QUEUE=1
# ... etc
```

**Effect:**
- **High prefetch (100):** Worker pulls many tasks at once, enables batching
- **Low prefetch (1):** Worker pulls one task at a time, prevents overload

### Concurrency Control

**Thread-Level:**
- Thread pool managed by servicelayer
- Each thread processes tasks sequentially
- Thread-safe batch accumulation via `threading.Lock()`

**Database:**
- Flask app context provides thread-local sessions
- Session cleanup via `db.session.remove()` in periodic tasks

**RabbitMQ:**
- Prefetch limits prevent worker overload
- Manual acknowledgment ensures at-least-once delivery
- Consumer timeout: Tasks re-queued if worker crashes

### Error Handling

**Elasticsearch Operations:**
```python
for attempt in service_retries():
    try:
        es.index(index=index, id=id, body=body)
        return body
    except TransportError as exc:
        if exc.status_code in ("400", "403"):
            raise  # Don't retry client errors
        log.warning("Index error: %s", exc)
        backoff(failures=attempt)  # Exponential backoff
```

**Retry Strategy:**
- **Transient errors:** Automatic retry with exponential backoff
- **Client errors (400, 403):** No retry, task fails immediately
- **Max retries:** 10 attempts

**Task Failure:**
- Task not acknowledged
- RabbitMQ re-queues after consumer timeout
- Worker can pick up again or different worker processes

---

## Indexing Pipeline

### Batch Accumulation

**Data Structure:**
```python
self.indexing_batches = {
    collection_id_1: [task1, task2, task3, ...],
    collection_id_2: [task4, task5, ...],
}
```

**Flow:**
1. Index task arrives with entity IDs
2. Lock acquired (`with indexing_lock`)
3. Task appended to collection's batch
4. Timestamp updated
5. Total batch size checked
6. If threshold met: flush all batches
7. Else: mark task for deferred ACK

**Thread-Safe ACK:**
```python
# ACK from worker thread via connection callback
channel = task._channel
delattr(task, "_channel")
task = copy.deepcopy(task)
task.context["skip_ack"] = False
cb = functools.partial(worker.ack_message, task, channel)
channel.connection.add_callback_threadsafe(cb)
```

### Flush Triggers

**Size Trigger:**
```python
total_size = sum(len(batch) for batch in self.indexing_batches.values())
if total_size >= INDEXING_BATCH_SIZE:  # Default: 100
    op_index(self.indexing_batches, worker=self)
```

**Time Trigger (Periodic):**
```python
# Called from periodic() every ~1 second
def run_indexing_batches(self):
    with indexing_lock:
        now = time.time()
        since_last_update = int(now - self.indexing_batch_last_updated)

        if since_last_update > INDEXING_TIMEOUT:  # Default: 10s
            if self.indexing_batches:
                op_index(self.indexing_batches, worker=self)
```

### Bulk Operations

**Elasticsearch Bulk API:**

**File:** `aleph/index/util.py:197-217`

```python
def bulk_actions(actions, chunk_size=BULK_PAGE, sync=False):
    stream = streaming_bulk(
        es,
        actions,
        chunk_size=chunk_size,  # 500 actions per request
        max_retries=10,
        yield_ok=False,
        raise_on_error=False,
        refresh=refresh_sync(sync),
        request_timeout=MAX_REQUEST_TIMEOUT,
        timeout=MAX_TIMEOUT,
    )

    for _, details in stream:
        if details.get("delete", {}).get("status") == 404:
            continue  # Ignore "not found" on delete
        log.warning("Bulk index error: %r", details)
```

**Bulk Format:**
```json
{"index": {"_index": "aleph-entity-v1", "_id": "entity-1"}}
{"properties": {...}, "text": [...]}
{"index": {"_index": "aleph-entity-v1", "_id": "entity-2"}}
{"properties": {...}, "text": [...]}
```

**Performance:**
- **Chunk Size:** 500 actions per bulk request
- **Parallel Requests:** streaming_bulk sends requests in parallel
- **Retry:** Up to 10 retries on transient failures

### Error Recovery

**Skip Errors Mode:**
```python
# aleph/worker.py:57
index_many(batch, sync=sync, skip_errors=True)
```

- Individual entity failures don't block batch
- Errors logged: `log.warning("Bulk index error: %r", details)`
- Useful for handling FTM merge errors or validation issues

**Batch Consistency:**
- If batch fails: tasks not ACKed, will retry
- Elasticsearch "at-least-once" semantics
- Idempotent operations (same entity_id overwrites)

---

## Monitoring

### Status Tracking (Redis)

**Dataset Concept:**
- Jobs grouped by: `dataset = f"collection:{collection_id}:{job_id}"`
- Redis stores task counts per dataset
- `Dataset.is_done()` checks if pending + running == 0

**Get Status:**
```python
from aleph.queues import get_status
from aleph.model import Collection

collection = Collection.by_id(1)
status = get_status(collection)

# Output:
# {
#   "pending": 150,    # Queued tasks
#   "running": 5,      # Active tasks
#   "finished": 5000   # Completed tasks
# }
```

**Active Datasets:**
```python
from aleph.queues import get_active_dataset_status

status = get_active_dataset_status()
# Returns dict of all active datasets and their status
```

### Collection Statistics

**Endpoint:** `GET /api/2/collections/<id>`

**Response includes:**
```json
{
  "id": 1,
  "label": "Panama Papers",
  "count": 152340,
  "statistics": {
    "schema": {
      "Document": 120000,
      "Person": 15000,
      "Company": 12000,
      "Email": 5340
    },
    "countries": {"PA": 50000, "BVI": 30000},
    "languages": {"eng": 100000, "spa": 52340}
  },
  "status": {
    "pending": 0,
    "running": 0,
    "finished": 152340
  }
}
```

**Status Endpoint:** `GET /api/2/collections/<id>/status`

Dedicated endpoint for just task status (no database hit).

### Progress Reporting

**Task Logging:**
```python
# Task start
log.info(f"Task [collection:{cid}]: op:{op} task_id:{tid} priority:{pri} (started)")

# Task batched (index only)
log.info(f"Task [collection:{cid}]: op:{op} ... (batched)")

# Task complete
log.info(f"Task [collection:{cid}]: op:{op} ... (done)")
```

**Batch Progress:**
```python
# Every 1000 entities during indexing
if idx > 0 and idx % 1000 == 0:
    log.debug("[%s] Index: %s entities...", collection, idx)
```

**Mapping Progress:**
```python
# Every 1000 rows during mapping
if idx > 0 and idx % 1000 == 0:
    log.info("[%s] Mapped %s rows...", mapping.id, idx)
```

### Logging Configuration

**Structured Logging:**
```python
# aleph/core.py
from servicelayer.logs import configure_logging

configure_logging(level=logging.DEBUG)
```

**Environment Variables:**
```bash
ALEPH_LOG_LEVEL=DEBUG  # DEBUG, INFO, WARNING, ERROR
LOG_FORMAT=json        # Enable JSON structured logs
```

**JSON Log Example:**
```json
{
  "timestamp": "2025-12-09T10:30:45.123Z",
  "level": "INFO",
  "message": "Task [collection:1]: op:index task_id:abc-123 priority:5 (done)",
  "collection_id": 1,
  "task_id": "abc-123",
  "operation": "index",
  "priority": 5
}
```

### RabbitMQ Monitoring

**Management UI:**
- URL: `http://localhost:15672`
- Username: `guest`
- Password: `guest`

**Queue Metrics:**
- **Ready:** Tasks waiting to be processed
- **Unacked:** Tasks currently being processed
- **Total:** Lifetime task count
- **Rate:** Tasks per second

**CLI Monitoring:**
```bash
# List queues with task counts
docker-compose exec rabbitmq rabbitmqctl list_queues

# Output:
# index      150
# xref       2
# reingest   0
# exportsearch 5
```

---

## Configuration

### Environment Variables

**Worker Configuration:**
```bash
# Number of worker threads (servicelayer)
WORKER_THREADS=4

# Stages to process (comma-separated)
ALEPH_WORKER_STAGES=index,xref,reingest,reindex,loadmapping,flushmapping,exportsearch,exportxref,updateentity,pruneentity

# Queue prefetch counts
ALEPH_RABBITMQ_QOS_INDEX_QUEUE=100
ALEPH_RABBITMQ_QOS_XREF_QUEUE=1
# ... (see QoS section for all)

# Queue priority
ALEPH_RABBITMQ_MAX_PRIORITY=10
```

**Indexing Configuration:**
```bash
# Batch size before flush
ALEPH_INDEXING_BATCH_SIZE=100

# Timeout before flush (seconds)
ALEPH_INDEXING_TIMEOUT=10

# Delete-by-query batch size
ALEPH_INDEX_DELETE_BY_QUERY_BATCHSIZE=100
```

**Xref Configuration:**
```bash
# Elasticsearch scroll settings
ALEPH_XREF_SCROLL=5m
ALEPH_XREF_SCROLL_SIZE=1000

# Optional ML model
FTM_COMPARE_MODEL=/path/to/model.pkl
```

**Export Configuration:**
```bash
# Max export file size (bytes)
EXPORT_MAX_SIZE=1073741824  # 1 GB

# Max export result count
EXPORT_MAX_RESULTS=100000
```

**Database Configuration:**
```bash
ALEPH_DATABASE_URI=postgresql://user:pass@host/database
SQLALCHEMY_POOL_SIZE=5
SQLALCHEMY_POOL_TIMEOUT=10
SQLALCHEMY_MAX_OVERFLOW=10
SQLALCHEMY_POOL_RECYCLE=3600
```

**Elasticsearch Configuration:**
```bash
ALEPH_ELASTICSEARCH_URI=http://localhost:9200
ELASTICSEARCH_TIMEOUT=60
```

### Performance Tuning

**Indexing Optimization:**
```bash
# Higher batch size = more throughput, more memory
ALEPH_INDEXING_BATCH_SIZE=200
ALEPH_INDEXING_TIMEOUT=15

# More threads for index workers
WORKER_THREADS=8
ALEPH_RABBITMQ_QOS_INDEX_QUEUE=200
```

**Database Connection Pool:**
```bash
# Pool size should be >= number of threads + 2
SQLALCHEMY_POOL_SIZE=10  # For 8 worker threads
SQLALCHEMY_MAX_OVERFLOW=5
```

**Elasticsearch:**
```bash
# Longer timeout for slow clusters
ELASTICSEARCH_TIMEOUT=120

# More replicas for production
ALEPH_INDEX_REPLICAS=1
```

---

## Scaling

### Horizontal Scaling

**Multiple Worker Deployment:**
```yaml
# docker-compose.yml
services:
  worker-index:
    image: aleph
    command: aleph worker --threads=8
    environment:
      ALEPH_WORKER_STAGES: index,reingest,reindex,updateentity,pruneentity
    deploy:
      replicas: 3

  worker-xref:
    image: aleph
    command: aleph worker --threads=2
    environment:
      ALEPH_WORKER_STAGES: xref
    deploy:
      replicas: 2

  worker-export:
    image: aleph
    command: aleph worker --threads=1
    environment:
      ALEPH_WORKER_STAGES: exportsearch,exportxref
    deploy:
      replicas: 1

  worker-mapping:
    image: aleph
    command: aleph worker --threads=1
    environment:
      ALEPH_WORKER_STAGES: loadmapping,flushmapping
    deploy:
      replicas: 1
```

**Benefits:**
- **Fault Tolerance:** Worker failure doesn't halt all processing
- **Resource Isolation:** Heavy operations don't block others
- **Parallelism:** Multiple workers consume from same queue
- **Specialization:** Tune resources per stage type

### Vertical Scaling

**More Threads per Worker:**
```bash
aleph worker --threads=16
```

**Benefits:**
- **Throughput:** More concurrent task processing
- **Resource Utilization:** Better CPU usage

**Considerations:**
- **Database Connections:** Need larger pool (threads + 2)
- **Memory:** More threads = more memory usage
- **Diminishing Returns:** Beyond 8-16 threads, I/O becomes bottleneck

**Recommended Thread Counts:**

| Worker Type | Threads | Rationale |
|-------------|---------|-----------|
| Index-only | 8-16 | I/O bound, benefits from parallelism |
| Xref-only | 2-4 | CPU + memory intensive |
| Export-only | 1-2 | I/O bound, limited by archive speed |
| Mapping-only | 1-2 | Memory intensive |
| General | 4-8 | Balanced mix |

### Stage-Specific Workers

**Index-Heavy Deployment:**
```bash
# Worker 1: Index focus (high volume)
ALEPH_WORKER_STAGES=index
WORKER_THREADS=16
ALEPH_RABBITMQ_QOS_INDEX_QUEUE=200
ALEPH_INDEXING_BATCH_SIZE=200
```

**Xref-Heavy Deployment:**
```bash
# Worker 2: Xref focus (CPU/memory)
ALEPH_WORKER_STAGES=xref
WORKER_THREADS=4
ALEPH_RABBITMQ_QOS_XREF_QUEUE=1
```

**Export Workers:**
```bash
# Worker 3: Export focus (I/O)
ALEPH_WORKER_STAGES=exportsearch,exportxref
WORKER_THREADS=2
```

### Load Balancing

**RabbitMQ Round-Robin:**
- Tasks distributed evenly across workers
- Each worker ACKs independently
- Failed workers: tasks re-queued after timeout

**Priority-Based:**
- All workers respect task priority
- High-priority tasks processed first
- Within priority: FIFO order

**Prefetch-Based:**
- Workers with lower active task count receive more tasks
- QoS prefetch prevents overload
- Automatic load balancing

---

## Troubleshooting

### Common Issues

#### 1. Index Batches Not Flushing

**Symptoms:** Documents uploaded but not searchable, tasks show "batched"

**Diagnosis:**
```bash
# Check worker logs
docker-compose logs worker | grep -i "batch"

# Check queue depth
docker-compose exec rabbitmq rabbitmqctl list_queues | grep index
```

**Causes:**
- Worker not running
- Worker not processing `STAGE_INDEX`
- Batch size threshold not met
- Timeout not triggering

**Solutions:**
```bash
# Verify worker is running
docker-compose ps worker

# Check worker stages
docker-compose exec worker env | grep ALEPH_WORKER_STAGES

# Restart worker
docker-compose restart worker

# Force immediate indexing (reduce batch size temporarily)
ALEPH_INDEXING_BATCH_SIZE=10 ALEPH_INDEXING_TIMEOUT=2 aleph worker
```

---

#### 2. High Memory Usage

**Symptoms:** Worker killed by OOM, container restarts

**Diagnosis:**
```bash
# Check memory usage
docker stats worker

# Check batch sizes in logs
docker-compose logs worker | grep "Index:"
```

**Causes:**
- Too many worker threads
- Large indexing batches
- Entity caching
- Xref memory usage

**Solutions:**
```bash
# Reduce worker threads
WORKER_THREADS=2 aleph worker

# Reduce batch size
ALEPH_INDEXING_BATCH_SIZE=50 aleph worker

# Increase container memory limit
# docker-compose.yml
services:
  worker:
    deploy:
      resources:
        limits:
          memory: 4G  # Increase from default

# Use stage-specific workers (smaller memory footprint)
ALEPH_WORKER_STAGES=index aleph worker --threads=4
```

---

#### 3. Tasks Stuck in Queue

**Symptoms:** RabbitMQ shows many ready tasks, but none processing

**Diagnosis:**
```bash
# Check worker is consuming
docker-compose exec rabbitmq rabbitmqctl list_consumers

# Check worker logs for errors
docker-compose logs worker | tail -100

# Check task queue depths
docker-compose exec rabbitmq rabbitmqctl list_queues
```

**Causes:**
- Worker crashed
- Database connection issues
- Elasticsearch unavailable
- Unhandled exceptions

**Solutions:**
```bash
# Restart worker
docker-compose restart worker

# Check dependencies
docker-compose ps  # Verify postgres, elasticsearch, redis, rabbitmq are up

# Check worker can connect to ES
docker-compose exec worker curl http://elasticsearch:9200

# View worker error logs
docker-compose logs worker --tail=100 | grep -i error
```

---

#### 4. Slow Xref Performance

**Symptoms:** Xref takes hours, high CPU usage

**Diagnosis:**
```bash
# Check collection size
curl "http://localhost:8080/api/2/collections/1" | jq .count

# Check xref progress in logs
docker-compose logs worker | grep -i xref

# Monitor ES queries
curl "http://localhost:9200/_cat/tasks?v&detailed" | grep xref
```

**Causes:**
- Large collection (100,000+ entities)
- Many candidate matches
- Slow ML model evaluation

**Solutions:**
```bash
# Increase xref workers
# docker-compose.yml
worker-xref:
  deploy:
    replicas: 4

# Increase scroll size (more entities per ES request)
ALEPH_XREF_SCROLL_SIZE=2000 aleph worker

# Use faster scoring (disable ML model)
unset FTM_COMPARE_MODEL

# Split into smaller collections
# (Run xref on subsets, merge results)
```

---

#### 5. Database Connection Exhaustion

**Symptoms:** `QueuePool limit exceeded`, worker hangs

**Diagnosis:**
```bash
# Check active connections
docker-compose exec postgres psql -U aleph -c "SELECT count(*) FROM pg_stat_activity;"

# Check pool settings
docker-compose exec worker python -c "from aleph.core import db; print(db.engine.pool.size())"
```

**Causes:**
- Too many worker threads
- Insufficient pool size
- Connections not released

**Solutions:**
```bash
# Increase pool size
SQLALCHEMY_POOL_SIZE=20 aleph worker

# Reduce worker threads
WORKER_THREADS=4 aleph worker

# Enable pool recycling
SQLALCHEMY_POOL_RECYCLE=3600 aleph worker
```

---

#### 6. Elasticsearch Bulk Errors

**Symptoms:** "Circuit breaker tripped", timeout errors in logs

**Diagnosis:**
```bash
# Check ES cluster health
curl "http://localhost:9200/_cluster/health?pretty"

# Check circuit breakers
curl "http://localhost:9200/_nodes/stats/breaker?pretty"

# Check bulk thread pool
curl "http://localhost:9200/_cat/thread_pool/bulk?v"
```

**Causes:**
- Too large bulk batches
- ES memory pressure
- Too many concurrent requests

**Solutions:**
```bash
# Reduce batch size
ALEPH_INDEXING_BATCH_SIZE=50 aleph worker

# Reduce chunk size
# aleph/settings.py (code change needed)
BULK_PAGE = 250  # Down from 500

# Increase ES heap
# docker-compose.yml
elasticsearch:
  environment:
    ES_JAVA_OPTS: "-Xms2g -Xmx2g"  # Increase from 1g

# Add more ES nodes for horizontal scaling
```

---

### Debugging Commands

**Inspect Worker State:**
```python
from aleph.worker import get_worker

worker = get_worker(num_threads=4)
print(f"Queues: {worker.queues}")
print(f"QoS: {worker.prefetch_count_mapping}")
print(f"Indexing batches: {len(worker.indexing_batches)}")
for collection_id, tasks in worker.indexing_batches.items():
    print(f"  Collection {collection_id}: {len(tasks)} tasks")
```

**Check Collection Status:**
```python
from aleph.queues import get_status
from aleph.model import Collection

collection = Collection.by_id(1)
status = get_status(collection)
print(f"Pending: {status['pending']}")
print(f"Running: {status['running']}")
print(f"Finished: {status['finished']}")
```

**View Active Datasets:**
```python
from aleph.queues import get_active_dataset_status

status = get_active_dataset_status()
for dataset, counts in status.items():
    print(f"{dataset}: {counts}")
```

**Monitor RabbitMQ:**
```bash
# List queues
docker-compose exec rabbitmq rabbitmqctl list_queues name messages consumers

# List consumers
docker-compose exec rabbitmq rabbitmqctl list_consumers

# Purge queue (CAUTION: deletes all tasks)
docker-compose exec rabbitmq rabbitmqctl purge_queue index
```

---

## CLI Commands

### Start Worker

**Basic:**
```bash
aleph worker
```

**With Threads:**
```bash
aleph worker --threads=8
```

**With Custom Stages:**
```bash
ALEPH_WORKER_STAGES="index,xref" aleph worker --threads=4
```

**Debug Mode:**
```bash
ALEPH_LOG_LEVEL=DEBUG aleph worker --threads=1
```

### Worker Entry Point

**File:** `aleph/manage.py:143-149`

```python
@cli.command()
@click.option("--threads", required=False, type=int)
def worker(threads=1):
    """Run the queue-based worker service."""
    worker = get_worker(num_threads=threads)
    code = worker.run()
    sys.exit(code)
```

### Deployment Examples

**Docker Compose:**
```yaml
services:
  worker:
    image: aleph:latest
    command: aleph worker --threads=4
    environment:
      - ALEPH_WORKER_STAGES=index,xref,reingest,reindex
      - WORKER_THREADS=4
    restart: unless-stopped
```

**Kubernetes:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aleph-worker
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: worker
        image: aleph:latest
        command: ["aleph", "worker", "--threads=4"]
        env:
        - name: ALEPH_WORKER_STAGES
          value: "index,reingest,reindex,updateentity"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
```

**Systemd:**
```ini
[Unit]
Description=Aleph Worker
After=network.target postgresql.service

[Service]
Type=simple
User=aleph
WorkingDirectory=/opt/aleph
Environment="ALEPH_WORKER_STAGES=index,xref"
Environment="WORKER_THREADS=4"
ExecStart=/opt/aleph/venv/bin/aleph worker --threads=4
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

---

## Best Practices

### 1. Worker Deployment Strategy

**Development:**
- Single worker with all stages
- 1-2 threads
- Minimal resource requirements

```bash
aleph worker --threads=2
```

**Small Production (< 100k entities):**
- 2-3 general workers
- 4 threads each
- Mixed stage handling

```yaml
worker:
  replicas: 3
  command: aleph worker --threads=4
```

**Large Production (> 100k entities):**
- Specialized workers per stage
- Optimized thread counts
- Horizontal scaling

```yaml
worker-index:
  replicas: 5
  command: aleph worker --threads=8
  environment:
    ALEPH_WORKER_STAGES: index,updateentity

worker-xref:
  replicas: 2
  command: aleph worker --threads=2
  environment:
    ALEPH_WORKER_STAGES: xref

worker-other:
  replicas: 2
  command: aleph worker --threads=2
  environment:
    ALEPH_WORKER_STAGES: reingest,reindex,loadmapping,exportsearch,exportxref
```

### 2. Resource Allocation

**Memory Guidelines:**
- **Index workers:** 2-4 GB per worker
- **Xref workers:** 4-8 GB per worker (memory-intensive)
- **Mapping workers:** 2-4 GB per worker
- **Export workers:** 2-4 GB per worker

**CPU Guidelines:**
- **Index workers:** 2-4 CPUs (I/O bound)
- **Xref workers:** 4-8 CPUs (CPU bound)
- **Other workers:** 1-2 CPUs

### 3. Monitoring Checklist

- ✅ RabbitMQ queue depths (alert if > 1000)
- ✅ Worker CPU/memory usage (alert if > 80%)
- ✅ Database connection pool usage (alert if > 90%)
- ✅ Elasticsearch health (alert if not green)
- ✅ Task completion rate (alert if 0 for > 5 min)
- ✅ Error logs (alert on ERROR level)

### 4. Maintenance Tasks

**Daily:**
- Check worker logs for errors
- Monitor queue depths
- Review collection statistics

**Weekly:**
- Analyze slow tasks
- Check database connection pool health
- Review Elasticsearch index sizes

**Monthly:**
- Optimize Elasticsearch indices
- Review and adjust worker scaling
- Performance benchmarking

---

## Related Documentation

- [Ingestion](./INGESTION.md) - Document upload and ingest-file integration
- [Search](./SEARCH.md) - Elasticsearch indexing and search
- [Cross-Reference](./XREF.md) - Entity matching algorithms
- [Collections](./COLLECTIONS.md) - Collection management and operations
- [Entities](./ENTITIES.md) - Entity model and FollowTheMoney
- [Configuration](./CONFIGURATION.md) - Environment variables
- [Performance](./PERFORMANCE.md) - Optimization guide

---

## File References

### Worker Core

| File | Lines | Description |
|------|-------|-------------|
| `aleph/worker.py` | 109-261 | AlephWorker class, task dispatcher, periodic tasks |
| `aleph/queues.py` | 25-79 | Task queuing, status tracking |
| `aleph/settings.py` | 254-294 | Stage configuration, QoS settings |

### Stage Handlers

| File | Lines | Description |
|------|-------|-------------|
| `aleph/logic/processing.py` | 18-59 | Index, bulk operations |
| `aleph/logic/collections.py` | 156-190 | Reingest, reindex |
| `aleph/logic/xref.py` | 268-381 | Cross-reference, export matches |
| `aleph/logic/mapping.py` | 74-115 | Load/flush mapping |
| `aleph/logic/export.py` | 58-103 | Export search results |
| `aleph/logic/entities.py` | 56-180 | Update/prune entity |

### Indexing

| File | Lines | Description |
|------|-------|-------------|
| `aleph/index/entities.py` | 158-235 | Entity indexing, bulk operations |
| `aleph/index/util.py` | 197-217 | Bulk actions, retry logic |
| `aleph/index/xref.py` | - | Xref index operations |

---

**Document Version:** 1.0
**Last Updated:** 2025-12-09
**Contributors:** Claude Code Documentation Assistant
