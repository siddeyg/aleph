# Investigations (EntitySets)

**Last Updated:** 2025-12-09

## Overview

Investigations in Aleph are built around **EntitySets** - collections of entities used for investigative analysis and collaboration. EntitySets provide four distinct ways to organize, visualize, and analyze entities within collections:

1. **Lists** - Simple entity collections for organization
2. **Diagrams** - Network visualizations with persistent layouts
3. **Timelines** - Temporal/chronological analysis
4. **Profiles** - Entity resolution and deduplication

EntitySets inherit permissions from their parent Collections, enabling secure sharing and collaboration across investigation teams.

## Table of Contents

- [EntitySet Types](#entityset-types)
- [Data Models](#data-models)
- [API Reference](#api-reference)
- [Workflows](#workflows)
- [Frontend Components](#frontend-components)
- [Permissions and Sharing](#permissions-and-sharing)
- [Examples](#examples)
- [Best Practices](#best-practices)

---

## EntitySet Types

### 1. Lists

**Purpose:** Basic organization of related entities without additional structure.

**Use Cases:**
- Grouping entities by category (e.g., "Key Witnesses", "Offshore Companies")
- Building investigation checklists
- Organizing entities for export or further analysis
- Creating simple entity collections for team review

**Storage:** Entity IDs only, no additional metadata.

**Example:**
```json
{
  "id": "entityset-123",
  "label": "Key People of Interest",
  "type": "list",
  "collection_id": "collection-456",
  "count": 42,
  "created_at": "2025-01-15T10:30:00Z"
}
```

### 2. Diagrams

**Purpose:** Visual network analysis with persistent node positioning.

**Use Cases:**
- Mapping relationships between people, companies, and assets
- Visualizing ownership structures
- Analyzing financial flows
- Creating presentation-ready network visualizations
- Understanding complex entity relationships

**Storage:**
- Entity IDs
- Layout coordinates (x, y positions) stored as JSONB
- Edge/relationship metadata

**Visualization:**
- D3.js force-directed layout
- Dagre automatic layout algorithms
- Manual drag-and-drop positioning (react-draggable)
- Persistent layout saves user arrangements

**Layout Structure:**
```json
{
  "layout": {
    "entity-1": {"x": 100, "y": 200},
    "entity-2": {"x": 300, "y": 200},
    "entity-3": {"x": 200, "y": 400}
  }
}
```

**Example:**
```json
{
  "id": "entityset-789",
  "label": "Offshore Network Diagram",
  "type": "diagram",
  "collection_id": "collection-456",
  "count": 25,
  "layout": {
    "entity-abc": {"x": 150, "y": 100},
    "entity-def": {"x": 350, "y": 250}
  },
  "created_at": "2025-01-20T14:00:00Z"
}
```

### 3. Timelines

**Purpose:** Chronological visualization of entities and events.

**Use Cases:**
- Tracking events over time
- Analyzing temporal patterns
- Identifying clusters of activity
- Creating chronological narratives
- Time-based filtering and analysis

**Storage:**
- Entity IDs
- Date properties extracted from entities
- Event metadata

**Visualization:**
- Recharts library for rendering
- Date range filtering
- Event clustering by time periods
- Interactive timeline navigation

**Example:**
```json
{
  "id": "entityset-321",
  "label": "Panama Papers Timeline",
  "type": "timeline",
  "collection_id": "collection-456",
  "count": 150,
  "created_at": "2025-02-01T09:00:00Z"
}
```

### 4. Profiles

**Purpose:** Entity resolution, deduplication, and manual review of cross-reference matches.

**Use Cases:**
- Reviewing entity match candidates from xref
- Making judgements on entity duplicates
- Merging confirmed duplicate entities
- Managing entity resolution workflow
- Quality control for entity data

**Storage:**
- Entity IDs (match candidates)
- Judgement records (POSITIVE/NEGATIVE/UNSURE)
- Match scores and metadata

**Judgement Types:**
- `POSITIVE` - Entities are the same (should be merged)
- `NEGATIVE` - Entities are different (keep separate)
- `UNSURE` - Needs further review or information
- `NO_JUDGEMENT` - No decision recorded (default state)

**Workflow Integration:**
Profiles integrate with the cross-reference matching system:
```
1. Xref generates match candidates (score > 0.5)
2. Matches appear in Profile EntitySet
3. User reviews each match
4. User records judgement
5. POSITIVE matches can be merged
```

**Example:**
```json
{
  "id": "entityset-654",
  "label": "Duplicate Person Review",
  "type": "profile",
  "collection_id": "collection-456",
  "count": 78,
  "created_at": "2025-01-25T11:00:00Z"
}
```

---

## Data Models

### EntitySet Model

**File:** `aleph/model/entityset.py`

**Schema:**
```python
class EntitySet(db.Model, IdModel, SoftDeleteModel, DatedModel):
    __tablename__ = 'entityset'

    # Core fields
    id = db.Column(db.String(32), primary_key=True)
    label = db.Column(db.Unicode, nullable=False)
    type = db.Column(db.Enum('list', 'diagram', 'timeline', 'profile'))
    summary = db.Column(db.Unicode, nullable=True)  # Optional description

    # Relationships
    collection_id = db.Column(db.Integer, db.ForeignKey('collection.id'))
    collection = db.relationship('Collection', backref='entitysets')

    role_id = db.Column(db.Integer, db.ForeignKey('role.id'))  # Owner/creator
    role = db.relationship('Role')

    parent_id = db.Column(db.String(32), db.ForeignKey('entityset.id'))  # For nesting
    parent = db.relationship('EntitySet', backref='children', remote_side=[id])

    # Type-specific data
    layout = db.Column(JSONB)  # For diagram coordinates

    # Metadata
    count = db.Column(db.Integer, default=0)  # Number of entities
    created_at = db.Column(db.DateTime)
    updated_at = db.Column(db.DateTime)
    deleted_at = db.Column(db.DateTime)  # Soft delete

    # Relationships
    items = db.relationship('EntitySetItem', backref='entityset')
```

**Key Properties:**

| Property | Type | Description |
|----------|------|-------------|
| `id` | String(32) | Unique identifier |
| `label` | Unicode | Display name |
| `type` | Enum | list, diagram, timeline, profile |
| `summary` | Unicode | Optional description text |
| `collection_id` | Integer | Parent collection FK |
| `role_id` | Integer | Owner/creator role FK |
| `parent_id` | String(32) | Parent EntitySet FK (for nesting) |
| `layout` | JSONB | Diagram coordinates |
| `count` | Integer | Number of entities |
| `created_at` | DateTime | Creation timestamp |
| `updated_at` | DateTime | Last modification |
| `deleted_at` | DateTime | Soft delete timestamp |

### EntitySetItem Model

**File:** `aleph/model/entityset_item.py`

**Schema:**
```python
class EntitySetItem(db.Model, IdModel, DatedModel):
    __tablename__ = 'entityset_item'

    # Relationships
    entityset_id = db.Column(db.String(32), db.ForeignKey('entityset.id'))
    entity_id = db.Column(db.String(32), db.ForeignKey('entity.id'))
    collection_id = db.Column(db.Integer, db.ForeignKey('collection.id'))

    # Profile judgements (for type='profile')
    compared_to_entity_id = db.Column(db.String(32))  # Match candidate
    added_by_id = db.Column(db.Integer, db.ForeignKey('role.id'))
    judgement = db.Column(db.Enum(Judgement))  # POSITIVE, NEGATIVE, UNSURE, NO_JUDGEMENT

    # Optional metadata
    position = db.Column(db.Integer)  # Order in list
    deleted_at = db.Column(db.DateTime)  # Soft delete

    # Relationships
    entityset = db.relationship('EntitySet', backref='items')
    entity = db.relationship('Entity')
    collection = db.relationship('Collection')
    added_by = db.relationship('Role')

    # Constraints
    __table_args__ = (
        db.UniqueConstraint('entityset_id', 'entity_id'),
    )
```

**Purpose:** Junction table enabling many-to-many relationship between EntitySets and Entities.

---

## API Reference

### Base Path

`/api/2/entitysets`

### List EntitySets

**Endpoint:** `GET /api/2/entitysets`

**Description:** Retrieve all EntitySets accessible to the current user.

**Query Parameters:**
- `type` (optional) - Filter by type (list, diagram, timeline, profile)
- `collection_id` (optional) - Filter by collection
- `limit` (optional) - Results per page (default: 30)
- `offset` (optional) - Pagination offset

**Request:**
```bash
GET /api/2/entitysets?type=diagram&collection_id=collection-123
```

**Response:**
```json
{
  "results": [
    {
      "id": "entityset-789",
      "label": "Corporate Network",
      "type": "diagram",
      "summary": "Ownership structure analysis for XYZ Corp",
      "collection_id": "collection-123",
      "role_id": "user-456",
      "count": 45,
      "created_at": "2025-01-15T10:30:00Z",
      "updated_at": "2025-01-20T14:22:00Z"
    }
  ],
  "total": 1,
  "page": 1,
  "limit": 30
}
```

**Authorization:** Requires READ permission on collections.

### Create EntitySet

**Endpoint:** `POST /api/2/entitysets`

**Description:** Create a new EntitySet.

**Request Body:**
```json
{
  "label": "My Investigation Diagram",
  "type": "diagram",
  "summary": "Network analysis for investigation #42",
  "collection_id": "collection-123",
  "entities": ["entity-1", "entity-2", "entity-3"],
  "layout": {
    "entity-1": {"x": 100, "y": 200},
    "entity-2": {"x": 300, "y": 200},
    "entity-3": {"x": 200, "y": 400}
  }
}
```

**Response:**
```json
{
  "id": "entityset-new",
  "label": "My Investigation Diagram",
  "type": "diagram",
  "summary": "Network analysis for investigation #42",
  "collection_id": "collection-123",
  "role_id": "user-current",
  "parent_id": null,
  "count": 3,
  "layout": {
    "entity-1": {"x": 100, "y": 200},
    "entity-2": {"x": 300, "y": 200},
    "entity-3": {"x": 200, "y": 400}
  },
  "created_at": "2025-12-09T15:00:00Z",
  "updated_at": "2025-12-09T15:00:00Z"
}
```

**Status Code:** `201 Created`

**Authorization:** Requires WRITE permission on the collection.

**Validation:**
- `label` is required (1-255 characters)
- `type` must be one of: list, diagram, timeline, profile
- `summary` is optional (description text)
- `collection_id` must reference an accessible collection
- `entities` array is optional (can be added later)
- `layout` only applies to type=diagram
- `role_id` is automatically set to current user
- `parent_id` is optional (for nested EntitySets)

### Get EntitySet

**Endpoint:** `GET /api/2/entitysets/:id`

**Description:** Retrieve details of a specific EntitySet.

**Request:**
```bash
GET /api/2/entitysets/entityset-789
```

**Response:**
```json
{
  "id": "entityset-789",
  "label": "Corporate Network",
  "type": "diagram",
  "summary": "Ownership structure analysis for XYZ Corp",
  "collection_id": "collection-123",
  "collection": {
    "id": "collection-123",
    "label": "Panama Papers",
    "category": "leak"
  },
  "role_id": "user-456",
  "parent_id": null,
  "count": 45,
  "layout": {
    "entity-abc": {"x": 150, "y": 100},
    "entity-def": {"x": 350, "y": 250}
  },
  "created_at": "2025-01-15T10:30:00Z",
  "updated_at": "2025-01-20T14:22:00Z"
}
```

**Authorization:** Requires READ permission on the collection.

### Update EntitySet

**Endpoint:** `PUT /api/2/entitysets/:id`

**Description:** Update EntitySet metadata and layout.

**Request Body:**
```json
{
  "label": "Updated Investigation Name",
  "summary": "Updated description with new findings",
  "layout": {
    "entity-abc": {"x": 200, "y": 150},
    "entity-def": {"x": 400, "y": 300}
  }
}
```

**Response:**
```json
{
  "id": "entityset-789",
  "label": "Updated Investigation Name",
  "type": "diagram",
  "summary": "Updated description with new findings",
  "collection_id": "collection-123",
  "role_id": "user-456",
  "count": 45,
  "layout": {
    "entity-abc": {"x": 200, "y": 150},
    "entity-def": {"x": 400, "y": 300}
  },
  "updated_at": "2025-12-09T15:30:00Z"
}
```

**Authorization:** Requires WRITE permission on the collection.

**Notes:**
- Only `label`, `summary`, and `layout` can be updated
- Cannot change `type`, `collection_id`, `role_id`, or `parent_id` after creation

### Delete EntitySet

**Endpoint:** `DELETE /api/2/entitysets/:id`

**Description:** Soft-delete an EntitySet.

**Response:**
```json
{
  "status": "ok"
}
```

**Status Code:** `204 No Content`

**Authorization:** Requires WRITE permission on the collection.

**Notes:**
- Performs soft delete (sets `deleted_at` timestamp)
- EntitySetItems are also soft-deleted
- Can be recovered by administrators

### Get Entities in EntitySet

**Endpoint:** `GET /api/2/entitysets/:id/entities`

**Description:** Retrieve all entities within an EntitySet.

**Query Parameters:**
- `limit` (optional) - Results per page
- `offset` (optional) - Pagination offset
- `q` (optional) - Search within entities

**Request:**
```bash
GET /api/2/entitysets/entityset-789/entities?limit=10&offset=0
```

**Response:**
```json
{
  "results": [
    {
      "id": "entity-abc",
      "schema": "Company",
      "properties": {
        "name": ["Mossack Fonseca"],
        "country": ["PA"],
        "incorporationDate": ["1977-01-01"]
      },
      "collection_id": "collection-123"
    },
    {
      "id": "entity-def",
      "schema": "Person",
      "properties": {
        "name": ["John Smith"],
        "nationality": ["GB"]
      },
      "collection_id": "collection-123"
    }
  ],
  "total": 45,
  "page": 1,
  "limit": 10
}
```

**Authorization:** Requires READ permission on the collection.

### Add Entities to EntitySet

**Endpoint:** `PUT /api/2/entitysets/:id/entities`

**Description:** Add entities to an EntitySet.

**Request Body:**
```json
{
  "entities": ["entity-3", "entity-4", "entity-5"]
}
```

**Response:**
```json
{
  "status": "ok",
  "count": 48
}
```

**Authorization:** Requires WRITE permission on the collection.

**Validation:**
- Entities must exist in the same collection
- Duplicates are ignored (idempotent operation)
- EntitySet count is automatically updated

### Remove Entity from EntitySet

**Endpoint:** `DELETE /api/2/entitysets/:id/entities/:entity_id`

**Description:** Remove a specific entity from an EntitySet.

**Response:**
```json
{
  "status": "ok",
  "count": 47
}
```

**Status Code:** `204 No Content`

**Authorization:** Requires WRITE permission on the collection.

**Notes:**
- For diagrams, layout data for the entity is also removed
- EntitySet count is automatically decremented

---

## Workflows

### Creating a List Investigation

**Scenario:** Organize entities for review or export.

**Steps:**

1. **Create List:**
```bash
POST /api/2/entitysets
{
  "label": "Suspicious Transactions",
  "type": "list",
  "collection_id": "collection-123"
}
```

2. **Add Entities:**
```bash
PUT /api/2/entitysets/entityset-new/entities
{
  "entities": ["entity-1", "entity-2", "entity-3"]
}
```

3. **Review Entities:**
```bash
GET /api/2/entitysets/entityset-new/entities
```

4. **Export or Share:**
- Share collection with team members
- Export entity list for further analysis

### Creating a Network Diagram

**Scenario:** Visualize relationships between entities.

**Steps:**

1. **Create Diagram:**
```bash
POST /api/2/entitysets
{
  "label": "Corporate Ownership Network",
  "type": "diagram",
  "collection_id": "collection-123",
  "entities": ["company-1", "person-1", "company-2"]
}
```

2. **Load in Frontend:**
- Open diagram in Aleph UI
- Automatic force-directed layout applied

3. **Manual Arrangement:**
- Drag nodes to desired positions
- Frontend auto-saves layout coordinates

4. **Update Layout via API:**
```bash
PUT /api/2/entitysets/diagram-id
{
  "layout": {
    "company-1": {"x": 100, "y": 200},
    "person-1": {"x": 300, "y": 200},
    "company-2": {"x": 200, "y": 400}
  }
}
```

5. **Add More Entities:**
```bash
PUT /api/2/entitysets/diagram-id/entities
{
  "entities": ["person-2", "company-3"]
}
```

6. **Export Visualization:**
- Screenshot from UI
- Export as SVG/PNG
- Share read-only access with stakeholders

### Entity Resolution with Profiles

**Scenario:** Review and resolve duplicate entities from cross-reference matching.

**Complete Workflow:**

1. **Run Cross-Reference:**
```bash
POST /api/2/xref
{
  "collections": ["collection-123", "collection-456"]
}
```

2. **System Generates Matches:**
- Entities with score > 0.5 stored in xref index
- Match candidates created

3. **Create Profile EntitySet:**
```bash
POST /api/2/entitysets
{
  "label": "Person Deduplication",
  "type": "profile",
  "collection_id": "collection-123"
}
```

4. **Load Xref Matches:**
- Profile EntitySet automatically populated with xref candidates
- Or manually add entity pairs for review

5. **Review Each Match:**

**Positive Match (Same Entity):**
```bash
POST /api/2/entitysets/profile-id/judgements
{
  "entity_id": "entity-1",
  "match_id": "entity-2",
  "judgement": "POSITIVE"
}
```

**Negative Match (Different Entities):**
```bash
POST /api/2/entitysets/profile-id/judgements
{
  "entity_id": "entity-1",
  "match_id": "entity-3",
  "judgement": "NEGATIVE"
}
```

**Unsure (Need More Info):**
```bash
POST /api/2/entitysets/profile-id/judgements
{
  "entity_id": "entity-1",
  "match_id": "entity-4",
  "judgement": "UNSURE"
}
```

6. **Merge Positive Matches:**
```bash
POST /api/2/entities/merge
{
  "source_id": "entity-1",
  "target_id": "entity-2"
}
```

7. **Track Progress:**
- Profile EntitySet count shows remaining reviews
- Filter by judgement type to see status

### Timeline Analysis

**Scenario:** Analyze events chronologically.

**Steps:**

1. **Create Timeline:**
```bash
POST /api/2/entitysets
{
  "label": "Financial Crisis Timeline",
  "type": "timeline",
  "collection_id": "collection-123"
}
```

2. **Add Entities with Dates:**
```bash
PUT /api/2/entitysets/timeline-id/entities
{
  "entities": [
    "transaction-1",  # date: 2008-03-15
    "meeting-1",      # date: 2008-04-20
    "contract-1"      # date: 2008-05-10
  ]
}
```

3. **View in UI:**
- Frontend extracts date properties
- Entities plotted chronologically
- Recharts renders interactive timeline

4. **Filter by Date Range:**
- UI provides date range selector
- Filter entities by time period
- Analyze temporal patterns

5. **Export Timeline:**
- Screenshot visualization
- Export filtered entity list
- Generate chronological report

---

## Frontend Components

### Component Architecture

**Location:** `/ui/src/`

**Main Screens:**

1. **InvestigationsScreen** (`screens/InvestigationsScreen/`)
   - Lists all user's EntitySets
   - Filters by type, collection
   - Create new EntitySet
   - Search and pagination

2. **EntitySetScreen** (`screens/EntitySetScreen/`)
   - Detail view for specific EntitySet
   - Type-specific rendering (delegates to below)
   - Entity management (add/remove)
   - Sharing and permissions

3. **DiagramScreen** (`screens/DiagramScreen/`)
   - Diagram visualization and editing
   - Graph rendering with D3.js
   - Node drag-and-drop
   - Layout persistence

4. **TimelineScreen** (`screens/TimelineScreen/`)
   - Timeline visualization
   - Date filtering
   - Recharts integration
   - Event clustering

### Key Components

#### DiagramEditor

**File:** `ui/src/components/DiagramEditor/`

**Features:**
- Node selection and editing
- Edge creation/deletion
- Zoom and pan controls
- Layout algorithm selection
- Entity addition from search

**Dependencies:**
- `react-draggable` - Node positioning
- `d3-force` - Force-directed layout
- `dagre` - Hierarchical layout

**Example Integration:**
```jsx
import DiagramEditor from './components/DiagramEditor';

<DiagramEditor
  entitySetId="diagram-123"
  entities={entities}
  layout={layout}
  onLayoutChange={handleLayoutUpdate}
  onEntityAdd={handleEntityAdd}
  onEntityRemove={handleEntityRemove}
/>
```

#### GraphRenderer

**File:** `ui/src/components/GraphRenderer/`

**Responsibilities:**
- D3.js visualization rendering
- Node and edge drawing
- Force simulation management
- Interaction handling (drag, click, hover)

**Props:**
```typescript
interface GraphRendererProps {
  nodes: Node[];
  edges: Edge[];
  layout: Layout;
  width: number;
  height: number;
  onNodeDrag: (nodeId: string, x: number, y: number) => void;
  onNodeClick: (nodeId: string) => void;
}
```

#### Timeline Component

**File:** `ui/src/components/Timeline/`

**Features:**
- Recharts-based visualization
- Date axis formatting
- Event markers
- Zoom to date range
- Tooltip with entity details

**Example:**
```jsx
import Timeline from './components/Timeline';

<Timeline
  entities={entities}
  dateField="properties.date"
  onDateRangeChange={handleDateFilter}
/>
```

#### XrefTable

**File:** `ui/src/components/XrefTable/`

**Purpose:** Display cross-reference match candidates in Profile EntitySets.

**Features:**
- Side-by-side entity comparison
- Match score display
- Judgement buttons (Positive/Negative/Unsure)
- Bulk operations
- Filtering by judgement status

**Example:**
```jsx
import XrefTable from './components/XrefTable';

<XrefTable
  profileId="profile-123"
  matches={xrefMatches}
  onJudgement={handleJudgement}
/>
```

### Redux State

**File:** `ui/src/reducers/`

**EntitySets State:**
```javascript
{
  entitySets: {
    'entityset-123': {
      id: 'entityset-123',
      label: 'My Diagram',
      type: 'diagram',
      collection_id: 'collection-456',
      count: 45,
      layout: {...},
      created_at: '2025-01-15T10:30:00Z',
      updated_at: '2025-01-20T14:22:00Z'
    }
  },
  entitySetItems: {
    'entityset-123': ['entity-1', 'entity-2', 'entity-3']
  }
}
```

**Actions:**
```javascript
// Fetch EntitySets
dispatch(fetchEntitySets({type: 'diagram', collection_id: '123'}));

// Create EntitySet
dispatch(createEntitySet({label: 'New Diagram', type: 'diagram'}));

// Update Layout
dispatch(updateEntitySetLayout(entitySetId, newLayout));

// Add Entities
dispatch(addEntitiesToSet(entitySetId, ['entity-4', 'entity-5']));
```

### Routing

**File:** `ui/src/app/Router.tsx`

**Routes:**
```jsx
<Route path="/investigations" element={<InvestigationsScreen />} />
<Route path="/investigations/:entitySetId" element={<EntitySetScreen />} />
<Route path="/diagrams/:entitySetId" element={<DiagramScreen />} />
<Route path="/timelines/:entitySetId" element={<TimelineScreen />} />
<Route path="/profiles/:entitySetId" element={<ProfileScreen />} />
```

---

## Permissions and Sharing

### Permission Model

**Inheritance:** EntitySets inherit all permissions from their parent Collection.

**Access Levels:**
- **READ:** Can view EntitySet and its entities
- **WRITE:** Can modify EntitySet, add/remove entities, update layout

### Checking Permissions

**Backend (Python):**
```python
from aleph.authz import Authz

def view_entityset(entityset_id):
    entityset = EntitySet.by_id(entityset_id)
    authz = request.authz

    # Check read permission
    authz.require(authz.can_read(entityset.collection_id))

    return jsonify(entityset)

def update_entityset(entityset_id):
    entityset = EntitySet.by_id(entityset_id)
    authz = request.authz

    # Check write permission
    authz.require(authz.can_write(entityset.collection_id))

    entityset.update(request.json)
    return jsonify(entityset)
```

**Frontend (JavaScript):**
```javascript
import { selectPermission } from 'selectors';

const permission = selectPermission(state, collectionId);

// Check permissions
if (permission.read) {
  // User can view EntitySet
}

if (permission.write) {
  // User can modify EntitySet
  <Button onClick={handleAddEntity}>Add Entity</Button>
}
```

### Sharing Workflow

**1. Share Collection (EntitySets inherit):**

```bash
POST /api/2/collections/collection-123/permissions
{
  "role": "user-789",
  "read": true,
  "write": false
}
```

**Result:** User 789 can now:
- View all EntitySets in collection-123
- View entities within those EntitySets
- Cannot modify EntitySets or add/remove entities

**2. Grant Write Access:**

```bash
PUT /api/2/collections/collection-123/permissions/user-789
{
  "read": true,
  "write": true
}
```

**Result:** User 789 can now:
- Create new EntitySets in collection-123
- Modify existing EntitySets
- Add/remove entities
- Update diagram layouts

**3. Remove Access:**

```bash
DELETE /api/2/collections/collection-123/permissions/user-789
```

**Result:** User 789 loses all access to collection-123 and its EntitySets.

### Team Collaboration

**Scenario:** Investigation team needs to collaborate on a diagram.

**Setup:**
1. Create investigation collection
2. Create diagram EntitySet
3. Add team members with WRITE permission
4. All team members can:
   - View real-time updates
   - Add entities to diagram
   - Rearrange nodes
   - Add annotations

**Conflict Resolution:**
- Last-write-wins for layout updates
- Entity additions are atomic
- UI provides optimistic updates with rollback

---

## Examples

### Example 1: Create List and Export

**Goal:** Create a list of shell companies for export.

```python
import requests

BASE_URL = "http://localhost:8080/api/2"
API_KEY = "your-api-key"
headers = {"Authorization": f"ApiKey {API_KEY}"}

# 1. Create list
response = requests.post(
    f"{BASE_URL}/entitysets",
    json={
        "label": "Shell Companies List",
        "type": "list",
        "collection_id": "collection-123"
    },
    headers=headers
)
entityset = response.json()
entityset_id = entityset['id']

# 2. Search for shell companies
response = requests.get(
    f"{BASE_URL}/entities",
    params={
        "q": "shell company",
        "filter:collection_id": "collection-123",
        "filter:schema": "Company"
    },
    headers=headers
)
entities = response.json()['results']

# 3. Add entities to list
entity_ids = [e['id'] for e in entities]
requests.put(
    f"{BASE_URL}/entitysets/{entityset_id}/entities",
    json={"entities": entity_ids},
    headers=headers
)

# 4. Get final list
response = requests.get(
    f"{BASE_URL}/entitysets/{entityset_id}/entities",
    headers=headers
)
final_list = response.json()

print(f"Created list with {len(final_list['results'])} companies")
```

### Example 2: Programmatic Diagram Creation

**Goal:** Generate ownership diagram from entity relationships.

```python
import requests
import networkx as nx

BASE_URL = "http://localhost:8080/api/2"
API_KEY = "your-api-key"
headers = {"Authorization": f"ApiKey {API_KEY}"}

# 1. Get entities with ownership relationships
response = requests.get(
    f"{BASE_URL}/entities",
    params={
        "filter:collection_id": "collection-123",
        "filter:schemata": "Ownership"
    },
    headers=headers
)
ownerships = response.json()['results']

# 2. Build graph
G = nx.DiGraph()
for ownership in ownerships:
    owner = ownership['properties'].get('owner', [None])[0]
    asset = ownership['properties'].get('asset', [None])[0]
    if owner and asset:
        G.add_edge(owner, asset)

# 3. Calculate layout using networkx
pos = nx.spring_layout(G, k=2, iterations=50)

# 4. Create diagram with layout
layout = {}
for node_id, (x, y) in pos.items():
    layout[node_id] = {
        "x": int(x * 500 + 250),  # Scale and center
        "y": int(y * 500 + 250)
    }

response = requests.post(
    f"{BASE_URL}/entitysets",
    json={
        "label": "Ownership Network",
        "type": "diagram",
        "collection_id": "collection-123",
        "entities": list(G.nodes()),
        "layout": layout
    },
    headers=headers
)

diagram = response.json()
print(f"Created diagram with {len(G.nodes())} nodes and {len(G.edges())} edges")
```

### Example 3: Automated Entity Resolution

**Goal:** Programmatically review xref matches and merge duplicates.

```python
import requests

BASE_URL = "http://localhost:8080/api/2"
API_KEY = "your-api-key"
headers = {"Authorization": f"ApiKey {API_KEY}"}

# 1. Create profile EntitySet
response = requests.post(
    f"{BASE_URL}/entitysets",
    json={
        "label": "Person Deduplication",
        "type": "profile",
        "collection_id": "collection-123"
    },
    headers=headers
)
profile_id = response.json()['id']

# 2. Get xref matches
response = requests.get(
    f"{BASE_URL}/xref",
    params={
        "collection_id": "collection-123",
        "filter:schema": "Person"
    },
    headers=headers
)
matches = response.json()['results']

# 3. Auto-approve high-confidence matches
for match in matches:
    if match['score'] > 0.9:  # Very high confidence
        # Record positive judgement
        requests.post(
            f"{BASE_URL}/entitysets/{profile_id}/judgements",
            json={
                "entity_id": match['entity_id'],
                "match_id": match['match_id'],
                "judgement": "POSITIVE"
            },
            headers=headers
        )

        # Merge entities
        requests.post(
            f"{BASE_URL}/entities/merge",
            json={
                "source_id": match['entity_id'],
                "target_id": match['match_id']
            },
            headers=headers
        )
        print(f"Merged {match['entity_id']} -> {match['match_id']}")

    elif match['score'] < 0.6:  # Low confidence
        # Record negative judgement
        requests.post(
            f"{BASE_URL}/entitysets/{profile_id}/judgements",
            json={
                "entity_id": match['entity_id'],
                "match_id": match['match_id'],
                "judgement": "NEGATIVE"
            },
            headers=headers
        )

    else:  # Medium confidence - mark for manual review
        requests.post(
            f"{BASE_URL}/entitysets/{profile_id}/judgements",
            json={
                "entity_id": match['entity_id'],
                "match_id": match['match_id'],
                "judgement": "UNSURE"
            },
            headers=headers
        )

# 4. Get remaining manual review items
response = requests.get(
    f"{BASE_URL}/entitysets/{profile_id}/entities",
    params={"filter:judgement": "UNSURE"},
    headers=headers
)
manual_review = response.json()

print(f"{len(manual_review['results'])} matches need manual review")
```

### Example 4: Timeline from Document Dates

**Goal:** Create timeline of documents by publication date.

```python
import requests
from datetime import datetime

BASE_URL = "http://localhost:8080/api/2"
API_KEY = "your-api-key"
headers = {"Authorization": f"ApiKey {API_KEY}"}

# 1. Search for documents with dates
response = requests.get(
    f"{BASE_URL}/entities",
    params={
        "filter:collection_id": "collection-123",
        "filter:schema": "Document",
        "filter:properties.date": "[2020-01-01 TO 2023-12-31]"
    },
    headers=headers
)
documents = response.json()['results']

# 2. Sort by date
documents_sorted = sorted(
    documents,
    key=lambda d: d['properties'].get('date', [''])[0]
)

# 3. Create timeline
response = requests.post(
    f"{BASE_URL}/entitysets",
    json={
        "label": "Document Publication Timeline",
        "type": "timeline",
        "collection_id": "collection-123",
        "entities": [d['id'] for d in documents_sorted]
    },
    headers=headers
)

timeline = response.json()
print(f"Created timeline with {len(documents_sorted)} documents")
print(f"Date range: {documents_sorted[0]['properties']['date'][0]} to {documents_sorted[-1]['properties']['date'][0]}")
```

---

## Best Practices

### 1. Choosing the Right EntitySet Type

**Use Lists when:**
- Simple organization needed
- No visualization required
- Preparing data for export
- Building entity checklists

**Use Diagrams when:**
- Visualizing relationships is important
- Network analysis needed
- Presenting findings to stakeholders
- Understanding complex structures

**Use Timelines when:**
- Temporal analysis is key
- Events need chronological context
- Tracking activity over time
- Identifying patterns in time

**Use Profiles when:**
- Entity deduplication needed
- Cross-reference results need review
- Quality control on entity data
- Merging duplicate records

### 2. Diagram Layout Management

**Automatic Layouts:**
- Start with force-directed layout for initial exploration
- Use Dagre for hierarchical structures (org charts, ownership trees)
- Let algorithm run to stabilization before manual adjustments

**Manual Adjustments:**
- Save layout frequently (auto-save in UI)
- Group related entities visually
- Use consistent spacing for clarity
- Position important nodes centrally

**Performance:**
- Limit diagrams to <200 nodes for optimal performance
- Split large networks into multiple focused diagrams
- Use entity search to add specific nodes
- Remove peripheral nodes if diagram becomes cluttered

### 3. Entity Resolution Workflow

**Efficient Profile Management:**

1. **Pre-filter:** Only add high-confidence matches (>0.5) to profiles
2. **Batch review:** Group similar entity types together
3. **Start high confidence:** Review highest scores first for quick wins
4. **Document decisions:** Add notes to explain negative judgements
5. **Merge incrementally:** Don't merge all at once; verify results

**Quality Control:**
```python
# Review judgements before merging
def review_profile_quality(profile_id):
    stats = {
        'total': 0,
        'positive': 0,
        'negative': 0,
        'unsure': 0
    }

    judgements = get_profile_judgements(profile_id)
    for j in judgements:
        stats['total'] += 1
        stats[j['judgement'].lower()] += 1

    # Quality metrics
    decision_rate = (stats['positive'] + stats['negative']) / stats['total']
    merge_rate = stats['positive'] / stats['total']

    print(f"Decision rate: {decision_rate:.1%}")
    print(f"Merge rate: {merge_rate:.1%}")

    return stats
```

### 4. Performance Optimization

**EntitySet Size:**
- Keep lists under 10,000 entities for fast loading
- Diagrams: optimal at <200 nodes, max ~500
- Timelines: can handle 1,000+ events with date filtering
- Profiles: batch into multiple profiles if >500 match pairs

**Pagination:**
```python
# Always paginate large EntitySets
def get_all_entities(entityset_id, batch_size=100):
    all_entities = []
    offset = 0

    while True:
        response = requests.get(
            f"{BASE_URL}/entitysets/{entityset_id}/entities",
            params={"limit": batch_size, "offset": offset},
            headers=headers
        )
        entities = response.json()['results']

        if not entities:
            break

        all_entities.extend(entities)
        offset += batch_size

    return all_entities
```

**Caching:**
- Frontend caches EntitySet metadata
- Layout data cached in Redux store
- Entities cached separately
- Invalidate cache on updates

### 5. Collaboration Guidelines

**Naming Conventions:**
- Use descriptive labels: "Q1 2023 Panama Papers Network"
- Include investigator initials: "JD-Corporate-Network"
- Add date stamps for versions: "Shell-Companies-v2-2023-12"

**Team Workflows:**
- Assign EntitySet ownership to lead investigator
- Document decisions in collection notes
- Regular sync meetings to review diagrams
- Use write permissions carefully (avoid conflicts)

**Version Control:**
- Create dated snapshots of important diagrams
- Export layouts periodically as backup
- Document major changes in collection updates

### 6. Security Considerations

**Sensitive Investigations:**
- Use restricted collections for confidential work
- Limit WRITE permissions to trusted team only
- Regularly audit collection access logs
- Export sensitive diagrams to secure storage

**Data Privacy:**
- Don't include PII in EntitySet labels
- Use collection-level encryption for sensitive data
- Review sharing permissions before adding entities
- Remove access promptly when team members leave

### 7. Documentation

**EntitySet Metadata:**
- Use descriptive labels
- Document purpose in collection notes
- Tag related EntitySets
- Track significant changes

**Example Documentation:**
```markdown
## Investigation: Offshore Network Analysis

**Lead:** Jane Doe
**Created:** 2025-01-15
**Status:** Active

### EntitySets

1. **Corporate Structure Diagram** (entityset-789)
   - Purpose: Map ownership relationships
   - Entities: 45 companies, 12 persons
   - Key findings: 3 shell company clusters identified

2. **Transaction Timeline** (entityset-321)
   - Purpose: Chronological analysis of transfers
   - Date range: 2020-2023
   - Key findings: Activity spike in Q4 2022

3. **Duplicate Person Review** (entityset-654)
   - Purpose: Deduplication across datasets
   - Status: 78 matches, 45 reviewed
   - Merge rate: 65%
```

---

## Troubleshooting

### Common Issues

#### 1. Diagram Not Loading

**Symptoms:** Diagram screen shows loading spinner indefinitely.

**Causes:**
- Too many entities (>500)
- Corrupted layout data
- Missing entity IDs

**Solutions:**
```bash
# Check entity count
GET /api/2/entitysets/{id}
# If count > 500, split into multiple diagrams

# Reset layout
PUT /api/2/entitysets/{id}
{
  "layout": {}
}
```

#### 2. Layout Changes Not Persisting

**Symptoms:** Node positions reset after page refresh.

**Causes:**
- No WRITE permission
- API request failing
- Redux state not syncing

**Solutions:**
- Verify write permission on collection
- Check browser console for API errors
- Clear Redux store and reload
- Manually save layout:
```bash
PUT /api/2/entitysets/{id}
{
  "layout": {...}  # Get from browser console
}
```

#### 3. Entities Not Appearing in Timeline

**Symptoms:** Timeline is empty despite added entities.

**Causes:**
- Entities lack date properties
- Date format incorrect
- Date out of visible range

**Solutions:**
```bash
# Check entity dates
GET /api/2/entities/{id}
# Look for properties.date, properties.authoredAt, etc.

# Ensure date format: YYYY-MM-DD
# UI auto-detects common date properties
```

#### 4. Profile Matches Not Loading

**Symptoms:** Profile EntitySet shows 0 entities despite xref matches.

**Causes:**
- Xref not completed
- Score threshold too high
- Different collection filters

**Solutions:**
```bash
# Check xref status
GET /api/2/xref?collection_id={id}

# Lower score threshold in UI
# Re-run xref with correct collections
POST /api/2/xref
{
  "collections": ["collection-123", "collection-456"]
}
```

#### 5. Permission Denied Errors

**Symptoms:** 403 Forbidden when accessing EntitySet.

**Causes:**
- No READ permission on collection
- EntitySet deleted (soft delete)
- Collection archived

**Solutions:**
```bash
# Check collection permissions
GET /api/2/collections/{id}/permissions

# Verify EntitySet exists
GET /api/2/entitysets/{id}

# Request access from collection owner
```

---

## Related Documentation

- [Collections](./COLLECTIONS.md) - Parent container and permissions
- [Entities](./ENTITIES.md) - Entity model and FollowTheMoney
- [Cross-Reference](./XREF.md) - Entity matching for profiles
- [Search](./SEARCH.md) - Finding entities to add to EntitySets
- [API Reference](./API.md) - Complete API documentation
- [Workflows](./WORKFLOWS.md) - Common investigation workflows

---

## File References

### Backend Files

| File | Lines | Description |
|------|-------|-------------|
| `aleph/model/entityset.py` | ~150 | EntitySet model definition |
| `aleph/model/entityset_item.py` | ~50 | Junction table model |
| `aleph/logic/entitysets.py` | ~200 | Business logic |
| `aleph/logic/profiles.py` | ~150 | Profile-specific logic |
| `aleph/logic/matching.py` | ~200 | Entity matching algorithms |
| `aleph/views/entitysets_api.py` | ~200 | REST API endpoints |

### Frontend Files

| File | Lines | Description |
|------|-------|-------------|
| `ui/src/screens/InvestigationsScreen/` | ~150 | List view |
| `ui/src/screens/EntitySetScreen/` | ~200 | Detail view |
| `ui/src/screens/DiagramScreen/` | ~300 | Diagram editor |
| `ui/src/components/DiagramEditor/` | ~200 | Edit component |
| `ui/src/components/GraphRenderer/` | ~250 | D3 visualization |
| `ui/src/components/Timeline/` | ~150 | Timeline component |
| `ui/src/reducers/entitySets.js` | ~100 | Redux state management |

---

**Document Version:** 1.0
**Last Updated:** 2025-12-09
**Contributors:** Claude Code Documentation Assistant
