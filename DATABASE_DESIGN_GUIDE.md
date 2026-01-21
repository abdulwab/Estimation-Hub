# Database Design Guide: US City Permit Data Platform

## Overview

This guide documents the database architecture for an intelligence-driven outreach platform storing building permit data from multiple US cities and API sources.

---

## 1. Database Architecture

### Design Philosophy

| Principle | Description |
|-----------|-------------|
| **API-Agnostic** | Core schema independent of any specific API structure |
| **Schema Flexibility** | JSONB columns for API-specific fields without migrations |
| **Auditability** | Full history tracking, no silent overwrites |
| **Horizontal Scalability** | Partitioning by state/date for multi-million record scale |

### Database Choice: PostgreSQL

**Justification:**
- Native JSONB for flexible schema storage
- Table partitioning for large datasets
- Full-text search capabilities
- Mature indexing (B-tree, GIN, GiST)
- UPSERT support for efficient sync

### Separation of Concerns

```
┌─────────────────────────────────────────────────────────────┐
│                      DATA LAYERS                             │
├─────────────────────────────────────────────────────────────┤
│  RAW LAYER          │ raw_permit_payloads (JSONB)           │
├─────────────────────────────────────────────────────────────┤
│  CORE LAYER         │ permit_records, cities, agencies,     │
│                     │ contractors, owners, addresses         │
├─────────────────────────────────────────────────────────────┤
│  METADATA LAYER     │ source_apis, api_sync_logs,           │
│                     │ field_mappings                         │
├─────────────────────────────────────────────────────────────┤
│  ENRICHMENT LAYER   │ contractor_intelligence, scores,      │
│                     │ outreach_contacts                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Core Tables

### 2.1 `source_apis`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `name` | VARCHAR(100) | API name (e.g., "socrata") | No | UNIQUE |
| `base_url` | TEXT | Base API URL | No | - |
| `auth_type` | VARCHAR(50) | none/api_key/oauth | No | - |
| `rate_limit_per_hour` | INT | Rate limit | Yes | - |
| `is_active` | BOOLEAN | Active status | No | - |
| `created_at` | TIMESTAMPTZ | Creation time | No | - |

### 2.2 `cities`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `name` | VARCHAR(255) | City name | No | - |
| `state_code` | CHAR(2) | US state code | No | IDX |
| `fips_code` | VARCHAR(10) | FIPS code | Yes | UNIQUE |
| `population` | INT | Population | Yes | - |
| `timezone` | VARCHAR(50) | Timezone | Yes | - |
| `metadata` | JSONB | Additional info | Yes | GIN |
| `created_at` | TIMESTAMPTZ | Creation time | No | - |

### 2.3 `agencies`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `city_id` | UUID | FK → cities | No | IDX |
| `source_api_id` | UUID | FK → source_apis | No | IDX |
| `external_domain` | VARCHAR(255) | API domain | No | - |
| `external_dataset_id` | VARCHAR(100) | Dataset ID | No | - |
| `name` | VARCHAR(255) | Agency name | No | - |
| `sync_frequency` | VARCHAR(20) | daily/hourly/weekly | No | - |
| `last_sync_at` | TIMESTAMPTZ | Last sync time | Yes | IDX |
| `is_active` | BOOLEAN | Active status | No | - |

**Unique Constraint:** `(source_api_id, external_domain, external_dataset_id)`

### 2.4 `addresses`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `street_number` | VARCHAR(50) | House number | Yes | - |
| `street_direction` | VARCHAR(5) | N/S/E/W | Yes | - |
| `street_name` | VARCHAR(255) | Street name | No | IDX |
| `street_suffix` | VARCHAR(20) | St/Ave/Blvd | Yes | - |
| `unit` | VARCHAR(50) | Unit/Apt | Yes | - |
| `city_id` | UUID | FK → cities | No | IDX |
| `zip_code` | VARCHAR(10) | ZIP code | Yes | IDX |
| `latitude` | DECIMAL(10,7) | Latitude | Yes | - |
| `longitude` | DECIMAL(10,7) | Longitude | Yes | - |
| `location` | GEOGRAPHY | PostGIS point | Yes | GIST |
| `full_address` | TEXT | Computed | No | IDX |

**Unique Constraint:** `(full_address, city_id)`

### 2.5 `contractors`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `name` | VARCHAR(255) | Company name | No | IDX |
| `license_number` | VARCHAR(100) | License # | Yes | UNIQUE |
| `license_type` | VARCHAR(50) | License type | Yes | - |
| `license_expiry` | DATE | Expiration date | Yes | - |
| `phone` | VARCHAR(20) | Phone | Yes | - |
| `email` | VARCHAR(255) | Email | Yes | IDX |
| `address_id` | UUID | FK → addresses | Yes | - |
| `metadata` | JSONB | Extra fields | Yes | GIN |
| `created_at` | TIMESTAMPTZ | Creation time | No | - |
| `updated_at` | TIMESTAMPTZ | Update time | No | - |

### 2.6 `owners`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `name` | VARCHAR(255) | Owner name | No | IDX |
| `owner_type` | VARCHAR(20) | individual/business | No | - |
| `phone` | VARCHAR(20) | Phone | Yes | - |
| `email` | VARCHAR(255) | Email | Yes | IDX |
| `address_id` | UUID | FK → addresses | Yes | - |
| `metadata` | JSONB | Extra fields | Yes | GIN |

### 2.7 `permit_records` (Main Table)

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `agency_id` | UUID | FK → agencies | No | IDX |
| `external_permit_id` | VARCHAR(100) | Source permit # | No | - |
| `permit_number` | VARCHAR(100) | Normalized # | No | IDX |
| `permit_type` | VARCHAR(100) | Type | No | IDX |
| `permit_subtype` | VARCHAR(100) | Subtype | Yes | - |
| `permit_class` | VARCHAR(50) | residential/commercial | Yes | IDX |
| `work_description` | TEXT | Description | Yes | FTS |
| `status` | VARCHAR(50) | Current status | No | IDX |
| `application_date` | DATE | Applied | Yes | IDX |
| `issue_date` | DATE | Issued | Yes | IDX |
| `expiration_date` | DATE | Expires | Yes | - |
| `completion_date` | DATE | Completed | Yes | - |
| `estimated_cost` | DECIMAL(15,2) | Project cost | Yes | IDX |
| `actual_cost` | DECIMAL(15,2) | Final cost | Yes | - |
| `square_footage` | INT | Sq ft | Yes | - |
| `housing_units` | INT | Unit count | Yes | - |
| `address_id` | UUID | FK → addresses | No | IDX |
| `contractor_id` | UUID | FK → contractors | Yes | IDX |
| `owner_id` | UUID | FK → owners | Yes | - |
| `api_specific_fields` | JSONB | Source extras | Yes | GIN |
| `created_at` | TIMESTAMPTZ | First ingested | No | - |
| `updated_at` | TIMESTAMPTZ | Last updated | No | IDX |
| `data_hash` | VARCHAR(64) | SHA-256 dedup | No | IDX |

**Unique Constraint:** `(agency_id, external_permit_id)`  
**Partitioning:** By `issue_date` (yearly) or `state_code` via agency join

### 2.8 `permit_status_history`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `permit_id` | UUID | FK → permit_records | No | IDX |
| `status` | VARCHAR(50) | Status value | No | - |
| `changed_at` | TIMESTAMPTZ | Change time | No | IDX |
| `changed_by` | VARCHAR(100) | Sync source | Yes | - |
| `notes` | TEXT | Context | Yes | - |

### 2.9 `raw_permit_payloads`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `agency_id` | UUID | FK → agencies | No | IDX |
| `external_permit_id` | VARCHAR(100) | Source ID | No | - |
| `payload` | JSONB | Raw JSON | No | GIN |
| `fetched_at` | TIMESTAMPTZ | Fetch time | No | IDX |
| `processed` | BOOLEAN | Processing flag | No | IDX |
| `processing_error` | TEXT | Error message | Yes | - |

**Partitioning:** By `fetched_at` (monthly)

### 2.10 `api_sync_logs`

| Field | Type | Description | Nullable | Index |
|-------|------|-------------|----------|-------|
| `id` | UUID | Primary key | No | PK |
| `agency_id` | UUID | FK → agencies | No | IDX |
| `started_at` | TIMESTAMPTZ | Start time | No | IDX |
| `completed_at` | TIMESTAMPTZ | End time | Yes | - |
| `status` | VARCHAR(20) | success/failed/partial | No | IDX |
| `records_fetched` | INT | Fetched count | Yes | - |
| `records_inserted` | INT | Inserted count | Yes | - |
| `records_updated` | INT | Updated count | Yes | - |
| `error_message` | TEXT | Error details | Yes | - |

---

## 3. Handling Multiple API Schemas

### Normalization Strategy

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Socrata API    │───▶│  FIELD MAPPER    │───▶│  Core Schema    │
│  (Chicago)      │    │  (Transform)     │    │  permit_records │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         │                     │
         ▼                     ▼
┌─────────────────┐    ┌──────────────────┐
│  raw_payloads   │    │  api_specific_   │
│  (JSONB)        │    │  fields (JSONB)  │
└─────────────────┘    └──────────────────┘
```

