# Document Ingestion

**Last Updated:** 2025-12-09

## Overview

Document ingestion is the process of uploading files to Aleph, extracting their content, and making them searchable. Aleph supports over 40 file formats including documents, spreadsheets, presentations, archives, emails, and images. The ingestion pipeline handles text extraction, OCR (Optical Character Recognition), metadata extraction, and full-text indexing.

### Ingestion Flow

```
Upload → Archive Storage → Queue Task → ingest-file Service →
Worker Processing → Entity Extraction → Indexing → Searchable
```

### Key Features

- **40+ file formats** - PDF, Office docs, emails, archives, images
- **Automatic OCR** - Extract text from scanned documents and images
- **Metadata extraction** - Author, dates, EXIF data, email headers
- **Content deduplication** - Same file uploaded twice stored once
- **Nested document support** - Archives, email attachments, PDF portfolios
- **Bulk upload** - CLI tools for uploading entire directories
- **Progress tracking** - Monitor upload and processing status
- **Language detection** - Automatic language identification for search

## Table of Contents

- [Architecture](#architecture)
- [Upload Methods](#upload-methods)
- [Supported Formats](#supported-formats)
- [Processing Pipeline](#processing-pipeline)
- [Data Models](#data-models)
- [API Reference](#api-reference)
- [CLI Commands](#cli-commands)
- [Configuration](#configuration)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)

---

## Architecture

### System Components

```
┌─────────────┐
│   Browser   │
└──────┬──────┘
       │ HTTP Upload
       ▼
┌─────────────┐
│   API       │ Upload endpoint: POST /api/2/collections/<id>/ingest
│   Server    │
└──────┬──────┘
       │ 1. Save to Archive (SHA1 hash)
       │ 2. Create Document record (PostgreSQL)
       │ 3. Queue task (RabbitMQ)
       ▼
┌─────────────┐
│  RabbitMQ   │ STAGE_INGEST queue
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ ingest-file │ External Docker service
│   Service   │ - Text extraction
│             │ - OCR processing
│             │ - Metadata extraction
│             │ - Entity generation
└──────┬──────┘
       │ Queues: STAGE_ANALYZE → STAGE_INDEX
       ▼
┌─────────────┐
│   Worker    │ Aleph worker process
│  (Python)   │ - Index to Elasticsearch
└──────┬──────┘
       │
       ▼
┌─────────────┐
│Elasticsearch│ Full-text search index
└─────────────┘
```

### Service Responsibilities

**API Server (`aleph/views/ingest_api.py`)**:
- Accept file uploads
- Validate metadata
- Store files in archive
- Create Document records
- Queue processing tasks

**Archive (`servicelayer`)**:
- Content-addressed storage (SHA1)
- Filesystem or S3 backend
- Automatic deduplication

**ingest-file Service** (separate Docker container):
- File format detection
- Text extraction (Apache Tika)
- OCR (Tesseract)
- Metadata extraction
- Entity generation (FollowTheMoney)

**Worker (`aleph/worker.py`)**:
- Task orchestration
- Elasticsearch indexing
- Cross-reference matching
- Status tracking

**PostgreSQL**:
- Document metadata
- Collection records
- Entity relationships

**Elasticsearch**:
- Full-text search
- Entity properties
- Faceted search

**RabbitMQ**:
- Task queue
- Multi-stage pipeline
- Priority handling

---

## Upload Methods

### 1. Web UI Upload

**Location:** Collections → Upload Documents

**Features:**
- Drag-and-drop interface
- Multiple file selection
- Metadata form (optional)
- Real-time progress tracking
- Automatic collection assignment

**Process:**
1. Select collection
2. Choose files or drag-and-drop
3. Optionally add metadata (author, date, etc.)
4. Click "Upload"
5. Monitor progress in UI

**File Size Limit:** Configured by `ALEPH_MAX_CONTENT_LENGTH` (default: unlimited)

### 2. API Upload

**Endpoint:** `POST /api/2/collections/<collection_id>/ingest`

**Authentication:** API key or session cookie required

**Request Format:** `multipart/form-data`

**Form Fields:**
- `file`: Binary file data (required)
- `meta`: JSON metadata object (optional)

**Example (cURL):**
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -F "file=@document.pdf" \
  -F 'meta={"foreign_id":"doc-123","title":"Annual Report","author":"John Smith"}' \
  https://aleph.example.org/api/2/collections/1/ingest
```

**Example (Python):**
```python
import requests

BASE_URL = "https://aleph.example.org"
API_KEY = "your-api-key"
headers = {"Authorization": f"ApiKey {API_KEY}"}

# Upload single file
with open("document.pdf", "rb") as f:
    files = {"file": f}
    data = {
        "meta": json.dumps({
            "foreign_id": "doc-123",
            "title": "Annual Report",
            "author": "John Smith",
            "date": "2023-01-15"
        })
    }
    response = requests.post(
        f"{BASE_URL}/api/2/collections/1/ingest",
        headers=headers,
        files=files,
        data=data
    )
    print(response.json())
```

**Metadata Schema:**

Available metadata fields:
```json
{
  "foreign_id": "unique-identifier",
  "title": "Document title",
  "summary": "Brief description",
  "author": "Author name",
  "publisher": "Publisher",
  "file_name": "Override filename",
  "mime_type": "Override MIME type",
  "source_url": "Original URL",
  "crawler": "Crawler name",
  "date": "2023-01-15",
  "authored_at": "2023-01-10",
  "modified_at": "2023-01-12",
  "published_at": "2023-01-15",
  "retrieved_at": "2023-01-20",
  "countries": ["US", "GB"],
  "languages": ["eng", "deu"],
  "keywords": ["finance", "audit"]
}
```

### 3. Bulk Upload (CLI)

**Command:** `aleph crawldir`

**Purpose:** Upload entire directories with nested structure

**Usage:**
```bash
aleph crawldir [OPTIONS] PATH
```

**Options:**
- `-l, --language` - ISO language codes for OCR (e.g., `eng`, `deu`, `fra`)
- `-f, --foreign_id` - Collection foreign ID (creates if doesn't exist)

**Example:**
```bash
# Upload directory with English OCR
aleph crawldir /path/to/documents -l eng

# Upload with custom collection
aleph crawldir /path/to/documents -f "investigation-2023"

# Upload with multiple OCR languages
aleph crawldir /path/to/documents -l eng -l deu -l fra
```

**Behavior:**
- Recursively processes all files and subdirectories
- Creates folder entities for directories
- Preserves directory structure
- Automatic deduplication by content hash
- Uses collection.languages for OCR if not specified

**Example Directory Structure:**
```
documents/
├── financial/
│   ├── report_2022.pdf
│   └── report_2023.pdf
└── correspondence/
    ├── email_001.msg
    └── email_002.msg
```

**Result in Aleph:**
```
Collection: documents
├── Folder: financial
│   ├── Document: report_2022.pdf
│   └── Document: report_2023.pdf
└── Folder: correspondence
    ├── Document: email_001.msg (with attachments)
    └── Document: email_002.msg (with attachments)
```

### 4. Bulk Entity Import

**Command:** `aleph load-entities`

**Purpose:** Import pre-structured entities (not files)

**Usage:**
```bash
aleph load-entities -i COLLECTION_ID FILE.json
```

**Example:**
```bash
# Import entities from JSON file
aleph load-entities -i 1 entities.json

# Import from stdin
cat entities.json | aleph load-entities -i 1 -
```

**Input Format (FTM JSON):**
```json
[
  {
    "id": "entity-123",
    "schema": "Person",
    "properties": {
      "name": ["John Smith"],
      "birthDate": ["1980-05-15"],
      "nationality": ["US"]
    }
  },
  {
    "id": "entity-456",
    "schema": "Company",
    "properties": {
      "name": ["Acme Corp"],
      "jurisdiction": ["US"],
      "incorporationDate": ["2010-01-01"]
    }
  }
]
```

**API Endpoint:** `POST /api/2/collections/<id>/_bulk`

**Example (Python):**
```python
entities = [
    {
        "id": "entity-123",
        "schema": "Person",
        "properties": {
            "name": ["John Smith"],
            "nationality": ["US"]
        }
    }
]

response = requests.post(
    f"{BASE_URL}/api/2/collections/1/_bulk",
    headers=headers,
    json=entities
)
```

---

## Supported Formats

### Documents

| Format | Extension | Text Extraction | OCR | Notes |
|--------|-----------|----------------|-----|-------|
| PDF | `.pdf` | ✅ | ✅ | Extractable text or OCR for scanned |
| Word | `.doc`, `.docx` | ✅ | ❌ | Microsoft Word |
| OpenDocument | `.odt` | ✅ | ❌ | LibreOffice/OpenOffice |
| Rich Text | `.rtf` | ✅ | ❌ | Rich Text Format |
| Plain Text | `.txt` | ✅ | ❌ | UTF-8, ASCII |
| HTML | `.html`, `.htm` | ✅ | ❌ | Web pages |
| XML | `.xml` | ✅ | ❌ | Structured data |

### Spreadsheets

| Format | Extension | Text Extraction | Tables | Notes |
|--------|-----------|----------------|--------|-------|
| Excel | `.xls`, `.xlsx` | ✅ | ✅ | Microsoft Excel |
| OpenDocument | `.ods` | ✅ | ✅ | LibreOffice Calc |
| CSV | `.csv` | ✅ | ✅ | Comma-separated values |

**Table Handling:**
- Each sheet becomes separate Table entity
- Cells indexed as text
- Structure preserved (rows/columns)

### Presentations

| Format | Extension | Text Extraction | Notes |
|--------|-----------|----------------|-------|
| PowerPoint | `.ppt`, `.pptx` | ✅ | Slide text and notes |
| OpenDocument | `.odp` | ✅ | LibreOffice Impress |

### Archives

| Format | Extension | Nested Processing | Notes |
|--------|-----------|-------------------|-------|
| ZIP | `.zip` | ✅ | Standard ZIP |
| RAR | `.rar` | ✅ | WinRAR format |
| 7-Zip | `.7z` | ✅ | 7-Zip format |
| TAR | `.tar`, `.tar.gz`, `.tgz` | ✅ | Unix TAR archives |
| GZIP | `.gz` | ✅ | GNU Zip |

**Archive Processing:**
- Automatically extracts all contents
- Creates child Document entities for each file
- Recursive extraction (ZIP inside ZIP)
- Preserves directory structure

### Email

| Format | Extension | Attachments | Headers | Notes |
|--------|-----------|-------------|---------|-------|
| PST | `.pst` | ✅ | ✅ | Outlook data file |
| MBOX | `.mbox` | ✅ | ✅ | Unix mailbox |
| EML | `.eml` | ✅ | ✅ | Email message |
| MSG | `.msg` | ✅ | ✅ | Outlook message |

**Email Metadata Extracted:**
- From, To, CC, BCC
- Subject, Date
- Message-ID, In-Reply-To
- Attachments (processed recursively)

### Images

| Format | Extension | OCR | EXIF | Notes |
|--------|-----------|-----|------|-------|
| JPEG | `.jpg`, `.jpeg` | ✅ | ✅ | Photographs |
| PNG | `.png` | ✅ | ❌ | Screenshots, graphics |
| TIFF | `.tiff`, `.tif` | ✅ | ✅ | High-quality scans |
| BMP | `.bmp` | ✅ | ❌ | Windows bitmap |
| GIF | `.gif` | ✅ | ❌ | Animated images |

**EXIF Data Extracted:**
- Camera make/model
- GPS coordinates
- Date/time taken
- Copyright information

### Other Formats

| Format | Extension | Notes |
|--------|-----------|-------|
| Outlook | `.pst`, `.ost` | Email archives |
| iCalendar | `.ics` | Calendar events |
| vCard | `.vcf` | Contact cards |
| DjVu | `.djvu` | Scanned documents |

---

## Processing Pipeline

### Stage 1: Upload & Archive

**Location:** `aleph/views/ingest_api.py:77-152`

**Process:**
1. **Authentication**: Verify API key or session
2. **Authorization**: Check WRITE permission on collection
3. **Validation**: Validate metadata schema
4. **Temporary Storage**: Save upload to `/tmp`
5. **Hash Calculation**: Compute SHA1 content hash
6. **Archive**: Store in content-addressed archive (S3 or filesystem)
7. **Database**: Create Document record with metadata
8. **Queue**: Send to STAGE_INGEST queue

**Content Hash:**
- SHA1 hash of file contents (64-character hex)
- Used for deduplication
- Archive key: `/<hash[0:2]>/<hash[2:4]>/<hash>`

**Example Archive Paths:**
```
Filesystem: /data/ab/cd/abcdef1234567890...
S3: s3://bucket/abcdef1234567890...
```

### Stage 2: Ingestion (ingest-file)

**Service:** External Docker container (`ghcr.io/alephdata/ingest-file:4.1.2`)

**Responsibilities:**
- File format detection (MIME type analysis)
- Text extraction (Apache Tika)
- OCR for images/scanned PDFs (Tesseract)
- Metadata extraction
- Nested file processing (archives, attachments)
- Entity generation (FollowTheMoney format)

**Process:**
1. **Pull Task**: Retrieve from STAGE_INGEST queue
2. **Load File**: Fetch from archive by content_hash
3. **Detect Format**: Identify file type (MIME)
4. **Route to Ingestor**: Select appropriate parser
5. **Extract Content**: Parse file and extract text
6. **OCR (if needed)**: Run Tesseract on images
7. **Generate Entities**: Create FTM entities
8. **Queue Results**: Send to STAGE_ANALYZE

**Supported Ingestors:**
- `PDFIngestor` - PDF documents
- `DocumentIngestor` - Office files (Word, Excel, PowerPoint)
- `EmailIngestor` - Email formats (PST, MBOX, EML, MSG)
- `ArchiveIngestor` - Archives (ZIP, RAR, 7Z, TAR)
- `ImageIngestor` - Image files with OCR
- `TablesIngestor` - Spreadsheets and CSVs
- `PlainTextIngestor` - Text files

**OCR Process:**
1. Check if text is extractable
2. If no text or low-quality, trigger OCR
3. Run Tesseract with configured languages
4. Extract text from image/scanned PDF
5. Store extracted text in entity

**OCR Languages:**
- Configured via collection.languages
- Fallback to `ALEPH_OCR_DEFAULTS` (default: `eng`)
- Multiple languages: `eng,deu,fra` (English, German, French)
- Tesseract supports 100+ languages

### Stage 3: Analysis (ingest-file)

**Process:**
1. **Language Detection**: Identify document language
2. **Entity Extraction**: Extract named entities (NER)
3. **Metadata Enhancement**: Add computed metadata
4. **Fragment Generation**: Create searchable fragments
5. **Queue Indexing**: Send to STAGE_INDEX

**Extracted Entities:**
- **Pages** (for PDFs) - Each page as separate entity
- **Tables** (for spreadsheets) - Each sheet as Table entity
- **Emails** - Email message entities with headers
- **Attachments** - Nested file entities

### Stage 4: Indexing (Aleph Worker)

**Location:** `aleph/worker.py:169-216`, `aleph/index/entities.py:158-175`

**Process:**
1. **Pull Tasks**: Worker retrieves from STAGE_INDEX queue
2. **Batch Tasks**: Collect up to 100 entities (configurable)
3. **Format Entities**: Convert to Elasticsearch format
4. **Bulk Index**: Send batch to Elasticsearch
5. **Update Stats**: Update collection document count

**Elasticsearch Document Structure:**
```json
{
  "id": "entity-id-signed",
  "collection_id": 1,
  "schema": "Document",
  "properties": {
    "title": ["Annual Report 2023"],
    "contentHash": ["abcdef123456..."],
    "fileName": ["report.pdf"],
    "mimeType": ["application/pdf"],
    "author": ["John Smith"],
    "date": ["2023-01-15"],
    "languages": ["eng"],
    "countries": ["US"]
  },
  "text": ["Full extracted text content..."],
  "created_at": "2023-01-15T10:00:00Z",
  "updated_at": "2023-01-15T10:05:00Z"
}
```

**Indexing Configuration:**
- **Batch Size**: `ALEPH_INDEXING_BATCH_SIZE` (default: 100)
- **Timeout**: `ALEPH_INDEXING_TIMEOUT` (default: 10 seconds)
- **Replicas**: `ALEPH_INDEX_REPLICAS` (default: 0)

### Stage 5: Post-Processing

**Optional Stages:**

**Cross-Reference Matching (`STAGE_XREF`):**
- Triggered manually or on collection completion
- Compares entities across collections
- Generates match candidates with scores
- See [XREF.md](./XREF.md) for details

**Collection Statistics:**
- Document count
- Entity count by schema
- Processing status
- Storage size

---

## Data Models

### Document Model

**File:** `aleph/model/document.py`

**Schema:**
```python
class Document(db.Model, DatedModel):
    SCHEMA = "Document"
    SCHEMA_FOLDER = "Folder"
    SCHEMA_TABLE = "Table"

    # Primary key
    id = db.Column(db.BigInteger, primary_key=True)

    # Content identification
    content_hash = db.Column(db.Unicode(65), nullable=True, index=True)  # SHA1
    foreign_id = db.Column(db.Unicode, unique=False, nullable=True, index=True)

    # Schema type
    schema = db.Column(db.String(255), nullable=False)  # Document, Folder, Table

    # Metadata (JSONB for flexibility)
    meta = db.Column(JSONB, default={})

    # Relationships
    role_id = db.Column(db.Integer, db.ForeignKey("role.id"), nullable=True)
    parent_id = db.Column(db.BigInteger, nullable=True, index=True)  # For nested docs
    collection_id = db.Column(db.Integer, db.ForeignKey("collection.id"), nullable=False)

    # Timestamps
    created_at = db.Column(db.DateTime)
    updated_at = db.Column(db.DateTime)
```

**Key Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `id` | BigInteger | Auto-incrementing primary key |
| `content_hash` | String(65) | SHA1 hash for deduplication |
| `foreign_id` | String | External/user-provided identifier |
| `schema` | String | Document type (Document, Folder, Table) |
| `meta` | JSONB | Flexible metadata storage |
| `role_id` | Integer | Uploader user ID |
| `parent_id` | BigInteger | Parent document (for nested files) |
| `collection_id` | Integer | Collection foreign key |

**Metadata Properties (stored in `meta` JSONB):**
- `title` - Document title
- `summary` - Brief description
- `author` - Author name
- `publisher` - Publisher
- `file_name` - Original filename
- `mime_type` - MIME type (e.g., `application/pdf`)
- `source_url` - Original URL
- `crawler` - Crawler that fetched the document
- `date` - General date field
- `authored_at` - Date authored/created
- `modified_at` - Date last modified
- `published_at` - Date published
- `retrieved_at` - Date retrieved/crawled
- `languages` - Array of ISO language codes
- `countries` - Array of ISO country codes
- `keywords` - Array of keywords/tags

**Key Methods:**

**`Document.save()`** - Create or update document:
```python
document = Document.save(
    collection=collection,
    parent=parent_document,  # Optional
    foreign_id="doc-123",
    content_hash="abcdef123...",
    meta={"title": "Report", "author": "John"},
    role_id=user_id
)
```

**`Document.to_proxy()`** - Convert to FollowTheMoney entity:
```python
proxy = document.to_proxy(ns=collection.ns)
# Returns followthemoney.proxy.EntityProxy
```

### Collection Model

**File:** `aleph/model/collection.py`

**Key Properties:**
```python
id = db.Column(db.Integer, primary_key=True)
foreign_id = db.Column(db.Unicode, unique=True)
label = db.Column(db.Unicode, nullable=False)
category = db.Column(db.Enum(...))  # news, leak, casefile, etc.
casefile = db.Column(db.Boolean, default=False)
languages = db.Column(ARRAY(db.Unicode(3)))  # ISO 639-2 codes
ns = db.Column(db.Unicode)  # Namespace for entity ID signing
```

**OCR Language Configuration:**
```python
# Set OCR languages for collection
collection.languages = ["eng", "deu", "fra"]
db.session.commit()

# These languages will be used for OCR on all uploaded documents
```

---

## API Reference

### Upload Document

**Endpoint:** `POST /api/2/collections/<collection_id>/ingest`

**Description:** Upload a file to a collection for processing.

**Authentication:** Required (API key or session)

**Authorization:** WRITE permission on collection

**Request:**
- **Content-Type:** `multipart/form-data`
- **Form Fields:**
  - `file`: Binary file data (required)
  - `meta`: JSON metadata object (optional)

**Query Parameters:**
- `sync` (boolean) - Wait for processing to complete (default: false)
- `index` (boolean) - Index document after processing (default: true)

**Example Request:**
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -F "file=@document.pdf" \
  -F 'meta={"title":"Report","author":"John Smith"}' \
  "https://aleph.example.org/api/2/collections/1/ingest"
```

**Response:**
```json
{
  "status": "ok",
  "id": "entity-id-signed"
}
```

**Status Code:** `201 Created`

**Notes:**
- File size limited by `ALEPH_MAX_CONTENT_LENGTH` (default: unlimited)
- Duplicate files (same content hash) create new Document record but don't re-store
- `sync=true` blocks until processing completes (use for small files only)

### Bulk Import Entities

**Endpoint:** `POST /api/2/collections/<collection_id>/_bulk`

**Description:** Bulk import pre-structured entities (not files).

**Authentication:** Required

**Authorization:**
- WRITE permission on collection
- `bulk_import` permission (admin-only by default)

**Request Body:** Array of FollowTheMoney entities (JSON)

**Query Parameters:**
- `safe` (boolean) - Remove content hashes (security, default: true)
- `clean` (boolean) - Validate entity properties (default: true)
- `mutable` (boolean) - Allow UI editing (default: false)

**Example Request:**
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "id": "entity-123",
      "schema": "Person",
      "properties": {
        "name": ["John Smith"],
        "nationality": ["US"]
      }
    }
  ]' \
  "https://aleph.example.org/api/2/collections/1/_bulk"
```

**Response:** `204 No Content`

**Notes:**
- For programmatic data import (not file upload)
- Entities must conform to FollowTheMoney schema
- See [ENTITIES.md](./ENTITIES.md) for entity structure

### Get Processing Status

**Endpoint:** `GET /api/2/collections/<collection_id>/status`

**Description:** Get upload and processing status for a collection.

**Authentication:** Required

**Authorization:** READ permission on collection

**Response:**
```json
{
  "pending": 150,
  "running": 5,
  "finished": 5000,
  "jobs": [
    {
      "job_id": "uuid-123",
      "stages": [
        {
          "stage": "ingest",
          "pending": 10,
          "running": 2,
          "finished": 100
        },
        {
          "stage": "index",
          "pending": 5,
          "running": 1,
          "finished": 105
        }
      ]
    }
  ]
}
```

**Status Fields:**
- `pending` - Tasks waiting in queue
- `running` - Tasks currently processing
- `finished` - Tasks completed

**Usage:**
```python
response = requests.get(
    f"{BASE_URL}/api/2/collections/1/status",
    headers=headers
)
status = response.json()
print(f"Pending: {status['pending']}, Running: {status['running']}")
```

### Reingest Collection

**Endpoint:** `POST /api/2/collections/<collection_id>/reingest`

**Description:** Re-process all documents in collection (useful after OCR config changes).

**Authentication:** Required

**Authorization:** WRITE permission on collection

**Query Parameters:**
- `index` (boolean) - Re-index after processing (default: true)

**Response:**
```json
{
  "status": "ok"
}
```

**Use Cases:**
- OCR language configuration changed
- ingest-file service updated with better extraction
- Fix documents that failed processing
- Refresh extracted entities

**Example:**
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.org/api/2/collections/1/reingest"
```

---

## CLI Commands

### aleph crawldir

**Purpose:** Upload entire directory structure to a collection.

**Usage:**
```bash
aleph crawldir [OPTIONS] PATH
```

**Options:**
- `-l, --language TEXT` - ISO 639-2 language codes for OCR (multiple allowed)
- `-f, --foreign_id TEXT` - Collection foreign ID (creates if doesn't exist)

**Examples:**

**Basic upload:**
```bash
aleph crawldir /data/documents
```

**With OCR languages:**
```bash
aleph crawldir /data/documents -l eng -l deu
```

**Custom collection:**
```bash
aleph crawldir /data/panama-papers -f "panama-papers-2023"
```

**Behavior:**
- Creates collection if foreign_id doesn't exist
- Recursively processes all files and subdirectories
- Preserves directory hierarchy (folders become Folder entities)
- Automatic content deduplication by SHA1 hash
- Uses `file_name` from path, preserves structure in `foreign_id`

**Output:**
```
INFO Starting crawl of /data/documents...
INFO Crawl [1]: /data/documents/file1.pdf -> 123
INFO Crawl [1]: /data/documents/file2.docx -> 124
INFO Crawl [1]: /data/documents/subdir/file3.pdf -> 125
INFO Complete. Make sure a worker is running :)
```

**Requirements:**
- Aleph CLI installed (`pip install alephclient`)
- Valid Aleph configuration (ALEPH_HOST, ALEPH_API_KEY)
- Worker must be running to process queued documents

### aleph load-entities

**Purpose:** Bulk import structured entities from JSON file.

**Usage:**
```bash
aleph load-entities -i COLLECTION_ID FILE
```

**Options:**
- `-i, --infile INTEGER` - Collection ID (required)
- `FILE` - JSON file path or `-` for stdin

**Examples:**

**From file:**
```bash
aleph load-entities -i 1 entities.json
```

**From stdin:**
```bash
cat entities.json | aleph load-entities -i 1 -
```

**Input Format:**
```json
[
  {
    "id": "person-1",
    "schema": "Person",
    "properties": {
      "name": ["John Smith"],
      "birthDate": ["1980-05-15"],
      "nationality": ["US"]
    }
  },
  {
    "id": "company-1",
    "schema": "Company",
    "properties": {
      "name": ["Acme Corp"],
      "jurisdiction": ["US"]
    }
  }
]
```

**Use Cases:**
- Import data from external systems
- Load FollowTheMoney entities from other tools
- Migrate data between Aleph instances
- Bulk entity creation

### Worker Commands

**Start worker:**
```bash
aleph worker
```

**Run in development (single-threaded):**
```bash
aleph worker --debug
```

**Environment Variables:**
- `WORKER_THREADS` - Number of concurrent threads (default: 8)
- `ALEPH_WORKER_STAGES` - Comma-separated stages to process

**Example (only index documents):**
```bash
ALEPH_WORKER_STAGES=index aleph worker
```

---

## Configuration

### Environment Variables

**Core Settings:**
```bash
# Archive Storage
ARCHIVE_TYPE=file  # or 's3'
ARCHIVE_PATH=/data
# For S3:
# ARCHIVE_BUCKET=my-aleph-docs
# AWS_REGION=us-east-1
# AWS_ACCESS_KEY_ID=xxx
# AWS_SECRET_ACCESS_KEY=xxx

# OCR Configuration
ALEPH_OCR_DEFAULTS=eng  # Default Tesseract languages (comma-separated)

# Upload Limits
ALEPH_MAX_CONTENT_LENGTH=0  # Max upload size in bytes (0 = unlimited)

# Worker Configuration
WORKER_THREADS=8  # Concurrent worker threads
ALEPH_INDEXING_BATCH_SIZE=100  # Entities indexed per batch
ALEPH_INDEXING_TIMEOUT=10  # Indexing timeout in seconds

# Queue Configuration
RABBITMQ_QOS_INDEX_QUEUE=100  # Prefetch count for indexing
RABBITMQ_QOS_REINGEST_QUEUE=1  # Prefetch count for reingestion
```

**ingest-file Service:**
```yaml
# docker-compose.yml
ingest-file:
  image: ghcr.io/alephdata/ingest-file:4.1.2
  tmpfs:
    - /tmp:mode=777
  volumes:
    - archive-data:/data
  env_file:
    - aleph.env
  environment:
    # Inherits: ARCHIVE_TYPE, ARCHIVE_PATH, FTM_STORE_URI
    # Add custom settings here
```

### OCR Language Configuration

**System-wide Default:**
```bash
# aleph.env
ALEPH_OCR_DEFAULTS=eng,deu,fra
```

**Per-Collection (via API):**
```python
# Set languages for specific collection
response = requests.put(
    f"{BASE_URL}/api/2/collections/1",
    headers=headers,
    json={"languages": ["eng", "ara", "rus"]}
)
```

**Supported Languages** (100+ via Tesseract):
- `eng` - English
- `deu` - German
- `fra` - French
- `spa` - Spanish
- `rus` - Russian
- `ara` - Arabic
- `zho` - Chinese
- `jpn` - Japanese
- Many more...

**Install Additional Languages:**
```bash
# In ingest-file container
apt-get update
apt-get install tesseract-ocr-<lang>
```

### Storage Configuration

**Filesystem Storage:**
```bash
ARCHIVE_TYPE=file
ARCHIVE_PATH=/data
```
- Files stored: `/data/<hash[0:2]>/<hash[2:4]>/<hash>`
- Requires persistent volume
- Simple setup, good for development

**S3 Storage (Recommended for Production):**
```bash
ARCHIVE_TYPE=s3
ARCHIVE_BUCKET=my-aleph-documents
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=AKIAXXXXXXXX
AWS_SECRET_ACCESS_KEY=xxxxxxxxx
```
- Scalable and durable
- Supports pre-signed URLs for direct download
- Automatic redundancy
- Cost-effective for large datasets

**Google Cloud Storage:**
```bash
ARCHIVE_TYPE=gs
ARCHIVE_BUCKET=my-aleph-documents
GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json
```

### Performance Tuning

**Worker Configuration:**
```bash
# Number of worker threads
WORKER_THREADS=16  # Increase for more parallelism

# Indexing batch size
ALEPH_INDEXING_BATCH_SIZE=200  # Higher = more throughput, more memory

# Queue prefetch counts
RABBITMQ_QOS_INDEX_QUEUE=200  # More tasks prefetched
```

**Elasticsearch:**
```bash
# Timeout for indexing operations
ELASTICSEARCH_TIMEOUT=120  # Increase for slow clusters

# Index replicas (for redundancy)
ALEPH_INDEX_REPLICAS=1  # 0 for dev, 1+ for production
```

**ingest-file Resources:**
```yaml
# docker-compose.yml
ingest-file:
  deploy:
    resources:
      limits:
        cpus: '4'
        memory: 8G
```

---

## Monitoring

### Processing Status

**UI Dashboard:**
1. Navigate to Collection
2. Click "Status" tab
3. View:
   - Total documents
   - Processing status (pending/running/finished)
   - Stage breakdown (ingest/analyze/index)
   - Job progress by job_id

**API Monitoring:**
```python
import requests
import time

def monitor_collection(collection_id):
    while True:
        response = requests.get(
            f"{BASE_URL}/api/2/collections/{collection_id}/status",
            headers=headers
        )
        status = response.json()

        print(f"Pending: {status['pending']}, "
              f"Running: {status['running']}, "
              f"Finished: {status['finished']}")

        if status['pending'] == 0 and status['running'] == 0:
            print("Processing complete!")
            break

        time.sleep(5)

monitor_collection(1)
```

### Worker Logs

**View worker output:**
```bash
# Docker Compose
docker-compose logs -f worker

# Kubernetes
kubectl logs -f deployment/aleph-worker

# Systemd
journalctl -u aleph-worker -f
```

**Log Levels:**
```bash
# Set log level
ALEPH_LOG_LEVEL=DEBUG  # DEBUG, INFO, WARNING, ERROR
```

**Structured Logging (JSON):**
```bash
ALEPH_LOG_JSON=true
```

**Example Log Entry:**
```json
{
  "timestamp": "2023-01-15T10:30:45.123Z",
  "level": "INFO",
  "message": "Task [collection:1]: op:index task_id:abc123 priority:5 (done)",
  "collection_id": 1,
  "task_id": "abc123",
  "operation": "index"
}
```

### Queue Monitoring

**RabbitMQ Management UI:**
- URL: `http://localhost:15672`
- Username: `guest`
- Password: `guest`

**View Queues:**
- `ingest` - Documents waiting for ingest-file
- `index` - Entities waiting for indexing
- `xref` - Collections waiting for cross-reference

**Key Metrics:**
- **Ready** - Tasks waiting to be processed
- **Unacked** - Tasks currently being processed
- **Total** - Lifetime task count
- **Rate** - Tasks per second

**CLI Monitoring:**
```bash
# List queues
docker-compose exec rabbitmq rabbitmqctl list_queues

# Output:
# Listing queues for vhost / ...
# ingest  150
# index   25
# xref    0
```

### Collection Statistics

**API Endpoint:** `GET /api/2/collections/<id>`

**Response includes:**
```json
{
  "id": 1,
  "label": "My Collection",
  "count": 1523,  // Total indexed entities
  "statistics": {
    "schema": {
      "Document": 1200,
      "Folder": 50,
      "Table": 30,
      "Email": 243
    }
  },
  "status": "active"
}
```

**Compute Statistics:**
```bash
# Recalculate all collection statistics
aleph update-collection-stats
```

---

## Troubleshooting

### Documents Not Processing

**Symptoms:** Documents uploaded but not appearing in search.

**Diagnosis:**

1. **Check Upload Success:**
```bash
# Check if Document record created
docker-compose exec api aleph shell
>>> from aleph.model import Document
>>> docs = Document.query.filter_by(collection_id=1).all()
>>> print(f"Found {len(docs)} documents")
```

2. **Check Queue Status:**
```bash
docker-compose exec rabbitmq rabbitmqctl list_queues

# Should show tasks in 'ingest' queue
```

3. **Check Worker Running:**
```bash
docker-compose ps worker

# Should show 'Up' status
```

4. **Check Worker Logs:**
```bash
docker-compose logs worker | grep ERROR
```

**Solutions:**

**Worker not running:**
```bash
docker-compose up -d worker
```

**Worker crashed:**
```bash
docker-compose restart worker
```

**Tasks stuck in queue:**
```bash
# Restart worker
docker-compose restart worker

# If still stuck, restart RabbitMQ (CAUTION: may lose in-flight tasks)
docker-compose restart rabbitmq
```

### OCR Not Working

**Symptoms:** Scanned PDFs or images not searchable.

**Diagnosis:**

1. **Check OCR Languages Configured:**
```python
# Via API
response = requests.get(f"{BASE_URL}/api/2/collections/1", headers=headers)
print(response.json().get('languages'))
# Should return: ['eng', 'deu', etc.]
```

2. **Check ingest-file Logs:**
```bash
docker-compose logs ingest-file | grep -i tesseract
```

3. **Verify Tesseract Installed:**
```bash
docker-compose exec ingest-file tesseract --version
docker-compose exec ingest-file tesseract --list-langs
```

**Solutions:**

**Languages not configured:**
```python
# Set via API
requests.put(
    f"{BASE_URL}/api/2/collections/1",
    headers=headers,
    json={"languages": ["eng"]}
)

# Then reingest
requests.post(f"{BASE_URL}/api/2/collections/1/reingest", headers=headers)
```

**Tesseract language pack missing:**
```bash
# Install in ingest-file container
docker-compose exec ingest-file apt-get update
docker-compose exec ingest-file apt-get install -y tesseract-ocr-deu
docker-compose restart ingest-file
```

**Reingest documents:**
```bash
curl -X POST \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  "https://aleph.example.org/api/2/collections/1/reingest"
```

### Upload Fails with 413 Error

**Symptoms:** `413 Payload Too Large` error on upload.

**Cause:** File size exceeds web server limit.

**Solution:**

**Nginx Configuration:**
```nginx
# nginx.conf
client_max_body_size 500M;  # Increase limit
```

**Aleph Configuration:**
```bash
# aleph.env
ALEPH_MAX_CONTENT_LENGTH=524288000  # 500MB in bytes
```

**Restart Services:**
```bash
docker-compose restart nginx api
```

### Archive Storage Full

**Symptoms:** Upload fails with storage error.

**Diagnosis:**
```bash
# Check disk space
df -h /data

# Check archive directory
du -sh /data/*
```

**Solutions:**

**Free up space:**
```bash
# Clean up dangling files (no longer referenced)
aleph cleanup-archive

# This removes files not in database or index
```

**Expand storage:**
```bash
# For filesystem: mount larger volume
# For S3: increase bucket quota (usually unlimited)
```

**Switch to S3:**
```bash
# Update aleph.env
ARCHIVE_TYPE=s3
ARCHIVE_BUCKET=my-bucket
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx

# Migrate existing files (custom script needed)
```

### ingest-file Service Crashes

**Symptoms:** Documents stuck in "ingest" stage, ingest-file container not running.

**Diagnosis:**
```bash
docker-compose ps ingest-file
docker-compose logs ingest-file | tail -100
```

**Common Causes:**

**Out of Memory:**
```bash
# Check logs for OOMKilled
docker-compose logs ingest-file | grep -i "killed"

# Solution: Increase memory limit
# docker-compose.yml
ingest-file:
  deploy:
    resources:
      limits:
        memory: 8G  # Increase from default 4G
```

**Corrupted File:**
```bash
# Find problematic file in logs
docker-compose logs ingest-file | grep -i "error"

# Delete problematic Document
docker-compose exec api aleph shell
>>> from aleph.model import Document
>>> doc = Document.query.get(123)
>>> db.session.delete(doc)
>>> db.session.commit()
```

**Restart Service:**
```bash
docker-compose restart ingest-file
```

### Slow Processing

**Symptoms:** Documents taking hours to process.

**Diagnosis:**

1. **Check Queue Backlog:**
```bash
docker-compose exec rabbitmq rabbitmqctl list_queues
# Large numbers indicate bottleneck
```

2. **Check Worker Load:**
```bash
docker stats worker
# High CPU/memory indicates worker bottleneck
```

3. **Check ingest-file Load:**
```bash
docker stats ingest-file
# High CPU indicates processing bottleneck
```

**Solutions:**

**Scale Workers:**
```bash
# docker-compose.yml
worker:
  deploy:
    replicas: 3  # Run 3 worker instances

# Or increase threads per worker
# aleph.env
WORKER_THREADS=16  # Increase from default 8
```

**Scale ingest-file:**
```bash
# docker-compose.yml
ingest-file:
  deploy:
    replicas: 2  # Run 2 ingest-file instances
    resources:
      limits:
        cpus: '4'  # Allocate more CPU
        memory: 8G
```

**Optimize Indexing:**
```bash
# aleph.env
ALEPH_INDEXING_BATCH_SIZE=200  # Increase batch size
RABBITMQ_QOS_INDEX_QUEUE=200  # Prefetch more tasks
```

**Check Elasticsearch Performance:**
```bash
# Check cluster health
curl http://localhost:9200/_cluster/health?pretty

# Optimize index
curl -X POST http://localhost:9200/aleph-v4-*/_forcemerge?max_num_segments=1
```

### Missing Text in Search

**Symptoms:** Document indexed but text not searchable.

**Diagnosis:**

1. **Check Elasticsearch Document:**
```bash
# Get document from index
curl "http://localhost:9200/aleph-v4-*/_search?q=id:YOUR_ENTITY_ID" | jq .
```

2. **Check `text` field:**
```json
{
  "properties": {...},
  "text": ["Should contain extracted text here"]
}
```

**Causes:**

**Text extraction failed:**
- Encrypted PDF (password-protected)
- Corrupted file
- Unsupported format variant

**OCR not triggered:**
- Image/scanned PDF but no OCR languages configured
- OCR failed silently

**Solutions:**

**Re-upload with OCR:**
```python
# Set collection languages
requests.put(
    f"{BASE_URL}/api/2/collections/1",
    headers=headers,
    json={"languages": ["eng"]}
)

# Reingest
requests.post(f"{BASE_URL}/api/2/collections/1/reingest", headers=headers)
```

**Check ingest-file logs:**
```bash
docker-compose logs ingest-file | grep -i "error\|fail"
```

### Duplicate Documents

**Symptoms:** Same file uploaded multiple times creates duplicate entries.

**Explanation:** This is expected behavior.

**How It Works:**
- Same content_hash = archive stores only once (deduplication)
- Different foreign_id or parent = separate Document records
- Saves storage, but creates multiple searchable entries

**To Avoid Duplicates:**

**Use consistent foreign_id:**
```python
# Always use same foreign_id for same source
requests.post(
    f"{BASE_URL}/api/2/collections/1/ingest",
    headers=headers,
    files={"file": file},
    data={"meta": json.dumps({"foreign_id": "unique-id-123"})}
)
```

**crawldir deduplication:**
```bash
# crawldir uses file path as foreign_id
# Re-running on same directory won't create duplicates
aleph crawldir /data/documents -f "my-collection"
```

**Find Duplicates:**
```python
# Query for duplicate content hashes
from aleph.model import Document
from sqlalchemy import func

duplicates = db.session.query(
    Document.content_hash,
    func.count(Document.id).label('count')
).group_by(
    Document.content_hash
).having(
    func.count(Document.id) > 1
).all()

for hash, count in duplicates:
    print(f"Hash {hash}: {count} documents")
```

---

## Best Practices

### 1. Organize Collections

**Strategy:**
- **One collection per source** - Separate "Leaked Emails" from "Financial Records"
- **Use categories** - Set appropriate collection.category (leak, news, casefile)
- **Set languages** - Configure collection.languages for OCR accuracy
- **Descriptive labels** - "Panama Papers 2023" not "Collection 1"

### 2. Metadata Enrichment

**Always provide:**
- `foreign_id` - For deduplication and external references
- `title` - Override filename with meaningful title
- `date` or `authored_at` - Enable timeline analysis
- `author` - Track document sources
- `source_url` - Link back to original

**Example:**
```python
meta = {
    "foreign_id": f"leak-{source_id}-{doc_id}",
    "title": "Board Meeting Minutes - Q4 2023",
    "author": "Corporate Secretary",
    "date": "2023-12-15",
    "source_url": "https://source.org/docs/123",
    "keywords": ["board", "minutes", "financial"]
}
```

### 3. Bulk Upload Strategy

**For large datasets:**

1. **Use crawldir** - Faster than UI upload
2. **Set OCR languages** - Before upload, not after
3. **Monitor progress** - Use status API endpoint
4. **Scale workers** - Add workers for large batches
5. **Upload off-peak** - Reduce impact on users

**Example Workflow:**
```bash
# 1. Prepare collection
curl -X POST \
  -H "Authorization: ApiKey $API_KEY" \
  -d '{"foreign_id":"leak-2023","label":"Leak 2023","languages":["eng","deu"]}' \
  $ALEPH_URL/api/2/collections

# 2. Start upload
aleph crawldir /data/leak-2023 -f "leak-2023"

# 3. Monitor progress
while true; do
  curl -s "$ALEPH_URL/api/2/collections/1/status" | jq .pending
  sleep 10
done
```

### 4. OCR Optimization

**Language Selection:**
- Be specific: `eng` for English, not generic
- Use multiple: `eng,deu,fra` for multilingual documents
- Limit count: 3-4 languages max for performance

**When to Use OCR:**
- Scanned documents
- Photographs of documents
- Screenshots
- Historical archives

**When to Skip:**
- Native digital PDFs (already have text)
- Office documents (DOCX, XLSX)
- Emails (text-based)

### 5. Storage Management

**Production Setup:**
- **Use S3** - Scalable, durable, cost-effective
- **Enable versioning** - Protect against accidental deletion
- **Set lifecycle** - Auto-archive old files to Glacier
- **Monitor costs** - Track storage growth

**Cleanup Strategy:**
```bash
# Periodically clean dangling files
aleph cleanup-archive

# Delete old exports
aleph delete-expired-exports

# Archive old collections (mark as deleted)
```

### 6. Performance Monitoring

**Key Metrics:**
- Queue depth (RabbitMQ)
- Processing rate (documents/hour)
- Index size (Elasticsearch)
- Storage size (S3/filesystem)
- Worker CPU/memory usage

**Alerting:**
```bash
# Check if queue is growing
QUEUE_SIZE=$(docker-compose exec rabbitmq rabbitmqctl list_queues | awk '{sum += $2} END {print sum}')
if [ $QUEUE_SIZE -gt 1000 ]; then
  echo "ALERT: Queue backlog is $QUEUE_SIZE"
fi
```

### 7. Error Handling

**Strategy:**
- Monitor worker logs for errors
- Set up alerts for crashes
- Keep failed documents for retry
- Document common issues

**Retry Failed Documents:**
```bash
# Reingest collection (retries all)
curl -X POST \
  -H "Authorization: ApiKey $API_KEY" \
  "$ALEPH_URL/api/2/collections/1/reingest"
```

---

## Related Documentation

- [Collections](./COLLECTIONS.md) - Collection management and permissions
- [Search](./SEARCH.md) - Searching uploaded documents
- [Entities](./ENTITIES.md) - Understanding document entities
- [Workers](./WORKERS.md) - Worker architecture and task processing
- [API Reference](./API.md) - Complete API documentation
- [Configuration](./CONFIGURATION.md) - Environment variables and settings

---

## File References

### Backend Files

| File | Lines | Description |
|------|-------|-------------|
| `aleph/views/ingest_api.py` | 77-152 | Upload endpoint implementation |
| `aleph/model/document.py` | 19-231 | Document model and methods |
| `aleph/logic/documents.py` | 14-61 | Document processing logic, crawldir |
| `aleph/logic/processing.py` | 18-59 | Bulk import and indexing |
| `aleph/worker.py` | 109-280 | Worker task processing |
| `aleph/queues.py` | 25-79 | Task queuing and status |
| `aleph/index/entities.py` | 158-235 | Elasticsearch indexing |
| `aleph/views/archive_api.py` | 13-60 | Archive file retrieval |
| `aleph/logic/archive.py` | 21-46 | Archive cleanup |
| `aleph/manage.py` | 152-166 | crawldir CLI command |

### Configuration Files

| File | Description |
|------|-------------|
| `aleph.env.tmpl` | Environment variable template |
| `docker-compose.yml` | Service configuration |
| `aleph/validation/schema/ingest.yml` | Upload metadata schema |

---

**Document Version:** 1.0
**Last Updated:** 2025-12-09
**Contributors:** Claude Code Documentation Assistant
