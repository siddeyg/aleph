# Elasticsearch Integration

Complete documentation for Aleph's Elasticsearch search and indexing architecture.

## Table of Contents

1. [Overview](#overview)
2. [Index Architecture](#index-architecture)
3. [Index Mappings](#index-mappings)
4. [Query System](#query-system)
5. [Search Implementation](#search-implementation)
6. [Facets and Aggregations](#facets-and-aggregations)
7. [Indexing Pipeline](#indexing-pipeline)
8. [Performance Optimization](#performance-optimization)
9. [Configuration](#configuration)
10. [Monitoring and Debugging](#monitoring-and-debugging)
11. [Troubleshooting](#troubleshooting)
12. [Best Practices](#best-practices)

---

## Overview

Aleph uses Elasticsearch as its primary search and indexing engine, providing powerful full-text search, faceted navigation, and analytics capabilities across investigative data.

### Key Features

- **Per-Schema Indices**: Separate indices for each entity type (Person, Company, Document, etc.)
- **Versioned Indices**: Zero-downtime reindexing with index versioning
- **Authorization-Aware**: Every query automatically filtered by user permissions
- **Multilingual Support**: Custom analyzers for Latin-script languages
- **Batch Processing**: Intelligent bulk indexing for optimal performance
- **Faceted Search**: Dynamic aggregations for result refinement

### Architecture Components

| Component | Purpose | Location |
|-----------|---------|----------|
| **Index Management** | Index creation, configuration, versioning | `aleph/index/` |
| **Query Building** | Search query construction | `aleph/search/query.py` |
| **Facet System** | Aggregations and result facets | `aleph/search/facet.py` |
| **Entity Indexing** | Entity index operations | `aleph/index/entities.py` |
| **Collection Indexing** | Collection index operations | `aleph/index/collections.py` |
| **Xref Indexing** | Cross-reference match indexing | `aleph/index/xref.py` |

### Elasticsearch Version

- **Supported Version**: Elasticsearch 7.17.0
- **Client Library**: `elasticsearch-py`
- **Connection**: Configured via `ALEPH_ELASTICSEARCH_URI`

---

## Index Architecture

### Index Naming Convention

Aleph uses a structured naming convention for indices:

```
Pattern: {prefix}-{type}-{schema}-{version}

Examples:
aleph-entity-person-v20231001
aleph-entity-company-v20231001
aleph-entity-document-v20231001
aleph-collection-v20231001
aleph-xref-v20231001
```

**Code Reference**: aleph/index/util.py:75-76

```python
def index_name(name, version):
    return "-".join((SETTINGS.INDEX_PREFIX, name, version))
```

### Per-Schema Index Strategy

Each entity schema (Person, Company, Document, etc.) gets its own index family. This provides:

1. **Optimized Mappings**: Field types tailored to each schema
2. **Independent Scaling**: Heavy schemas (Page, Table) get more shards
3. **Faster Queries**: Narrower search space
4. **Schema Evolution**: Update mappings per type without affecting others

**Code Reference**: aleph/index/indexes.py:24-29

```python
def schema_index(schema, version):
    """Convert a schema object to an index name."""
    if schema.abstract:
        raise InvalidData("Cannot index abstract schema: %s" % schema)
    name = "entity-%s" % schema.name.lower()
    return index_name(name, version=version)
```

### Read vs Write Indices

Aleph separates read and write operations to support zero-downtime reindexing:

**Read Indices**: Multiple versions can be queried simultaneously

```python
# aleph/index/indexes.py:46-55
def entities_read_index(schema=None, expand=True):
    """Combined index to run all queries against."""
    indexes = []
    for schema in schema_scope(schema, expand=expand):
        for version in SETTINGS.INDEX_READ:  # e.g., ['v20231001', 'v20230801']
            indexes.append(schema_index(schema, version))
    return ",".join(indexes)
```

**Write Index**: Single current version for new/updated entities

```python
# aleph/index/indexes.py:58-61
def entities_write_index(schema):
    """Index that is currently written by new queries."""
    schema = model.get(schema)
    return schema_index(schema, SETTINGS.INDEX_WRITE)  # e.g., 'v20231001'
```

### Zero-Downtime Reindexing

**Process**:

1. Create new index version with updated mappings
2. Start indexing entities to new version
3. Keep reading from both old and new versions
4. Switch write operations to new index
5. Remove old version from read list
6. Delete old index

**Configuration**:
```bash
# settings.py or environment
INDEX_READ = ['v20231001', 'v20230801']  # Read from multiple versions
INDEX_WRITE = 'v20231001'  # Write to current version
```

### Shard Strategy

Different schema types get different shard counts based on expected volume:

**Code Reference**: aleph/index/util.py:36-56

```python
SHARDS_LIGHT = 1    # Low-volume schemas
SHARDS_DEFAULT = 5  # Standard schemas
SHARDS_HEAVY = 10   # High-volume schemas

SHARD_WEIGHTS = {
    "Folder": SHARDS_LIGHT,
    "Package": SHARDS_LIGHT,
    "Workbook": SHARDS_LIGHT,
    "Video": SHARDS_LIGHT,
    "Audio": SHARDS_LIGHT,
    "Page": SHARDS_HEAVY,         # Many pages from documents
    "Email": SHARDS_HEAVY,        # Email collections can be large
    "PlainText": SHARDS_HEAVY,    # Extracted text documents
    "Pages": SHARDS_HEAVY,
    "Table": SHARDS_HEAVY,        # Spreadsheet rows
}

def get_shard_weight(schema):
    if SETTINGS.TESTING:
        return 1
    return SHARD_WEIGHTS.get(schema.name, SHARDS_DEFAULT)
```

**Benefits**:
- High-volume schemas distributed across more shards
- Better parallelization for heavy workloads
- Lighter schemas don't waste resources

---

## Index Mappings

### Entity Index Mapping

Complete mapping structure for entity indices:

**Code Reference**: aleph/index/indexes.py:83-127

```python
mapping = {
    "date_detection": False,  # Disable automatic date detection
    "dynamic": False,         # Strict schema enforcement
    "_source": {
        "excludes": ["text", "fingerprints"]  # Exclude from _source for efficiency
    },
    "properties": {
        # Core entity fields
        "caption": KEYWORD,              # Display name
        "schema": KEYWORD,               # Entity type (Person, Company, etc.)
        "schemata": KEYWORD,             # All parent schemas

        # Property groups (FTM type groups)
        registry.entity.group: KEYWORD,      # Entity references
        registry.language.group: KEYWORD,    # Language codes
        registry.country.group: KEYWORD,     # Country codes
        registry.checksum.group: KEYWORD,    # File checksums
        registry.ip.group: KEYWORD,          # IP addresses
        registry.url.group: KEYWORD,         # URLs
        registry.iban.group: KEYWORD,        # Bank account numbers
        registry.email.group: KEYWORD,       # Email addresses
        registry.phone.group: KEYWORD,       # Phone numbers
        registry.mimetype.group: KEYWORD,    # MIME types
        registry.identifier.group: KEYWORD,  # Generic identifiers
        registry.date.group: PARTIAL_DATE,   # Dates with flexible format
        registry.address.group: KEYWORD,     # Physical addresses
        registry.name.group: KEYWORD,        # Names

        # Fingerprints for entity matching
        "fingerprints": {
            "type": "keyword",
            "normalizer": "latin_index",  # Normalize for matching
            "copy_to": "text",            # Copy to full-text field
            "fields": {
                "text": LATIN_TEXT        # Analyzed version
            }
        },

        # Full-text search field
        "text": {
            "type": "text",
            "analyzer": "latin_index",
            "search_analyzer": "latin_query",
            "search_quote_analyzer": "latin_index",
            "term_vector": "with_positions_offsets"  # For highlighting
        },

        # Schema-specific properties (dynamic per entity type)
        "properties": {
            "type": "object",
            "properties": schema_mapping  # Generated per schema
        },

        # Numeric properties for sorting/range queries
        "numeric": {
            "type": "object",
            "properties": numeric_mapping  # Date and number fields
        },

        # Authorization and metadata
        "role_id": KEYWORD,          # Entity owner
        "profile_id": KEYWORD,       # Profile reference
        "collection_id": KEYWORD,    # Parent collection
        "origin": KEYWORD,           # Source identifier
        "created_at": {"type": "date"},
        "updated_at": {"type": "date"}
    }
}
```

### Field Type Definitions

**Code Reference**: aleph/index/util.py:24-34

```python
# Date format supporting partial dates
DATE_FORMAT = "yyyy-MM-dd'T'HH:mm:ss||yyyy-MM-dd||yyyy-MM||yyyy"
PARTIAL_DATE = {"type": "date", "format": DATE_FORMAT}

# Text field with Latin script analysis
LATIN_TEXT = {
    "type": "text",
    "analyzer": "latin_index",
    "search_analyzer": "latin_query"
}

# Keyword for exact matching, filtering, aggregations
KEYWORD = {"type": "keyword"}
KEYWORD_COPY = {"type": "keyword", "copy_to": "text"}

# Numeric fields
NUMERIC = {"type": "double"}
```

### Dynamic Property Mapping

Properties are mapped dynamically based on FollowTheMoney schema:

**Code Reference**: aleph/index/indexes.py:74-81

```python
def configure_schema(schema, version):
    schema_mapping = {}
    numeric_mapping = {registry.date.group: NUMERIC}

    for prop in schema.properties.values():
        # Get type mapping (text, date, keyword)
        config = deepcopy(TYPE_MAPPINGS.get(prop.type, KEYWORD))
        config["copy_to"] = ["text"]  # Copy all properties to text field
        schema_mapping[prop.name] = config

        if prop.type in NUMERIC_TYPES:
            numeric_mapping[prop.name] = deepcopy(NUMERIC)
```

**Example**: Person schema properties
```json
{
    "properties": {
        "properties": {
            "name": {"type": "text", "copy_to": ["text"]},
            "birthDate": {"type": "date", "format": "yyyy-MM-dd'T'HH:mm:ss||yyyy-MM-dd||yyyy-MM||yyyy", "copy_to": ["text"]},
            "nationality": {"type": "keyword", "copy_to": ["text"]},
            "passport": {"type": "keyword", "copy_to": ["text"]},
            "idNumber": {"type": "keyword", "copy_to": ["text"]}
        },
        "numeric": {
            "dates": {"type": "double"},
            "birthDate": {"type": "double"}
        }
    }
}
```

### Custom Analyzers

#### Latin Index Analyzer

Normalizes text for multilingual search across Latin-script languages:

**Code Reference**: aleph/index/util.py:340-365

```python
"analysis": {
    "analyzer": {
        "latin_index": {
            "tokenizer": "standard",
            "filter": ["latinize"]
        },
        "latin_query": {
            "tokenizer": "standard",
            "filter": ["latinize", "synonames"]
        }
    },
    "normalizer": {
        "latin_index": {
            "type": "custom",
            "filter": ["latinize"]
        }
    },
    "filter": {
        "latinize": {
            "type": "icu_transform",
            # Transliterate to Latin, decompose, lowercase, remove accents
            "id": "Any-Latin; NFKD; Lower(); [:Nonspacing Mark:] Remove; NFKC"
        },
        "synonames": {
            "type": "synonym",
            "lenient": "true",
            "synonyms_path": "synonames.txt"  # Common name variations
        }
    }
}
```

**Transformations**:
- **Any-Latin**: Transliterate non-Latin scripts to Latin
- **NFKD**: Decompose characters (é → e + ´)
- **Lower()**: Convert to lowercase
- **[:Nonspacing Mark:] Remove**: Remove accent marks
- **NFKC**: Recompose characters

**Examples**:
```
"Müller" → "muller"
"Café" → "cafe"
"Москва" (Moscow in Cyrillic) → "moskva"
"中国" (China in Chinese) → "zhong guo"
```

#### Synonames Filter

Provides synonym expansion for common name variations:

```
# synonames.txt examples
Corp, Corporation, Corp., Inc., Incorporated, Inc
Ltd, Limited, Ltd.
Mr, Mister, Mr.
Dr, Doctor, Dr.
```

---

## Query System

### Query Class Hierarchy

**Base Query Class**: aleph/search/query.py:30-311

```python
class Query(object):
    """Base query builder for Elasticsearch."""

    TEXT_FIELDS = ["text"]                    # Fields for full-text search
    PREFIX_FIELD = "name"                     # Field for prefix matching
    SKIP_FILTERS = []                         # Filters to skip
    AUTHZ_FIELD = "collection_id"             # Field for authorization
    HIGHLIGHT_FIELD = "text"                  # Field to highlight
    SORT_FIELDS = {                           # Named sort options
        "label": "label.kw",
        "score": "_score",
    }
    SORT_DEFAULT = ["_score"]                 # Default sort order
    SOURCE = {}                               # Fields to return

    def __init__(self, parser):
        self.parser = parser

    def get_query(self):
        """Build complete query."""
        return {
            "bool": {
                "should": self.get_text_query(),
                "must": [],
                "must_not": self.get_negative_filters(),
                "filter": self.get_filters(),
                "minimum_should_match": 1
            }
        }

    def get_body(self):
        """Build complete request body."""
        return {
            "query": self.get_query(),
            "post_filter": self.get_post_filters(),
            "from": self.parser.offset,
            "size": self.parser.limit,
            "aggregations": self.get_aggregations(),
            "sort": self.get_sort(),
            "highlight": self.get_highlight(),
            "_source": self.get_source()
        }

    def search(self):
        """Execute the query."""
        result = es.search(index=self.get_index(), body=self.get_body())
        return result
```

### Text Query Construction

**Code Reference**: aleph/search/query.py:46-65

```python
def get_text_query(self):
    """Build text search part of query."""
    query = []

    # Full-text search
    if self.parser.text:
        qs = {
            "query_string": {
                "query": self.parser.text,
                "lenient": True,                   # Don't fail on parse errors
                "fields": self.TEXT_FIELDS,        # ["text"]
                "default_operator": "AND",         # All terms must match
                "minimum_should_match": "66%"      # Or 66% for long queries
            }
        }
        query.append(qs)

    # Prefix matching (autocomplete)
    if self.parser.prefix:
        query.append({
            "match_phrase_prefix": {
                self.PREFIX_FIELD: self.parser.prefix
            }
        })

    # Default to match all if no text query
    if not len(query):
        query.append({"match_all": {}})

    return query
```

### Filter Application

**Authorization Filter**: Always applied to restrict results to readable collections

**Code Reference**: aleph/search/query.py:88-100

```python
def get_filters(self):
    """Apply query filters from the user interface."""
    skip = [*self.SKIP_FILTERS, *self.parser.facet_names]
    filters = self.get_filters_list(skip)

    if self.AUTHZ_FIELD is not None:
        # This enforces the authorization (access control) rules on
        # a particular query by comparing the collections a user is
        # authorized for with the one on the document.
        if self.parser.authz and not self.parser.authz.is_admin:
            authz = authz_query(self.parser.authz, field=self.AUTHZ_FIELD)
            filters.append(authz)

    return filters
```

**Authorization Query Builder**: aleph/index/util.py:101-109

```python
def authz_query(authz, field="collection_id"):
    """Generate a search query filter from an authz object."""
    # Hot-wire authorization entirely for admins.
    if authz.is_admin:
        return {"match_all": {}}

    collections = authz.collections(authz.READ)
    if not len(collections):
        return {"match_none": {}}

    return {"terms": {field: collections}}
```

### Range Filters

Support for range queries on dates and numbers:

**Code Reference**: aleph/search/query.py:67-86

```python
def get_filters_list(self, skip):
    filters = []
    range_filters = dict()

    for field, values in self.parser.filters.items():
        if field in skip:
            continue

        # Collect all range query filters for a field in a single query
        if field.startswith(("gt:", "gte:", "lt:", "lte:")):
            op, field = field.split(":", 1)
            if range_filters.get(field) is None:
                range_filters[field] = {op: list(values)[0]}
            else:
                range_filters[field][op] = list(values)[0]
            continue

        filters.append(field_filter_query(field, values))

    # Build range filters
    for field, ops in range_filters.items():
        filters.append(range_filter_query(field, ops))

    return filters
```

**Usage**:
```python
filters = {
    "gte:properties.birthDate": "1950-01-01",
    "lte:properties.birthDate": "2000-12-31"
}
# Generates: {"range": {"properties.birthDate": {"gte": "1950-01-01", "lte": "2000-12-31"}}}
```

### Negative Filters

**Code Reference**: aleph/search/query.py:110-118

```python
def get_negative_filters(self):
    """Apply negative filters."""
    filters = []

    # Exclude documents where field exists
    for field, _ in self.parser.empties.items():
        filters.append({"exists": {"field": field}})

    # Exclude specific values
    for field, values in self.parser.excludes.items():
        filters.append(field_filter_query(field, values))

    return filters
```

---

## Search Implementation

### Complete Query Example

Typical entity search query structure:

```json
{
    "query": {
        "bool": {
            "should": [
                {
                    "query_string": {
                        "query": "corruption scandal",
                        "lenient": true,
                        "fields": ["text"],
                        "default_operator": "AND",
                        "minimum_should_match": "66%"
                    }
                }
            ],
            "must": [],
            "must_not": [
                {
                    "term": {"properties.status": "inactive"}
                }
            ],
            "filter": [
                {
                    "term": {"schema": "Person"}
                },
                {
                    "terms": {
                        "collection_id": [1, 5, 12]
                    }
                },
                {
                    "terms": {
                        "countries": ["US", "UK"]
                    }
                }
            ],
            "minimum_should_match": 1
        }
    },
    "post_filter": {
        "bool": {
            "filter": [
                {
                    "term": {"schemata": "LegalEntity"}
                }
            ]
        }
    },
    "from": 0,
    "size": 30,
    "sort": [
        {"_score": {"order": "desc"}},
        {"created_at": {"order": "desc", "missing": "_last"}}
    ],
    "highlight": {
        "encoder": "html",
        "fields": {
            "text": {
                "highlight_query": {
                    "query_string": {
                        "query": "corruption scandal",
                        "lenient": true,
                        "default_operator": "AND",
                        "minimum_should_match": "66%"
                    }
                },
                "require_field_match": false,
                "number_of_fragments": 5,
                "fragment_size": 150,
                "max_analyzed_offset": 100000
            }
        }
    },
    "aggregations": {
        "countries.values": {
            "terms": {
                "field": "countries",
                "size": 100,
                "execution_hint": "map"
            }
        },
        "schema.filtered": {
            "filter": {
                "bool": {
                    "filter": [
                        {
                            "terms": {"collection_id": [1, 5, 12]}
                        }
                    ]
                }
            },
            "aggregations": {
                "schema.values": {
                    "terms": {
                        "field": "schema",
                        "size": 100
                    }
                }
            }
        }
    },
    "_source": {
        "includes": ["schema", "properties", "collection_id", "caption"],
        "excludes": []
    }
}
```

### Sorting

**Code Reference**: aleph/search/query.py:205-224

```python
def get_sort(self):
    """Pick one of a set of named result orderings."""
    if not len(self.parser.sorts):
        return self.SORT_DEFAULT

    sort_fields = ["_score"]
    for field, direction in self.parser.sorts:
        field = self.SORT_FIELDS.get(field, field)
        type_ = get_field_type(field)

        config = {"order": direction, "missing": "_last"}
        es_type = get_index_field_type(type_)
        if es_type:
            config["unmapped_type"] = es_type

        # Sort dates by minimum value in numeric field
        if field == registry.date.group:
            field = "numeric.dates"
            config["mode"] = "min"

        # Use numeric field for sorting numeric properties
        if type_ in NUMERIC_TYPES:
            field = field.replace("properties.", "numeric.")

        sort_fields.append({field: config})

    return list(reversed(sort_fields))
```

**Sort Options**:
- `_score`: Relevance ranking (default)
- `created_at`: Creation timestamp
- `updated_at`: Modification timestamp
- `properties.<field>`: Any entity property

### Highlighting

**Code Reference**: aleph/search/query.py:226-247

```python
def get_highlight(self):
    if not self.parser.highlight:
        return {}

    return {
        "encoder": "html",
        "fields": {
            self.HIGHLIGHT_FIELD: {
                "highlight_query": {
                    "query_string": {
                        "query": self.parser.highlight_text,
                        "lenient": True,
                        "default_operator": "AND",
                        "minimum_should_match": "66%"
                    }
                },
                "require_field_match": False,
                "number_of_fragments": self.parser.highlight_count,
                "fragment_size": self.parser.highlight_length,
                "max_analyzed_offset": self.parser.max_highlight_analyzed_offset
            }
        }
    }
```

**Highlighting Configuration**:
- **encoder**: `html` (wraps matches in HTML tags)
- **number_of_fragments**: How many snippets to return (default: 5)
- **fragment_size**: Characters per snippet (default: 150)
- **max_analyzed_offset**: Maximum text length to analyze (default: 100000)

### Source Filtering

Control which fields are returned in results:

**Code Reference**: aleph/index/entities.py:19-29

```python
PROXY_INCLUDES = [
    "schema",
    "properties",
    "collection_id",
    "profile_id",
    "role_id",
    "mutable",
    "created_at",
    "updated_at"
]
ENTITY_SOURCE = {"includes": PROXY_INCLUDES}
```

**Benefits**:
- Reduces network transfer
- Improves query performance
- text and fingerprints fields excluded from _source for efficiency

---

## Facets and Aggregations

### Facet Architecture

Facets provide dynamic result refinement through aggregations:

**Code Reference**: aleph/search/query.py:138-203

```python
def get_aggregations(self):
    """Aggregate the query in order to generate faceted results."""
    aggregations = {}

    for facet_name in self.parser.facet_names:
        facet_aggregations = {}

        # Standard facet values (terms aggregation)
        if self.parser.get_facet_values(facet_name):
            agg_name = "%s.values" % facet_name
            terms = {
                "field": facet_name,
                "size": self.parser.get_facet_size(facet_name),
                "execution_hint": "map"
            }
            facet_aggregations[agg_name] = {"terms": terms}

        # Cardinality (count of distinct values)
        if self.parser.get_facet_total(facet_name):
            agg_name = "%s.cardinality" % facet_name
            facet_aggregations[agg_name] = {
                "cardinality": {"field": facet_name}
            }

        # Date histogram (timeline facets)
        interval = self.parser.get_facet_interval(facet_name)
        if interval is not None:
            agg_name = "%s.intervals" % facet_name
            facet_aggregations[agg_name] = {
                "date_histogram": {
                    "field": facet_name,
                    "calendar_interval": interval,  # day, week, month, year
                    "format": DATE_FORMAT,
                    "min_doc_count": 0  # Include empty buckets
                }
            }

            # Extended bounds for empty buckets in range
            filters = self.parser.filters
            min_val = filters.get("gte:%s" % facet_name) or filters.get("gt:%s" % facet_name)
            max_val = filters.get("lte:%s" % facet_name) or filters.get("lt:%s" % facet_name)
            if min_val or max_val:
                extended_bounds = {}
                if min_val:
                    extended_bounds["min"] = ensure_list(min_val)[0]
                if max_val:
                    extended_bounds["max"] = ensure_list(max_val)[0]
                facet_aggregations[agg_name]["date_histogram"]["extended_bounds"] = extended_bounds

        # Post-filter aggregations (exclude selected facet from filter)
        other_filters = self.get_post_filters(exclude=facet_name)
        if len(other_filters["bool"]["filter"]):
            agg_name = "%s.filtered" % facet_name
            aggregations[agg_name] = {
                "filter": other_filters,
                "aggregations": facet_aggregations
            }
        else:
            aggregations.update(facet_aggregations)

    return aggregations
```

### Facet Types

#### 1. Terms Facet (Standard)

Count documents by field value:

```json
{
    "schema.values": {
        "terms": {
            "field": "schema",
            "size": 100,
            "execution_hint": "map"
        }
    }
}
```

**Response**:
```json
{
    "buckets": [
        {"key": "Person", "doc_count": 1547},
        {"key": "Company", "doc_count": 892},
        {"key": "Document", "doc_count": 3421}
    ]
}
```

#### 2. Cardinality Aggregation

Count distinct values:

```json
{
    "countries.cardinality": {
        "cardinality": {
            "field": "countries"
        }
    }
}
```

**Response**:
```json
{
    "value": 143
}
```

#### 3. Date Histogram

Timeline aggregation by date intervals:

```json
{
    "dates.intervals": {
        "date_histogram": {
            "field": "properties.date",
            "calendar_interval": "year",
            "format": "yyyy-MM-dd'T'HH:mm:ss||yyyy-MM-dd||yyyy-MM||yyyy",
            "min_doc_count": 0,
            "extended_bounds": {
                "min": "2010-01-01",
                "max": "2023-12-31"
            }
        }
    }
}
```

**Response**:
```json
{
    "buckets": [
        {"key_as_string": "2010", "key": 1262304000000, "doc_count": 45},
        {"key_as_string": "2011", "key": 1293840000000, "doc_count": 67},
        {"key_as_string": "2012", "key": 1325376000000, "doc_count": 89}
    ]
}
```

### Facet Classes

Different facet types with custom label resolution:

**Code Reference**: aleph/search/facet.py:12-133

```python
class Facet(object):
    """Base facet class."""

    def to_dict(self):
        active = list(self.parser.filters.get(self.name, []))
        data = {"filters": active}

        if self.parser.get_facet_total(self.name):
            data["total"] = self.cardinality.get("value")

        if self.parser.get_facet_values(self.name):
            results = []
            for bucket in self.data.get("buckets", []):
                key = self.get_key(bucket)
                results.append({
                    "id": key,
                    "label": key,
                    "count": bucket.pop("doc_count", 0),
                    "active": key in active
                })

            self.expand([r.get("id") for r in results])
            for result in results:
                self.update(result, result.get("id"))

            data["values"] = results

        return data

class SchemaFacet(Facet):
    """Entity schema facet with plural labels."""
    def update(self, result, key):
        try:
            result["label"] = model.get(key).plural
        except AttributeError:
            result["label"] = key

class CountryFacet(Facet):
    """Country facet with country names."""
    def update(self, result, key):
        result["label"] = registry.country.names.get(key, key)

class CollectionFacet(Facet):
    """Collection facet with collection labels."""
    def expand(self, keys):
        for key in keys:
            if self.parser.authz.can(key, self.parser.authz.READ):
                resolver.queue(self.parser, Collection, key)
        resolver.resolve(self.parser)

    def update(self, result, key):
        collection = resolver.get(self.parser, Collection, key)
        if collection is not None:
            result["label"] = collection.get("label")
            result["category"] = collection.get("category")

class EntityFacet(Facet):
    """Entity facet with entity captions."""
    def expand(self, keys):
        for key in keys:
            resolver.queue(self.parser, Entity, key)
        resolver.resolve(self.parser)

    def update(self, result, key):
        entity = resolver.get(self.parser, Entity, key)
        if entity is not None:
            proxy = model.get_proxy(entity)
            result["label"] = proxy.caption
```

### Post-Filter Aggregations

Facets use post-filters to exclude their own selected values from the aggregation, allowing users to see alternative options:

```json
{
    "countries.filtered": {
        "filter": {
            "bool": {
                "filter": [
                    {"term": {"schema": "Person"}},
                    {"terms": {"collection_id": [1, 5, 12]}}
                ]
            }
        },
        "aggregations": {
            "countries.values": {
                "terms": {"field": "countries", "size": 100}
            }
        }
    }
}
```

**Effect**: If "US" is selected, the countries facet still shows counts including "US", allowing users to deselect it.

---

## Indexing Pipeline

### Bulk Indexing Strategy

Aleph uses streaming bulk operations for efficient indexing:

**Code Reference**: aleph/index/util.py:197-217

```python
def bulk_actions(actions, chunk_size=BULK_PAGE, sync=False):
    """Bulk indexing with timeouts, bells and whistles."""
    stream = streaming_bulk(
        es,
        actions,
        chunk_size=chunk_size,          # Default: 500 documents per request
        max_retries=10,
        yield_ok=False,
        raise_on_error=False,
        refresh=refresh_sync(sync),
        request_timeout=MAX_REQUEST_TIMEOUT,  # 84600 seconds (23.5 hours)
        timeout=MAX_TIMEOUT                   # 700 minutes
    )

    for _, details in stream:
        if details.get("delete", {}).get("status") == 404:
            continue
        log.warning("Bulk index error: %r", details)
```

**Configuration**:
- `BULK_PAGE = 500`: Documents per bulk request
- `MAX_REQUEST_TIMEOUT = 84600`: Request timeout in seconds
- `MAX_TIMEOUT = "700m"`: Elasticsearch timeout

### Entity Indexing

**Code Reference**: Referenced in aleph/index/entities.py

```python
def iter_entities(
    authz=None,
    collection_id=None,
    schemata=None,
    includes=PROXY_INCLUDES,
    excludes=None,
    filters=None,
    sort=None,
    es_scroll="5m",
    es_scroll_size=1000
):
    """Scan all entities matching the given criteria."""
    query = {
        "query": _entities_query(filters, authz, collection_id, schemata),
        "_source": _source_spec(includes, excludes)
    }
    preserve_order = False
    if sort is not None:
        query["sort"] = ensure_list(sort)
        preserve_order = True

    index = entities_read_index(schema=schemata)
    for res in scan(
        es,
        index=index,
        query=query,
        timeout=MAX_TIMEOUT,
        request_timeout=MAX_REQUEST_TIMEOUT,
        preserve_order=preserve_order,
        scroll=es_scroll,
        size=es_scroll_size
    ):
        entity = unpack_result(res)
        if entity is not None:
            yield entity
```

### Index Operations

#### Create/Update Entity

```python
def index_safe(index, id, body, sync=False, **kwargs):
    """Index a single document and retry until it has been stored."""
    for attempt in service_retries():
        try:
            refresh = refresh_sync(sync)
            es.index(index=index, id=id, body=body, refresh=refresh, **kwargs)
            body["id"] = str(id)
            body.pop("text", None)
            return body
        except TransportError as exc:
            if exc.status_code in ("400", "403"):
                raise
            log.warning("Index error [%s:%s]: %s", index, id, exc)
            backoff(failures=attempt)
```

#### Delete Entity

```python
def delete_safe(index, id, sync=False):
    es.delete(
        index=index,
        id=str(id),
        ignore=[404],
        refresh=refresh_sync(sync)
    )
```

#### Delete by Query

```python
def query_delete(index, query, sync=False, **kwargs):
    """Delete all documents matching the given query inside the index."""
    for attempt in service_retries():
        try:
            es.delete_by_query(
                index=index,
                body={"query": query},
                _source=False,
                slices="auto",              # Parallelize across shards
                conflicts="proceed",         # Continue on version conflicts
                wait_for_completion=sync,
                refresh=refresh_sync(sync),
                request_timeout=MAX_REQUEST_TIMEOUT,
                timeout=MAX_TIMEOUT,
                scroll_size=SETTINGS.INDEX_DELETE_BY_QUERY_BATCHSIZE
            )
            return
        except TransportError as exc:
            if exc.status_code in ("400", "403"):
                raise
            log.warning("Query delete failed: %s", exc)
            backoff(failures=attempt)
```

### Refresh Strategy

**Code Reference**: aleph/index/util.py:69-72

```python
def refresh_sync(sync):
    if SETTINGS.TESTING:
        return True
    return True if sync else False
```

**Refresh Modes**:
- `False`: Don't refresh (fastest, eventual visibility)
- `True`: Refresh immediately (slower, immediate visibility)
- `"wait_for"`: Wait for refresh interval (balanced)

**Best Practices**:
- Use `sync=False` for bulk operations
- Use `sync=True` for interactive operations where user expects immediate results
- Testing mode always uses `sync=True` for consistency

---

## Performance Optimization

### Query Performance

#### 1. Authorization Pre-filtering

Collections are filtered at query time, not post-processing:

```python
# Efficient: Filter in query
authz_filter = {"terms": {"collection_id": [1, 5, 12]}}  # From Redis cache

# Inefficient: Post-processing
# results = all_results.filter(lambda r: r.collection_id in [1, 5, 12])
```

#### 2. Source Filtering

Exclude large fields from results:

```python
"_source": {
    "includes": ["schema", "properties", "collection_id", "caption"],
    "excludes": ["text", "fingerprints"]  # Excluded from index _source
}
```

**Savings**: 50-90% reduction in network transfer for large documents

#### 3. Routing

Route queries to specific shards (when possible):

```python
es.search(
    index=index_name,
    routing=collection_id,  # Only query shards containing this collection
    body=query
)
```

#### 4. Request Cache

Enable request cache for filtered queries:

```json
{
    "size": 0,
    "query": {...},
    "aggregations": {...}
}
```

**Note**: `size: 0` makes queries cacheable

### Indexing Performance

#### 1. Bulk Operations

Always use bulk API for multiple documents:

```python
# Good: Bulk index 500 documents in one request
actions = [
    {"_index": index, "_id": id, "_source": doc}
    for id, doc in documents
]
bulk_actions(actions, chunk_size=500)

# Bad: Individual index calls
for id, doc in documents:
    index_safe(index, id, doc)  # 500 separate requests!
```

**Performance**: 10-100x faster with bulk operations

#### 2. Disable Refresh During Bulk Load

```python
# Disable refresh
es.indices.put_settings(
    index=index_name,
    body={"index": {"refresh_interval": "-1"}}
)

# Bulk index documents
bulk_actions(actions, chunk_size=1000, sync=False)

# Force refresh
es.indices.refresh(index=index_name)

# Re-enable refresh
es.indices.put_settings(
    index=index_name,
    body={"index": {"refresh_interval": "1s"}}
)
```

**Performance**: 2-5x faster bulk indexing

#### 3. Increase Bulk Size

For large imports, increase chunk size:

```python
bulk_actions(actions, chunk_size=1000, sync=False)  # Default: 500
```

**Considerations**:
- Larger chunks = fewer requests = faster
- Too large = memory pressure, timeouts
- Recommended: 500-2000 documents per chunk

#### 4. Connection Pooling

Configure ES client with connection pool:

```python
from elasticsearch import Elasticsearch

es = Elasticsearch(
    hosts=[ELASTICSEARCH_URI],
    max_retries=3,
    retry_on_timeout=True,
    maxsize=25,  # Connection pool size
    timeout=60
)
```

### Index Configuration

#### Shard Count

**Code Reference**: aleph/index/util.py:36-66

```python
SHARDS_LIGHT = 1    # < 1M documents
SHARDS_DEFAULT = 5  # 1-10M documents
SHARDS_HEAVY = 10   # > 10M documents
```

**Guidelines**:
- Each shard should contain 10-50 GB of data
- More shards = better parallelization but more overhead
- Fewer shards = less overhead but limited scalability

#### Replica Count

```python
"number_of_replicas": "0"  # Development: No replicas
"number_of_replicas": "1"  # Production: 1 replica for HA
"number_of_replicas": "2"  # Critical: 2 replicas
```

**Trade-offs**:
- More replicas = higher availability, faster reads
- More replicas = slower writes, more storage

#### Refresh Interval

```python
"refresh_interval": "1s"    # Default: Near real-time
"refresh_interval": "5s"    # Balanced: Less overhead
"refresh_interval": "30s"   # High-throughput: Bulk loads
"refresh_interval": "-1"    # Disabled: Maximum performance
```

### Mapping Optimizations

#### Exclude Fields from _source

**Code Reference**: aleph/index/indexes.py:86

```python
"_source": {
    "excludes": ["text", "fingerprints"]
}
```

**Benefits**:
- Smaller index size (text field can be very large)
- Faster indexing
- Lower network transfer
- Fields still searchable, just not retrievable

#### Disable Norms

For fields not used in scoring:

```json
{
    "properties": {
        "collection_id": {
            "type": "keyword",
            "norms": false
        }
    }
}
```

**Savings**: ~1 byte per document per field

---

## Configuration

### Environment Variables

```bash
# Elasticsearch connection
ALEPH_ELASTICSEARCH_URI=http://localhost:9200

# Index configuration
ALEPH_INDEX_PREFIX=aleph
ALEPH_INDEX_WRITE=v20231001
ALEPH_INDEX_READ=v20231001,v20230801
ALEPH_INDEX_REPLICAS=0

# Bulk operations
ALEPH_INDEX_DELETE_BY_QUERY_BATCHSIZE=1000

# Testing
ALEPH_TESTING=false
```

### Index Settings

**Code Reference**: aleph/index/util.py:330-365

```python
def index_settings(shards=5, replicas=SETTINGS.INDEX_REPLICAS):
    """Configure an index in ES with support for text transliteration."""
    if SETTINGS.TESTING:
        shards = 1
        replicas = 0

    return {
        "index": {
            "number_of_shards": str(shards),
            "number_of_replicas": str(replicas),
            "analysis": {
                "analyzer": {
                    "latin_index": {
                        "tokenizer": "standard",
                        "filter": ["latinize"]
                    },
                    "icu_latin": {
                        "tokenizer": "standard",
                        "filter": ["latinize"]
                    },
                    "latin_query": {
                        "tokenizer": "standard",
                        "filter": ["latinize", "synonames"]
                    }
                },
                "normalizer": {
                    "latin_index": {
                        "type": "custom",
                        "filter": ["latinize"]
                    }
                },
                "filter": {
                    "latinize": {
                        "type": "icu_transform",
                        "id": "Any-Latin; NFKD; Lower(); [:Nonspacing Mark:] Remove; NFKC"
                    },
                    "synonames": {
                        "type": "synonym",
                        "lenient": "true",
                        "synonyms_path": "synonames.txt"
                    }
                }
            }
        }
    }
```

### Elasticsearch Client Configuration

```python
from aleph.core import es

# Client is configured in aleph/core.py
es = Elasticsearch(
    hosts=[SETTINGS.ELASTICSEARCH_URI],
    max_retries=3,
    retry_on_timeout=True,
    timeout=60
)
```

### Performance Tuning

| Setting | Development | Production | Bulk Load |
|---------|-------------|------------|-----------|
| `number_of_shards` | 1 | 5-10 | 10 |
| `number_of_replicas` | 0 | 1 | 0 |
| `refresh_interval` | 1s | 1s | -1 |
| `bulk_chunk_size` | 500 | 500 | 1000-2000 |
| `max_result_window` | 10000 | 10000 | N/A |

---

## Monitoring and Debugging

### Cluster Health

```python
health = es.cluster.health()
print(f"Status: {health['status']}")  # green, yellow, red
print(f"Nodes: {health['number_of_nodes']}")
print(f"Active shards: {health['active_shards']}")
print(f"Unassigned shards: {health['unassigned_shards']}")
```

### Index Statistics

```python
stats = es.indices.stats(index='aleph-entity-*')
print(f"Total documents: {stats['_all']['primaries']['docs']['count']}")
print(f"Index size: {stats['_all']['primaries']['store']['size_in_bytes']}")
```

### Query Profiling

Enable profiling to analyze query performance:

```json
{
    "profile": true,
    "query": {
        "match": {
            "text": "corruption"
        }
    }
}
```

**Response includes**:
- Time spent in each query component
- Shard-level breakdown
- Rewrite time
- Lucene-level details

### Slow Query Logging

Configure slow query thresholds:

```json
{
    "index": {
        "search": {
            "slowlog": {
                "threshold": {
                    "query": {
                        "warn": "10s",
                        "info": "5s",
                        "debug": "2s"
                    },
                    "fetch": {
                        "warn": "1s",
                        "info": "500ms",
                        "debug": "200ms"
                    }
                }
            }
        }
    }
}
```

### Common Debugging Commands

```bash
# Check cluster health
curl -X GET "localhost:9200/_cluster/health?pretty"

# List all indices
curl -X GET "localhost:9200/_cat/indices?v"

# Get index mapping
curl -X GET "localhost:9200/aleph-entity-person-v20231001/_mapping?pretty"

# Get index settings
curl -X GET "localhost:9200/aleph-entity-person-v20231001/_settings?pretty"

# Count documents
curl -X GET "localhost:9200/aleph-entity-person-v20231001/_count?pretty"

# Sample documents
curl -X GET "localhost:9200/aleph-entity-person-v20231001/_search?pretty&size=1"

# Check shard allocation
curl -X GET "localhost:9200/_cat/shards?v"

# Force merge segments
curl -X POST "localhost:9200/aleph-entity-person-v20231001/_forcemerge?max_num_segments=1"
```

---

## Troubleshooting

### Issue 1: "Result window is too large"

**Error**:
```
Result window is too large, from + size must be less than or equal to: [10000]
```

**Cause**: Trying to paginate beyond 10,000 results

**Solutions**:

1. **Use scroll API** for large result sets:
```python
for entity in iter_entities(authz=authz, collection_id=collection_id):
    process(entity)
```

2. **Use search_after** for pagination:
```json
{
    "size": 100,
    "sort": [{"created_at": "desc"}, {"_id": "asc"}],
    "search_after": ["2023-10-01T12:00:00", "entity-123"]
}
```

3. **Increase max_result_window** (not recommended):
```json
{
    "index": {
        "max_result_window": 50000
    }
}
```

### Issue 2: Slow Queries

**Symptoms**: Queries taking > 5 seconds

**Debugging**:

1. **Enable profiling**:
```json
{"profile": true, "query": {...}}
```

2. **Check slow logs**:
```bash
tail -f /var/log/elasticsearch/aleph_index_search_slowlog.log
```

**Common Causes**:

1. **Too many shards queried**: Narrow schema filter
2. **Large result sets**: Reduce size or use aggregations
3. **Complex nested queries**: Simplify query structure
4. **Unoptimized mappings**: Use keyword instead of text for filters

**Solutions**:

1. **Add schema filter**:
```python
filters.append({"term": {"schema": "Person"}})
```

2. **Use _source filtering**:
```python
"_source": {"includes": ["schema", "caption", "collection_id"]}
```

3. **Leverage query cache**:
```json
{"size": 0, "aggregations": {...}}
```

### Issue 3: Index Red/Yellow Status

**Check status**:
```bash
curl "localhost:9200/_cluster/health?pretty"
```

**Red Status** (data loss):
- Primary shard unassigned
- Solution: Check disk space, node status, restore from backup

**Yellow Status** (no data loss):
- Replica shard unassigned
- Solution: Add nodes or reduce replica count

**Fix yellow status**:
```bash
curl -X PUT "localhost:9200/aleph-*/_settings" -H 'Content-Type: application/json' -d'
{
    "number_of_replicas": 0
}
'
```

### Issue 4: Out of Memory (OOM)

**Symptoms**: Elasticsearch crashes with OOM errors

**Causes**:
1. Heap size too small
2. Too many field data in memory
3. Large aggregations

**Solutions**:

1. **Increase heap size** (max 32GB):
```bash
export ES_JAVA_OPTS="-Xms4g -Xmx4g"
```

2. **Use keyword fields** instead of text for aggregations:
```json
{"aggregations": {"countries": {"terms": {"field": "countries"}}}}
```
Not:
```json
{"aggregations": {"countries": {"terms": {"field": "countries.text"}}}}
```

3. **Reduce aggregation size**:
```json
{"terms": {"size": 100}}  # Not 10000
```

### Issue 5: Indexing Failures

**Check bulk errors**:
```python
# aleph/index/util.py logs bulk errors
log.warning("Bulk index error: %r", details)
```

**Common Causes**:
1. **Mapping conflicts**: Field type mismatch
2. **Document too large**: > 100MB
3. **Timeout**: Document processing too slow

**Solutions**:

1. **Fix mapping conflicts**: Reindex with correct mapping
2. **Split large documents**: Break into smaller entities
3. **Increase timeout**:
```python
MAX_REQUEST_TIMEOUT = 300  # 5 minutes
```

### Issue 6: Search Not Finding Results

**Debugging steps**:

1. **Check if document is indexed**:
```bash
curl "localhost:9200/aleph-entity-person-v20231001/_doc/entity-123?pretty"
```

2. **Check analyzer**:
```bash
curl -X GET "localhost:9200/aleph-entity-person-v20231001/_analyze?pretty" -H 'Content-Type: application/json' -d'
{
    "analyzer": "latin_index",
    "text": "Müller"
}
'
```

3. **Test query directly**:
```bash
curl -X GET "localhost:9200/aleph-entity-person-v20231001/_search?pretty" -H 'Content-Type: application/json' -d'
{
    "query": {
        "query_string": {
            "query": "muller",
            "fields": ["text"]
        }
    }
}
'
```

**Common Issues**:
1. **Case sensitivity**: Analyzer not applied
2. **Authorization filter**: User doesn't have access
3. **Wrong index**: Querying old version
4. **Field not indexed**: Check mapping

---

## Best Practices

### Query Design

1. **Always apply authorization filters**
   ```python
   if not authz.is_admin:
       filters.append(authz_query(authz))
   ```

2. **Use specific schema filters**
   ```python
   filters.append({"term": {"schema": "Person"}})
   ```

3. **Leverage keyword fields for filtering**
   ```python
   {"term": {"countries": "US"}}  # Fast
   # Not: {"match": {"countries.text": "US"}}  # Slow
   ```

4. **Limit result size**
   ```python
   {"size": 30}  # Reasonable page size
   # Not: {"size": 10000}  # Too large
   ```

5. **Use _source filtering**
   ```python
   "_source": {"includes": ["schema", "caption", "collection_id"]}
   ```

### Indexing Best Practices

1. **Use bulk operations**
   ```python
   bulk_actions(actions, chunk_size=500)
   ```

2. **Disable refresh during bulk load**
   ```python
   es.indices.put_settings(body={"refresh_interval": "-1"})
   ```

3. **Force merge after bulk load**
   ```python
   es.indices.forcemerge(index=index, max_num_segments=1)
   ```

4. **Monitor bulk errors**
   ```python
   for _, details in streaming_bulk(...):
       log.warning("Bulk error: %r", details)
   ```

### Mapping Best Practices

1. **Use keyword for filters and aggregations**
   ```json
   {"type": "keyword"}  # Exact match, aggregations
   ```

2. **Use text for full-text search**
   ```json
   {"type": "text", "analyzer": "latin_index"}
   ```

3. **Exclude large fields from _source**
   ```json
   {"_source": {"excludes": ["text", "fingerprints"]}}
   ```

4. **Copy fields for multi-purpose**
   ```json
   {"type": "keyword", "copy_to": "text"}
   ```

5. **Version indices for migrations**
   ```
   aleph-entity-person-v20231001  # New version
   aleph-entity-person-v20230801  # Old version
   ```

### Performance Best Practices

1. **Cache permission lists in Redis**
2. **Use connection pooling** (maxsize=25)
3. **Set appropriate timeouts** (request_timeout=60)
4. **Monitor heap usage** (target < 75%)
5. **Use appropriate shard counts** (10-50GB per shard)
6. **Enable request cache** for aggregations
7. **Use routing** when possible

### Operational Best Practices

1. **Monitor cluster health** regularly
2. **Set up slow query logging**
3. **Configure index lifecycle policies**
4. **Implement backup strategy**
5. **Test reindexing procedures**
6. **Document custom analyzers**
7. **Version control mapping changes**

---

## Summary

Aleph's Elasticsearch integration provides a sophisticated, scalable search architecture with:

### Key Strengths

1. **Authorization-First Design**: Every query automatically filtered by permissions
2. **Per-Schema Indices**: Optimized mappings and independent scaling
3. **Zero-Downtime Migrations**: Versioned indices with read/write separation
4. **Multilingual Support**: ICU-based analyzers for Latin scripts
5. **Batch Processing**: Streaming bulk operations for high throughput
6. **Faceted Navigation**: Dynamic aggregations with post-filtering
7. **Flexible Mappings**: FollowTheMoney schema integration

### Architecture Overview

```
User Query
    ↓
SearchQueryParser (parse request parameters)
    ↓
Query Builder (build ES DSL query)
    ├── Text Query (query_string)
    ├── Filters (schema, collection_id, properties)
    ├── Authorization Filter (collection_id terms)
    ├── Aggregations (facets with post-filtering)
    ├── Sort (relevance, dates, properties)
    └── Highlight (text snippets)
    ↓
Elasticsearch Query
    ├── Read Indices (multiple versions)
    ├── Per-Schema Indices (Person, Company, Document)
    ├── Shard Distribution (1-10 shards per schema)
    └── Custom Analyzers (latin_index, latin_query)
    ↓
SearchQueryResult (format and return)
```

### Performance Characteristics

- **Query Latency**: 50-500ms (depending on complexity)
- **Indexing Throughput**: 1,000-10,000 docs/sec (bulk mode)
- **Bulk Chunk Size**: 500 documents (default)
- **Max Result Window**: 10,000 documents
- **Refresh Interval**: 1s (near real-time)

### File Reference

| File | Purpose | Key Classes/Functions |
|------|---------|----------------------|
| `aleph/index/indexes.py` | Index schemas and versioning | `schema_index()`, `configure_schema()`, `entities_read_index()` |
| `aleph/index/util.py` | Index utilities and helpers | `bulk_actions()`, `authz_query()`, `index_settings()` |
| `aleph/index/entities.py` | Entity indexing operations | `iter_entities()`, `entities_by_ids()` |
| `aleph/search/query.py` | Query builder base class | `Query`, `get_query()`, `get_aggregations()` |
| `aleph/search/facet.py` | Facet implementations | `Facet`, `SchemaFacet`, `CountryFacet` |
| `aleph/search/parser.py` | Query parameter parsing | `SearchQueryParser` |
| `aleph/search/result.py` | Result formatting | `SearchQueryResult` |

### Related Documentation

- [ARCHITECTURE.md](./ARCHITECTURE.md) - Overall system architecture
- [API.md](./API.md) - Search API endpoints
- [SEARCH.md](./SEARCH.md) - User-facing search features
- [ENTITIES.md](./ENTITIES.md) - Entity data model
- [XREF.md](./XREF.md) - Cross-reference matching
- [WORKERS.md](./WORKERS.md) - Background indexing workers
- [PERFORMANCE.md](./PERFORMANCE.md) - Performance tuning guide

---

**Version**: Based on Aleph 4.1.7 with Elasticsearch 7.17.0
**Last Updated**: 2025-12-09
