# Aleph Configuration Guide

Complete reference for configuring Aleph via environment variables and configuration files.

## Table of Contents
- [Configuration Overview](#configuration-overview)
- [Required Settings](#required-settings)
- [Application Settings](#application-settings)
- [Security & Authentication](#security--authentication)
- [Database & Search](#database--search)
- [Email Configuration](#email-configuration)
- [Content Processing](#content-processing)
- [Worker Configuration](#worker-configuration)
- [Monitoring & Observability](#monitoring--observability)
- [Feature Flags](#feature-flags)
- [Environment Examples](#environment-examples)
- [Configuration Best Practices](#configuration-best-practices)
- [Troubleshooting](#troubleshooting)

---

## Configuration Overview

### How Configuration Works

Aleph uses environment variables for all configuration. Settings are loaded from:

1. **Environment variables** - Primary configuration method
2. **aleph.env file** - Docker Compose loads this file
3. **Default values** - Defined in `aleph/settings.py`

**Configuration Hierarchy** (highest priority first):
```
1. Environment Variables (OS level)
2. aleph.env file (Docker Compose)
3. Default values (aleph/settings.py)
```

### Configuration File Location

**Development**:
```bash
/path/to/aleph/aleph.env
```

**Docker Compose**:
```yaml
services:
  api:
    env_file:
      - aleph.env  # Loaded automatically
```

**Kubernetes**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aleph-config
data:
  ALEPH_SECRET_KEY: "..."
  ALEPH_APP_TITLE: "My Aleph"
```

---

## Required Settings

These settings **MUST** be configured before Aleph can run:

### ALEPH_SECRET_KEY

**Purpose**: Session encryption and CSRF protection
**Type**: String (minimum 32 characters)
**Required**: Yes
**Default**: None

**Generate a secure key**:
```bash
# Linux/macOS
openssl rand -hex 32

# Python
python -c "import secrets; print(secrets.token_hex(32))"
```

**Example**:
```bash
ALEPH_SECRET_KEY=a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z6a7b8c9d0
```

⚠️ **Security Warning**: Never commit this to version control. Use secrets management in production.

### ALEPH_DATABASE_URI

**Purpose**: PostgreSQL database connection
**Type**: Connection string
**Required**: Yes
**Default**: None

**Format**:
```
postgresql://[user]:[password]@[host]:[port]/[database]
```

**Examples**:
```bash
# Local development
ALEPH_DATABASE_URI=postgresql://aleph:aleph@localhost:5432/aleph

# Production with password
ALEPH_DATABASE_URI=postgresql://aleph:SecureP@ssw0rd@db.example.com:5432/aleph_prod

# With SSL
ALEPH_DATABASE_URI=postgresql://user:pass@host:5432/db?sslmode=require
```

---

## Application Settings

### Basic Application Info

#### ALEPH_APP_TITLE

**Purpose**: Application display name
**Type**: String
**Default**: "Aleph"

```bash
ALEPH_APP_TITLE=OCCRP Data Platform
```

#### ALEPH_APP_NAME

**Purpose**: Internal application name (used in index names)
**Type**: String (lowercase, no spaces)
**Default**: "aleph"

```bash
ALEPH_APP_NAME=myorg_aleph
```

#### ALEPH_UI_URL

**Purpose**: Public URL where UI is accessible
**Type**: URL
**Default**: "http://localhost:8080/"

```bash
# Development
ALEPH_UI_URL=http://localhost:8080/

# Production
ALEPH_UI_URL=https://aleph.example.org/
```

### Branding

#### ALEPH_LOGO

**Purpose**: Path to custom logo image
**Type**: Path (relative to static/)
**Default**: "/static/logo.png"

```bash
ALEPH_LOGO=/static/custom_logo.png
```

#### ALEPH_LOGO_AR

**Purpose**: Logo for Arabic/RTL languages
**Type**: Path
**Default**: Same as ALEPH_LOGO

```bash
ALEPH_LOGO_AR=/static/logo_arabic.png
```

#### ALEPH_FAVICON

**Purpose**: Browser favicon
**Type**: Path
**Default**: "/static/favicon.png"

```bash
ALEPH_FAVICON=/static/favicon.ico
```

### System Messages

#### ALEPH_APP_BANNER

**Purpose**: Display system-wide banner message
**Type**: HTML string
**Default**: None

```bash
ALEPH_APP_BANNER="<strong>System Maintenance</strong>: Scheduled downtime on Saturday 2-4 AM UTC"
```

#### ALEPH_APP_MESSAGES_URL

**Purpose**: External JSON feed for system messages
**Type**: URL
**Default**: None

```bash
ALEPH_APP_MESSAGES_URL=https://example.org/aleph-messages.json
```

### Security Headers

#### ALEPH_FORCE_HTTPS

**Purpose**: Enforce HTTPS redirects
**Type**: Boolean
**Default**: true (if ALEPH_UI_URL starts with https)

```bash
ALEPH_FORCE_HTTPS=true
```

#### ALEPH_CONTENT_POLICY

**Purpose**: Content Security Policy header
**Type**: String
**Default**: "default-src: 'self' 'unsafe-inline' 'unsafe-eval' data: *"

```bash
ALEPH_CONTENT_POLICY="default-src 'self'; img-src 'self' data: https:; script-src 'self'"
```

#### ALEPH_CORS_ORIGINS

**Purpose**: Allowed CORS origins (pipe-separated)
**Type**: List
**Default**: "*"

```bash
# Single origin
ALEPH_CORS_ORIGINS=https://example.org

# Multiple origins
ALEPH_CORS_ORIGINS=https://example.org|https://example.com|http://localhost:3000
```

### Operating Modes

#### ALEPH_DEBUG

**Purpose**: Enable debug mode (verbose errors)
**Type**: Boolean
**Default**: false

```bash
# Development only!
ALEPH_DEBUG=true
```

⚠️ **Never enable in production** - exposes sensitive information

#### ALEPH_CACHE

**Purpose**: Enable HTTP caching headers
**Type**: Boolean
**Default**: true (false if DEBUG=true)

```bash
ALEPH_CACHE=true
```

#### ALEPH_MAINTENANCE

**Purpose**: Put system in read-only mode with warning banner
**Type**: Boolean
**Default**: false

```bash
ALEPH_MAINTENANCE=true
```

---

## Security & Authentication

### Authentication Methods

#### ALEPH_PASSWORD_LOGIN

**Purpose**: Enable password-based authentication
**Type**: Boolean
**Default**: true (false if OAUTH=true)

```bash
# Disable for SSO-only environments
ALEPH_PASSWORD_LOGIN=false
```

#### ALEPH_SINGLE_USER

**Purpose**: Disable authentication (everyone is admin)
**Type**: Boolean
**Default**: false

```bash
# Development/private instances only
ALEPH_SINGLE_USER=true
```

⚠️ **Security Warning**: Only use for local development or completely isolated instances

#### ALEPH_REQUIRE_LOGGED_IN

**Purpose**: Require authentication for all access (no anonymous)
**Type**: Boolean
**Default**: false

```bash
ALEPH_REQUIRE_LOGGED_IN=true
```

### OAuth/OIDC Configuration

#### ALEPH_OAUTH

**Purpose**: Enable OAuth authentication
**Type**: Boolean
**Default**: false

```bash
ALEPH_OAUTH=true
```

#### ALEPH_OAUTH_HANDLER

**Purpose**: OAuth provider type
**Type**: String (oidc, keycloak, google, cognito, azure)
**Default**: "oidc"

```bash
ALEPH_OAUTH_HANDLER=keycloak
```

#### ALEPH_OAUTH_KEY

**Purpose**: OAuth client ID
**Type**: String
**Required**: If OAUTH=true

```bash
ALEPH_OAUTH_KEY=aleph-client-id
```

#### ALEPH_OAUTH_SECRET

**Purpose**: OAuth client secret
**Type**: String
**Required**: If OAUTH=true

```bash
ALEPH_OAUTH_SECRET=abc123xyz456
```

#### ALEPH_OAUTH_METADATA_URL

**Purpose**: OIDC discovery endpoint
**Type**: URL
**Required**: For OIDC providers

```bash
# Keycloak example
ALEPH_OAUTH_METADATA_URL=https://keycloak.example.org/realms/aleph/.well-known/openid-configuration

# Google example
ALEPH_OAUTH_METADATA_URL=https://accounts.google.com/.well-known/openid-configuration
```

#### ALEPH_OAUTH_SCOPE

**Purpose**: OAuth scopes to request
**Type**: Space-separated string
**Default**: "openid email profile"

```bash
ALEPH_OAUTH_SCOPE=openid email profile groups
```

#### ALEPH_OAUTH_AUDIENCE

**Purpose**: OAuth audience claim
**Type**: String
**Default**: None

```bash
ALEPH_OAUTH_AUDIENCE=aleph-api
```

#### ALEPH_OAUTH_ADMIN_GROUP

**Purpose**: Group name for automatic admin access
**Type**: String
**Default**: "superuser"

```bash
ALEPH_OAUTH_ADMIN_GROUP=aleph-admins
```

### User Management

#### ALEPH_ADMINS

**Purpose**: Email addresses to automatically grant admin rights
**Type**: Comma-separated list
**Default**: None

```bash
ALEPH_ADMINS=admin@example.org,superuser@example.org
```

#### ALEPH_SYSTEM_USER

**Purpose**: Foreign ID for system user
**Type**: String
**Default**: "system:aleph"

```bash
ALEPH_SYSTEM_USER=system:myaleph
```

#### ALEPH_SESSION_EXPIRE

**Purpose**: Session duration in seconds
**Type**: Integer
**Default**: 60,000 (800,000 if SINGLE_USER)

```bash
# 24 hours
ALEPH_SESSION_EXPIRE=86400

# 7 days
ALEPH_SESSION_EXPIRE=604800
```

#### ALEPH_ROLE_INACTIVE

**Purpose**: Days until inactive users stop receiving notifications
**Type**: Integer (days)
**Default**: 180 (6 months)

```bash
# 90 days
ALEPH_ROLE_INACTIVE=90
```

### Blocked User Messages

#### ALEPH_ROLE_BLOCKED_MESSAGE

**Purpose**: Message shown to blocked users
**Type**: String
**Default**: "Your account has been blocked."

```bash
ALEPH_ROLE_BLOCKED_MESSAGE="Your access has been suspended. Contact support@example.org"
```

#### ALEPH_ROLE_BLOCKED_LINK

**Purpose**: URL for blocked users
**Type**: URL
**Default**: None

```bash
ALEPH_ROLE_BLOCKED_LINK=https://support.example.org/account-blocked
```

---

## Database & Search

### PostgreSQL

See [ALEPH_DATABASE_URI](#aleph_database_uri) above.

### Elasticsearch

#### ALEPH_ELASTICSEARCH_URI

**Purpose**: Elasticsearch cluster URL
**Type**: URL
**Default**: "http://localhost:9200"

```bash
# Single node
ALEPH_ELASTICSEARCH_URI=http://elasticsearch:9200

# Multiple nodes
ALEPH_ELASTICSEARCH_URI=http://es-node1:9200,http://es-node2:9200

# With authentication
ALEPH_ELASTICSEARCH_URI=http://elastic:password@es.example.org:9200

# HTTPS
ALEPH_ELASTICSEARCH_URI=https://es.example.org:9200
```

#### ELASTICSEARCH_TLS_CA_CERTS

**Purpose**: Path to CA certificate file
**Type**: Path
**Default**: None

```bash
ELASTICSEARCH_TLS_CA_CERTS=/etc/aleph/certs/ca.crt
```

#### ELASTICSEARCH_TLS_VERIFY_CERTS

**Purpose**: Verify TLS certificates
**Type**: Boolean
**Default**: false

```bash
ELASTICSEARCH_TLS_VERIFY_CERTS=true
```

#### ELASTICSEARCH_TLS_CLIENT_CERT

**Purpose**: Client certificate for mTLS
**Type**: Path
**Default**: None

```bash
ELASTICSEARCH_TLS_CLIENT_CERT=/etc/aleph/certs/client.crt
```

#### ELASTICSEARCH_TLS_CLIENT_KEY

**Purpose**: Client certificate key
**Type**: Path
**Default**: None

```bash
ELASTICSEARCH_TLS_CLIENT_KEY=/etc/aleph/certs/client.key
```

#### ELASTICSEARCH_TIMEOUT

**Purpose**: Request timeout in seconds
**Type**: Integer
**Default**: 60

```bash
ELASTICSEARCH_TIMEOUT=120
```

### Index Configuration

#### ALEPH_INDEX_PREFIX

**Purpose**: Prefix for all index names
**Type**: String
**Default**: Value of ALEPH_APP_NAME

```bash
ALEPH_INDEX_PREFIX=prod_aleph
```

**Results in indices like**: `prod_aleph-entity-person-v1`

#### ALEPH_INDEX_WRITE

**Purpose**: Index version for writes
**Type**: String
**Default**: "v1"

```bash
ALEPH_INDEX_WRITE=v2
```

#### ALEPH_INDEX_READ

**Purpose**: Index versions for reads (comma-separated)
**Type**: List
**Default**: [ALEPH_INDEX_WRITE]

```bash
# Read from both v1 and v2 during migration
ALEPH_INDEX_READ=v1,v2
```

#### ALEPH_INDEX_REPLICAS

**Purpose**: Number of ES index replicas
**Type**: Integer
**Default**: 0

```bash
# 1 replica = 2 total copies
ALEPH_INDEX_REPLICAS=1

# 2 replicas = 3 total copies (recommended for production)
ALEPH_INDEX_REPLICAS=2
```

### Indexing Performance

#### ALEPH_INDEXING_BATCH_SIZE

**Purpose**: Number of entities to index in batch
**Type**: Integer
**Default**: 100

```bash
# Larger batches = better throughput, more memory
ALEPH_INDEXING_BATCH_SIZE=500
```

#### ALEPH_INDEXING_TIMEOUT

**Purpose**: Seconds to collect batch before indexing
**Type**: Integer
**Default**: 10

```bash
ALEPH_INDEXING_TIMEOUT=5
```

### Cross-Reference Settings

#### ALEPH_XREF_SCROLL

**Purpose**: ES scroll timeout for xref
**Type**: Duration string
**Default**: "5m"

```bash
ALEPH_XREF_SCROLL=10m
```

#### ALEPH_XREF_SCROLL_SIZE

**Purpose**: Entities per scroll batch
**Type**: Integer
**Default**: 1000

```bash
ALEPH_XREF_SCROLL_SIZE=2000
```

#### FTM_COMPARE_MODEL

**Purpose**: Path to ML model for entity matching
**Type**: Path
**Default**: Auto-downloaded during build

```bash
FTM_COMPARE_MODEL=/opt/ftm-compare/model.pkl
```

### Redis

Redis is configured via servicelayer (FTM_STORE_URI).

#### REDIS_URL

**Purpose**: Redis connection URL
**Type**: URL
**Default**: "redis://localhost:6379/0"

```bash
REDIS_URL=redis://redis:6379/0

# With password
REDIS_URL=redis://:password@redis:6379/0

# With database selection
REDIS_URL=redis://redis:6379/1
```

---

## Email Configuration

### SMTP Settings

#### ALEPH_MAIL_FROM

**Purpose**: Sender email address
**Type**: Email
**Default**: "aleph@domain.com"

```bash
ALEPH_MAIL_FROM=noreply@example.org
```

#### ALEPH_MAIL_HOST

**Purpose**: SMTP server hostname
**Type**: String
**Default**: "localhost"

```bash
ALEPH_MAIL_HOST=smtp.gmail.com
```

#### ALEPH_MAIL_PORT

**Purpose**: SMTP server port
**Type**: Integer
**Default**: 465

```bash
# SSL/TLS
ALEPH_MAIL_PORT=465

# STARTTLS
ALEPH_MAIL_PORT=587
```

#### ALEPH_MAIL_USERNAME

**Purpose**: SMTP authentication username
**Type**: String
**Default**: None

```bash
ALEPH_MAIL_USERNAME=apikey
```

#### ALEPH_MAIL_PASSWORD

**Purpose**: SMTP authentication password
**Type**: String
**Default**: None

```bash
ALEPH_MAIL_PASSWORD=your-smtp-password
```

#### ALEPH_MAIL_SSL

**Purpose**: Use SSL/TLS (implicit)
**Type**: Boolean
**Default**: false

```bash
# For port 465
ALEPH_MAIL_SSL=true
```

#### ALEPH_MAIL_TLS

**Purpose**: Use STARTTLS (explicit)
**Type**: Boolean
**Default**: true

```bash
# For port 587
ALEPH_MAIL_TLS=true
```

### Email Examples

**Gmail**:
```bash
ALEPH_MAIL_FROM=aleph@example.org
ALEPH_MAIL_HOST=smtp.gmail.com
ALEPH_MAIL_PORT=587
ALEPH_MAIL_USERNAME=your-email@gmail.com
ALEPH_MAIL_PASSWORD=your-app-password
ALEPH_MAIL_TLS=true
ALEPH_MAIL_SSL=false
```

**SendGrid**:
```bash
ALEPH_MAIL_FROM=noreply@example.org
ALEPH_MAIL_HOST=smtp.sendgrid.net
ALEPH_MAIL_PORT=587
ALEPH_MAIL_USERNAME=apikey
ALEPH_MAIL_PASSWORD=SG.your-api-key-here
ALEPH_MAIL_TLS=true
```

**AWS SES**:
```bash
ALEPH_MAIL_FROM=aleph@example.org
ALEPH_MAIL_HOST=email-smtp.us-east-1.amazonaws.com
ALEPH_MAIL_PORT=587
ALEPH_MAIL_USERNAME=AKIAIOSFODNN7EXAMPLE
ALEPH_MAIL_PASSWORD=your-smtp-password
ALEPH_MAIL_TLS=true
```

---

## Content Processing

### Language Settings

#### ALEPH_DEFAULT_LANGUAGE

**Purpose**: Default content language
**Type**: ISO 639-1 code
**Default**: "en"

```bash
ALEPH_DEFAULT_LANGUAGE=en
```

#### ALEPH_UI_LANGUAGES

**Purpose**: Available UI languages (comma-separated)
**Type**: List
**Default**: "ru,es,de,en,ar,fr"

```bash
ALEPH_UI_LANGUAGES=en,es,fr,de
```

### Search & Display

#### ALEPH_RESULT_HIGHLIGHT

**Purpose**: Enable result highlighting
**Type**: Boolean
**Default**: true

```bash
ALEPH_RESULT_HIGHLIGHT=false
```

#### ALEPH_MAX_EXPAND_ENTITIES

**Purpose**: Max entities returned when expanding properties
**Type**: Integer
**Default**: 200

```bash
ALEPH_MAX_EXPAND_ENTITIES=500
```

### API Limits

#### ALEPH_API_RATE_LIMIT

**Purpose**: Requests per window for anonymous users
**Type**: Integer
**Default**: 30

```bash
ALEPH_API_RATE_LIMIT=100
```

#### ALEPH_API_RATE_WINDOW

**Purpose**: Rate limit window in minutes
**Type**: Integer
**Default**: 15

```bash
ALEPH_API_RATE_WINDOW=10
```

### Export Limits

#### EXPORT_MAX_SIZE

**Purpose**: Maximum export file size in bytes
**Type**: Integer
**Default**: 1073741824 (1 GB)

```bash
# 5 GB
EXPORT_MAX_SIZE=5368709120
```

#### EXPORT_MAX_RESULTS

**Purpose**: Maximum search results to export
**Type**: Integer
**Default**: 100,000

```bash
EXPORT_MAX_RESULTS=500000
```

### Notifications

#### ALEPH_NOTIFICATIONS_DELETE

**Purpose**: Delete notifications after N days
**Type**: Integer
**Default**: 90 (3 months)

```bash
# 30 days
ALEPH_NOTIFICATIONS_DELETE=30
```

---

## Worker Configuration

### Queue Settings

#### ALEPH_BROKER_URI

**Purpose**: RabbitMQ connection URL
**Type**: URL
**Default**: Derived from REDIS_URL

```bash
# RabbitMQ
ALEPH_BROKER_URI=amqp://guest:guest@rabbitmq:5672

# With credentials
ALEPH_BROKER_URI=amqp://aleph:password@rabbitmq:5672/vhost

# Redis (alternative)
ALEPH_BROKER_URI=redis://redis:6379/1
```

#### ALEPH_RABBITMQ_MAX_PRIORITY

**Purpose**: Maximum task priority
**Type**: Integer
**Default**: 10

```bash
ALEPH_RABBITMQ_MAX_PRIORITY=5
```

### Worker Stages

#### ALEPH_WORKER_STAGES

**Purpose**: Which stages this worker handles (comma-separated)
**Type**: List
**Default**: All stages

**Available stages**:
- `index` - Entity indexing
- `xref` - Cross-reference matching
- `reingest` - Re-process documents
- `reindex` - Reindex collection
- `loadmapping` - Load mapping data
- `flushmapping` - Execute mapping
- `exportsearch` - Export search results
- `exportxref` - Export xref results
- `updateentity` - Update entity
- `pruneentity` - Delete entity

**Examples**:
```bash
# Worker 1: Only indexing and xref
ALEPH_WORKER_STAGES=index,xref

# Worker 2: Only exports
ALEPH_WORKER_STAGES=exportsearch,exportxref

# Worker 3: Everything else
ALEPH_WORKER_STAGES=reingest,reindex,loadmapping,flushmapping,updateentity,pruneentity
```

### Queue Prefetch (QOS) Settings

Control how many tasks each worker grabs at once:

```bash
# Indexing (high throughput)
ALEPH_RABBITMQ_QOS_INDEX_QUEUE=100

# Xref (resource-intensive, low concurrency)
ALEPH_RABBITMQ_QOS_XREF_QUEUE=1

# Reingest
ALEPH_RABBITMQ_QOS_REINGEST_QUEUE=1

# Reindex
ALEPH_RABBITMQ_QOS_REINDEX_QUEUE=1

# Mappings
ALEPH_RABBITMQ_QOS_LOAD_MAPPING_QUEUE=1
ALEPH_RABBITMQ_QOS_FLUSH_MAPPING_QUEUE=1

# Exports
ALEPH_RABBITMQ_QOS_EXPORT_SEARCH_QUEUE=1
ALEPH_RABBITMQ_QOS_EXPORT_XREF_QUEUE=1

# Entity operations
ALEPH_RABBITMQ_QOS_UPDATE_ENTITY_QUEUE=1
ALEPH_RABBITMQ_QOS_PRUNE_ENTITY_QUEUE=1
```

---

## Monitoring & Observability

### Sentry Error Tracking

#### SENTRY_DSN

**Purpose**: Sentry project DSN
**Type**: URL
**Default**: None

```bash
SENTRY_DSN=https://abc123@o123.ingest.sentry.io/456
```

#### SENTRY_ENVIRONMENT

**Purpose**: Environment name for Sentry
**Type**: String
**Default**: ""

```bash
SENTRY_ENVIRONMENT=production
```

### Prometheus Metrics

#### PROMETHEUS_ENABLED

**Purpose**: Enable Prometheus metrics endpoint
**Type**: Boolean
**Default**: false

```bash
PROMETHEUS_ENABLED=true
```

#### PROMETHEUS_PORT

**Purpose**: Prometheus exporter port
**Type**: Integer
**Default**: 9100

```bash
PROMETHEUS_PORT=9090
```

**Metrics endpoint**: `http://api:9100/metrics`

---

## Feature Flags

### Experimental Features

#### ALEPH_ENABLE_EXPERIMENTAL_BOOKMARKS_FEATURE

**Purpose**: Enable bookmarks (experimental)
**Type**: Boolean
**Default**: false

```bash
ALEPH_ENABLE_EXPERIMENTAL_BOOKMARKS_FEATURE=true
```

### Feedback URLs

#### ALEPH_FEEDBACK_URL_DOCUMENTS

**Purpose**: External feedback form URL for documents
**Type**: URL
**Default**: None

```bash
ALEPH_FEEDBACK_URL_DOCUMENTS=https://forms.gle/abc123
```

#### ALEPH_FEEDBACK_URL_TIMELINES

**Purpose**: External feedback form URL for timelines
**Type**: URL
**Default**: None

```bash
ALEPH_FEEDBACK_URL_TIMELINES=https://forms.gle/xyz789
```

---

## Environment Examples

### Development (aleph.env)

```bash
# Security
ALEPH_SECRET_KEY=dev-secret-key-not-for-production

# Application
ALEPH_APP_TITLE=Aleph Dev
ALEPH_UI_URL=http://localhost:8080/
ALEPH_DEBUG=true
ALEPH_CACHE=false

# Database
ALEPH_DATABASE_URI=postgresql://aleph:aleph@postgres:5432/aleph
ALEPH_ELASTICSEARCH_URI=http://elasticsearch:9200
REDIS_URL=redis://redis:6379/0

# Queue
ALEPH_BROKER_URI=amqp://guest:guest@rabbitmq:5672

# Authentication (password only)
ALEPH_PASSWORD_LOGIN=true
ALEPH_OAUTH=false

# Email (console only)
ALEPH_MAIL_FROM=dev@localhost

# Archive
ARCHIVE_TYPE=file
ARCHIVE_PATH=/data
```

### Production (aleph.env)

```bash
# Security
ALEPH_SECRET_KEY=${SECRET_KEY}  # From secrets manager
ALEPH_FORCE_HTTPS=true

# Application
ALEPH_APP_TITLE=OCCRP Aleph
ALEPH_UI_URL=https://aleph.example.org/
ALEPH_DEBUG=false
ALEPH_CACHE=true
ALEPH_MAINTENANCE=false

# Database
ALEPH_DATABASE_URI=postgresql://aleph:${DB_PASSWORD}@postgres.internal:5432/aleph_prod
ALEPH_ELASTICSEARCH_URI=https://es-node1.internal:9200,https://es-node2.internal:9200
REDIS_URL=redis://:${REDIS_PASSWORD}@redis.internal:6379/0

# Queue
ALEPH_BROKER_URI=amqp://aleph:${RABBITMQ_PASSWORD}@rabbitmq.internal:5672/aleph

# Authentication (OAuth)
ALEPH_PASSWORD_LOGIN=false
ALEPH_OAUTH=true
ALEPH_OAUTH_HANDLER=keycloak
ALEPH_OAUTH_KEY=${OAUTH_CLIENT_ID}
ALEPH_OAUTH_SECRET=${OAUTH_CLIENT_SECRET}
ALEPH_OAUTH_METADATA_URL=https://auth.example.org/realms/aleph/.well-known/openid-configuration
ALEPH_OAUTH_ADMIN_GROUP=aleph-admins

# Admin users
ALEPH_ADMINS=admin@example.org

# Email (SendGrid)
ALEPH_MAIL_FROM=noreply@example.org
ALEPH_MAIL_HOST=smtp.sendgrid.net
ALEPH_MAIL_PORT=587
ALEPH_MAIL_USERNAME=apikey
ALEPH_MAIL_PASSWORD=${SENDGRID_API_KEY}
ALEPH_MAIL_TLS=true

# Archive (S3)
ARCHIVE_TYPE=s3
ARCHIVE_BUCKET=aleph-prod-documents
AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY}
AWS_SECRET_ACCESS_KEY=${AWS_SECRET_KEY}
AWS_REGION=us-east-1

# Indexing
ALEPH_INDEX_PREFIX=prod_aleph
ALEPH_INDEX_REPLICAS=2
ALEPH_INDEXING_BATCH_SIZE=500

# Monitoring
SENTRY_DSN=${SENTRY_DSN}
SENTRY_ENVIRONMENT=production
PROMETHEUS_ENABLED=true

# Rate limiting
ALEPH_API_RATE_LIMIT=100
ALEPH_REQUIRE_LOGGED_IN=true
```

### Staging (aleph.env)

```bash
# Security
ALEPH_SECRET_KEY=${STAGING_SECRET_KEY}

# Application
ALEPH_APP_TITLE=Aleph Staging
ALEPH_UI_URL=https://staging-aleph.example.org/
ALEPH_DEBUG=false
ALEPH_CACHE=true
ALEPH_APP_BANNER="<strong>STAGING ENVIRONMENT</strong> - Test data only"

# Database (smaller instances)
ALEPH_DATABASE_URI=postgresql://aleph:${DB_PASSWORD}@postgres-staging:5432/aleph_staging
ALEPH_ELASTICSEARCH_URI=http://elasticsearch-staging:9200
REDIS_URL=redis://redis-staging:6379/0

# Queue
ALEPH_BROKER_URI=amqp://guest:guest@rabbitmq-staging:5672

# Authentication (password + OAuth)
ALEPH_PASSWORD_LOGIN=true
ALEPH_OAUTH=true
ALEPH_OAUTH_HANDLER=keycloak
ALEPH_OAUTH_KEY=${STAGING_OAUTH_CLIENT_ID}
ALEPH_OAUTH_SECRET=${STAGING_OAUTH_CLIENT_SECRET}
ALEPH_OAUTH_METADATA_URL=https://staging-auth.example.org/realms/aleph/.well-known/openid-configuration

# Email (test SMTP)
ALEPH_MAIL_FROM=staging@example.org
ALEPH_MAIL_HOST=mailhog
ALEPH_MAIL_PORT=1025

# Archive
ARCHIVE_TYPE=s3
ARCHIVE_BUCKET=aleph-staging-documents

# Monitoring
SENTRY_DSN=${SENTRY_DSN}
SENTRY_ENVIRONMENT=staging
```

---

## Configuration Best Practices

### Security

1. **Never commit secrets to version control**
   ```bash
   # Bad
   ALEPH_SECRET_KEY=abc123xyz...  # In git

   # Good
   ALEPH_SECRET_KEY=${SECRET_KEY}  # From environment/secrets manager
   ```

2. **Use strong random secrets**
   ```bash
   # Minimum 32 characters
   openssl rand -hex 32
   ```

3. **Enable HTTPS in production**
   ```bash
   ALEPH_FORCE_HTTPS=true
   ALEPH_UI_URL=https://aleph.example.org/
   ```

4. **Disable debug mode**
   ```bash
   ALEPH_DEBUG=false
   ```

5. **Use OAuth for authentication**
   ```bash
   ALEPH_OAUTH=true
   ALEPH_PASSWORD_LOGIN=false  # Disable password auth
   ```

### Performance

1. **Increase batch sizes for large datasets**
   ```bash
   ALEPH_INDEXING_BATCH_SIZE=1000
   ALEPH_INDEXING_TIMEOUT=30
   ```

2. **Configure ES replicas**
   ```bash
   # Production: 2 replicas (3 total copies)
   ALEPH_INDEX_REPLICAS=2
   ```

3. **Tune worker QOS settings**
   ```bash
   # Allow indexing worker to grab many tasks
   ALEPH_RABBITMQ_QOS_INDEX_QUEUE=500

   # Limit resource-intensive xref to 1 at a time
   ALEPH_RABBITMQ_QOS_XREF_QUEUE=1
   ```

4. **Enable caching**
   ```bash
   ALEPH_CACHE=true
   ```

### Reliability

1. **Enable monitoring**
   ```bash
   SENTRY_DSN=https://...
   PROMETHEUS_ENABLED=true
   ```

2. **Configure email for alerts**
   ```bash
   ALEPH_MAIL_FROM=alerts@example.org
   ALEPH_MAIL_HOST=smtp.example.org
   ```

3. **Use connection pooling**
   - PostgreSQL: Built into SQLAlchemy
   - Elasticsearch: Built into client
   - Redis: Automatic

### Scalability

1. **Use external services**
   ```bash
   # Managed Elasticsearch
   ALEPH_ELASTICSEARCH_URI=https://es-cluster.example.org:9200

   # Managed PostgreSQL
   ALEPH_DATABASE_URI=postgresql://user:pass@rds.amazonaws.com:5432/aleph

   # S3 for documents
   ARCHIVE_TYPE=s3
   ARCHIVE_BUCKET=my-aleph-documents
   ```

2. **Separate worker types**
   ```bash
   # Worker Pod 1: Indexing only (many replicas)
   ALEPH_WORKER_STAGES=index

   # Worker Pod 2: Xref only (fewer replicas, more resources)
   ALEPH_WORKER_STAGES=xref

   # Worker Pod 3: Everything else
   ALEPH_WORKER_STAGES=reingest,reindex,loadmapping,flushmapping
   ```

---

## Troubleshooting

### Common Issues

#### "ALEPH_SECRET_KEY is required"

**Cause**: ALEPH_SECRET_KEY not set
**Solution**:
```bash
# Generate a key
openssl rand -hex 32

# Set it
export ALEPH_SECRET_KEY=<generated-key>
```

#### "Could not connect to database"

**Cause**: Invalid DATABASE_URI or database not running
**Solution**:
```bash
# Check database is running
docker-compose ps postgres

# Test connection
psql postgresql://aleph:aleph@localhost:5432/aleph

# Check environment variable
echo $ALEPH_DATABASE_URI
```

#### "Elasticsearch connection failed"

**Cause**: ES not running or wrong URL
**Solution**:
```bash
# Check ES is running
docker-compose ps elasticsearch

# Test connection
curl http://localhost:9200

# Check environment variable
echo $ALEPH_ELASTICSEARCH_URI
```

#### "Redis connection refused"

**Cause**: Redis not running
**Solution**:
```bash
# Check Redis
docker-compose ps redis

# Test connection
redis-cli -h localhost ping

# Should return: PONG
```

#### "OAuth login fails"

**Cause**: Wrong OAuth configuration
**Solution**:
```bash
# Verify metadata URL
curl $ALEPH_OAUTH_METADATA_URL

# Check client ID and secret
echo $ALEPH_OAUTH_KEY
echo $ALEPH_OAUTH_SECRET

# Verify redirect URI is registered
# Should be: https://your-aleph.org/api/2/sessions/callback
```

#### "Email not sending"

**Cause**: SMTP misconfiguration
**Solution**:
```bash
# Test SMTP connection
telnet $ALEPH_MAIL_HOST $ALEPH_MAIL_PORT

# Check credentials
echo $ALEPH_MAIL_USERNAME
echo $ALEPH_MAIL_PASSWORD

# Verify TLS/SSL settings
# Port 465 = SSL (ALEPH_MAIL_SSL=true)
# Port 587 = TLS (ALEPH_MAIL_TLS=true)
```

### Debugging Configuration

**View loaded configuration**:
```python
from aleph.core import settings

# In Flask shell
flask shell
>>> from aleph.core import settings
>>> settings.DATABASE_URI
>>> settings.ELASTICSEARCH_URL
>>> settings.OAUTH
```

**Check environment variables**:
```bash
# All ALEPH_ variables
env | grep ALEPH_

# Specific variable
echo $ALEPH_SECRET_KEY

# Docker Compose
docker-compose exec api env | grep ALEPH_
```

**Validate aleph.env file**:
```bash
# Check for syntax errors
cat aleph.env

# Load and display
export $(cat aleph.env | xargs)
env | grep ALEPH_
```

---

**Last Updated:** 2025-12-08
**Version:** 4.1.7
**Completeness:** 100%
