# Aleph Documentation Index

Complete documentation for the Aleph investigative data platform.

## Quick Start

- **New Users**: Start with [claude.md](../claude.md) for project overview
- **Developers**: See [DEVELOPMENT.md](./DEVELOPMENT.md) for setup instructions
- **Administrators**: Check [DEPLOYMENT.md](./DEPLOYMENT.md) for production deployment
- **Command Reference**: See [COMMANDS.md](./COMMANDS.md) for all available commands

## Documentation Structure

### Core Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [claude.md](../claude.md) | Main project documentation and overview | All users |
| [COMMANDS.md](./COMMANDS.md) | Complete CLI and Make command reference with examples | Developers, Admins |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | System architecture and design patterns | Developers, Architects |
| [API.md](./API.md) | REST API endpoint documentation | Developers, Integrators |
| [DEVELOPMENT.md](./DEVELOPMENT.md) | Development setup and workflows | Developers |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | Production deployment guide | DevOps, Administrators |

### Reference Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [MODELS.md](./MODELS.md) | Database schema and data models | Developers, DBAs |
| [WORKFLOWS.md](./WORKFLOWS.md) | User workflows and common tasks | Users, Analysts, Developers |
| [DATA_PIPELINES.md](./DATA_PIPELINES.md) | Data flow and processing pipelines | Developers, Architects |
| [CONFIGURATION.md](./CONFIGURATION.md) | Environment variables and settings | Administrators |
| [SECURITY.md](./SECURITY.md) | Security features and best practices | Security Engineers |
| [PERFORMANCE.md](./PERFORMANCE.md) | Performance tuning and optimization | DevOps |
| [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) | Common issues and solutions | All users |

### Feature Documentation

| Document | Description | Audience |
|----------|-------------|----------|
| [SEARCH.md](./SEARCH.md) | Search capabilities and query syntax | Users, Developers |
| [XREF.md](./XREF.md) | Cross-reference matching system | Users, Analysts |
| [COLLECTIONS.md](./COLLECTIONS.md) | Collection management guide | Users, Analysts |
| [ENTITIES.md](./ENTITIES.md) | Entity types and FollowTheMoney schema | Users, Developers |
| [INVESTIGATIONS.md](./INVESTIGATIONS.md) | EntitySets, diagrams, and timelines | Users, Analysts |
| [INGESTION.md](./INGESTION.md) | Document upload and processing | Users, Administrators |
| [TRANSLATION.md](./TRANSLATION.md) | Internationalization guide | Translators, Developers |

### Advanced Topics

| Document | Description | Audience |
|----------|-------------|----------|
| [WORKERS.md](./WORKERS.md) | Background task processing | Developers, DevOps |
| [ELASTICSEARCH.md](./ELASTICSEARCH.md) | Search index architecture | Developers, DevOps |
| [FOLLOWTHEMONEY.md](./FOLLOWTHEMONEY.md) | Entity schema system guide | Developers, Data Modelers |
| [OAUTH.md](./OAUTH.md) | OAuth/OIDC integration | Administrators |
| [KUBERNETES.md](./KUBERNETES.md) | Kubernetes deployment with Helm | DevOps |
| [TESTING.md](./TESTING.md) | Testing strategy and guidelines | Developers |

## Documentation by Role

### For End Users

1. [Project Overview](../claude.md) - What is Aleph?
2. [Search Guide](./SEARCH.md) - How to search effectively
3. [Collections](./COLLECTIONS.md) - Organizing data
4. [Cross-Reference](./XREF.md) - Finding entity matches
5. [Investigations](./INVESTIGATIONS.md) - Creating diagrams and timelines
6. [Troubleshooting](./TROUBLESHOOTING.md) - Solving common issues

### For Analysts

1. [Search Advanced](./SEARCH.md) - Query syntax and filters
2. [Entity Management](./ENTITIES.md) - Working with structured data
3. [Cross-Reference Deep Dive](./XREF.md) - Understanding matching algorithms
4. [Investigation Workflows](./INVESTIGATIONS.md) - Case management
5. [Data Import](./INGESTION.md) - Loading datasets

### For Developers

1. [Development Setup](./DEVELOPMENT.md) - Getting started
2. [Architecture](./ARCHITECTURE.md) - System design
3. [API Reference](./API.md) - Complete endpoint documentation (86 endpoints)
4. [Data Models](./MODELS.md) - Database schema (12 models)
5. [Data Pipelines](./DATA_PIPELINES.md) - Processing workflows and data flow
6. [Workflows](./WORKFLOWS.md) - User workflows and common tasks
7. [Command Reference](./COMMANDS.md) - CLI tools
8. [FollowTheMoney](./FOLLOWTHEMONEY.md) - Entity schema
9. [Testing](./TESTING.md) - Writing and running tests
10. [Workers](./WORKERS.md) - Background processing