### Field Mapping Table: `field_mappings`

| Field | Type | Description |
|-------|------|-------------|
| `id` | UUID | Primary key |
| `agency_id` | UUID | FK → agencies |
| `source_field` | VARCHAR(100) | API field name |
| `target_field` | VARCHAR(100) | Core field name |
| `transform_fn` | VARCHAR(50) | Transform type |
| `is_required` | BOOLEAN | Required flag |

### Socrata → Core Mapping Example

| Socrata Field (Chicago) | Core Field | Transform |
|-------------------------|------------|-----------|
| `permit_` | `permit_number` | direct |
| `permit_type` | `permit_type` | direct |
| `issue_date` | `issue_date` | parse_date |
| `reported_cost` | `estimated_cost` | to_decimal |
| `work_description` | `work_description` | direct |
| `street_number` | `addresses.street_number` | direct |
| `latitude` | `addresses.latitude` | to_decimal |
| `contact_1_name` | `api_specific_fields.contacts[0]` | json_nest |

### Handling Missing/Extra Fields

```python
def map_permit(raw: dict, mappings: list) -> dict:
    core = {}
    extras = {}
    
    for key, value in raw.items():
        mapping = find_mapping(key, mappings)
        if mapping:
            core[mapping.target_field] = transform(value, mapping.transform_fn)
        else:
            extras[key] = value
    
    core['api_specific_fields'] = extras
    return core
```

