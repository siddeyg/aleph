# Aleph Development Guide

Complete guide for setting up a local development environment and contributing to Aleph.

## Table of Contents
- [Quick Start](#quick-start)
- [Prerequisites](#prerequisites)
- [Development Setup](#development-setup)
- [Running Aleph Locally](#running-aleph-locally)
- [Frontend Development](#frontend-development)
- [Backend Development](#backend-development)
- [Testing](#testing)
- [Code Quality](#code-quality)
- [Database Migrations](#database-migrations)
- [Debugging](#debugging)
- [Common Development Tasks](#common-development-tasks)
- [Contributing](#contributing)
- [Troubleshooting](#troubleshooting)

---

## Quick Start

**TL;DR - Get started in 5 minutes**:

```bash
# 1. Clone and enter directory
git clone https://github.com/alephdata/aleph.git
cd aleph

# 2. Copy environment template
cp aleph.env.tmpl aleph.env

# 3. Start infrastructure services
make services

# 4. Run database migrations
make upgrade

# 5. Start API and UI
make web

# Access at http://localhost:8080
```

---

## Prerequisites

### Required Software

| Software | Minimum Version | Purpose |
|----------|----------------|---------|
| **Docker** | 20.10+ | Container runtime |
| **Docker Compose** | 2.0+ | Multi-container orchestration |
| **Git** | 2.x | Version control |
| **Python** | 3.10 | Backend development (native) |
| **Node.js** | 16.x | Frontend development (native) |
| **Make** | Any | Build automation |

### Operating System

- **Linux** (Ubuntu 20.04+, Debian 11+) - Recommended
- **macOS** (Big Sur 11+)
- **Windows** (WSL2) - Use Ubuntu 20.04+ in WSL2

### Hardware Requirements

**Minimum**:
- 8 GB RAM
- 20 GB disk space
- 2 CPU cores

**Recommended**:
- 16 GB RAM
- 50 GB disk space
- 4+ CPU cores

---

## Development Setup

### 1. Clone Repository

```bash
git clone https://github.com/alephdata/aleph.git
cd aleph
```

### 2. Configuration

**Copy environment template**:
```bash
cp aleph.env.tmpl aleph.env
```

**Edit aleph.env** (minimal required):
```bash
# Security (generate with: openssl rand -hex 32)
ALEPH_SECRET_KEY=your-secret-key-here

# Database
ALEPH_DATABASE_URI=postgresql://aleph:aleph@postgres:5432/aleph

# Search
ALEPH_ELASTICSEARCH_URI=http://elasticsearch:9200

# Cache & Queue
REDIS_URL=redis://redis:6379/0

# Debug mode
ALEPH_DEBUG=true
ALEPH_CACHE=false
```

### 3. Start Infrastructure

**Using Docker Compose** (Recommended):
```bash
make services
```

This starts:
- PostgreSQL (port 5432)
- Elasticsearch (port 9200)
- Redis (port 6379)
- RabbitMQ (port 5672, management: 15672)
- ingest-file service

**Verify services are running**:
```bash
docker compose -f docker-compose.dev.yml ps
```

### 4. Initialize Database

```bash
# Run migrations
make upgrade

# Create admin user
docker compose -f docker-compose.dev.yml run --rm app \
  aleph createuser --admin admin@example.org
```

### 5. Start Development Servers

**Option A: Docker-based (easier)**:
```bash
# Start both API and UI
make web

# Or separately:
make api  # API only (port 5000)
make worker  # Background worker
```

**Option B: Native (faster iteration)**:
```bash
# Install dependencies
make dev

# Backend
FLASK_ENV=development FLASK_APP=aleph.manage:app flask run --port 5000

# Worker (in new terminal)
aleph worker

# Frontend (in new terminal)
cd ui
npm install
npm start  # Starts on port 3000
```

**Access application**:
- **UI**: http://localhost:8080 (Docker) or http://localhost:3000 (native)
- **API**: http://localhost:5000
- **API Docs**: http://localhost:5000/api/openapi.json

---

## Running Aleph Locally

### Development Architecture

```
┌─────────────────────────────────────────┐
│  Browser: http://localhost:8080        │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────┐  ┌────────────────────┐
│  React UI (port 3000)       │  │  API (port 5000)   │
│  npm start                  │  │  flask run         │
└─────────────────────────────┘  └──────┬─────────────┘
                                        │
              ┌─────────────────────────┼─────────────┐
              │                         │             │
         ┌────▼────┐  ┌────────────┐ ┌─▼──────┐  ┌───▼──────┐
         │Postgres │  │Elasticsearch│ │ Redis │  │ RabbitMQ │
         │(Docker) │  │  (Docker)  │ │(Docker│  │ (Docker) │
         └─────────┘  └────────────┘ └───────┘  └──────────┘
```

### Make Commands

| Command | Description |
|---------|-------------|
| `make services` | Start infrastructure (DB, ES, Redis, etc.) |
| `make upgrade` | Run database migrations |
| `make web` | Start API + UI (Docker) |
| `make api` | Start API only |
| `make worker` | Start background worker with debugger |
| `make shell` | Open bash shell in app container |
| `make test` | Run backend tests |
| `make test-ui` | Run frontend tests |
| `make lint` | Check Python code style (ruff) |
| `make lint-ui` | Check TypeScript code style (ESLint) |
| `make format` | Format Python code (black) |
| `make format-ui` | Format TypeScript code (Prettier) |
| `make tail` | Follow logs from all containers |
| `make stop` | Stop all containers |
| `make clean` | Remove build artifacts |
| `make build` | Rebuild Docker images |

### Environment Files

**aleph.env** - Main configuration:
```bash
# Required
ALEPH_SECRET_KEY=...
ALEPH_DATABASE_URI=...

# Optional development settings
ALEPH_DEBUG=true
ALEPH_CACHE=false
ALEPH_PASSWORD_LOGIN=true
ALEPH_SINGLE_USER=false  # Set to true for no-auth mode
```

**docker-compose.dev.yml** - Development services:
- Mounts local code into containers for live reloading
- Exposes debug ports
- Uses development settings

---

## Frontend Development

### Setup

```bash
cd ui
npm install
npm start
```

**Runs on**: http://localhost:3000
**Proxies API requests to**: http://localhost:5000

### Project Structure

```
ui/
├── src/
│   ├── app/              # App setup (store, router)
│   │   ├── store.js      # Redux store configuration
│   │   ├── api.js        # API client
│   │   └── Router.jsx    # Route definitions
│   ├── actions/          # Redux actions (30+ files)
│   ├── reducers/         # Redux reducers
│   ├── selectors.js      # Redux selectors
│   ├── components/       # Reusable components (38 dirs)
│   │   ├── common/       # Generic UI components
│   │   ├── Entity/       # Entity display components
│   │   ├── Collection/   # Collection components
│   │   └── ...
│   ├── screens/          # Full-page views (26 screens)
│   │   ├── HomeScreen/
│   │   ├── CollectionScreen/
│   │   ├── EntityScreen/
│   │   └── ...
│   ├── dialogs/          # Modal dialogs (22 dialogs)
│   ├── viewers/          # Document viewers (PDF, text, etc.)
│   └── util/             # Utility functions
├── public/               # Static assets
└── package.json          # Dependencies
```

### Tech Stack

- **React** 17.0.2 - UI framework
- **Redux** 4.2.1 - State management
- **React Router** 6.28.1 - Routing
- **Blueprint.js** 4.18.0 - UI components
- **Axios** 0.30.0 - HTTP client
- **TypeScript** - Type safety
- **Intl** - Internationalization

### Development Workflow

**1. Make changes** to any file in `ui/src/`

**2. Hot reload** automatically applies changes

**3. Check browser console** for errors

**4. Run tests**:
```bash
npm test            # Run tests once
npm test -- --watch # Watch mode
```

**5. Lint and format**:
```bash
npm run lint        # Check code style
npm run format      # Auto-format code
npm run type-check  # TypeScript checking
```

### Adding a New Feature

**1. Create component**:
```typescript
// ui/src/components/MyFeature/MyFeature.tsx
import React from 'react';
import { Button } from '@blueprintjs/core';

export function MyFeature() {
  return (
    <div>
      <Button text="Click me" />
    </div>
  );
}
```

**2. Create actions**:
```javascript
// ui/src/actions/myFeatureActions.js
import { createAction } from 'redux-act';

export const doSomething = createAction('MY_FEATURE_DO_SOMETHING');
```

**3. Create reducer**:
```javascript
// ui/src/reducers/myFeature.js
import { createReducer } from 'redux-act';
import { doSomething } from 'actions/myFeatureActions';

const initialState = {};

export default createReducer({
  [doSomething]: (state, payload) => ({ ...state, ...payload })
}, initialState);
```

**4. Add route** (if needed):
```typescript
// ui/src/app/Router.jsx
<Route path="/my-feature" element={<MyFeatureScreen />} />
```

### Building for Production

```bash
cd ui
npm run build
```

**Output**: `ui/build/` directory with optimized static files

---

## Backend Development

### Setup

**Docker-based**:
```bash
make services
make shell  # Opens bash in app container
```

**Native Python**:
```bash
# Create virtualenv
python3 -m venv env
source env/bin/activate

# Install dependencies
make dev
# Or manually:
pip install -r requirements.txt
pip install -r requirements-dev.txt
pip install -e .
```

### Project Structure

```
aleph/
├── __init__.py
├── core.py                # Flask app factory
├── settings.py            # Configuration
├── authz.py               # Authorization system
├── worker.py              # Background worker
├── manage.py              # CLI commands
├── wsgi.py                # WSGI entry point
├── model/                 # SQLAlchemy models (12 files)
│   ├── common.py          # Base models
│   ├── role.py            # Users/groups
│   ├── collection.py      # Collections
│   ├── entity.py          # Entities
│   └── ...
├── logic/                 # Business logic (23 files)
│   ├── collections.py
│   ├── entities.py
│   ├── xref.py            # Cross-reference
│   └── ...
├── views/                 # API endpoints (28 files)
│   ├── base_api.py
│   ├── collections_api.py
│   ├── entities_api.py
│   └── ...
├── search/                # Elasticsearch queries
│   ├── query.py
│   ├── parser.py
│   └── facet.py
├── index/                 # Index management
│   ├── indexes.py
│   ├── entities.py
│   └── collections.py
├── migrate/               # Database migrations
│   └── versions/          # Alembic migration files
├── tests/                 # Test suite
│   ├── factories/
│   ├── fixtures/
│   └── test_*.py
└── translations/          # i18n files
```

### Tech Stack

- **Python** 3.10
- **Flask** 2.3.3 - Web framework
- **SQLAlchemy** 2.0.21 - ORM
- **Elasticsearch** 7.17.0 - Search
- **FollowTheMoney** 3.5.9 - Entity schema
- **Marshmallow** 3.21.1 - Serialization
- **Gunicorn** 22.0.0 - WSGI server

### Development Workflow

**1. Make changes** to Python files

**2. Restart API** (if using Docker):
```bash
# Ctrl+C to stop, then:
make api
```

**Native development** (auto-reload):
```bash
FLASK_ENV=development FLASK_APP=aleph.manage:app flask run
```

**3. Check logs**:
```bash
make tail
# Or specific service:
docker compose -f docker-compose.dev.yml logs -f api
```

**4. Run tests**:
```bash
make test
# Or specific test:
make test file=aleph/tests/test_collections_api.py
```

**5. Lint and format**:
```bash
make lint           # Ruff linting
make format         # Black formatting
make format-check   # Check formatting without changing
```

### Adding a New API Endpoint

**1. Create view function**:
```python
# aleph/views/myresource_api.py
from flask import Blueprint, request
from aleph.authz import Authz
from aleph.views.util import jsonify

blueprint = Blueprint('myresource', __name__)

@blueprint.route('/api/2/myresource', methods=['GET'])
def list_resources():
    """List my resources."""
    authz = request.authz
    # ... logic here
    return jsonify({'results': []})
```

**2. Register blueprint**:
```python
# aleph/views/__init__.py
from aleph.views.myresource_api import blueprint as myresource

# In mount_app_blueprints():
app.register_blueprint(myresource)
```

**3. Add logic layer**:
```python
# aleph/logic/myresource.py
from aleph.core import db

def create_resource(data, authz):
    """Create a new resource."""
    # Business logic here
    db.session.commit()
    return resource
```

**4. Add tests**:
```python
# aleph/tests/test_myresource_api.py
from aleph.tests.util import TestCase

class MyResourceApiTestCase(TestCase):
    def test_list_resources(self):
        res = self.client.get('/api/2/myresource')
        assert res.status_code == 200
```

### CLI Commands

**User management**:
```bash
aleph createuser admin@example.org --admin
aleph updateuser admin@example.org --password newpass
aleph deleteuser user@example.org
```

**Collection management**:
```bash
aleph crawldir -f mycollection /path/to/documents
aleph reindex mycollection
aleph delete mycollection
```

**Index management**:
```bash
aleph index              # Reindex all
aleph upgrade            # Run migrations
aleph resetindex         # Delete and recreate indices
```

**Development**:
```bash
aleph worker             # Start worker
aleph shell              # Python REPL with app context
```

---

## Testing

### Backend Tests

**Run all tests**:
```bash
make test
```

**Run specific test file**:
```bash
make test file=aleph/tests/test_entities_api.py
```

**Run specific test function**:
```bash
docker compose -f docker-compose.dev.yml run --rm app \
  pytest aleph/tests/test_entities_api.py::TestEntitiesAPI::test_create_entity
```

**With coverage**:
```bash
docker compose -f docker-compose.dev.yml run --rm app \
  pytest --cov=aleph --cov-report=html
```

**Test Structure**:
```python
from aleph.tests.util import TestCase

class MyTestCase(TestCase):
    def setUp(self):
        super().setUp()
        # Setup test data
        self.user = self.create_user()
        self.collection = self.create_collection()

    def test_something(self):
        """Test description."""
        # Arrange
        data = {'label': 'Test'}

        # Act
        res = self.client.post('/api/2/collections',
                               json=data,
                               headers=self.get_authz_headers(self.user))

        # Assert
        assert res.status_code == 200
        assert res.json['label'] == 'Test'
```

**Test Fixtures**:
```python
from aleph.tests.factories import RoleFactory, CollectionFactory

def test_with_factories(self):
    user = RoleFactory.create(name='John Doe')
    collection = CollectionFactory.create(creator=user)
```

### Frontend Tests

**Run tests**:
```bash
make test-ui
# Or:
cd ui && npm test
```

**Watch mode**:
```bash
cd ui && npm test -- --watch
```

**Coverage**:
```bash
cd ui && npm test -- --coverage
```

**Test Structure**:
```typescript
import { render, screen } from '@testing-library/react';
import { MyComponent } from './MyComponent';

describe('MyComponent', () => {
  it('renders correctly', () => {
    render(<MyComponent title="Test" />);
    expect(screen.getByText('Test')).toBeInTheDocument();
  });
});
```

---

## Code Quality

### Python Code Style

**Linting** (Ruff):
```bash
make lint
# Or:
ruff check .
```

**Formatting** (Black):
```bash
make format           # Auto-format
make format-check     # Check only
```

**Configuration**: `pyproject.toml`, `ruff.toml`

### TypeScript Code Style

**Linting** (ESLint):
```bash
make lint-ui
# Or:
cd ui && npm run lint
```

**Formatting** (Prettier):
```bash
make format-ui          # Auto-format
make format-check-ui    # Check only
```

**Type checking**:
```bash
cd ui && npm run type-check
```

**Configuration**: `ui/.eslintrc.js`, `ui/.prettierrc`

### Pre-commit Checks

**Before committing, run**:
```bash
# Backend
make format
make lint
make test

# Frontend
cd ui
npm run format
npm run lint
npm test
```

---

## Database Migrations

### Creating a Migration

**1. Make model changes**:
```python
# aleph/model/collection.py
class Collection(db.Model, ...):
    new_field = db.Column(db.String)  # Add new field
```

**2. Generate migration**:
```bash
docker compose -f docker-compose.dev.yml run --rm app \
  aleph db migrate -m "Add new_field to collection"
```

**3. Review migration file**:
```bash
# Check: aleph/migrate/versions/XXXX_add_new_field_to_collection.py
```

**4. Edit if needed** (for complex changes)

**5. Apply migration**:
```bash
make upgrade
# Or:
aleph db upgrade
```

### Migration Best Practices

**1. Test in fresh database**:
```bash
# Drop database
docker compose -f docker-compose.dev.yml down -v
docker compose -f docker-compose.dev.yml up -d postgres

# Run migrations
make upgrade
```

**2. Reversible migrations**:
```python
def upgrade():
    op.add_column('collection', sa.Column('new_field', sa.String()))

def downgrade():
    op.drop_column('collection', 'new_field')
```

**3. Data migrations**:
```python
from alembic import op
import sqlalchemy as sa

def upgrade():
    # Schema change
    op.add_column('collection', sa.Column('status', sa.String()))

    # Data migration
    connection = op.get_bind()
    connection.execute("UPDATE collection SET status = 'active' WHERE deleted_at IS NULL")
```

### Migration Commands

```bash
aleph db current        # Show current revision
aleph db history        # Show migration history
aleph db upgrade        # Apply all pending migrations
aleph db upgrade +1     # Apply one migration
aleph db downgrade -1   # Rollback one migration
aleph db stamp head     # Mark as up-to-date without running
```

---

## Debugging

### Backend Debugging

**Visual Studio Code** (launch.json):
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: Flask",
      "type": "python",
      "request": "launch",
      "module": "flask",
      "env": {
        "FLASK_APP": "aleph.manage:app",
        "FLASK_ENV": "development"
      },
      "args": ["run", "--no-debugger", "--no-reload"],
      "jinja": true
    }
  ]
}
```

**PyCharm**:
1. Run → Edit Configurations
2. Add Python configuration
3. Script path: `flask`
4. Parameters: `run`
5. Environment variables: `FLASK_APP=aleph.manage:app`

**debugpy (Remote debugging)**:
```bash
# Start worker with debugger (port 5679)
make worker
```

**VSCode attach** (launch.json):
```json
{
  "name": "Attach to Worker",
  "type": "python",
  "request": "attach",
  "connect": {
    "host": "localhost",
    "port": 5679
  }
}
```

**Print debugging**:
```python
import structlog
log = structlog.get_logger(__name__)

log.info("Debug message", entity_id=entity.id, schema=entity.schema)
```

### Frontend Debugging

**React DevTools**:
- Install browser extension
- View component tree, props, state

**Redux DevTools**:
- Install browser extension
- View actions, state changes, time-travel

**Browser DevTools**:
- Console: View logs, errors
- Network: Monitor API requests
- Sources: Set breakpoints

**VSCode debugging** (launch.json):
```json
{
  "name": "Chrome: Launch",
  "type": "chrome",
  "request": "launch",
  "url": "http://localhost:3000",
  "webRoot": "${workspaceFolder}/ui/src"
}
```

### Database Debugging

**pgAdmin**:
```bash
# Access Postgres directly
docker compose -f docker-compose.dev.yml exec postgres psql -U aleph aleph
```

**Query logs**:
```python
# Enable SQLAlchemy query logging
import logging
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)
```

### Elasticsearch Debugging

**Query logs**:
```python
# In Python code
import logging
logging.getLogger('elasticsearch').setLevel(logging.DEBUG)
```

**Direct queries**:
```bash
# Check indices
curl http://localhost:9200/_cat/indices?v

# Search
curl http://localhost:9200/aleph-entity-*/_search?pretty
```

---

## Common Development Tasks

### Add a New Model

**1. Create model file**:
```python
# aleph/model/mymodel.py
from aleph.core import db
from aleph.model.common import IdModel, DatedModel

class MyModel(db.Model, IdModel, DatedModel):
    __tablename__ = 'my_model'

    name = db.Column(db.Unicode, nullable=False)
    description = db.Column(db.Unicode)
```

**2. Import in model init**:
```python
# aleph/model/__init__.py
from aleph.model.mymodel import MyModel  # noqa
```

**3. Create migration**:
```bash
aleph db migrate -m "Add MyModel"
```

**4. Apply migration**:
```bash
make upgrade
```

### Add Translation

**1. Mark strings for translation**:
```python
from flask_babel import lazy_gettext

message = lazy_gettext("This will be translated")
```

**2. Extract messages**:
```bash
make translate
```

**3. Update translations**:
```bash
# Edit: aleph/translations/{lang}/LC_MESSAGES/aleph.po
```

**4. Compile**:
```bash
pybabel compile -f -d aleph/translations -D aleph
```

### Update Dependencies

**Backend**:
```bash
# Edit requirements.txt
pip install -r requirements.txt
```

**Frontend**:
```bash
cd ui
# Edit package.json
npm install
```

**Rebuild Docker images**:
```bash
make build
```

---

## Contributing

### Contribution Workflow

**1. Fork repository** on GitHub

**2. Clone your fork**:
```bash
git clone https://github.com/YOUR_USERNAME/aleph.git
cd aleph
git remote add upstream https://github.com/alephdata/aleph.git
```

**3. Create feature branch**:
```bash
git checkout -b feature/my-feature
```

**4. Make changes and commit**:
```bash
git add .
git commit -m "feat: Add my feature"
```

**5. Push to your fork**:
```bash
git push origin feature/my-feature
```

**6. Create Pull Request** on GitHub

### Commit Message Format

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>: <description>

[optional body]

[optional footer]
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `test`: Adding or updating tests
- `refactor`: Code refactoring
- `style`: Code style changes
- `chore`: Build process, dependencies

**Examples**:
```
feat: Add entity merge functionality
fix: Resolve xref scoring edge case
docs: Update API documentation for entitysets
test: Add tests for alert notifications
refactor: Simplify query builder pattern
```

### Code Review Checklist

- [ ] Tests pass (`make test`, `make test-ui`)
- [ ] Code is formatted (`make format`, `make format-ui`)
- [ ] Linting passes (`make lint`, `make lint-ui`)
- [ ] Documentation updated
- [ ] Migration created (if models changed)
- [ ] No console warnings
- [ ] Works in Chrome, Firefox, Safari
- [ ] Commit messages follow convention

---

## Troubleshooting

### "Port already in use"

**Cause**: Another service is using the port

**Solution**:
```bash
# Find process using port
lsof -i :5000  # Or :3000, :5432, etc.

# Kill process
kill -9 <PID>

# Or use different ports
FLASK_RUN_PORT=5001 flask run
```

### "Cannot connect to database"

**Cause**: PostgreSQL not running or wrong connection string

**Solution**:
```bash
# Check if running
docker compose -f docker-compose.dev.yml ps postgres

# Start if stopped
make services

# Check connection
docker compose -f docker-compose.dev.yml exec postgres psql -U aleph aleph
```

### "Elasticsearch connection failed"

**Cause**: ES not ready or out of memory

**Solution**:
```bash
# Check status
curl http://localhost:9200/_cluster/health?pretty

# Check logs
docker compose -f docker-compose.dev.yml logs elasticsearch

# Increase heap size (docker-compose.dev.yml)
environment:
  - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
```

### "npm install fails"

**Cause**: Node version mismatch or corrupt cache

**Solution**:
```bash
# Clear cache
rm -rf ui/node_modules ui/package-lock.json
cd ui && npm cache clean --force

# Use correct Node version (nvm)
nvm install 16
nvm use 16

# Reinstall
npm install
```

### "Migration fails"

**Cause**: Database state doesn't match migration

**Solution**:
```bash
# Check current revision
aleph db current

# Force stamp to specific version
aleph db stamp <revision>

# Or reset database
docker compose -f docker-compose.dev.yml down -v
make upgrade
```

### "Worker not processing tasks"

**Cause**: RabbitMQ connection or worker not running

**Solution**:
```bash
# Check RabbitMQ
docker compose -f docker-compose.dev.yml ps rabbitmq

# Check queue size
curl -u guest:guest http://localhost:15672/api/queues

# Start worker
make worker
```

### "Slow indexing performance"

**Cause**: Small batch sizes or ES resource constraints

**Solution**:
```bash
# Increase batch size (aleph.env)
ALEPH_INDEXING_BATCH_SIZE=1000
ALEPH_INDEXING_TIMEOUT=30

# Increase ES resources
# Edit docker-compose.dev.yml:
ES_JAVA_OPTS=-Xms4g -Xmx4g
```

### "Cannot import name X"

**Cause**: Circular import or missing dependency

**Solution**:
```bash
# Reinstall in development mode
pip install -e .

# Check for circular imports
import aleph  # Should not error
```

### Getting Help

- **GitHub Discussions**: https://github.com/alephdata/aleph/discussions
- **GitHub Issues**: https://github.com/alephdata/aleph/issues
- **Documentation**: https://docs.alephdata.org
- **Community**: Join OCCRP community

---

**Last Updated:** 2025-12-08
**Version:** 4.1.7
**Completeness:** 100%
