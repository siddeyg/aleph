# Aleph Command Reference

Complete reference guide for all Aleph CLI commands and Make commands with usage examples.

## Table of Contents
- [CLI Commands](#cli-commands)
  - [Collection Management](#collection-management)
  - [Data Management](#data-management)
  - [User & Role Management](#user--role-management)
  - [Indexing & Search](#indexing--search)
  - [System Administration](#system-administration)
  - [Queue Management](#queue-management)
- [Make Commands](#make-commands)
  - [Development](#development)
  - [Testing](#testing)
  - [Building](#building)
  - [Deployment](#deployment)
- [API Commands (curl)](#api-commands-curl)

---

## CLI Commands

All CLI commands are accessed via the `aleph` command. In a Docker environment, prefix with the container execution:

```bash
# Docker environment
docker-compose run --rm api aleph <command>

# Local environment
aleph <command>
```

### Version Information

#### `aleph --version` / `aleph -v`
Display version information for Aleph and dependencies.

```bash
$ aleph --version
Aleph 4.1.7
Python 3.10.13
Flask 2.3.3
Werkzeug 3.0.1
```

---

## Collection Management

### `aleph collections`
List all collections in the system.

**Options:**
- `--secret BOOL` - Filter by secret flag (True/False/None)
- `--casefile BOOL` - Filter by casefile flag (True/False/None)

**Examples:**
```bash
# List all collections
aleph collections

# List only secret collections
aleph collections --secret true

# List only casefiles
aleph collections --casefile true

# List non-casefile collections
aleph collections --casefile false
```

**Output:**
```
Foreign ID              ID  Label
--------------------  ----  ---------------------------
my-collection           12  My Investigation
leak-2024               45  Panama Papers Subset
watchlist-osint         78  OSINT Watchlist
```

---

### `aleph crawldir`
Import documents from a local directory into a collection.

**Syntax:**
```bash
aleph crawldir [OPTIONS] PATH
```

**Options:**
- `-l, --language LANG` - ISO language code for OCR (can specify multiple times)
- `-f, --foreign_id ID` - Foreign ID for the collection (auto-generated if omitted)

**Examples:**
```bash
# Crawl directory with auto-generated collection ID
aleph crawldir /data/documents

# Crawl with specific collection ID
aleph crawldir -f my-docs-2024 /data/documents

# Crawl with multiple OCR languages
aleph crawldir -l eng -l fra -l deu /data/multilingual

# Crawl directory (auto-creates collection named after directory)
aleph crawldir /home/user/investigation_files
# Creates collection: directory:investigation-files
```

**Notes:**
- Creates collection automatically if it doesn't exist
- Processes files recursively
- Supported formats: PDF, DOCX, XLSX, CSV, HTML, TXT, images (for OCR)
- Make sure a worker is running to process uploaded files

---

### `aleph delete`
Delete a collection and all its contents.

**Syntax:**
```bash
aleph delete [OPTIONS] FOREIGN_ID
```

**Options:**
- `--sync` - Wait for deletion to complete (default: async)
- `--async` - Delete asynchronously

**Examples:**
```bash
# Delete collection asynchronously
aleph delete my-old-collection

# Delete and wait for completion
aleph delete --sync temporary-collection
```

**Warning:** This is destructive and cannot be undone (soft delete).

---

### `aleph flush`
Remove all contents from a collection but keep the collection metadata.

**Syntax:**
```bash
aleph flush [OPTIONS] FOREIGN_ID
```

**Options:**
- `--sync` - Wait for flush to complete
- `--async` - Flush asynchronously (default)

**Examples:**
```bash
# Flush collection contents
aleph flush my-collection

# Flush and wait for completion
aleph flush --sync my-collection
```

---

### `aleph touch`
Mark a collection as changed to trigger reprocessing and reindexing.

**Syntax:**
```bash
aleph touch [OPTIONS] FOREIGN_ID
```

**Options:**
- `--sync` - Wait for operation to complete (default: true)
- `--async` - Run asynchronously

**Examples:**
```bash
# Touch collection (trigger recompute)
aleph touch my-collection

# Touch asynchronously
aleph touch --async my-collection
```

---

### `aleph publish`
Make a collection publicly visible to all users (including anonymous).

**Syntax:**
```bash
aleph publish FOREIGN_ID
```

**Examples:**
```bash
# Make collection public
aleph publish public-data-2024
```

**Notes:**
- Grants read permission to system guest role
- Does not grant write permission
- Can be reversed via API or UI

---

## Data Management

### `aleph load-entities`
Import FollowTheMoney entities from a JSON lines file.

**Syntax:**
```bash
aleph load-entities [OPTIONS] FOREIGN_ID
```

**Options:**
- `-i, --infile FILE` - Input file (default: stdin)
- `--safe/--unsafe` - Allow references to archive hashes (default: safe)
- `--mutable/--immutable` - Mark entities as mutable (default: immutable)
- `--clean/--unclean` - Enable server-side validation (default: clean)

**Examples:**
```bash
# Load from file
aleph load-entities -i entities.ijson my-collection

# Load from stdin
cat entities.ijson | aleph load-entities my-collection

# Load with mutable entities
aleph load-entities --mutable -i entities.ijson my-collection

# Load unsafe entities (with archive references)
aleph load-entities --unsafe -i entities.ijson my-collection
```

**Input Format (JSON Lines):**
```json
{"id": "entity-1", "schema": "Person", "properties": {"name": ["John Doe"], "birthDate": ["1980-01-15"]}}
{"id": "entity-2", "schema": "Company", "properties": {"name": ["ACME Corp"], "jurisdiction": ["US"]}}
```

---

### `aleph dump-entities`
Export all entities from a collection to JSON lines format.

**Syntax:**
```bash
aleph dump-entities [OPTIONS] FOREIGN_ID
```

**Options:**
- `-o, --outfile FILE` - Output file (default: stdout)

**Examples:**
```bash
# Export to stdout
aleph dump-entities my-collection

# Export to file
aleph dump-entities -o export.ijson my-collection

# Export and compress
aleph dump-entities my-collection | gzip > export.ijson.gz
```

---

### `aleph dump-profiles`
Export profile entityset items (entity judgements) from collections.

**Syntax:**
```bash
aleph dump-profiles [OPTIONS]
```

**Options:**
- `-o, --outfile FILE` - Output file (default: stdout)
- `-f, --foreign_id ID` - Specific collection (default: all collections)

**Examples:**
```bash
# Export all profiles
aleph dump-profiles -o profiles.jsonl

# Export profiles from specific collection
aleph dump-profiles -f my-investigation -o profiles.jsonl
```

---

## User & Role Management

### `aleph createuser`
Create a new user account.

**Syntax:**
```bash
aleph createuser [OPTIONS] EMAIL
```

**Options:**
- `-p, --password PASSWORD` - Set user password
- `-n, --name NAME` - Set display name
- `-a, --admin` - Grant admin privileges

**Examples:**
```bash
# Create basic user
aleph createuser user@example.com

# Create user with password and name
aleph createuser -p secret123 -n "Jane Doe" jane@example.com

# Create admin user
aleph createuser --admin -p admin123 admin@example.com

# Create user (interactive password prompt)
aleph createuser analyst@example.com
```

**Output:**
```
User created. ID: 42, API Key: aleph-abc123def456...
```

**Notes:**
- Save the API key - it's only shown once
- Email must be unique
- If password omitted, user must reset via email

---

### `aleph renameuser`
Rename an existing user.

**Syntax:**
```bash
aleph renameuser EMAIL NEW_NAME
```

**Examples:**
```bash
# Rename user
aleph renameuser user@example.com "John Smith"
```

---

### `aleph users`
List all users with their details.

**Examples:**
```bash
$ aleph users
Foreign ID       ID  E-Mail              Name         is admin  groups
-------------  ----  ------------------  -----------  --------  ----------
admin@test       12  admin@test.com      Admin User   True
analyst@test     45  analyst@test.com    Analyst      False     investigators
```

---

### `aleph creategroup`
Create a new user group.

**Syntax:**
```bash
aleph creategroup NAME
```

**Examples:**
```bash
# Create group
aleph creategroup investigators

# Create group for team
aleph creategroup "Data Journalism Team"
```

---

### `aleph groups`
List all groups in the system.

**Examples:**
```bash
$ aleph groups
Foreign ID          ID  Name
----------------  ----  -------------------
investigators       23  investigators
editors             45  editors
```

---

### `aleph useradd`
Add a user to a group.

**Syntax:**
```bash
aleph useradd GROUP USER
```

**Examples:**
```bash
# Add user to group (by foreign ID)
aleph useradd investigators analyst@example.com

# Add multiple users
aleph useradd investigators user1@example.com
aleph useradd investigators user2@example.com
```

---

### `aleph userdel`
Remove a user from a group.

**Syntax:**
```bash
aleph userdel GROUP USER
```

**Examples:**
```bash
# Remove user from group
aleph userdel investigators former-analyst@example.com
```

---

### `aleph deleterole`
Permanently delete a role (user or group) from the database.

**Syntax:**
```bash
aleph deleterole FOREIGN_ID
```

**Examples:**
```bash
# Delete user
aleph deleterole old-user@example.com

# Delete group
aleph deleterole old-group
```

**Warning:** This is a hard delete and cannot be undone.

---

## Indexing & Search

### `aleph reindex`
Reindex all entities in a collection to Elasticsearch.

**Syntax:**
```bash
aleph reindex [OPTIONS] FOREIGN_ID
```

**Options:**
- `--flush` - Clear existing index before reindexing

**Examples:**
```bash
# Reindex collection
aleph reindex my-collection

# Reindex with flush (clear existing data)
aleph reindex --flush my-collection
```

**Use Cases:**
- After changing entity data
- After Elasticsearch mapping changes
- Fixing search inconsistencies

---

### `aleph reindex-full`
Reindex all collections in the system.

**Syntax:**
```bash
aleph reindex-full [OPTIONS]
```

**Options:**
- `--flush` - Clear all indices before reindexing

**Examples:**
```bash
# Reindex all collections
aleph reindex-full

# Reindex all with flush
aleph reindex-full --flush
```

**Warning:** This can take hours for large deployments.

---

### `aleph reindex-casefiles`
Reindex only casefile collections (investigations).

**Syntax:**
```bash
aleph reindex-casefiles [OPTIONS]
```

**Options:**
- `--flush` - Clear indices before reindexing

**Examples:**
```bash
# Reindex all casefiles
aleph reindex-casefiles

# Reindex casefiles with flush
aleph reindex-casefiles --flush
```

---

### `aleph reingest`
Re-process and optionally reindex documents and entities.

**Syntax:**
```bash
aleph reingest [OPTIONS] FOREIGN_ID
```

**Options:**
- `--index` - Also reindex after reingesting
- `--flush/--no-flush` - Flush existing data (default: flush)
- `--include_ingest` - Re-process documents through ingest-file service

**Examples:**
```bash
# Reingest collection
aleph reingest my-collection

# Reingest and reindex
aleph reingest --index my-collection

# Reingest without flushing
aleph reingest --no-flush my-collection

# Full reingest including document processing
aleph reingest --include_ingest --index my-collection
```

---

### `aleph reingest-casefiles`
Re-ingest all casefile collections.

**Syntax:**
```bash
aleph reingest-casefiles [OPTIONS]
```

**Options:**
- `--index` - Also reindex after reingesting

**Examples:**
```bash
# Reingest all casefiles
aleph reingest-casefiles

# Reingest and reindex all casefiles
aleph reingest-casefiles --index
```

---

### `aleph xref`
Cross-reference entities within a collection or against other collections.

**Syntax:**
```bash
aleph xref FOREIGN_ID
```

**Examples:**
```bash
# Cross-reference collection
aleph xref my-collection
```

**What it does:**
- Compares all entities in the collection
- Uses machine learning to find similar entities
- Stores matches in xref index
- Accessible via UI cross-reference tab

---

## System Administration

### `aleph upgrade`
Create or upgrade the database schema and Elasticsearch indices.

**Examples:**
```bash
# Run migrations and create indices
aleph upgrade
```

**Notes:**
- Run after updating Aleph version
- Safe to run multiple times
- Requires database connectivity

---

### `aleph downgrade-database`
Downgrade database to a previous migration.

**Syntax:**
```bash
aleph downgrade-database [OPTIONS]
```

**Options:**
- `-r, --revision REV` - Target revision (default: -1 for previous)
- `-x, --execute` - Actually execute (default: dry run)

**Examples:**
```bash
# Dry run (show SQL only)
aleph downgrade-database

# Downgrade to previous revision
aleph downgrade-database --execute

# Downgrade to specific revision
aleph downgrade-database -r abc123 --execute

# Downgrade to head
aleph downgrade-database -r head --execute
```

**Warning:** Downgrades can cause data loss. Test in staging first.

---

### `aleph resetindex`
Delete and recreate all Elasticsearch indices.

**Examples:**
```bash
aleph resetindex
```

**Warning:** This deletes all indexed data. Data must be reindexed from database.

---

### `aleph resetcache`
Clear the Redis cache.

**Examples:**
```bash
aleph resetcache
```

**Use Cases:**
- Clearing stale permission cache
- Fixing cache-related bugs
- Performance testing

---

### `aleph flushdeleted`
Permanently remove soft-deleted records from the database.

**Examples:**
```bash
aleph flushdeleted
```

**What it does:**
- Removes collections with `deleted_at` timestamp
- Removes entities with `deleted_at` timestamp
- Frees up database space

---

### `aleph cleanup-archive`
Remove orphaned files from archive storage.

**Syntax:**
```bash
aleph cleanup-archive [OPTIONS]
```

**Options:**
- `-p, --prefix PREFIX` - Only scan files with specific prefix

**Examples:**
```bash
# Clean all archive files
aleph cleanup-archive

# Clean specific prefix
aleph cleanup-archive -p 2023/
```

**What it does:**
- Scans archive storage (S3 or local)
- Removes files not referenced in database
- Frees up storage space

---

### `aleph update`
Re-index collections and clear caches after system updates.

**Examples:**
```bash
aleph update
```

**What it does:**
- Updates roles from OAuth providers
- Upgrades collection schemas
- Cleans up orphaned mappings

---

### `aleph evilshit`
**DANGEROUS:** Delete all data and recreate the database.

**Examples:**
```bash
aleph evilshit
```

**Warning:**
- Deletes ALL data permanently
- Drops database and indices
- Only use for testing/development
- Cannot be undone

---

### `aleph reset-api-key-expiration`
Reset expiration dates for legacy non-expiring API keys.

**Examples:**
```bash
aleph reset-api-key-expiration
```

---

### `aleph hash-plaintext-api-keys`
Convert legacy plaintext API keys to hashed format.

**Examples:**
```bash
aleph hash-plaintext-api-keys
```

---

## Queue Management

### `aleph status`
Check queue status and worker progress.

**Syntax:**
```bash
aleph status [FOREIGN_ID]
```

**Examples:**
```bash
# Check all queues
aleph status

# Check specific collection
aleph status my-collection
```

**Output:**
```
Collection         Job      Stage     Pending  Running  Finished
---------------  -------  --------  --------  -------  ---------
my-collection                            120        5        450
my-collection    job-123  ingest         45        2        200
my-collection    job-123  index          35        1        150
my-collection    job-456  xref           40        2        100
```

---

### `aleph cancel`
Cancel all pending tasks for a collection.

**Syntax:**
```bash
aleph cancel FOREIGN_ID
```

**Examples:**
```bash
# Cancel collection tasks
aleph cancel problematic-collection
```

---

### `aleph cancel-user`
Cancel all pending tasks not related to any dataset.

**Examples:**
```bash
aleph cancel-user
```

---

### `aleph worker`
Start a worker process to handle background tasks.

**Syntax:**
```bash
aleph worker [OPTIONS]
```

**Options:**
- `--threads N` - Number of worker threads (default: 1)

**Examples:**
```bash
# Start worker with 1 thread
aleph worker

# Start worker with 8 threads
aleph worker --threads 8
```

**Notes:**
- Run continuously in background
- Multiple workers can run concurrently
- Each worker processes tasks from RabbitMQ

---

### `aleph flushqueue`
Flush all tasks from the RabbitMQ queue.

**Examples:**
```bash
aleph flushqueue
```

**Warning:** Cancels all pending background jobs.

---

### `aleph retry-exports`
Retry failed export jobs.

**Examples:**
```bash
aleph retry-exports
```

---

## Make Commands

Make commands are used for development workflows. Run from the project root directory.

### Development

#### `make services`
Start infrastructure services (PostgreSQL, Elasticsearch, Redis, RabbitMQ, ingest-file).

```bash
make services
```

**What it starts:**
- postgres (port 5432)
- elasticsearch (port 9200)
- redis (port 6379)
- rabbitmq (port 5672, management on 15672)
- ingest-file (document processor)

---

#### `make shell`
Open a bash shell in the API container with all dependencies.

```bash
make shell
```

**Use cases:**
- Run aleph commands interactively
- Debug issues
- Explore container environment

---

#### `make upgrade`
Start database services and run migrations.

```bash
make upgrade
```

**Equivalent to:**
```bash
docker-compose up -d postgres elasticsearch
aleph upgrade
```

---

#### `make web`
Start both API and UI services for development.

```bash
make web
```

**Access:**
- UI: http://localhost:8080
- API: http://localhost:5000

---

#### `make api`
Start only the API service.

```bash
make api
```

---

#### `make worker`
Start a worker process with debugging enabled.

```bash
make worker
```

**Notes:**
- Debugger listens on port 5679
- Use with VS Code or PyCharm remote debugging

---

#### `make tail`
Follow logs from all Docker containers.

```bash
make tail
```

---

#### `make stop`
Stop all Docker containers.

```bash
make stop
```

---

### Testing

#### `make test`
Run backend Python tests.

```bash
# Run all tests
make test

# Run specific test file
make test file=aleph/tests/test_xref.py

# Run specific test
make test file=aleph/tests/test_xref.py::test_xref_collection
```

---

#### `make test-ui`
Run frontend React tests.

```bash
make test
```

---

#### `make e2e`
Run end-to-end tests with Playwright.

```bash
make e2e
```

**What it does:**
- Starts full stack in Docker
- Creates test admin user
- Runs Playwright browser tests
- Saves screenshots/videos on failure

---

#### `make e2e-local-setup`
Install dependencies for local E2E testing.

```bash
make e2e-local-setup
```

---

#### `make e2e-local`
Run E2E tests locally (without Docker).

```bash
make e2e-local
```

---

### Code Quality

#### `make lint`
Run Python linter (Ruff).

```bash
make lint
```

---

#### `make lint-ui`
Run JavaScript/TypeScript linter (ESLint).

```bash
make lint-ui
```

---

#### `make format`
Format Python code with Black.

```bash
make format
```

---

#### `make format-ui`
Format JavaScript/TypeScript code with Prettier.

```bash
make format-ui
```

---

#### `make format-check`
Check Python code formatting without making changes.

```bash
make format-check
```

---

#### `make format-check-ui`
Check JavaScript/TypeScript formatting without making changes.

```bash
make format-check-ui
```

---

### Building

#### `make build`
Build Docker images for development.

```bash
make build
```

---

#### `make build-ui`
Build production UI Docker image.

```bash
make build-ui
```

---

#### `make build-e2e`
Build E2E test Docker image.

```bash
make build-e2e
```

---

#### `make build-full`
Build all Docker images (dev, UI, E2E).

```bash
make build-full
```

---

### Maintenance

#### `make clean`
Remove build artifacts and temporary files.

```bash
make clean
```

**Removes:**
- `dist/`, `build/`, `.eggs/`
- `*.pyc`, `*.pyo`
- `__pycache__/`
- UI build files

---

#### `make translate`
Extract strings and compile translations.

```bash
make translate
```

**What it does:**
1. Extract UI strings to messages.json
2. Extract backend strings to messages.pot
3. Push to Transifex
4. Pull translations
5. Compile translation files

---

#### `make dev`
Install Python development dependencies locally.

```bash
make dev
```

---

#### `make fixtures`
Create test fixture data.

```bash
make fixtures
```

---

#### `make ingest-restart`
Restart the ingest-file service.

```bash
make ingest-restart
```

---

## API Commands (curl)

### Authentication

#### Login
```bash
curl -X POST http://localhost:5000/api/2/sessions/login \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "secret"}'
```

#### Get Session
```bash
curl http://localhost:5000/api/2/sessions \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

#### Logout
```bash
curl -X DELETE http://localhost:5000/api/2/sessions/logout \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

---

### Collections

#### List Collections
```bash
curl http://localhost:5000/api/2/collections \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

#### Get Collection
```bash
curl http://localhost:5000/api/2/collections/COLLECTION_ID \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

#### Create Collection
```bash
curl -X POST http://localhost:5000/api/2/collections \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "foreign_id": "my-collection",
    "label": "My Collection",
    "category": "leak",
    "summary": "Description of my collection"
  }'
```

#### Update Collection
```bash
curl -X PUT http://localhost:5000/api/2/collections/COLLECTION_ID \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "Updated Label",
    "summary": "Updated description"
  }'
```

#### Delete Collection
```bash
curl -X DELETE http://localhost:5000/api/2/collections/COLLECTION_ID \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

---

### Search

#### Search Entities
```bash
# Basic search
curl "http://localhost:5000/api/2/search?q=corruption" \
  -H "Authorization: ApiKey YOUR_API_KEY"

# Search with filters
curl "http://localhost:5000/api/2/search?q=john+doe&schema=Person&countries=US" \
  -H "Authorization: ApiKey YOUR_API_KEY"

# Paginated search
curl "http://localhost:5000/api/2/search?q=company&limit=50&offset=100" \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

#### Get Entity
```bash
curl http://localhost:5000/api/2/entities/ENTITY_ID \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

---

### Entities

#### Create Entity
```bash
curl -X POST http://localhost:5000/api/2/entities \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "schema": "Person",
    "properties": {
      "name": ["John Doe"],
      "birthDate": ["1980-01-15"],
      "nationality": ["US"]
    },
    "collection_id": "COLLECTION_ID"
  }'
```

#### Update Entity
```bash
curl -X PUT http://localhost:5000/api/2/entities/ENTITY_ID \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "schema": "Person",
    "properties": {
      "name": ["John Doe"],
      "birthDate": ["1980-01-15"],
      "email": ["john@example.com"]
    }
  }'
```

#### Delete Entity
```bash
curl -X DELETE http://localhost:5000/api/2/entities/ENTITY_ID \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

---

### Ingest

#### Upload Document
```bash
curl -X POST http://localhost:5000/api/2/collections/COLLECTION_ID/ingest \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -F "file=@/path/to/document.pdf" \
  -F "foreign_id=my-doc-001"
```

#### Upload Multiple Files
```bash
for file in /path/to/docs/*; do
  curl -X POST http://localhost:5000/api/2/collections/COLLECTION_ID/ingest \
    -H "Authorization: ApiKey YOUR_API_KEY" \
    -F "file=@$file"
done
```

---

### Cross-Reference

#### Generate Xref
```bash
curl -X POST http://localhost:5000/api/2/xref \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "collection_ids": ["COLLECTION_A", "COLLECTION_B"]
  }'
```

#### Get Xref Results
```bash
curl "http://localhost:5000/api/2/collections/COLLECTION_ID/xref" \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

---

### Exports

#### Create Export
```bash
curl -X POST http://localhost:5000/api/2/exports \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "corruption",
    "collection_id": "COLLECTION_ID"
  }'
```

#### List Exports
```bash
curl http://localhost:5000/api/2/exports \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

#### Download Export
```bash
curl http://localhost:5000/api/2/exports/EXPORT_ID/download \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -o export.zip
```

---

### Alerts

#### Create Alert
```bash
curl -X POST http://localhost:5000/api/2/alerts \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "money laundering",
    "collection_id": "COLLECTION_ID"
  }'
```

#### List Alerts
```bash
curl http://localhost:5000/api/2/alerts \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

#### Delete Alert
```bash
curl -X DELETE http://localhost:5000/api/2/alerts/ALERT_ID \
  -H "Authorization: ApiKey YOUR_API_KEY"
```

---

## Common Workflows

### Complete Setup from Scratch

```bash
# 1. Start services
make services

# 2. Run migrations
make upgrade

# 3. Create admin user
docker-compose run --rm api aleph createuser --admin --password admin admin@admin.com

# 4. Start web interface
make web

# 5. Access UI at http://localhost:8080
```

---

### Import Data Pipeline

```bash
# 1. Create collection
docker-compose run --rm api aleph collections

# 2. Upload documents
docker-compose run --rm api aleph crawldir -f my-docs /data/documents

# 3. Load entities
docker-compose run --rm api aleph load-entities -i entities.ijson my-docs

# 4. Start worker to process
docker-compose run --rm api aleph worker --threads 4

# 5. Verify status
docker-compose run --rm api aleph status my-docs
```

---

### Cross-Reference Workflow

```bash
# 1. Ensure collections are indexed
docker-compose run --rm api aleph reindex collection-a
docker-compose run --rm api aleph reindex collection-b

# 2. Run cross-reference
docker-compose run --rm api aleph xref collection-a

# 3. Check status
docker-compose run --rm api aleph status collection-a

# 4. Export matches via API
curl "http://localhost:5000/api/2/collections/collection-a/xref" \
  -H "Authorization: ApiKey YOUR_API_KEY" \
  > xref-results.json
```

---

### Backup and Restore

```bash
# Backup database
docker-compose exec postgres pg_dump -U aleph aleph > backup.sql

# Backup entities
docker-compose run --rm api aleph dump-entities my-collection > entities.ijson

# Restore database
docker-compose exec -T postgres psql -U aleph aleph < backup.sql

# Restore entities
cat entities.ijson | docker-compose run --rm -T api aleph load-entities my-collection
```

---

### Performance Troubleshooting

```bash
# Check queue status
docker-compose run --rm api aleph status

# Check Elasticsearch health
curl http://localhost:9200/_cluster/health?pretty

# Check Redis connectivity
docker-compose exec redis redis-cli ping

# Check RabbitMQ
open http://localhost:15672  # guest/guest

# Clear caches
docker-compose run --rm api aleph resetcache

# Restart services
make stop
make services
make web
```

---

## Environment Variables

Common environment variables used in commands:

```bash
# Database
ALEPH_DATABASE_URI=postgresql://user:pass@localhost/aleph

# Elasticsearch
ALEPH_ELASTICSEARCH_URI=http://localhost:9200

# Redis
REDIS_URL=redis://localhost:6379/0

# RabbitMQ
RABBITMQ_URL=amqp://guest:guest@localhost:5672/

# Application
ALEPH_SECRET_KEY=your-secret-key
ALEPH_APP_TITLE="My Aleph"
ALEPH_UI_URL=http://localhost:8080/

# Authentication
ALEPH_PASSWORD_LOGIN=true
ALEPH_OAUTH=false
ALEPH_SINGLE_USER=false

# Workers
WORKER_THREADS=4

# OCR
ALEPH_OCR_DEFAULTS=eng,fra,deu
```

---

## Quick Reference

### Most Common Commands

```bash
# Development
make services          # Start infrastructure
make upgrade          # Run migrations
make web              # Start API + UI
make test             # Run tests

# Collections
aleph collections     # List collections
aleph crawldir PATH   # Import documents
aleph reindex ID      # Reindex collection

# Users
aleph createuser EMAIL    # Create user
aleph users              # List users

# Data
aleph load-entities ID   # Import entities
aleph dump-entities ID   # Export entities
aleph xref ID           # Cross-reference

# System
aleph upgrade           # Migrate database
aleph worker            # Start worker
aleph status            # Check queues
```

---

**Last Updated:** 2025-11-16
**Version:** 4.1.7
