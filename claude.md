# Aleph Project - Claude Context Document

**Date:** 2025-12-08
**Working Directory:** `/media/Daten1/projects/aleph`
**GitHub Fork:** `https://github.com/siddeyg/aleph`
**Upstream:** `https://github.com/alephdata/aleph`

## Project Overview

Aleph is OCCRP's (Organized Crime and Corruption Reporting Project) investigative data platform used by journalists worldwide to search through documents and structured data for investigative journalism.

### Key Capabilities
- Document ingestion and OCR
- Full-text search across 400+ million documents
- Cross-referencing entities
- Network graph visualization
- Investigation management
- Data source integration

### Technology Stack
- **Backend:** Python, Flask, SQLAlchemy
- **Frontend:** React, TypeScript
- **Search:** Elasticsearch
- **Database:** PostgreSQL
- **Queue:** Redis
- **Storage:** S3-compatible blob storage
- **Data Model:** FollowTheMoney (FtM)

## Current Session Context

### What We're Doing
1. Setting up proper working environment at `/media/Daten1/projects/aleph`
2. Creating comprehensive documentation in `cdocs/` subdirectory
3. Analyzing all existing documentation to understand the project
4. Preparing to generate missing documentation (like MODELS.md)

### Previous Investigation
- User had Claude Code Terminal analyze the aleph project ~3 weeks ago
- CCT created documentation files that weren't saved locally
- Files like MODELS.md were generated but lost
- Goal: Properly document the project and preserve knowledge

### Documentation Structure
```
aleph/
├── claude.md              # This file - Claude's context document
├── cdocs/                 # Our comprehensive documentation
│   ├── PROJECT_SUMMARY.md # High-level project overview
│   ├── MODELS.md          # Data models and schemas
│   ├── SETUP.md           # Setup and configuration guide
│   ├── ANALYSIS.md        # Technical analysis and insights
│   └── ...                # Additional documentation as needed
├── docs/                  # Official Aleph documentation (Astro site)
├── aleph/                 # Backend Python application
├── ui/                    # Frontend React application
└── ...
```

## Repository Structure

### Main Directories
- `aleph/` - Python backend application
  - `model/` - Database models
  - `views/` - API endpoints
  - `logic/` - Business logic
  - `queues/` - Background task processing
  - `search/` - Elasticsearch integration

- `ui/` - React frontend
  - `src/` - Source code
  - `public/` - Static assets

- `docs/` - Astro documentation site
  - `src/pages/` - Documentation pages (MDX format)
  - Organized by: users, developers, operations

- `contrib/` - Contributed tools and utilities
- `helm/` - Kubernetes deployment charts
- `mappings/` - Entity type mappings
- `e2e/` - End-to-end tests

### Key Files
- `README.rst` - Main project readme
- `CONTRIBUTING.md` - Contribution guidelines
- `setup.py` - Python package setup
- `docker-compose.yml` - Local development setup
- `Makefile` - Common development commands
- `requirements.txt` - Python dependencies
- `aleph.env.tmpl` - Environment variable template

## Git Workflow

### Remotes
```bash
origin: https://github.com/siddeyg/aleph (your fork)
upstream: https://github.com/alephdata/aleph (original repo)
```

### Branching Strategy
- Work on feature branches
- Sync with upstream main regularly
- Push to fork for backup
- Create PRs when ready to contribute upstream

### Current Branch
- Main branch: `master`
- Working on documentation improvements

## Data Model (FollowTheMoney)

Aleph uses the FollowTheMoney (FtM) data model:
- Entities represent people, companies, assets, etc.
- Properties define entity attributes
- Schema defines entity types and their valid properties
- Cross-references link related entities

### Core Entity Types
- Person
- Company
- Asset
- Document
- Email
- Payment
- Contract
- And many more...

## Important Concepts

### Collections
- Containers for related documents and entities
- Access control boundary
- Can be private or shared with groups

### Investigations
- Special collections for investigative work
- Support cross-referencing across collections
- Network diagram visualization

### Cross-Referencing (Xref)
- Automated entity matching across collections
- Finds potential duplicates and connections
- Uses fuzzy matching algorithms

### Ingestion Pipeline
1. Document upload
2. File type detection
3. Text extraction (OCR if needed)
4. Entity extraction (NER)
5. Indexing in Elasticsearch
6. Cross-referencing

## Development Environment

### Prerequisites
- Docker and Docker Compose
- Python 3.10+
- Node.js 18+
- Make

### Local Setup
```bash
# Start services
make services

# Run backend
make api

# Run frontend
cd ui && npm start

# Run workers
make worker

# Access at http://localhost:8080
```

### Environment Variables
See `aleph.env.tmpl` for configuration options

## API Endpoints

### Main API Routes
- `/api/2/` - API version 2
- `/api/2/collections` - Collections management
- `/api/2/entities` - Entity CRUD operations
- `/api/2/search` - Search endpoints
- `/api/2/notifications` - User notifications
- `/api/2/exports` - Export management

## Testing

```bash
# Run backend tests
make test

# Run e2e tests
make test-e2e

# Run frontend tests
cd ui && npm test
```

## Deployment

### Production Deployment
- Uses Kubernetes (Helm charts in `helm/` directory)
- Requires: PostgreSQL, Elasticsearch, Redis, S3 storage
- Supports horizontal scaling
- See `docs/` for detailed deployment guides

## Next Steps

1. ✅ Clone repository to working directory
2. ✅ Create claude.md for context
3. ⏳ Create cdocs/ directory structure
4. ⏳ Read and analyze all documentation files
5. ⏳ Create comprehensive documentation in cdocs/
6. ⏳ Generate MODELS.md with detailed schema documentation
7. ⏳ Commit and push changes to fork

## Resources

- **Official Docs:** https://docs.aleph.occrp.org
- **GitHub:** https://github.com/alephdata/aleph
- **FollowTheMoney:** https://followthemoney.tech
- **OCCRP:** https://www.occrp.org

## Notes for Future Context Recovery

### Key Information to Remember
- This is a fork for contributing improvements
- Documentation should be comprehensive and beginner-friendly
- Focus on clarity and practical examples
- All work is tracked in git and synced to GitHub fork
- Documentation lives in `cdocs/` subdirectory

### Session Information
- User: `cy` on Linux Mint
- Working from: `/media/Daten1/projects/aleph`
- GitHub username: `siddeyg`
- Email: `github@diemachtderworte.de`

---

**Last Updated:** 2025-12-08 15:58 UTC
**Status:** Initial setup complete, beginning documentation analysis
