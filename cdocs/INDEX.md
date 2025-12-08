# Aleph Documentation Index

**Project:** Aleph - OCCRP Investigative Data Platform
**Repository:** `/media/Daten1/projects/aleph`
**Fork:** https://github.com/siddeyg/aleph
**Upstream:** https://github.com/alephdata/aleph
**Last Updated:** 2025-12-08

---

## About This Documentation

This is a comprehensive documentation collection for the Aleph project, created to provide deep technical insights into the platform's architecture, data models, and implementation details. This documentation supplements the official Aleph documentation with additional technical depth and analysis.

---

## Quick Navigation

### For Developers
- [Database Models Reference](./MODELS.md) - Complete schema documentation
- [Security Audit Report](../bug-fixes/SECURITY_AUDIT.md) - Security analysis & patches
- [Official Getting Started](/docs/src/pages/developers/getting-started/development-environment/index.mdx)
- [Architecture Overview](/docs/src/pages/developers/explanation/architecture/index.mdx)
- [Contributing Guidelines](/CONTRIBUTING.md)

### For Users
- [Official User Guides](/docs/src/pages/users/)
- [Search Basics](/docs/src/pages/users/search/basics/index.mdx)
- [Investigations](/docs/src/pages/users/investigations/overview/index.mdx)

### For Operators
- [Production Deployment](/docs/src/pages/developers/getting-started/production-deployment/index.mdx)
- [Configuration Options](/docs/src/pages/developers/reference/configuration-options/index.mdx)
- [Operations Guides](/docs/src/pages/developers/how-to/operations/)

---

## Documentation Structure

### cdocs/ - Enhanced Documentation

Our enhanced documentation in the `cdocs/` directory:

| File | Description | Status |
|------|-------------|--------|
| [INDEX.md](./INDEX.md) | This file - complete documentation index | ✅ Complete |
| [MODELS.md](./MODELS.md) | Comprehensive database schema reference | ✅ Complete |

### bug-fixes/ - Security Audit & Patches

Security audit and bug fixes documentation:

