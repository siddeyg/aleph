# Aleph - Project Documentation

## Quick Reference
- [Complete Documentation Index](./docs/INDEX.md)
- [Command Reference](./docs/COMMANDS.md)
- [Architecture Overview](./docs/ARCHITECTURE.md)
- [API Reference](./docs/API.md)
- [Development Guide](./docs/DEVELOPMENT.md)

## Project Overview

**Aleph** is an open-source, web-based platform for indexing, searching, and analyzing large collections of documents and structured data. Built by OCCRP (Organized Crime and Corruption Reporting Project), it's specifically designed for investigative journalism and compliance research.

**Version:** 4.1.7
**License:** MIT
**Repository:** `/media/cy/Daten1/projects/aleph`
**Primary Use Case:** Document sifting and cross-referencing for investigative reporting

### Core Capabilities

1. **Document Management** - Index and search massive document collections (PDF, Word, HTML, etc.)
2. **Entity Analysis** - Structure and cross-reference entities (people, companies, organizations)
3. **Full-Text Search** - Powerful Elasticsearch-backed search with fuzzy matching
4. **Cross-Reference (Xref)** - Automatic entity matching across datasets using ML
5. **Investigations** - Create diagrams, timelines, and profiles from entities
6. **Alerts** - Get notified when new data matches saved searches
7. **Access Control** - Fine-grained permissions per collection

## Technology Stack

### Backend
- **Language:** Python 3.10
- **Framework:** Flask 2.3.3
- **Database:** PostgreSQL 10+
- **Search:** Elasticsearch 7.17.0
- **Queue:** RabbitMQ 3.9 / Redis
- **Cache:** Redis (alpine)
- **ORM:** SQLAlchemy 2.0.21
- **Schema:** FollowTheMoney 3.5.9 (OCCRP entity model)

### Frontend
- **Framework:** React 17.0.2 + TypeScript
- **State:** Redux 4.2.1 + Redux-Thunk
- **UI Library:** Blueprint.js 4.18.0
- **Build:** Create React App + Craco
- **Routing:** React Router 6.28.1

### Infrastructure
- **Container:** Docker + Docker Compose
- **Orchestration:** Kubernetes (Helm charts)
- **Web Server:** Gunicorn (backend), Nginx (frontend)
- **Document Processing:** ingest-file 4.1.2

## Quick Start

### Development Environment

```bash
# Start infrastructure services
make services

# Run database migrations
make upgrade

# Start API + UI
make web

# Access the application
open http://localhost:8080
```

### Production Deployment

```bash
# Using Docker Compose
docker-compose up -d

# Using Kubernetes
helm install aleph ./helm
```

## Project Structure