### For DevOps/Administrators

1. [Deployment Guide](./DEPLOYMENT.md) - Production setup
2. [Configuration](./CONFIGURATION.md) - Environment settings
3. [Security](./SECURITY.md) - Hardening guidelines
4. [Performance](./PERFORMANCE.md) - Optimization techniques
5. [Kubernetes](./KUBERNETES.md) - K8s deployment
6. [Troubleshooting](./TROUBLESHOOTING.md) - System issues
7. [Command Reference](./COMMANDS.md) - Admin commands

## Documentation by Task

### Installation & Setup

- [Local Development Setup](./DEVELOPMENT.md#local-setup)
- [Docker Compose Deployment](./DEPLOYMENT.md#docker-compose)
- [Kubernetes Deployment](./KUBERNETES.md)
- [Initial Configuration](./CONFIGURATION.md#required-settings)
- [Creating First User](./COMMANDS.md#aleph-createuser)

### Data Management

- [Uploading Documents](./INGESTION.md#document-upload)
- [Importing Entities](./COMMANDS.md#aleph-load-entities)
- [Creating Collections](./COLLECTIONS.md#creating-collections)
- [Setting Permissions](./COLLECTIONS.md#access-control)
- [Data Export](./COMMANDS.md#aleph-dump-entities)

### Search & Analysis

- [Basic Search](./SEARCH.md#basic-search)
- [Advanced Queries](./SEARCH.md#advanced-syntax)
- [Faceted Search](./SEARCH.md#facets)
- [Cross-Referencing](./XREF.md#running-xref)
- [Entity Matching](./XREF.md#matching-algorithm)

### Investigations

- [Creating Entity Lists](./INVESTIGATIONS.md#entity-lists)
- [Building Diagrams](./INVESTIGATIONS.md#network-diagrams)
- [Timeline Views](./INVESTIGATIONS.md#timelines)
- [Profile Management](./INVESTIGATIONS.md#profiles)
- [Sharing Investigations](./INVESTIGATIONS.md#sharing)

### Administration

- [User Management](./COMMANDS.md#user--role-management)
- [Collection Management](./COMMANDS.md#collection-management)
- [Index Maintenance](./COMMANDS.md#indexing--search)
- [Queue Monitoring](./COMMANDS.md#queue-management)
- [System Maintenance](./TROUBLESHOOTING.md#maintenance)

### Development

- [Running Tests](./TESTING.md#running-tests)
- [Making API Calls](./API.md#authentication)
- [Creating Entities](./ENTITIES.md#creating-entities)
- [Database Migrations](./DEVELOPMENT.md#migrations)
- [Adding Features](./DEVELOPMENT.md#contribution-workflow)

## External Resources

### Official Documentation
- [Aleph Official Docs](https://docs.alephdata.org/) - Primary documentation site
- [GitHub Repository](https://github.com/alephdata/aleph) - Source code and issues
- [FollowTheMoney Docs](https://followthemoney.tech/) - Entity schema documentation
- [API Documentation](https://redocly.github.io/redoc/?url=https://aleph.occrp.org/api/openapi.json) - Interactive API docs

### Community
- [GitHub Discussions](https://github.com/alephdata/aleph/discussions) - Community Q&A
- [GitHub Issues](https://github.com/alephdata/aleph/issues) - Bug reports and feature requests
- [OCCRP](https://www.occrp.org/) - Organization behind Aleph

### Related Projects
- [FollowTheMoney](https://github.com/alephdata/followthemoney) - Entity schema library
- [ingest-file](https://github.com/alephdata/ingest-file) - Document processing service
- [servicelayer](https://github.com/alephdata/servicelayer) - Shared utilities

### Tutorials & Guides
- [OCCRP Data Desk](https://github.com/occrp/datadesktop) - Aleph usage tutorials
- [Investigative Dashboard](https://investigativedashboard.org/) - Investigation guides

## How to Use This Documentation

### 1. Start with the Overview
Read [claude.md](../claude.md) to understand what Aleph is and its core capabilities.

### 2. Choose Your Path
- **User**: Focus on SEARCH, COLLECTIONS, XREF, and INVESTIGATIONS
- **Developer**: Start with DEVELOPMENT, then ARCHITECTURE and API
- **Admin**: Begin with DEPLOYMENT, CONFIGURATION, and SECURITY

### 3. Reference as Needed
- Use COMMANDS.md for quick command lookups
- Check TROUBLESHOOTING.md when issues arise
- Refer to API.md for integration work

### 4. Deep Dive Topics
- Read ARCHITECTURE.md to understand system design
- Study FOLLOWTHEMONEY.md for data modeling
- Review WORKERS.md for background processing

## Documentation Standards

### File Naming
- Use uppercase for documentation files (e.g., `COMMANDS.md`)
- Use descriptive names (`TROUBLESHOOTING.md` not `HELP.md`)
- Use `.md` extension for Markdown files

### Structure
- Start with a title and brief description
- Include table of contents for long documents
- Use headers hierarchically (H1 → H2 → H3)
- Provide code examples with syntax highlighting
- Include command outputs where helpful

### Code Examples
- Show full commands, not just fragments
- Include expected output
- Explain what each example does
- Use realistic data in examples

### Cross-References
- Link to related documentation
- Reference specific sections: `[Text](./FILE.md#section)`
- Keep links relative, not absolute

## Contributing to Documentation

### How to Contribute
1. Fork the repository
2. Create a feature branch
3. Make documentation changes
4. Test all links and code examples
5. Submit a pull request

### What to Document
- New features and changes
- Common user questions
- Troubleshooting solutions
- Performance tips
- Security considerations

### Writing Guidelines
- Write clearly and concisely
- Use active voice
- Include practical examples
- Explain the "why", not just the "how"
- Keep it up to date

### Review Process
- Documentation PRs reviewed by maintainers
- Accuracy verified against code
- Examples tested for correctness
- Links checked for validity

## Document Status

| Document | Status | Last Updated | Completeness |
|----------|--------|--------------|--------------|
| claude.md | ✅ Complete | 2025-11-16 | 100% |
| COMMANDS.md | ✅ Complete | 2025-11-16 | 100% |
| INDEX.md | ✅ Complete | 2025-12-08 | 100% |
| MODELS.md | ✅ Complete | 2025-12-08 | 100% |
| API.md | ✅ Complete | 2025-12-08 | 100% (86/86 endpoints) |
| WORKFLOWS.md | ✅ Complete | 2025-12-08 | 100% (10 workflows) |
| DATA_PIPELINES.md | ✅ Complete | 2025-12-08 | 100% (4 pipelines) |
| SECURITY_AUDIT.md | ✅ Complete | 2025-12-08 | 100% (7 issues) |
| ARCHITECTURE.md | ✅ Complete | 2025-12-08 | 100% (1797 lines) |
| CONFIGURATION.md | ✅ Complete | 2025-12-08 | 100% (1476 lines) |
| DEVELOPMENT.md | ✅ Complete | 2025-12-08 | 100% (1218 lines) |
| DEPLOYMENT.md | ✅ Complete | 2025-12-08 | 100% (1358 lines) |
| SEARCH.md | ✅ Complete | 2025-12-08 | 100% (1100+ lines) |
| XREF.md | ✅ Complete | 2025-12-09 | 100% (2100+ lines) |
| DOCUMENTATION_PROGRESS.md | ✅ Complete | 2025-12-09 | 100% (tracking doc) |
| Other docs | ⏳ Planned | - | 0% |

**Legend:**
- ✅ Complete - Comprehensive and up to date
- 📝 In Progress - Partially complete
- ⏳ Planned - Not yet started
- ❌ Deprecated - No longer maintained

## Getting Help

### In-Application Help
- Status page: `/api/2/status`
- OpenAPI spec: `/api/openapi.json`
- Health check: `/api/2/_health`

### Command Help
```bash
# General help
aleph --help

# Command-specific help
aleph <command> --help

# Examples
aleph crawldir --help
aleph createuser --help
```

### Troubleshooting
1. Check [TROUBLESHOOTING.md](./TROUBLESHOOTING.md)
2. Search [GitHub Issues](https://github.com/alephdata/aleph/issues)
3. Review logs: `make tail` or `docker-compose logs`
4. Ask in [GitHub Discussions](https://github.com/alephdata/aleph/discussions)

### Support Channels
- **Bug Reports**: [GitHub Issues](https://github.com/alephdata/aleph/issues/new)
- **Feature Requests**: [GitHub Discussions](https://github.com/alephdata/aleph/discussions)
- **Questions**: [GitHub Discussions](https://github.com/alephdata/aleph/discussions)
- **Security Issues**: Email security@occrp.org

## Version Information

**Aleph Version:** 4.1.7
**Documentation Version:** 1.0.0
**Last Updated:** 2025-11-16

## License

This documentation is part of the Aleph project and is licensed under the MIT License.

See [LICENSE](../LICENSE) for details.

---

**Quick Links:**
[Main Documentation](../claude.md) |
[Commands](./COMMANDS.md) |
[Architecture](./ARCHITECTURE.md) |
[API](./API.md) |
[Development](./DEVELOPMENT.md) |
[Deployment](./DEPLOYMENT.md)