| File | Description | Status |
|------|-------------|--------|
| [SECURITY_AUDIT.md](../bug-fixes/SECURITY_AUDIT.md) | Comprehensive security code review | ✅ Complete |
| [patches/*.patch](../bug-fixes/patches/) | 7 ready-to-apply patches | ✅ Complete |

### docs/ - Official Documentation

The `docs/` directory contains the official Aleph documentation site (Astro-based):

#### Developer Documentation

##### Explanations - Core Concepts
- [Architecture Overview](/docs/src/pages/developers/explanation/architecture/index.mdx)
- [FollowTheMoney Data Model](/docs/src/pages/developers/explanation/followthemoney/index.mdx)
- [Ingest Pipeline](/docs/src/pages/developers/explanation/ingest-pipeline/index.mdx)
- [Search Implementation](/docs/src/pages/developers/explanation/search/index.mdx)
- [Cross-Referencing](/docs/src/pages/developers/explanation/cross-referencing/index.mdx)
- [Permissions Model](/docs/src/pages/developers/explanation/permissions/index.mdx)
- [Entity Extraction](/docs/src/pages/developers/explanation/entity-extraction/index.mdx)
- [Design Premises](/docs/src/pages/developers/explanation/design-premises/index.mdx)

##### Getting Started
- [Development Environment Setup](/docs/src/pages/developers/getting-started/development-environment/index.mdx)
- [Production Deployment](/docs/src/pages/developers/getting-started/production-deployment/index.mdx)
- [Docker Compose Deployment](/docs/src/pages/developers/getting-started/production-deployment/docker-compose/index.mdx)
- [Finding Datasets](/docs/src/pages/developers/getting-started/find-datasets/index.mdx)

##### How-To Guides - Data Operations
- [Data Operations Overview](/docs/src/pages/developers/how-to/data/index.mdx)
- [Install alephclient](/docs/src/pages/developers/how-to/data/install-alephclient/index.mdx)
- [Install FtM](/docs/src/pages/developers/how-to/data/install-ftm/index.mdx)
- [Import FtM Data](/docs/src/pages/developers/how-to/data/import-ftm-data/index.mdx)
- [Import Tabular Data](/docs/src/pages/developers/how-to/data/import-tabular-data/index.mdx)
- [Import OCDS Data](/docs/src/pages/developers/how-to/data/import-ocds-data/index.mdx)
- [Export FtM Data](/docs/src/pages/developers/how-to/data/export-ftm-data/index.mdx)
- [Export Mentions](/docs/src/pages/developers/how-to/data/export-mentions/index.mdx)
- [Export Network Graphs](/docs/src/pages/developers/how-to/data/export-network-graphs/index.mdx)
- [Mixed Graphs](/docs/src/pages/developers/how-to/data/mixed-graphs/index.mdx)
- [Download Files](/docs/src/pages/developers/how-to/data/download-files/index.mdx)
- [Upload Directory](/docs/src/pages/developers/how-to/data/upload-directory/index.mdx)

##### How-To Guides - Development
- [Development Overview](/docs/src/pages/developers/how-to/development/index.mdx)
- [Configure Identity Provider](/docs/src/pages/developers/how-to/development/identity-provider/index.mdx)
- [Configure Blob Storage](/docs/src/pages/developers/how-to/development/blob-storage/index.mdx)
- [Inspect Elasticsearch](/docs/src/pages/developers/how-to/development/inspect-elasticsearch/index.mdx)
- [Add New Languages](/docs/src/pages/developers/how-to/development/new-languages/index.mdx)
- [Translate Aleph](/docs/src/pages/developers/how-to/development/translate-aleph/index.mdx)
- [Custom Text Processor](/docs/src/pages/developers/how-to/development/custom-text-processor/index.mdx)

##### How-To Guides - Operations
- [Operations Overview](/docs/src/pages/developers/how-to/operations/index.mdx)
- [Configure OAuth](/docs/src/pages/developers/how-to/operations/oauth/index.mdx)
- [Configure Identity Provider](/docs/src/pages/developers/how-to/operations/identity-provider/index.mdx)
- [Configure Blob Storage](/docs/src/pages/developers/how-to/operations/blob-storage/index.mdx)
- [Configure Reverse Proxy](/docs/src/pages/developers/how-to/operations/reverse-proxy/index.mdx)
- [Configure Sentry](/docs/src/pages/developers/how-to/operations/sentry/index.mdx)
- [Configure Prometheus](/docs/src/pages/developers/how-to/operations/prometheus/index.mdx)
- [Structured Logging](/docs/src/pages/developers/how-to/operations/structured-logging/index.mdx)
- [Manage Users and Groups](/docs/src/pages/developers/how-to/operations/users-and-groups/index.mdx)
- [Backups](/docs/src/pages/developers/how-to/operations/backups/index.mdx)
- [Upgrades](/docs/src/pages/developers/how-to/operations/upgrades/index.mdx)
- [Custom Data Model](/docs/src/pages/developers/how-to/operations/custom-data-model/index.mdx)
- [Custom Pages](/docs/src/pages/developers/how-to/operations/custom-pages/index.mdx)
- [Message Banner](/docs/src/pages/developers/how-to/operations/message-banner/index.mdx)

##### Reference Documentation
- [Configuration Options](/docs/src/pages/developers/reference/configuration-options/index.mdx)
- [Glossary](/docs/src/pages/developers/reference/glossary/index.mdx)
- [Supported File Types](/docs/src/pages/developers/reference/file-types/index.mdx)
- [Supported Languages](/docs/src/pages/developers/reference/languages/index.mdx)
- [Prometheus Metrics](/docs/src/pages/developers/reference/prometheus-metrics/index.mdx)

#### User Documentation

##### Getting Started
- [User Overview](/docs/src/pages/users/index.mdx)
- [Create Account](/docs/src/pages/users/getting-started/account/create/index.mdx)
- [Account Management](/docs/src/pages/users/getting-started/account/index.mdx)
- [Reset Password](/docs/src/pages/users/getting-started/account/reset-password/index.mdx)
- [Homepage Overview](/docs/src/pages/users/getting-started/homepage/index.mdx)
- [Key Terms](/docs/src/pages/users/getting-started/key-terms/index.mdx)

##### Search
- [Search Basics](/docs/src/pages/users/search/basics/index.mdx)
- [Advanced Search](/docs/src/pages/users/search/advanced/index.mdx)
- [Browse Datasets](/docs/src/pages/users/search/datasets/index.mdx)

##### Investigations
- [Investigation Overview](/docs/src/pages/users/investigations/overview/index.mdx)
- [Create Investigation](/docs/src/pages/users/investigations/create/index.mdx)
- [Upload Documents](/docs/src/pages/users/investigations/uploading-documents/index.mdx)
- [Entity Editor](/docs/src/pages/users/investigations/entity-editor/index.mdx)
- [Network Diagrams](/docs/src/pages/users/investigations/network-diagrams/index.mdx)
- [Cross-Referencing](/docs/src/pages/users/investigations/cross-referencing/index.mdx)
- [Manage Access](/docs/src/pages/users/investigations/manage-access/index.mdx)

##### Advanced Features
- [Reconciliation](/docs/src/pages/users/advanced/reconciliation/index.mdx)

##### FAQ
- [Account FAQ](/docs/src/pages/users/faq/account/index.mdx)

#### General
- [About](/docs/src/pages/about/index.mdx)
- [Get in Touch](/docs/src/pages/get-in-touch/index.mdx)

### Root Documentation Files

| File | Description |
|------|-------------|
| [README.rst](../README.rst) | Main project readme |
| [CONTRIBUTING.md](../CONTRIBUTING.md) | Contribution guidelines |
| [CODE_OF_CONDUCT.md](../CODE_OF_CONDUCT.md) | Community code of conduct |
| [SUPPORT.md](../SUPPORT.md) | Support policy |
| [SECURITY.md](../SECURITY.md) | Security policy |
| [CHANGELOG.md](https://github.com/alephdata/docs/blob/master/developers/changelog.md) | Version history (external) |

---

## Technology Stack Reference

### Backend
- **Language:** Python 3.10+
- **Framework:** Flask 2.3.3
- **ORM:** SQLAlchemy 2.0.21
- **Database:** PostgreSQL 10+
- **Search:** Elasticsearch 7.17.0
- **Queue:** RabbitMQ 3.9 / Redis
- **Cache:** Redis (alpine)
- **Data Model:** FollowTheMoney 3.5.9

### Frontend
- **Framework:** React 17.0.2
- **Language:** TypeScript
- **State:** Redux 4.2.1 + Redux-Thunk
- **UI:** Blueprint.js 4.18.0
- **Build:** Create React App + Craco
- **Routing:** React Router 6.28.1

### Infrastructure
- **Containerization:** Docker + Docker Compose
- **Orchestration:** Kubernetes (Helm charts)
- **Web Server:** Gunicorn (backend), Nginx (frontend)
- **Document Processing:** ingest-file 4.1.2

---

## Key Concepts

### Collections
Organizational units for data with access control:
- **Datasets**: Source material (leaks, company registries, sanctions lists)
- **Casefiles**: Investigations and analysis work
- 19 predefined categories
- Per-collection read/write permissions

### FollowTheMoney Entities
Structured data following the FtM schema:
- **Common Types**: Person, Company, Asset, Document, Email, Payment
- **Properties**: Type-safe attributes (name, date, country, etc.)
- **References**: Entity-to-entity relationships
- **Schemas**: 40+ built-in entity types

### Cross-Referencing (Xref)
Automated entity matching:
- **Fingerprinting**: Normalized name matching
- **Machine Learning**: GLM Bernoulli scoring model
- **Judgements**: positive, negative, unsure
- **Manual Review**: User confirmation workflow

### Entity Sets
Collections of entities for analysis:
- **Lists**: Simple collections
- **Diagrams**: Network visualizations
- **Timelines**: Chronological views
- **Profiles**: Detailed entity dossiers

### Ingest Pipeline
Multi-stage document processing:
1. **Upload**: API receives file
2. **Archive**: Store in S3/filesystem by SHA1 hash
3. **Ingest**: Extract metadata, convert to PDF, extract text/OCR
4. **Analyze**: Language detection, NER, pattern extraction
5. **Index**: Merge fragments, index in Elasticsearch

---

## Architecture Overview

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

### Component Responsibilities

| Component | Purpose | Technologies |
|-----------|---------|--------------|
| **UI** | User interface | React, TypeScript, Blueprint.js |
| **API** | REST endpoints, business logic | Flask, SQLAlchemy |
| **Workers** | Background jobs | Python, RabbitMQ, Redis |
| **ingest-file** | Document processing | Python, LibreOffice, PyMuPDF, SpaCy |
| **PostgreSQL** | Relational data storage | App database + FtM Store |
| **Elasticsearch** | Full-text search | Per-schema indexes |
| **Redis** | Caching, sessions, job metadata | Key-value store |
| **RabbitMQ** | Task queue | Message broker |
| **Archive** | File storage | S3-compatible or filesystem |

---

## Development Workflow

### Setup

```bash
# Clone repository
git clone https://github.com/siddeyg/aleph.git
cd aleph

# Configure environment
cp aleph.env.tmpl aleph.env
# Edit aleph.env with ALEPH_SECRET_KEY and other settings

# Start services
make services

# Build containers
make build

# Run migrations
make upgrade

# Start development server
make web

# Start worker (in separate terminal)
make worker

# Access at http://localhost:8080
```

### Common Tasks

```bash
# Create admin user
make shell
aleph createuser --name="Admin" --admin --password="password" admin@example.com

# Load test data
aleph crawldir /aleph/contrib/testdata

# Run backend tests
make test

# Run frontend tests
make test-ui

# Linting
make lint
make lint-ui

# Format code
make format
make format-ui

# Database migrations
aleph db revision -m "description"
aleph db upgrade
aleph db downgrade

# Reindex collection
aleph reindex --foreign-id collection-name

# Check system status
aleph status
```

### Git Workflow

- **develop** branch: Active development
- **main** branch: Stable releases
- Fork → Feature branch → Pull request to develop
- Follows Gitflow methodology

---

## API Quick Reference

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

---

## Configuration Reference

Key environment variables:

### Required
- `ALEPH_SECRET_KEY` - Session encryption (generate with `openssl rand -hex 32`)
- `ALEPH_DATABASE_URI` - PostgreSQL connection
- `ALEPH_ELASTICSEARCH_URI` - Elasticsearch cluster
- `REDIS_URL` - Redis cache
- `RABBITMQ_URL` - RabbitMQ queue

### Application
- `ALEPH_APP_TITLE` - UI title
- `ALEPH_UI_URL` - Frontend URL
- `ALEPH_APP_NAME` - Instance name

### Authentication
- `ALEPH_SINGLE_USER` - Disable auth (dev only)
- `ALEPH_OAUTH` - Enable OAuth
- `ALEPH_PASSWORD_LOGIN` - Enable password login
- `ALEPH_ADMINS` - Auto-admin emails

### Storage
- `ARCHIVE_TYPE` - "file" or "s3"
- `ARCHIVE_PATH` - Local storage path
- `ARCHIVE_BUCKET` - S3 bucket

### Processing
- `ALEPH_OCR_DEFAULTS` - Tesseract languages
- `WORKER_THREADS` - Worker thread count

See [Configuration Options](/docs/src/pages/developers/reference/configuration-options/index.mdx) for complete list.

---

## External Resources

### Official Links
- **Website:** https://aleph.occrp.org
- **Documentation:** https://docs.aleph.occrp.org
- **GitHub:** https://github.com/alephdata/aleph
- **OCCRP:** https://www.occrp.org

### Related Projects
- **FollowTheMoney:** https://followthemoney.tech
- **followthemoney-store:** https://github.com/alephdata/followthemoney-store
- **ingest-file:** https://github.com/alephdata/ingest-file
- **alephclient:** https://github.com/alephdata/alephclient
- **servicelayer:** https://github.com/alephdata/servicelayer
- **Memorious:** https://alephdata.github.io/memorious

### Community
- **GitHub Issues:** https://github.com/alephdata/aleph/issues
- **Discussions:** https://github.com/alephdata/aleph/discussions
- **Get in Touch:** https://docs.aleph.occrp.org/get-in-touch

---

## Maintenance

### Document Updates

When updating documentation:

1. Update relevant `.mdx` files in `/docs/src/pages/`
2. Update `cdocs/` files for enhanced documentation
3. Update version numbers and timestamps
4. Run documentation site locally to verify:
   ```bash
   cd docs
   npm install
   npm run dev
   ```
5. Commit changes with descriptive message
6. Submit pull request to develop branch

### Documentation Standards

- Use clear, concise language
- Include code examples where applicable
- Add file references with `file_path:line_number` format
- Keep table of contents up to date
- Use proper Markdown formatting
- Include diagrams for complex concepts
- Cross-reference related documentation

---

## Version Information

- **Aleph Version:** 4.1.7
- **Documentation Version:** 1.0.0
- **Last Full Review:** 2025-12-08
- **Python:** 3.10+
- **Node.js:** 18+
- **PostgreSQL:** 10+
- **Elasticsearch:** 7.17.0

---

**Generated:** 2025-12-08
**Maintainer:** OCCRP Development Team
**Contributors:** Community contributors (see CONTRIBUTING.md)

---

*This documentation is maintained as part of the Aleph project. For questions or improvements, please file an issue on GitHub or consult the contribution guidelines.*