---

## 4. Data Ingestion Pipeline

### Pipeline Flow

```
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│  FETCH   │──▶│   RAW    │──▶│ VALIDATE │──▶│NORMALIZE │──▶│  UPSERT  │
│  (API)   │   │  STORE   │   │          │   │          │   │  (Core)  │
└──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
     │              │              │              │              │
     ▼              ▼              ▼              ▼              ▼
  Paginate    raw_permit_    Schema      Field         permit_
  + Retry     payloads       Check       Mapping       records
```

### Step Details

1. **Fetch**: Paginated API calls with retry/backoff
2. **Raw Store**: Insert into `raw_permit_payloads` (JSONB)
3. **Validate**: Schema validation, required fields check
4. **Normalize**: Apply field mappings, transforms
5. **Upsert**: Insert or update `permit_records`

### Deduplication Logic

```sql
-- Using data_hash for change detection
INSERT INTO permit_records (...)
VALUES (...)
ON CONFLICT (agency_id, external_permit_id)
DO UPDATE SET
    status = EXCLUDED.status,
    updated_at = NOW()
WHERE permit_records.data_hash != EXCLUDED.data_hash;
```

### Upsert vs Insert Strategy

| Scenario | Strategy |
|----------|----------|
| New permit | INSERT |
| Existing + changed hash | UPDATE core fields |
| Existing + same hash | SKIP |
| Status change | UPDATE + INSERT into history |

---

## 5. Data Freshness & Versioning

### Tracking Last Update

```sql
-- Per-agency freshness
SELECT 
    a.name,
    a.last_sync_at,
    COUNT(p.id) as permit_count,
    MAX(p.updated_at) as latest_permit_update
FROM agencies a
LEFT JOIN permit_records p ON p.agency_id = a.id
GROUP BY a.id;
```

### Status History Tracking

```sql
-- Trigger for status changes
CREATE OR REPLACE FUNCTION track_status_change()
RETURNS TRIGGER AS $$
BEGIN
    IF OLD.status IS DISTINCT FROM NEW.status THEN
        INSERT INTO permit_status_history (permit_id, status, changed_at)
        VALUES (NEW.id, NEW.status, NOW());
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER permit_status_trigger
AFTER UPDATE ON permit_records
FOR EACH ROW EXECUTE FUNCTION track_status_change();
```

### Audit Strategy

- **No deletes**: Use `is_active` flag for soft deletes
- **No overwrites**: Status changes logged to history table
- **Version tracking**: `data_hash` changes trigger updates

---

## 6. Scalability & Growth

### Partitioning Strategy

```sql
-- Partition by issue_date (yearly)
CREATE TABLE permit_records (
    ...
) PARTITION BY RANGE (issue_date);

CREATE TABLE permit_records_2024 
    PARTITION OF permit_records
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE permit_records_2025
    PARTITION OF permit_records
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

### Indexing for Analytics

```sql
-- Composite indexes for common queries
CREATE INDEX idx_permits_city_date 
    ON permit_records (agency_id, issue_date DESC);