```
aleph/
├── aleph/              # Backend Python application
│   ├── model/          # SQLAlchemy ORM models
│   ├── views/          # Flask API blueprints
│   ├── logic/          # Business logic layer
│   ├── search/         # Elasticsearch query builders
│   ├── index/          # Index management
│   ├── worker.py       # Async task processing
│   └── manage.py       # CLI management commands
├── ui/                 # Frontend React application
│   └── src/
│       ├── components/ # Reusable React components
│       ├── screens/    # Full-page views
│       ├── dialogs/    # Modal dialogs
│       ├── actions/    # Redux actions
│       └── reducers/   # Redux reducers
├── helm/               # Kubernetes deployment
├── e2e/                # End-to-end tests
├── docs/               # Documentation (see INDEX.md)
└── Makefile            # Development commands
```

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 React UI (Browser)                      │
│              Redux State Management                     │
└────────────────────┬────────────────────────────────────┘
                     │ REST API (/api/2/*)
┌────────────────────▼────────────────────────────────────┐
│              Flask API Backend                          │
│  Authentication │ Authorization │ Business Logic        │
└─────┬──────────┬──────────┬──────────┬─────────────────┘
      │          │          │          │
   ┌──▼──┐  ┌───▼───┐  ┌───▼───┐  ┌──▼──────┐
   │ DB  │  │  ES   │  │Worker │  │ Archive │
   │ PG  │  │ Index │  │ Queue │  │ S3/File │
   └─────┘  └───────┘  └───────┘  └─────────┘
```

### Key Components

1. **API Layer** (`aleph/views/`) - REST endpoints for all operations
2. **Model Layer** (`aleph/model/`) - Data models and persistence
3. **Logic Layer** (`aleph/logic/`) - Business logic and orchestration
4. **Search Layer** (`aleph/search/`, `aleph/index/`) - Elasticsearch operations
5. **Worker Layer** (`aleph/worker.py`) - Background job processing
6. **UI Layer** (`ui/`) - React single-page application

## Core Concepts

### Collections
Organizational units for data. Each collection can contain:
- Documents (files, folders)
- Entities (structured data)
- Mappings (data imports)

Collections have:
- **Category** - 20 predefined types (news, leak, court, sanctions, etc.)
- **Permissions** - Read/write access per user/group
- **Metadata** - Publisher, update frequency, countries, languages

### Entities
Structured data objects following the FollowTheMoney schema:
- **Person** - Individuals
- **Company** - Organizations
- **LegalEntity** - Generic legal entities
- **Document** - File references
- **Event** - Temporal occurrences
- And 40+ more schemas

### Cross-Reference (Xref)
Automatic entity matching using:
- Fingerprint extraction (normalized names)
- Machine learning scoring (GLM Bernoulli model)
- Manual review workflow
- Configurable thresholds

### EntitySets (Investigations)
Collections of entities for analysis:
- **Lists** - Simple entity lists
- **Diagrams** - Network visualization
- **Timelines** - Chronological view
- **Profiles** - Detailed entity profiles

### Alerts
Saved searches that notify users when new matching data appears.

## Common Tasks

### Search Documents
```bash
# Via CLI
aleph search "corruption case"

# Via API
curl http://localhost:5000/api/2/search?q=corruption

# Via UI
Navigate to Search page, enter query
```

### Upload Documents
```bash
# Via CLI
aleph crawldir /path/to/documents --foreign-id my-upload

# Via API
curl -X POST http://localhost:5000/api/2/collections/{id}/ingest \
  -F file=@document.pdf

# Via UI
Collections → Upload button
```

### Create Entity
```bash
# Via CLI
aleph write-entity collection-id entity.json

# Via API
curl -X POST http://localhost:5000/api/2/entities \
  -H "Content-Type: application/json" \
  -d '{"schema": "Person", "properties": {"name": ["John Doe"]}}'

# Via UI
Collection → New Entity button
```

### Run Cross-Reference
```bash
# Via CLI
aleph xref collection-a collection-b

# Via API
POST /api/2/xref
{"collection_ids": ["collection-a", "collection-b"]}

# Via UI
Collection → Cross-reference tab → Select collections
```

## API Endpoints

Base URL: `/api/2/`

### Authentication
- `GET /api/2/sessions` - Current session
- `POST /api/2/sessions/login` - Login
- `DELETE /api/2/sessions/logout` - Logout

### Search
- `GET /api/2/search` - Search entities
- `GET /api/2/entities/{id}` - Get entity
- `POST /api/2/entities` - Create entity
- `PUT /api/2/entities/{id}` - Update entity
- `DELETE /api/2/entities/{id}` - Delete entity

### Collections
- `GET /api/2/collections` - List collections
- `GET /api/2/collections/{id}` - Get collection
- `POST /api/2/collections` - Create collection
- `PUT /api/2/collections/{id}` - Update collection
- `DELETE /api/2/collections/{id}` - Delete collection

### Ingestion
- `POST /api/2/collections/{id}/ingest` - Upload file
- `GET /api/2/collections/{id}/status` - Ingest status

### Cross-Reference
- `POST /api/2/xref` - Generate xref matches
- `GET /api/2/collections/{id}/xref` - Get xref results

See [API Reference](./docs/API.md) for complete endpoint documentation.

## Configuration

Key environment variables (see `aleph.env.tmpl`):

### Required
- `ALEPH_SECRET_KEY` - Session encryption (generate with `openssl rand -hex 32`)
- `ALEPH_DATABASE_URI` - PostgreSQL connection string
- `ALEPH_ELASTICSEARCH_URI` - Elasticsearch cluster URL
- `REDIS_URL` - Redis cache URL
- `RABBITMQ_URL` - RabbitMQ queue URL

### Application
- `ALEPH_APP_TITLE` - UI page title (default: "Aleph")
- `ALEPH_UI_URL` - Frontend URL (default: `http://localhost:8080/`)
- `ALEPH_APP_NAME` - Instance name (default: "aleph")

### Authentication
- `ALEPH_SINGLE_USER` - Disable auth (dev only)
- `ALEPH_OAUTH` - Enable OAuth login
- `ALEPH_PASSWORD_LOGIN` - Enable password login (default: true)
- `ALEPH_ADMINS` - Auto-admin email addresses (comma-separated)

### Storage
- `ARCHIVE_TYPE` - "file" or "s3"
- `ARCHIVE_PATH` - Local storage path (default: "/data")
- `ARCHIVE_BUCKET` - S3 bucket name (if using S3)

### Processing
- `ALEPH_OCR_DEFAULTS` - Tesseract languages (default: "eng")
- `WORKER_THREADS` - Worker thread count per process

See [Configuration Guide](./docs/CONFIGURATION.md) for complete details.

## Development

### Prerequisites
- Docker & Docker Compose
- Python 3.10+
- Node.js 16+
- Make

### Setup
```bash
# Clone repository
git clone https://github.com/alephdata/aleph.git
cd aleph

# Start services
make services

# Install dependencies (handled by Docker)
make build

# Run migrations
make upgrade

# Start development server
make web
```

### Testing
```bash
# Backend tests
make test

# Frontend tests
make test-ui

# E2E tests
make e2e

# Linting
make lint
make lint-ui

# Format code
make format
make format-ui
```

### Database Migrations
```bash
# Create new migration
aleph db revision -m "description"

# Apply migrations
aleph db upgrade

# Rollback
aleph db downgrade
```

See [Development Guide](./docs/DEVELOPMENT.md) for detailed instructions.

## Security

### Authentication Methods
1. **Password Login** - Email + password
2. **OAuth/OIDC** - Google, Azure AD, Cognito, KeyCloak
3. **API Keys** - For programmatic access

### Authorization
- **Role-Based Access Control (RBAC)**
- Per-collection read/write permissions
- User groups for team access
- Admin role for system-wide access

### Security Headers
- HSTS (Force HTTPS)
- CSP (Content Security Policy)
- CORS (Configurable origins)
- Feature Policy (Disable unused browser features)

### Data Protection
- Password hashing (werkzeug.security)
- API key digest storage
- Session token encryption
- Audit logging (events table)

## Monitoring

### Metrics
- Prometheus metrics (`prometheus-client`)
- Custom metrics: request count, latency, worker jobs

### Error Tracking
- Sentry integration (`SENTRY_DSN`)
- Structured JSON logging
- Request profiling (`ALEPH_PROFILE=true`)

### Health Checks
- `GET /api/2/status` - System status
- Database connectivity
- Elasticsearch cluster health
- Worker queue status

## Troubleshooting

### Common Issues

**Database connection errors**
```bash
# Check PostgreSQL is running
docker-compose ps postgres

# Check connection string
echo $ALEPH_DATABASE_URI

# Test connection
psql $ALEPH_DATABASE_URI -c "SELECT 1"
```

**Elasticsearch not indexing**
```bash
# Check ES health
curl http://localhost:9200/_cluster/health

# Reindex collection
aleph reindex --foreign-id collection-name

# Check worker logs
docker-compose logs worker
```

**Worker not processing jobs**
```bash
# Check RabbitMQ
docker-compose ps rabbitmq

# Check queue status
aleph status

# Restart worker
docker-compose restart worker
```

See [Troubleshooting Guide](./docs/TROUBLESHOOTING.md) for more solutions.

## Performance Tuning

### Elasticsearch
- Increase heap size: `ES_JAVA_OPTS=-Xms4g -Xmx4g`
- Adjust shard count in `aleph/index/indexes.py`
- Enable request cache

### PostgreSQL
- Increase shared_buffers
- Tune connection pool size
- Add database indices for custom queries

### Workers
- Increase worker count: `docker-compose up --scale worker=4`
- Adjust thread count: `WORKER_THREADS=8`
- Enable batch indexing: `INDEXING_TIMEOUT=10`

### Caching
- Enable response cache: `ALEPH_CACHE=true`
- Increase Redis memory: `maxmemory 4gb`
- Configure cache TTL in settings

## Deployment

### Docker Compose (Production)
```bash
# Configure environment
cp aleph.env.tmpl aleph.env
# Edit aleph.env with production values

# Start stack
docker-compose up -d

# Initialize database
docker-compose run --rm api aleph upgrade

# Create admin user
docker-compose run --rm api aleph createuser --admin admin@example.com
```

### Kubernetes (Helm)
```bash
# Configure values
cp helm/values.yaml helm/production-values.yaml
# Edit production-values.yaml

# Install
helm install aleph ./helm -f helm/production-values.yaml

# Upgrade
helm upgrade aleph ./helm -f helm/production-values.yaml
```

See [Deployment Guide](./docs/DEPLOYMENT.md) for production best practices.

## Additional Resources

### Documentation
- [Complete Documentation Index](./docs/INDEX.md) - All documentation files
- [Architecture Deep Dive](./docs/ARCHITECTURE.md) - System design details
- [API Reference](./docs/API.md) - Complete API documentation
- [Data Models](./docs/MODELS.md) - Database schema reference
- [Command Reference](./docs/COMMANDS.md) - CLI command guide

### External Links
- [Official Documentation](https://docs.alephdata.org/)
- [FollowTheMoney Documentation](https://followthemoney.tech/)
- [GitHub Repository](https://github.com/alephdata/aleph)
- [OCCRP](https://www.occrp.org/)

### Community
- [GitHub Issues](https://github.com/alephdata/aleph/issues)
- [Discussions](https://github.com/alephdata/aleph/discussions)

## Contributing

### Code Contributions
1. Fork the repository
2. Create feature branch: `git checkout -b feature-name`
3. Make changes and test
4. Format code: `make format && make format-ui`
5. Run tests: `make test && make test-ui`
6. Submit pull request

### Translation
- Translations managed via [Transifex](https://www.transifex.com/aleph/)
- 12+ languages supported
- See [Translation Guide](./docs/TRANSLATION.md)

### Documentation
- Documentation in Markdown format
- Located in `docs/` directory
- Submit PRs for improvements

## License

MIT License - See [LICENSE](LICENSE) file for details.

## Credits

Developed by OCCRP (Organized Crime and Corruption Reporting Project)

Built with:
- Flask (Python web framework)
- React (UI framework)
- Elasticsearch (search engine)
- FollowTheMoney (entity schema)
- Blueprint.js (UI components)
- And many other open-source projects

---

## Documentation Completion Summary

### Phase 1: API Documentation (100% Complete)

**API.md** - Complete REST API Reference
- **Status**: ✅ Complete (86/86 endpoints documented)
- **Date Completed**: 2025-12-08
- **Coverage**:
  - Collections & Permissions (12 endpoints)
  - Roles & Authentication (10 endpoints)
  - Entities & Search (8 endpoints)
  - Alerts & Bookmarks (10 endpoints)
  - Mappings (7 endpoints)
  - EntitySets/Investigations (9 endpoints)
  - Profiles (5 endpoints)
  - Reconciliation/OpenRefine (5 endpoints)
  - Notifications & Status (2 endpoints)
  - Archive & Streaming (3 endpoints)
  - Exports & Cross-Reference (4 endpoints)
  - Ingestion (11 endpoints)

Each endpoint includes:
- Authentication requirements
- Request/response examples with JSON
- Query parameters and path parameters
- curl command examples
- Usage notes and best practices

### Phase 2: Workflow Documentation (100% Complete)

**WORKFLOWS.md** - User Workflows and Common Tasks
- **Status**: ✅ Complete (10 workflows documented)
- **Date Completed**: 2025-12-08
- **Workflows**:
  1. Authentication & Registration - User signup and login flow
  2. Collection Management - Creating and configuring collections
  3. Document Upload & Processing - Ingestion pipeline
  4. Entity Management - Creating and editing structured data
  5. Investigation Workflows - Diagrams, timelines, lists
  6. Alert Management - Setting up search notifications
  7. Cross-Reference Workflow - Finding entity matches
  8. Data Import via Mappings - CSV/Excel bulk import
  9. Search & Discovery - Full-text search workflows
  10. Profile Management - Entity resolution and merging

Each workflow includes:
- Step-by-step process
- API endpoints involved
- Sequence diagrams
- Key files and components
- Example requests/responses

### Phase 3: Data Pipeline Documentation (100% Complete)

**DATA_PIPELINES.md** - Technical Data Flow Documentation
- **Status**: ✅ Complete (4 pipelines documented)
- **Date Completed**: 2025-12-08
- **Pipelines**:
  1. **Document Ingestion Pipeline** (6 stages)
     - Upload & validation → Archive storage → Job queuing
     - Worker processing → Entity extraction → ES indexing
  2. **Entity Extraction & Cross-Reference** (5 stages)
     - Pair generation → Feature extraction → ML scoring
     - Match storage → User review
  3. **Background Job Processing Architecture**
     - Queue system (RabbitMQ/Redis)
     - Worker architecture and scaling
     - Job stages, status tracking, error handling
  4. **Search & Query Execution Pipeline** (5 stages)
     - Query parsing → Query building → ES execution
     - Result processing → Response serialization

Each pipeline includes:
- Technical implementation details
- Code paths and key files
- Performance metrics and benchmarks
- Configuration parameters
- Monitoring and debugging guidance

### Previously Completed Documentation

**MODELS.md** - Database Schema Reference
- **Status**: ✅ Complete (12 models documented)
- **Date Completed**: 2025-11-16
- **Models**: Role, Alert, Collection, Permission, EntitySet, Bookmark, Event, Mapping, Diagram, Entity, Document, Export
- Includes relationships, methods, indexes, and security considerations

**COMMANDS.md** - CLI Command Reference
- **Status**: ✅ Complete (100% coverage)
- **Date Completed**: 2025-11-16
- All Make commands and Aleph CLI commands documented

**SECURITY_AUDIT.md** - Security Analysis
- **Status**: ✅ Complete (7 issues identified)
- **Date Completed**: 2025-12-08
- **Issues**: 1 CRITICAL, 2 HIGH, 1 MEDIUM, 3 LOW severity
- Includes patches in `bug-fixes/patches/` directory

### Phase 4: Core Documentation (100% Complete)

**ARCHITECTURE.md** - System Architecture & Design Patterns
- **Status**: ✅ Complete (1797 lines)
- **Date Completed**: 2025-12-08
- **Coverage**:
  - System overview and component architecture
  - Backend architecture (Model-Logic-View layers)
  - Frontend architecture (React, Redux)
  - Data flow (ingestion, xref, search pipelines)
  - Search architecture (Elasticsearch)
  - Worker architecture (task queue system)
  - Design patterns (soft delete, authz caching, proxy pattern)
  - Deployment architecture (Docker Compose, Kubernetes)
  - Database architecture (PostgreSQL schema, indexes, migrations)
  - API design patterns (REST, pagination, error handling)
  - Testing architecture (unit, integration, factories)
  - Monitoring & observability (Prometheus, logging, Sentry)
  - Configuration management
  - Development workflow

**CONFIGURATION.md** - Complete Configuration Reference
- **Status**: ✅ Complete (1476 lines)
- **Date Completed**: 2025-12-08
- **Coverage**:
  - Configuration overview and hierarchy
  - Required settings (SECRET_KEY, DATABASE_URI)
  - Application settings (branding, security headers)
  - Security & authentication (OAuth, password, session)
  - Database & search (PostgreSQL, Elasticsearch, Redis)
  - Email configuration (SMTP examples for Gmail, SendGrid, AWS SES)
  - Content processing (language, limits, notifications)
  - Worker configuration (queue settings, stages, QOS)
  - Monitoring (Sentry, Prometheus)
  - Feature flags
  - Environment examples (development, staging, production)
  - Best practices (security, performance, reliability)
  - Troubleshooting guide

**DEVELOPMENT.md** - Developer Guide
- **Status**: ✅ Complete (1218 lines)
- **Date Completed**: 2025-12-08
- **Coverage**:
  - Quick start and prerequisites
  - Development setup (Docker and native)
  - Running Aleph locally
  - Frontend development (React, Redux, TypeScript)
  - Backend development (Flask, SQLAlchemy, API)
  - Testing (backend pytest, frontend Jest)
  - Code quality (ruff, black, ESLint, Prettier)
  - Database migrations (Alembic)
  - Debugging (VSCode, PyCharm, debugpy, DevTools)
  - Common development tasks
  - Contributing guidelines (workflow, commit format)
  - Troubleshooting guide

**DEPLOYMENT.md** - Production Deployment Guide
- **Status**: ✅ Complete (1358 lines)
- **Date Completed**: 2025-12-08
- **Coverage**:
  - Deployment overview and architecture
  - Prerequisites and hardware requirements
  - Docker Compose deployment (step-by-step)
  - Kubernetes/Helm deployment with values
  - SSL/TLS configuration (Let's Encrypt, custom certs)
  - Production configuration (DB, ES, archive)
  - Backup and restore (automated scripts)
  - Monitoring and logging (Prometheus, Grafana, ELK)
  - Scaling (horizontal, vertical, load balancing)
  - Security hardening (network, app, database, secrets)
  - Updating Aleph and rollback procedures
  - Troubleshooting guide

### Documentation Statistics

| Metric | Count | Status |
|--------|-------|--------|
| **API Endpoints Documented** | 86/86 | 100% ✅ |
| **User Workflows Documented** | 10 | Complete ✅ |
| **Data Pipelines Documented** | 4 | Complete ✅ |
| **Database Models Documented** | 12/12 | 100% ✅ |
| **Security Issues Identified** | 7 | Patched ✅ |
| **Core Documentation Files** | 11 | Complete ✅ |
| **Total Lines of Documentation** | ~13,000+ | - |
| **Code Examples Included** | 400+ | - |

### Work Summary

#### Current Session Accomplishments (2025-12-08)
1. **Completed ARCHITECTURE.md** from 80% to 100% (1797 lines)
   - Added deployment architecture (Docker Compose, Kubernetes)
   - Added database architecture (PostgreSQL, indexes, migrations)
   - Added API design patterns (REST, error handling, auth)
   - Added testing architecture (unit, integration, fixtures)
   - Added monitoring & observability (Prometheus, logging, Sentry)
   - Added configuration management and development workflow

2. **Created CONFIGURATION.md** from scratch (1476 lines)
   - Documented all 80+ environment variables
   - Added required vs optional settings
   - Included SMTP examples for Gmail, SendGrid, AWS SES
   - Added environment examples (dev, staging, production)
   - Added best practices and troubleshooting guide

3. **Created DEVELOPMENT.md** from scratch (1218 lines)
   - Complete development setup guide (Docker and native)
   - Frontend development (React, Redux, TypeScript)
   - Backend development (Flask, SQLAlchemy, API)
   - Testing, code quality, database migrations
   - Debugging with VSCode, PyCharm, debugpy
   - Contributing guidelines and troubleshooting

4. **Created DEPLOYMENT.md** from scratch (1358 lines)
   - Docker Compose deployment (step-by-step)
   - Kubernetes/Helm deployment with values
   - SSL/TLS configuration (Let's Encrypt, custom certs)
   - Backup and restore with automated scripts
   - Monitoring and logging (Prometheus, Grafana, ELK)
   - Scaling and security hardening
   - Update procedures and troubleshooting

#### Previous Session Accomplishments (2025-12-08)
1. **Completed API.md** from 40% (34 endpoints) to 100% (86 endpoints)
   - Added 52 endpoints with full documentation
   - Included request/response examples for all endpoints
   - Added curl commands and usage notes

2. **Created WORKFLOWS.md** from scratch
   - Documented 10 major user workflows
   - Included step-by-step processes
   - Added sequence diagrams and API flow

3. **Created DATA_PIPELINES.md** from scratch
   - Documented 4 core data pipelines
   - Included technical implementation details
   - Added performance metrics and tuning guidance

4. **Updated INDEX.md**
   - Added new documentation files
   - Updated completion status
   - Improved navigation structure

### Repository Structure

```
/media/Daten1/projects/aleph/
├── docs/
│   ├── INDEX.md                  ✅ 100% - Documentation index
│   ├── API.md                    ✅ 100% - 86 API endpoints
│   ├── WORKFLOWS.md              ✅ 100% - 10 user workflows
│   ├── DATA_PIPELINES.md         ✅ 100% - 4 data pipelines
│   ├── MODELS.md                 ✅ 100% - 12 database models
│   ├── COMMANDS.md               ✅ 100% - CLI reference
│   ├── ARCHITECTURE.md           📝  80% - System architecture
│   └── ...
├── bug-fixes/
│   ├── SECURITY_AUDIT.md         ✅ 100% - Security analysis
│   └── patches/                  ✅ 7 patch files
├── claude.md                     ✅ Updated - Project context
└── [aleph codebase...]
```

### Git History

Branch: `claude/document-project-analysis-01QNAg9nXr6bAYWEv9vixQif`

**Recent Commits**:
1. `c789f28f4` - Create comprehensive DATA_PIPELINES.md documentation
2. `2487b5575` - Create comprehensive WORKFLOWS.md documentation
3. `81e17cca0` - Complete API.md with all remaining endpoints - 100% coverage!
4. `b6831337e` - Add EntitySets API endpoint documentation
5. `b2b9bac29` - Add Mappings API endpoint documentation
6. `2272121e3` - Add Alerts and Bookmarks API endpoint documentation
7. `e5656732d` - Add Roles, Groups, and Permissions API endpoints

All changes pushed to remote: `origin/claude/document-project-analysis-01QNAg9nXr6bAYWEv9vixQif`

### Next Steps for Future Work

**Remaining Documentation** (in priority order):
1. Complete ARCHITECTURE.md (currently 80%)
2. Complete DEVELOPMENT.md (currently 75%)
3. Complete DEPLOYMENT.md (currently 70%)
4. Complete CONFIGURATION.md (currently 80%)
5. Create additional specialized docs as needed

### How to Use This Documentation

1. **Context Recovery**: Read this file (claude.md) first for project overview
2. **Quick Reference**: Check INDEX.md for document navigation
3. **API Integration**: Use API.md for endpoint reference (100% complete)
4. **Understanding Workflows**: See WORKFLOWS.md for user task flows
5. **Technical Deep-Dive**: Read DATA_PIPELINES.md for system internals
6. **Database Schema**: Reference MODELS.md for data structure
7. **Security Review**: See SECURITY_AUDIT.md for security analysis

---

**Last Updated:** 2025-12-08
**Documentation Version:** 2.0.0
**Aleph Version:** 4.1.7
