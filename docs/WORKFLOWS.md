# Aleph Workflows

This document describes the key user workflows in Aleph, showing how different components and API endpoints work together to accomplish common tasks.

## Table of Contents

1. [Authentication & Registration](#authentication--registration)
2. [Collection Management](#collection-management)
3. [Document Upload & Processing](#document-upload--processing)
4. [Entity Management](#entity-management)
5. [Investigation Workflows](#investigation-workflows)
6. [Alert Management](#alert-management)
7. [Cross-Reference Workflow](#cross-reference-workflow)
8. [Data Import via Mappings](#data-import-via-mappings)
9. [Search & Discovery](#search--discovery)
10. [Profile Management & Entity Resolution](#profile-management--entity-resolution)

---

## Authentication & Registration

### User Registration Flow

**Steps**:

1. **Request Registration Code** (`POST /api/2/roles/code`)
   - User enters email address
   - System sends verification code to email
   - Code expires after configurable time

2. **Create Account** (`POST /api/2/roles`)
   - User submits code + email + password
   - System validates code
   - Creates user account with `user` role
   - Returns session token

3. **Login** (`POST /api/2/sessions/login`)
   - User provides email + password
   - System validates credentials
   - Returns session token + user object

4. **Maintain Session** (`GET /api/2/sessions`)
   - Frontend calls on page load
   - Returns current user or null if not authenticated
   - Session maintained via cookies or Authorization header

**Sequence**:
```
User → POST /roles/code (email)
     → Email sent with code
User → POST /roles (email, code, password)
     → Account created, session started
User → GET /sessions
     → Returns authenticated user
```

**Key Files**:
- `aleph/views/roles_api.py:64-105` - Registration endpoints
- `aleph/views/sessions_api.py` - Session management
- `aleph/logic/roles.py` - User creation logic

---

## Collection Management

### Creating a Collection

**Steps**:

1. **Create Collection** (`POST /api/2/collections`)
   - User specifies: label, category, languages, countries
   - System validates authz (user must have permission)
   - Creates collection with creator as admin
   - Initializes empty Elasticsearch index

2. **Configure Permissions** (`PUT /api/2/collections/:id/permissions`)
   - Add users/groups with read or write access
   - System checks collection ownership
   - Updates permission records in database

3. **Upload Documents** (see [Document Upload](#document-upload--processing))

**Sequence**:
```
User → POST /collections {"label": "My Investigation", "category": "leak"}
     → Collection created with ID 123
User → PUT /collections/123/permissions
     → Add team members with read/write access
User → POST /collections/123/ingest
     → Upload documents for processing
```

**Authorization**:
- Collection creator automatically gets write access
- Only users with write access can modify permissions
- Admin users can access all collections

**Key Files**:
- `aleph/views/collections_api.py:117-149` - Collection creation
- `aleph/views/permissions_api.py` - Permission management
- `aleph/model/collection.py` - Collection model
- `aleph/logic/collections.py` - Collection business logic

---

## Document Upload & Processing

### Document Ingestion Pipeline

**Steps**:

1. **Upload File** (`POST /api/2/collections/:id/ingest`)
   - User uploads file (PDF, Word, Excel, etc.)
   - System validates file type and size
   - Stores blob in archive (S3 or local filesystem)
   - Creates parent Document entity

2. **Queue Processing Job**
   - System queues `ingest` job to RabbitMQ
   - Job includes: collection_id, content_hash, metadata

3. **Worker Processing** (Background)
   - Worker picks up job from queue
   - Extracts metadata (title, date, author)
   - Performs OCR if needed (via Tesseract)
   - Extracts text content
   - Generates child Document entities (pages, attachments)
   - Extracts entities using FollowTheMoney

4. **Index to Elasticsearch**
   - Worker indexes document to Elasticsearch
   - Updates collection statistics
   - Marks processing as complete

5. **User Notification** (Optional)
   - If alert matches document, create notification
   - User can view document via `/entities/:id`

**Sequence**:
```
User → POST /collections/123/ingest (file: document.pdf)
     → File stored, job queued
     → Returns 202 Accepted

Worker → Pick job from queue
       → Extract text: "..." (via ingest-file)
       → OCR pages if needed
       → Extract entities
       → Index to Elasticsearch
       → Mark complete

User → GET /collections/123/status
     → Check processing progress
User → GET /entities/:doc_id
     → View processed document
```

**Processing States**:
- `pending` - Queued for processing
- `running` - Currently being processed
- `success` - Completed successfully
- `failure` - Processing failed (see error message)

**Key Files**:
- `aleph/views/ingest_api.py` - Upload endpoint
- `aleph/worker.py` - Background worker
- `aleph/ingest/` - Document processing logic
- `aleph/index/entities.py` - Elasticsearch indexing
- `aleph/logic/documents.py` - Document management

---

## Entity Management

### Creating and Editing Entities

**Steps**:

1. **Create Entity** (`POST /api/2/entities`)
   - User provides schema (Person, Company, etc.)
   - User fills in properties (name, date, addresses)
   - System validates against FollowTheMoney schema
   - Generates entity ID
   - Stores in Elasticsearch

2. **Update Entity** (`PUT /api/2/entities/:id`)
   - User modifies properties
   - System validates changes
   - Updates Elasticsearch document
   - Queues reindex job for connected entities

3. **Link Entities** (via properties)
   - Set relationship properties (e.g., `directorship` links Person → Company)
   - System validates property types
   - Creates bidirectional reference

**Sequence**:
```
User → POST /entities
     {
       "schema": "Person",
       "properties": {
         "name": ["John Doe"],
         "birthDate": ["1980-05-15"]
       },
       "collection_id": 123
     }
     → Entity created with ID "abc123"

User → PUT /entities/abc123
     {
       "properties": {
         "nationality": ["US"]
       }
     }
     → Entity updated, reindexed
```

**Validation**:
- Properties must match schema definition
- Dates must be in ISO format or fuzzy (e.g., "1980")
- Countries use ISO 3166-1 alpha-2 codes
- Related entities must exist and be accessible

**Key Files**:
- `aleph/views/entities_api.py` - Entity CRUD endpoints
- `aleph/logic/entities.py` - Entity business logic
- `aleph/index/entities.py` - Elasticsearch operations
- FollowTheMoney library - Schema validation

---

## Investigation Workflows

### Creating a Network Diagram

**Steps**:

1. **Create EntitySet** (`POST /api/2/entitysets`)
   - User creates diagram with label and summary
   - System creates EntitySet of type `diagram`
   - Initializes empty layout

2. **Add Entities** (`POST /api/2/entitysets/:id/entities`)
   - User searches for entities
   - Adds relevant entities to diagram
   - Each addition creates EntitySetItem with `positive` judgement

3. **Arrange Layout** (`PUT /api/2/entitysets/:id`)
   - User drags entities in UI
   - Frontend updates layout coordinates
   - Layout saved: `{"entities": {"entity-1": {"x": 100, "y": 200}}}`

4. **Export Diagram**
   - User exports as image (frontend rendering)
   - Or exports entities as Excel/CSV

**Sequence**:
```
User → POST /entitysets
     {
       "type": "diagram",
       "label": "Corruption Network",
       "collection": {"collection_id": 123}
     }
     → Diagram created with ID "diagram-xyz"

User → POST /entitysets/diagram-xyz/entities
     {
       "id": "person-1",
       "schema": "Person",
       "properties": {"name": ["John Doe"]}
     }
     → Entity added to diagram

User → PUT /entitysets/diagram-xyz
     {
       "layout": {
         "entities": {
           "person-1": {"x": 150, "y": 200}
         }
       }
     }
     → Layout saved
```

**EntitySet Types**:
- `list` - Simple entity list
- `diagram` - Network visualization with layout
- `timeline` - Chronological arrangement
- `profile` - Entity merging/deduplication

**Key Files**:
- `aleph/views/entitysets_api.py` - EntitySet CRUD
- `aleph/logic/entitysets.py` - EntitySet business logic
- `aleph/model/entityset.py` - EntitySet model
- `ui/src/components/Diagram/` - Frontend diagram editor

---

## Alert Management

### Setting Up Search Alerts

**Steps**:

1. **Perform Search** (`GET /api/2/search?q=Putin+offshore`)
   - User tests query to ensure it returns relevant results

2. **Create Alert** (`POST /api/2/alerts`)
   - User saves query as alert
   - System stores query text and parameters
   - Alert set to active

3. **Background Monitoring**
   - Daily cron job runs all active alerts
   - Compares new index content against alert queries
   - If new matches found, creates notification

4. **User Receives Notification** (`GET /api/2/notifications`)
   - User sees notification count in UI
   - Clicks to view matched entities
   - Can click through to view entities

5. **Manage Alert** (`DELETE /api/2/alerts/:id`)
   - User can delete alert when no longer needed

**Sequence**:
```
User → GET /search?q=Putin+AND+offshore
     → Test query, see 50 results

User → POST /alerts
     {
       "query": "Putin AND offshore",
       "query_text": "Putin AND offshore"
     }
     → Alert created with ID 42

Cron Job (daily) →
     Run all alerts against index
     → Alert 42 finds 3 new entities
     → Create notification for user

User → GET /notifications
     → See: "3 new results for 'Putin AND offshore'"
User → Click → See matched entities
```

**Notification Types**:
- `match` - New entities match alert query
- `export` - Export job completed
- `xref` - Cross-reference job completed

**Key Files**:
- `aleph/views/alerts_api.py` - Alert CRUD
- `aleph/logic/alerts.py` - Alert processing
- `aleph/logic/notifications.py` - Notification creation
- `aleph/model/alert.py` - Alert model

---

## Cross-Reference Workflow

### Finding Entity Matches Across Collections

**Steps**:

1. **Select Source Collection** (e.g., "Leaked Documents")
   - User navigates to collection

2. **Initiate Xref** (`POST /api/2/collections/123/xref`)
   - User selects target collections to match against
   - System queues background xref job
   - Returns 202 Accepted

3. **Background Processing**
   - Worker generates entity pairs
   - Extracts fingerprints (normalized names, IDs)
   - Scores pairs using ML model (GLM Bernoulli)
   - Stores matches with scores

4. **Review Matches** (`GET /api/2/collections/123/xref`)
   - User sees matched entity pairs
   - Pairs sorted by similarity score (0.0 - 1.0)
   - Default threshold: 0.5 (configurable)

5. **Make Judgements** (`POST /api/2/profiles/_pairwise`)
   - User reviews each match
   - Accepts (`positive`) → Creates/updates profile
   - Rejects (`negative`) → Marks as different entities
   - Unsure → Flags for later review

6. **View Merged Profile** (`GET /api/2/profiles/:id`)
   - System merges accepted matches into profile
   - Combined properties from all matched entities
   - Can expand to see related entities

**Sequence**:
```
User → POST /collections/123/xref
     {
       "against_collection_ids": [456, 789]
     }
     → Job queued, returns 202 Accepted

Worker → Generate pairs
       → Score using ML
       → Store 250 matches

User → GET /collections/123/xref
     → See matched pairs:
       entity-1 <-> entity-2 (score: 0.95)
       entity-3 <-> entity-4 (score: 0.87)

User → POST /profiles/_pairwise
     {
       "entity_id": "entity-1",
       "match_id": "entity-2",
       "judgement": "positive"
     }
     → Profile created, entities merged

User → GET /profiles/profile-abc
     → View merged entity with combined properties
```

**Matching Algorithm**:
1. Extract fingerprints (names, IDs, phone numbers)
2. Generate candidate pairs (only entities with shared fingerprints)
3. Score each pair using GLM Bernoulli classifier
4. Filter by threshold (default 0.5)
5. Store as matches for user review

**Key Files**:
- `aleph/views/xref_api.py` - Xref endpoints
- `aleph/logic/xref.py` - Xref job logic
- `aleph/logic/matching.py` - ML scoring
- `aleph/views/profiles_api.py` - Profile management

---

## Data Import via Mappings

### Importing Structured Data (CSV/Excel)

**Steps**:

1. **Upload Table File** (`POST /api/2/collections/:id/ingest`)
   - User uploads CSV or Excel file
   - System processes as Document entity
   - Extracts table structure

2. **Create Mapping** (`POST /api/2/collections/:id/mappings`)
   - User defines how columns map to FTM entities
   - Example: CSV column "full_name" → Person.name
   - Mapping uses FollowTheMoney mapping format

3. **Trigger Import** (`POST /api/2/collections/:id/mappings/:mapping_id/trigger`)
   - System queues mapping job
   - Sets mapping status to `pending`

4. **Background Processing**
   - Worker loads mapping and table
   - For each row, creates entities based on mapping
   - Validates against FTM schema
   - Indexes entities to Elasticsearch
   - Updates mapping status

5. **View Imported Entities**
   - User searches collection
   - Sees entities created from table rows

**Sequence**:
```
User → POST /collections/123/ingest (file: people.csv)
     → Table uploaded as entity "table-xyz"

User → POST /collections/123/mappings
     {
       "table_id": "table-xyz",
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
           }
         }
       }
     }
     → Mapping created with ID 456

User → POST /collections/123/mappings/456/trigger
     → Import job queued

Worker → Process each row
       → Create Person entities
       → Index to Elasticsearch
       → Mark mapping as successful

User → GET /search?filter:collection_id=123
     → See imported Person entities
```

**Mapping Features**:
- **Keys**: Columns that uniquely identify entities (used for deduplication)
- **Properties**: Column-to-property mappings
- **Multiple Entities**: One table can create multiple entity types
- **Relationships**: Can create links between entities in same import

**Key Files**:
- `aleph/views/mappings_api.py` - Mapping CRUD
- `aleph/logic/mappings.py` - Mapping execution
- `aleph/model/mapping.py` - Mapping model
- FollowTheMoney mapping system

---

## Search & Discovery

### Full-Text Search Workflow

**Steps**:

1. **Enter Query** (`GET /api/2/search?q=corruption`)
   - User types search terms
   - System builds Elasticsearch query
   - Applies filters (collections, schemas, countries)

2. **Execute Search**
   - Elasticsearch searches across all indexed content
   - Matches in: entity names, document text, property values
   - Applies fuzzy matching for typos
   - Ranks results by relevance

3. **Apply Facets** (filters)
   - User narrows by collection, country, entity type
   - System rebuilds query with filters
   - Returns filtered results

4. **View Entity Details** (`GET /api/2/entities/:id`)
   - User clicks entity in results
   - See full entity properties
   - View related entities
   - Download source documents

**Sequence**:
```
User → GET /search?q=Putin+offshore
     → Returns 500 results across all collections

User → Add filter: filter:schema=Company
     → GET /search?q=Putin+offshore&filter:schema=Company
     → Returns 120 Company entities

User → Add filter: filter:countries=cy
     → GET /search?q=Putin+offshore&filter:schema=Company&filter:countries=cy
     → Returns 45 Cyprus companies

User → Click entity → GET /entities/company-xyz
     → View company details, directors, documents
```

**Search Features**:
- **Fuzzy Matching**: Handles typos automatically
- **Phrase Search**: Use quotes: `"John Doe"`
- **Boolean Operators**: AND, OR, NOT
- **Wildcards**: `Doe*` matches Doe, Doerr, etc.
- **Field Search**: `name:Putin` searches only names
- **Nested Search**: Finds entities via relationships

**Key Files**:
- `aleph/search/query.py` - Query builder
- `aleph/search/parser.py` - Query parsing
- `aleph/views/search_api.py` - Search endpoint
- `aleph/index/entities.py` - Elasticsearch mapping

---

## Profile Management & Entity Resolution

### Merging Duplicate Entities

**Steps**:

1. **Find Duplicates**
   - Via cross-reference (see [Xref Workflow](#cross-reference-workflow))
   - Or manual search and comparison

2. **Make Pairwise Judgement** (`POST /api/2/profiles/_pairwise`)
   - User decides entity-1 and entity-2 are same
   - Judgement: `positive`
   - System creates or updates profile

3. **Profile Creation**
   - System creates Profile (special EntitySet type)
   - Adds both entities as items
   - Merges properties into single pseudo-entity
   - Property merging rules: combine all values

4. **View Merged Profile** (`GET /api/2/profiles/:id`)
   - See combined entity with all properties
   - View source entities (items)
   - See which properties came from which source

5. **Find Related Entities** (`GET /api/2/profiles/:id/expand`)
   - Expand profile to see adjacent entities
   - Aggregates relationships from all source entities
   - E.g., if person-1 owns company-A and person-2 owns company-B,
     profile shows both ownerships

**Sequence**:
```
User finds duplicates:
  entity-1: "John Doe" (from collection 10)
  entity-2: "J. Doe" (from collection 20)

User → POST /profiles/_pairwise
     {
       "entity_id": "entity-1",
       "match_id": "entity-2",
       "judgement": "positive"
     }
     → Profile created: "profile-abc123"

System merges:
  - name: ["John Doe", "J. Doe"]
  - birthDate: ["1980-05-15"] (from entity-1)
  - nationality: ["US", "USA"] (combined from both)

User → GET /profiles/profile-abc123
     → View merged profile

User → GET /profiles/profile-abc123/expand
     → See all companies/addresses from both source entities
```

**Use Cases**:
- **Data Integration**: Merge entities from multiple sources
- **Deduplication**: Clean up duplicate data imports
- **Entity Resolution**: Link entities across datasets
- **Investigations**: Build complete picture of person/company

**Key Files**:
- `aleph/views/profiles_api.py` - Profile endpoints
- `aleph/logic/profiles.py` - Profile merging logic
- `aleph/model/entityset.py` - EntitySet (profile is special type)
- FollowTheMoney - Property merging rules

---

## Workflow Summary

| Workflow | Primary APIs | Key Components | Duration |
|----------|-------------|----------------|----------|
| User Registration | `/roles/code`, `/roles`, `/sessions` | authz, roles | < 1 min |
| Create Collection | `/collections`, `/permissions` | collections, authz | < 1 min |
| Upload Documents | `/ingest`, `/status` | worker, archive, ES | Minutes-Hours |
| Create Entities | `/entities` | FTM, ES | < 1 min |
| Build Diagram | `/entitysets`, `/entitysets/:id/entities` | entitysets | Ongoing |
| Set Alert | `/alerts` | alerts, notifications | < 1 min |
| Cross-Reference | `/xref`, `/profiles/_pairwise` | xref, matching, profiles | Hours |
| Import Data | `/mappings`, `/mappings/:id/trigger` | worker, FTM mappings | Minutes-Hours |
| Search | `/search`, `/entities/:id` | ES | < 1 second |
| Merge Entities | `/profiles/_pairwise`, `/profiles/:id` | profiles, FTM | < 1 min |

---

## Next Steps

For detailed technical implementation:
- [Architecture Documentation](./ARCHITECTURE.md) - System design and components
- [API Reference](./API.md) - Complete endpoint documentation
- [Data Pipelines](./DATA_PIPELINES.md) - Background processing flows
- [Data Models](./MODELS.md) - Database schema reference

---

**Last Updated**: 2025-12-08