CREATE INDEX idx_permits_type_status 
    ON permit_records (permit_type, status);

CREATE INDEX idx_permits_cost 
    ON permit_records (estimated_cost DESC) 
    WHERE estimated_cost > 100000;

-- Full-text search
CREATE INDEX idx_permits_fts 
    ON permit_records 
    USING GIN (to_tsvector('english', work_description));
```

### Growth Estimates

| Metric | Year 1 | Year 3 | Year 5 |
|--------|--------|--------|--------|
| Cities | 50 | 200 | 500+ |
| APIs | 1 (Socrata) | 5 | 10+ |
| Permits | 5M | 25M | 100M+ |
| Storage | 50GB | 250GB | 1TB+ |

---

## 7. Practical Examples

### Normalized Permit Record

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "agency_id": "123e4567-e89b-12d3-a456-426614174000",
  "external_permit_id": "BLD2025-12345",
  "permit_number": "BLD2025-12345",
  "permit_type": "PERMIT - NEW CONSTRUCTION",
  "permit_class": "residential",
  "work_description": "Construct new 3-story SFD",
  "status": "ISSUED",
  "application_date": "2025-01-10",
  "issue_date": "2025-01-15",
  "estimated_cost": 450000.00,
  "housing_units": 1,
  "address_id": "789e0123-e89b-12d3-a456-426614174000",
  "contractor_id": "abc12345-e89b-12d3-a456-426614174000",
  "api_specific_fields": {
    "ward": 42,
    "community_area": 8,
    "processing_time": 5,
    "contact_1_name": "SMITH CONSTRUCTION",
    "contact_1_type": "CONTRACTOR"
  }
}
```

### Raw API Payload (Socrata Chicago)

```json
{
  "id": "12345678",
  "permit_": "BLD2025-12345",
  "permit_type": "PERMIT - NEW CONSTRUCTION",
  "issue_date": "2025-01-15T00:00:00.000",
  "street_number": "1234",
  "street_direction": "N",
  "street_name": "MICHIGAN",
  "reported_cost": "450000",
  "work_description": "Construct new 3-story SFD",
  "ward": "42",
  "latitude": "41.8781",
  "longitude": "-87.6298",
  "contact_1_name": "SMITH CONSTRUCTION"
}
```

---

## 8. Assumptions & Constraints

| Assumption | Mitigation |
|------------|------------|
| APIs may change schemas | JSONB storage + field mappings |
| Inconsistent date formats | Transform layer with parsing |
| Missing required fields | Nullable core fields + validation |
| Duplicate permits across syncs | Hash-based deduplication |
| API rate limits | Backoff + throttling |
| Partial sync failures | Transaction rollback + retry queue |

### Data Quality Rules

- `permit_number` must be non-empty
- `issue_date` must be valid date or NULL
- `estimated_cost` must be ≥ 0 if present
- `latitude/longitude` must be valid coordinates if present

---

## ER Diagram (Simplified)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ source_apis  │     │   cities     │     │  addresses   │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       ▼                    ▼                    │
┌──────────────┐     ┌──────────────┐           │
│   agencies   │────▶│permit_records│◀──────────┘
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ├──────────▶ contractors
┌──────────────┐            │
│ api_sync_logs│            └──────────▶ owners
└──────────────┘
       │
       ▼
┌──────────────┐     ┌──────────────────┐
│raw_payloads  │     │permit_status_hist│
└──────────────┘     └──────────────────┘
```

---

## Quick Start SQL

```sql
-- Create extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "postgis";

-- Create tables (abbreviated)
CREATE TABLE cities (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    state_code CHAR(2) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE permit_records (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    agency_id UUID NOT NULL REFERENCES agencies(id),
    external_permit_id VARCHAR(100) NOT NULL,
    permit_number VARCHAR(100) NOT NULL,
    permit_type VARCHAR(100) NOT NULL,
    status VARCHAR(50) NOT NULL,
    issue_date DATE,
    estimated_cost DECIMAL(15,2),
    address_id UUID REFERENCES addresses(id),
    api_specific_fields JSONB,
    data_hash VARCHAR(64) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(agency_id, external_permit_id)
);

CREATE INDEX idx_permits_issue_date ON permit_records(issue_date DESC);
CREATE INDEX idx_permits_agency ON permit_records(agency_id);
CREATE INDEX idx_permits_type ON permit_records(permit_type);
CREATE INDEX idx_permits_api_fields ON permit_records USING GIN(api_specific_fields);
```
