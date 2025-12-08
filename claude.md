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
│   ├── INDEX.md           # Complete documentation index (500+ lines)
│   └── MODELS.md          # Database schema reference (600+ lines)
├── bug-fixes/             # Security audit and patches
│   ├── SECURITY_AUDIT.md  # Comprehensive security audit report
│   └── patches/           # 7 patch files ready to apply
│       ├── 001-secret-key-validation.patch
│       ├── 002-secure-csp.patch
│       ├── 003-restrict-cors.patch
│       ├── 004-jwt-api-update.patch
│       ├── 005-fix-query-exhaustion.patch
│       ├── 006-fix-typo.patch
│       └── 007-improve-db-check.patch
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

## Completed Work

1. ✅ Cloned repository to working directory
2. ✅ Created claude.md for context preservation
3. ✅ Created cdocs/ directory structure
4. ✅ Read and analyzed all 73 documentation files
5. ✅ Created comprehensive documentation:
   - `cdocs/MODELS.md` (600+ lines) - Complete database schema reference
   - `cdocs/INDEX.md` (500+ lines) - Central documentation hub
6. ✅ Conducted comprehensive security audit
7. ✅ Created bug-fixes/ directory with patches
8. ✅ Identified and documented 7 security/bug issues
9. ✅ Generated patch files for all issues

## Security Audit (2025-12-08)

A comprehensive security code review was conducted, identifying:
- **1 CRITICAL issue**: Missing SECRET_KEY validation
- **2 HIGH issues**: Insecure CSP and CORS defaults
- **1 MEDIUM issue**: Deprecated JWT API usage
- **3 LOW issues**: Code quality and potential bugs

**Documentation:** `bug-fixes/SECURITY_AUDIT.md`
**Patches:** `bug-fixes/patches/*.patch` (7 patches ready to apply)

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

## Security Audit Summary

### Issues Identified

| ID | Severity | File | Description |
|----|----------|------|-------------|
| CRITICAL-001 | CRITICAL | settings.py:76 | Missing SECRET_KEY validation |
| HIGH-001 | HIGH | settings.py:64-67 | Insecure CSP allows unsafe-inline/eval |
| HIGH-002 | HIGH | settings.py:70 | CORS allows all origins by default |
| MEDIUM-001 | MEDIUM | logic/util.py:59 | Deprecated JWT decode parameter |
| LOW-001 | LOW | logic/api_keys.py:96 | Potential query exhaustion |
| LOW-002 | LOW | logic/api_keys.py:134 | Typo in log message |
| LOW-003 | LOW | core.py:70 | Weak database type check |

### Applying Patches

```bash
cd /media/Daten1/projects/aleph

# Apply all patches
for patch in bug-fixes/patches/*.patch; do
    git apply "$patch"
done

# Or apply individual patches
git apply bug-fixes/patches/001-secret-key-validation.patch
```

---

**Last Updated:** 2025-12-08 17:45 UTC
**Status:** Documentation complete, security audit complete, patches ready
