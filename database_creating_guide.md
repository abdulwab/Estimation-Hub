# Database Creation Guide - Permit Intelligence Platform

This guide explains how to create the permit database schema and handle data from multiple city APIs with different field structures.

---

## Table of Contents

1. [Database Overview](#database-overview)
2. [Complete Field Reference](#complete-field-reference)
3. [Handling Different Schemas](#handling-different-schemas)
4. [Field Mapping Examples](#field-mapping-examples)
5. [SQL Migration Scripts](#sql-migration-scripts)

---

## Database Overview

The database uses PostgreSQL with:
- **14 tables** organized into geographic, contact, permit, and pipeline management
- **Partitioning** by city for scalability
- **JSONB columns** for storing source-specific fields without schema changes
- **PostGIS** for geolocation queries

### Table Summary

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `states` | US/Canada states/provinces | code, name, fips_code |
| `counties` | County/parish data | state_id, name, fips_code |
| `cities` | City information | state_id, county_id, name, slug |
| `data_sources` | API/data source config | city_id, source_type, field_mappings |
| `addresses` | Normalized addresses | street components, lat/lng, geom |
| `contacts` | People/companies | name, email, phone, license_number |
| `permits` | Core permit records | All permit data (partitioned by city) |
| `permit_contacts` | Permit-contact links | permit_id, contact_id, role |
| `permit_status_history` | Status changes | permit_id, status, status_date |
| `sync_jobs` | Pipeline job tracking | source_id, status, metrics |
| `sync_job_logs` | Detailed sync logs | job_id, level, message |

---

## Complete Field Reference

### Geographic Tables

#### `states`
| Field | Type | Description | Example |
|-------|------|-------------|---------|
| id | SERIAL | Primary key | 1 |
| code | VARCHAR(2) | State abbreviation | 'CA', 'TX', 'NY' |
| name | VARCHAR(100) | Full state name | 'California' |
| fips_code | VARCHAR(2) | Federal code | '06' |
| timezone | VARCHAR(50) | Default timezone | 'America/Los_Angeles' |

#### `counties`
| Field | Type | Description | Example |
|-------|------|-------------|---------|
| id | SERIAL | Primary key | 1 |
| state_id | INTEGER | FK to states | 1 |
| name | VARCHAR(255) | County name | 'Los Angeles County' |
| fips_code | VARCHAR(5) | Federal code | '06037' |

#### `cities`
| Field | Type | Description | Example |
|-------|------|-------------|---------|
| id | SERIAL | Primary key | 1 |
| state_id | INTEGER | FK to states | 1 |
| county_id | INTEGER | FK to counties | 1 |
| name | VARCHAR(255) | City name | 'Los Angeles' |
| slug | VARCHAR(255) | URL-friendly name | 'los-angeles' |
| population | INTEGER | City population | 3900000 |
| is_active | BOOLEAN | Sync enabled | true |
| metadata | JSONB | Extra city data | {"mayor": "...", "area_sq_mi": 469} |

---

### Data Source Configuration

#### `data_sources`
| Field | Type | Description | Example |
|-------|------|-------------|---------|
| id | SERIAL | Primary key | 1 |
| city_id | INTEGER | FK to cities | 1 |
| name | VARCHAR(255) | Source name | 'Chicago Building Permits' |
| source_type | ENUM | Type of source | 'socrata', 'accela', 'csv' |
| status | ENUM | Current status | 'active', 'inactive' |
| base_url | TEXT | API base URL | 'https://data.cityofchicago.org' |
| dataset_id | VARCHAR(100) | Dataset identifier | 'ydr8-5enu' |
| api_key_ref | VARCHAR(100) | Secrets manager key | 'chicago_api_key' |
| sync_frequency | INTERVAL | How often to sync | '1 day' |
| **field_mappings** | **JSONB** | **Source→Canonical mapping** | See below |
| config | JSONB | Extra config | {"date_format": "ISO8601"} |

**field_mappings Example:**
```json
{
  "permit_number": "permit_",
  "issued_date": "issue_date",
  "application_date": "application_start_date",
  "description": "work_description",
  "estimated_cost": "reported_cost",
  "street_number": "street_number",
  "street_name": "street_name",
  "latitude": "latitude",
  "longitude": "longitude"
}
```

---

### Address Table

#### `addresses`
| Field | Type | Description | Example |
|-------|------|-------------|---------|
| id | BIGSERIAL | Primary key | 1 |
| street_number | VARCHAR(50) | House number | '123' |
| street_direction | VARCHAR(10) | Direction prefix | 'N', 'S', 'E', 'W' |
| street_name | VARCHAR(255) | Street name | 'MICHIGAN' |
| street_suffix | VARCHAR(50) | Street type | 'AVE', 'ST', 'BLVD' |
| unit_type | VARCHAR(20) | Unit designator | 'APT', 'STE', 'UNIT' |
| unit_number | VARCHAR(50) | Unit number | '4B' |
| city_id | INTEGER | FK to cities | 1 |
| state_id | INTEGER | FK to states | 1 |
| zip_code | VARCHAR(10) | ZIP code | '60601' |
| zip_plus4 | VARCHAR(4) | ZIP+4 extension | '1234' |
| latitude | DECIMAL(10,8) | Latitude | 41.8781136 |
| longitude | DECIMAL(11,8) | Longitude | -87.6297982 |
| geom | GEOMETRY | PostGIS point | POINT(-87.629 41.878) |
| geocode_accuracy | VARCHAR(50) | Geocode quality | 'rooftop', 'street' |
| census_tract | VARCHAR(20) | Census tract ID | '170318391001' |
| parcel_number | VARCHAR(100) | Tax parcel ID | '14-33-200-015-0000' |
| address_hash | VARCHAR(64) | Dedup hash | SHA256 of normalized address |

---

### Contact Table

#### `contacts`
| Field | Type | Description | Example |
|-------|------|-------------|---------|
| id | BIGSERIAL | Primary key | 1 |
| contact_type | ENUM | Individual or company | 'individual', 'company' |
| first_name | VARCHAR(100) | First name | 'John' |
| middle_name | VARCHAR(100) | Middle name | 'Robert' |
| last_name | VARCHAR(100) | Last name | 'Smith' |
| company_name | VARCHAR(500) | Company name | 'ABC Construction LLC' |
| dba_name | VARCHAR(500) | Doing business as | 'ABC Builders' |
| email | VARCHAR(255) | Email address | 'john@abc.com' |
| phone | VARCHAR(50) | Primary phone | '312-555-1234' |
| address_id | BIGINT | FK to addresses | 1 |
| license_number | VARCHAR(100) | Contractor license | 'B-123456' |
| license_type | VARCHAR(100) | License class | 'General Contractor' |
| license_state | VARCHAR(2) | License state | 'IL' |
| license_expiry | DATE | License expiration | '2025-12-31' |
| enrichment_data | JSONB | ML/enrichment signals | {"lead_score": 0.85} |

---

### Core Permit Table (Partitioned)

#### `permits`
| Field | Type | Description | Maps From Sources |
|-------|------|-------------|-------------------|
| **Identity** ||||
| id | BIGSERIAL | Primary key | auto-generated |
| source_id | INTEGER | FK to data_sources | system-assigned |
| source_permit_id | VARCHAR(255) | Original ID from source | varies by city |
| city_id | INTEGER | FK to cities (partition key) | system-assigned |
| permit_hash | VARCHAR(64) | Cross-source dedup hash | computed |
| **Permit Identification** ||||
| permit_number | VARCHAR(255) | Permit tracking number | `permit_`, `permitnum`, `permit_no` |
| permit_category | ENUM | Category | `permit_type`, `permitclass` |
| permit_type | VARCHAR(255) | Source-specific type | `permit_type`, `permittypemapped` |
| permit_subtype | VARCHAR(255) | Sub-type | `permit_subtype`, `permittypedesc` |
| work_class | VARCHAR(100) | Work class | `work_type`, `workclass` |
| **Location** ||||
| address_id | BIGINT | FK to addresses | built from address fields |
| **Status** ||||
| status | ENUM | Current status | `permit_status`, `statuscurrent` |
| status_detail | VARCHAR(255) | Status details | `permit_milestone` |
| **Dates** ||||
| application_date | DATE | Application submitted | `application_start_date`, `applieddate` |
| issued_date | DATE | Permit issued | `issue_date`, `issueddate` |
| expiration_date | DATE | Permit expires | `expiresdate` |
| completed_date | DATE | Work completed | `completeddate` |
| finaled_date | DATE | Final inspection | `finaled_date` |
| **Project Details** ||||
| description | TEXT | Work description | `work_description`, `description` |
| **Valuation** ||||
| estimated_cost | DECIMAL(15,2) | Estimated cost | `reported_cost`, `estprojectcost` |
| permit_fee | DECIMAL(12,2) | Permit fee | `total_fee`, varies |
| total_fees | DECIMAL(12,2) | All fees | `subtotal_paid + subtotal_unpaid` |
| **Building Specifics** ||||
| square_footage | INTEGER | Project size | `totalsqft`, `total_sq_ft` |
| stories | INTEGER | Number of stories | `of_stories` |
| units | INTEGER | Dwelling units | `housingunits`, `of_residential_dwelling_units` |
| occupancy_type | VARCHAR(100) | Occupancy type | `occupancy` |
| **Extensible Fields** ||||
| raw_data | JSONB | Original source data | entire source record |
| extra_fields | JSONB | Unmapped fields | fields not in canonical schema |
| **Versioning** ||||
| version | INTEGER | Record version | 1, 2, 3... |
| is_current | BOOLEAN | Current version flag | true/false |
| previous_version_id | BIGINT | Link to prior version | FK to permits |

---

## Handling Different Schemas

### The Problem

Each city API returns data with different field names and structures:

**Chicago** (`data.cityofchicago.org`):
```json
{
  "permit_": "BLD2024-12345",
  "issue_date": "2024-01-15",
  "application_start_date": "2024-01-01",
  "work_description": "New 3-story residential building",
  "reported_cost": 450000,
  "street_number": "123",
  "street_name": "MICHIGAN",
  "latitude": 41.8781
}
```

**Seattle** (`cos-data.seattle.gov`):
```json
{
  "permitnum": "6789012-CN",
  "issueddate": "2024-01-15T00:00:00.000",
  "applieddate": "2024-01-01T00:00:00.000",
  "description": "Construct new 3-story dwelling",
  "estprojectcost": 450000,
  "originaladdress1": "123 Main St",
  "latitude": 47.6062
}
```

**Calgary** (`data.calgary.ca`):
```json
{
  "permitnum": "BP2024-98765",
  "issueddate": "2024-01-15",
  "applieddate": "2024-01-01",
  "description": "New residential construction",
  "estprojectcost": 600000,
  "originaladdress": "123 Centre St SW",
  "latitude": 51.0447
}
```

### The Solution: Field Mappings + JSONB

#### 1. Store Field Mappings Per Source

Each `data_sources` record has a `field_mappings` JSONB column:

```sql
-- Chicago field mapping
INSERT INTO data_sources (city_id, name, source_type, base_url, dataset_id, field_mappings)
VALUES (1, 'Chicago Building Permits', 'socrata', 'https://data.cityofchicago.org', 'ydr8-5enu', '{
  "permit_number": "permit_",
  "issued_date": "issue_date",
  "application_date": "application_start_date",
  "description": "work_description",
  "estimated_cost": "reported_cost",
  "street_number": "street_number",
  "street_name": "street_name",
  "latitude": "latitude",
  "longitude": "longitude",
  "status": "permit_status"
}');

-- Seattle field mapping
INSERT INTO data_sources (city_id, name, source_type, base_url, dataset_id, field_mappings)
VALUES (2, 'Seattle Building Permits', 'socrata', 'https://cos-data.seattle.gov', '76t5-zqzr', '{
  "permit_number": "permitnum",
  "issued_date": "issueddate",
  "application_date": "applieddate",
  "description": "description",
  "estimated_cost": "estprojectcost",
  "units": "housingunits",
  "status": "statuscurrent"
}');
```

#### 2. Map During Ingestion

```python
class FieldMapper:
    """Maps source fields to canonical schema using stored mappings."""
    
    def __init__(self, field_mappings: dict):
        self.mappings = field_mappings
    
    def transform(self, raw_record: dict) -> dict:
        """Transform source record to canonical format."""
        canonical = {}
        mapped_fields = set()
        
        # Map known fields
        for canonical_field, source_field in self.mappings.items():
            if source_field in raw_record:
                value = raw_record[source_field]
                canonical[canonical_field] = self._transform_value(
                    canonical_field, value
                )
                mapped_fields.add(source_field)
        
        # Store unmapped fields in extra_fields (preserves all data!)
        canonical['extra_fields'] = {
            k: v for k, v in raw_record.items()
            if k not in mapped_fields
        }
        
        # Store complete raw data for debugging
        canonical['raw_data'] = raw_record
        
        return canonical
    
    def _transform_value(self, field: str, value):
        """Apply field-specific transformations."""
        
        # Date normalization
        if field in ('issued_date', 'application_date', 'expiration_date'):
            return self._parse_date(value)
        
        # Currency normalization
        if field in ('estimated_cost', 'permit_fee', 'total_fees'):
            return self._parse_currency(value)
        
        # Status normalization
        if field == 'status':
            return self._normalize_status(value)
        
        return value
    
    def _parse_date(self, value) -> str:
        """Normalize various date formats to YYYY-MM-DD."""
        if not value:
            return None
        
        # Handle ISO 8601 with time
        if 'T' in str(value):
            return str(value).split('T')[0]
        
        # Handle common US formats
        from dateutil import parser
        try:
            parsed = parser.parse(str(value))
            return parsed.strftime('%Y-%m-%d')
        except:
            return None
    
    def _parse_currency(self, value) -> float:
        """Extract numeric value from currency string."""
        if value is None:
            return None
        
        # Remove $, commas, etc.
        cleaned = str(value).replace('$', '').replace(',', '').strip()
        try:
            return float(cleaned)
        except:
            return None
    
    def _normalize_status(self, value) -> str:
        """Map source-specific status to canonical enum."""
        if not value:
            return 'applied'
        
        value_lower = str(value).lower().strip()
        
        STATUS_MAP = {
            # Chicago statuses
            'issued': 'issued',
            'complete': 'completed',
            'permit issued': 'issued',
            
            # Seattle statuses
            'application accepted': 'applied',
            'in review': 'under_review',
            'permit issued': 'issued',
            'permit finaled': 'finaled',
            
            # Generic mappings
            'approved': 'approved',
            'expired': 'expired',
            'cancelled': 'cancelled',
            'void': 'void',
        }
        
        return STATUS_MAP.get(value_lower, 'other')
```

#### 3. Query Both Canonical and Source-Specific Fields

```sql
-- Query using canonical fields (works for all cities)
SELECT permit_number, issued_date, estimated_cost
FROM permits
WHERE status = 'issued' AND issued_date > '2024-01-01';

-- Query source-specific field from Chicago using JSONB
SELECT 
    permit_number,
    raw_data->>'community_area' as community_area,
    raw_data->>'ward' as ward
FROM permits
WHERE city_id = 1  -- Chicago
  AND (raw_data->>'community_area')::int = 32;

-- Query extra_fields for unmapped data
SELECT 
    permit_number,
    extra_fields->>'processing_time' as review_days
FROM permits
WHERE city_id = 1;
```

---

## Field Mapping Examples

### Chicago (Socrata) → Canonical

| Source Field (Chicago) | Canonical Field | Transform |
|------------------------|-----------------|-----------|
| `permit_` | `permit_number` | Direct |
| `issue_date` | `issued_date` | Parse date |
| `application_start_date` | `application_date` | Parse date |
| `work_description` | `description` | Direct |
| `reported_cost` | `estimated_cost` | Parse currency |
| `total_fee` | `total_fees` | Parse currency |
| `permit_type` | `permit_type` | Direct |
| `permit_status` | `status` | Normalize to enum |
| `street_number` | → `addresses.street_number` | Build address |
| `street_direction` | → `addresses.street_direction` | Build address |
| `street_name` | → `addresses.street_name` | Build address |
| `latitude` | → `addresses.latitude` | Direct |
| `longitude` | → `addresses.longitude` | Direct |
| `community_area` | → `raw_data.community_area` | Store in JSONB |
| `ward` | → `raw_data.ward` | Store in JSONB |
| `census_tract` | → `addresses.census_tract` | Direct |

### Seattle (Socrata) → Canonical

| Source Field (Seattle) | Canonical Field | Transform |
|-----------------------|-----------------|-----------|
| `permitnum` | `permit_number` | Direct |
| `issueddate` | `issued_date` | Parse ISO date |
| `applieddate` | `application_date` | Parse ISO date |
| `completeddate` | `completed_date` | Parse ISO date |
| `expiresdate` | `expiration_date` | Parse ISO date |
| `description` | `description` | Direct |
| `estprojectcost` | `estimated_cost` | Direct (already numeric) |
| `permitclass` | `permit_category` | Map to enum |
| `statuscurrent` | `status` | Normalize to enum |
| `housingunits` | `units` | Direct |
| `housingunitsremoved` | → `extra_fields.units_removed` | Store in JSONB |
| `housingunitsadded` | → `extra_fields.units_added` | Store in JSONB |
| `originaladdress1` | → Parse into address components | Complex transform |
| `latitude` | → `addresses.latitude` | Direct |
| `longitude` | → `addresses.longitude` | Direct |

### Calgary (Socrata) → Canonical

| Source Field (Calgary) | Canonical Field | Transform |
|------------------------|-----------------|-----------|
| `permitnum` | `permit_number` | Direct |
| `issueddate` | `issued_date` | Parse date |
| `applieddate` | `application_date` | Parse date |
| `completeddate` | `completed_date` | Parse date |
| `description` | `description` | Direct |
| `estprojectcost` | `estimated_cost` | Parse CAD currency |
| `permittype` | `permit_type` | Direct |
| `workclass` | `work_class` | Direct |
| `statuscurrent` | `status` | Normalize to enum |
| `housingunits` | `units` | Direct |
| `communityname` | → `extra_fields.community` | Store in JSONB |
| `originaladdress` | → Parse into address | Complex transform |

---

## SQL Migration Scripts

### Step 1: Create Enums

```sql
-- Create all enum types
CREATE TYPE source_type AS ENUM ('socrata', 'accela', 'opengov', 'csv', 'api', 'scraper');
CREATE TYPE source_status AS ENUM ('active', 'inactive', 'deprecated', 'testing');
CREATE TYPE contact_type AS ENUM ('individual', 'company');
CREATE TYPE permit_contact_role AS ENUM ('applicant', 'owner', 'contractor', 'architect', 'engineer', 'expeditor', 'tenant', 'agent', 'other');
CREATE TYPE permit_status AS ENUM ('applied', 'under_review', 'approved', 'issued', 'active', 'expired', 'completed', 'finaled', 'cancelled', 'revoked', 'void', 'on_hold');
CREATE TYPE permit_category AS ENUM ('building', 'electrical', 'plumbing', 'mechanical', 'hvac', 'demolition', 'grading', 'fire', 'sign', 'roofing', 'solar', 'pool', 'fence', 'accessory', 'commercial', 'residential', 'mixed_use', 'other');
CREATE TYPE sync_job_status AS ENUM ('pending', 'running', 'completed', 'failed', 'cancelled');
CREATE TYPE sync_job_type AS ENUM ('full', 'incremental', 'backfill');
CREATE TYPE log_level AS ENUM ('debug', 'info', 'warning', 'error', 'critical');
```

### Step 2: Create Geographic Tables

```sql
-- States
CREATE TABLE states (
    id              SERIAL PRIMARY KEY,
    code            VARCHAR(2) NOT NULL UNIQUE,
    name            VARCHAR(100) NOT NULL,
    fips_code       VARCHAR(2),
    timezone        VARCHAR(50),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Counties
CREATE TABLE counties (
    id              SERIAL PRIMARY KEY,
    state_id        INTEGER NOT NULL REFERENCES states(id),
    name            VARCHAR(255) NOT NULL,
    fips_code       VARCHAR(5),
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(state_id, name)
);

-- Cities
CREATE TABLE cities (
    id              SERIAL PRIMARY KEY,
    state_id        INTEGER NOT NULL REFERENCES states(id),
    county_id       INTEGER REFERENCES counties(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    population      INTEGER,
    timezone        VARCHAR(50),
    is_active       BOOLEAN DEFAULT true,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(state_id, slug)
);
```

### Step 3: Create Data Sources Table

```sql
CREATE TABLE data_sources (
    id              SERIAL PRIMARY KEY,
    city_id         INTEGER NOT NULL REFERENCES cities(id),
    name            VARCHAR(255) NOT NULL,
    source_type     source_type NOT NULL,
    status          source_status DEFAULT 'active',
    base_url        TEXT,
    dataset_id      VARCHAR(100),
    api_key_ref     VARCHAR(100),
    sync_enabled    BOOLEAN DEFAULT true,
    sync_frequency  INTERVAL DEFAULT '1 day',
    last_sync_at    TIMESTAMPTZ,
    next_sync_at    TIMESTAMPTZ,
    field_mappings  JSONB NOT NULL DEFAULT '{}',
    config          JSONB DEFAULT '{}',
    total_records   BIGINT DEFAULT 0,
    last_error      TEXT,
    error_count     INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### Step 4: Create Addresses Table

```sql
-- Enable PostGIS extension
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE addresses (
    id                  BIGSERIAL PRIMARY KEY,
    street_number       VARCHAR(50),
    street_direction    VARCHAR(10),
    street_name         VARCHAR(255),
    street_suffix       VARCHAR(50),
    unit_type           VARCHAR(20),
    unit_number         VARCHAR(50),
    city_id             INTEGER REFERENCES cities(id),
    state_id            INTEGER REFERENCES states(id),
    zip_code            VARCHAR(10),
    zip_plus4           VARCHAR(4),
    county_id           INTEGER REFERENCES counties(id),
    full_address        TEXT GENERATED ALWAYS AS (
        COALESCE(street_number, '') || ' ' ||
        COALESCE(street_direction || ' ', '') ||
        COALESCE(street_name, '') || ' ' ||
        COALESCE(street_suffix, '')
    ) STORED,
    latitude            DECIMAL(10, 8),
    longitude           DECIMAL(11, 8),
    geom                GEOMETRY(Point, 4326),
    geocode_accuracy    VARCHAR(50),
    geocoded_at         TIMESTAMPTZ,
    census_tract        VARCHAR(20),
    census_block        VARCHAR(20),
    parcel_number       VARCHAR(100),
    address_hash        VARCHAR(64) UNIQUE,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Create spatial index
CREATE INDEX idx_addresses_geom ON addresses USING GIST(geom);
```

### Step 5: Create Contacts Table

```sql
CREATE TABLE contacts (
    id                  BIGSERIAL PRIMARY KEY,
    contact_type        contact_type NOT NULL,
    first_name          VARCHAR(100),
    middle_name         VARCHAR(100),
    last_name           VARCHAR(100),
    company_name        VARCHAR(500),
    dba_name            VARCHAR(500),
    display_name        TEXT GENERATED ALWAYS AS (
        CASE 
            WHEN contact_type = 'company' THEN COALESCE(company_name, dba_name)
            ELSE TRIM(COALESCE(first_name, '') || ' ' || COALESCE(last_name, ''))
        END
    ) STORED,
    email               VARCHAR(255),
    phone               VARCHAR(50),
    phone_alt           VARCHAR(50),
    fax                 VARCHAR(50),
    website             TEXT,
    address_id          BIGINT REFERENCES addresses(id),
    license_number      VARCHAR(100),
    license_type        VARCHAR(100),
    license_state       VARCHAR(2),
    license_expiry      DATE,
    contact_hash        VARCHAR(64),
    enrichment_data     JSONB DEFAULT '{}',
    enriched_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);
```

### Step 6: Create Permits Table (Partitioned)

```sql
CREATE TABLE permits (
    id                      BIGSERIAL,
    source_id               INTEGER NOT NULL REFERENCES data_sources(id),
    source_permit_id        VARCHAR(255) NOT NULL,
    city_id                 INTEGER NOT NULL REFERENCES cities(id),
    address_id              BIGINT REFERENCES addresses(id),
    permit_number           VARCHAR(255),
    permit_category         permit_category,
    permit_type             VARCHAR(255),
    permit_subtype          VARCHAR(255),
    work_class              VARCHAR(100),
    status                  permit_status DEFAULT 'applied',
    status_detail           VARCHAR(255),
    application_date        DATE,
    issued_date             DATE,
    expiration_date         DATE,
    completed_date          DATE,
    finaled_date            DATE,
    description             TEXT,
    work_description        TEXT,
    estimated_cost          DECIMAL(15, 2),
    declared_valuation      DECIMAL(15, 2),
    permit_fee              DECIMAL(12, 2),
    total_fees              DECIMAL(12, 2),
    fees_paid               DECIMAL(12, 2),
    square_footage          INTEGER,
    stories                 INTEGER,
    units                   INTEGER,
    bedrooms                INTEGER,
    bathrooms               DECIMAL(3, 1),
    occupancy_type          VARCHAR(100),
    construction_type       VARCHAR(100),
    review_type             VARCHAR(100),
    processing_days         INTEGER,
    raw_data                JSONB DEFAULT '{}',
    extra_fields            JSONB DEFAULT '{}',
    version                 INTEGER DEFAULT 1,
    is_current              BOOLEAN DEFAULT true,
    previous_version_id     BIGINT,
    permit_hash             VARCHAR(64),
    ingested_at             TIMESTAMPTZ DEFAULT NOW(),
    last_seen_at            TIMESTAMPTZ DEFAULT NOW(),
    created_at              TIMESTAMPTZ DEFAULT NOW(),
    updated_at              TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (id, city_id)
) PARTITION BY LIST (city_id);

-- Create indexes
CREATE INDEX idx_permits_source ON permits(source_id, source_permit_id);
CREATE INDEX idx_permits_number ON permits(permit_number);
CREATE INDEX idx_permits_status ON permits(status);
CREATE INDEX idx_permits_issued ON permits(issued_date DESC);
CREATE INDEX idx_permits_hash ON permits(permit_hash);
CREATE INDEX idx_permits_raw ON permits USING GIN(raw_data jsonb_path_ops);
CREATE INDEX idx_permits_extra ON permits USING GIN(extra_fields jsonb_path_ops);
```

### Step 7: Create City Partitions

```sql
-- Function to create city partition
CREATE OR REPLACE FUNCTION create_city_partition(p_city_id INTEGER, p_city_slug TEXT)
RETURNS void AS $$
BEGIN
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS permits_%s PARTITION OF permits FOR VALUES IN (%s)',
        p_city_slug, p_city_id
    );
END;
$$ LANGUAGE plpgsql;

-- Create partitions for initial cities
SELECT create_city_partition(1, 'chicago');
SELECT create_city_partition(2, 'seattle');
SELECT create_city_partition(3, 'los_angeles');
SELECT create_city_partition(4, 'calgary');
-- Add more as needed
```

### Step 8: Create Relationship & History Tables

```sql
-- Permit-Contact junction
CREATE TABLE permit_contacts (
    id              BIGSERIAL PRIMARY KEY,
    permit_id       BIGINT NOT NULL,
    city_id         INTEGER NOT NULL,
    contact_id      BIGINT NOT NULL REFERENCES contacts(id),
    role            permit_contact_role NOT NULL,
    is_primary      BOOLEAN DEFAULT false,
    raw_name        TEXT,
    raw_data        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    FOREIGN KEY (permit_id, city_id) REFERENCES permits(id, city_id)
);

-- Status history
CREATE TABLE permit_status_history (
    id              BIGSERIAL PRIMARY KEY,
    permit_id       BIGINT NOT NULL,
    city_id         INTEGER NOT NULL,
    status          permit_status NOT NULL,
    status_detail   VARCHAR(255),
    status_date     DATE,
    notes           TEXT,
    changed_by      VARCHAR(255),
    source_id       INTEGER REFERENCES data_sources(id),
    raw_data        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    FOREIGN KEY (permit_id, city_id) REFERENCES permits(id, city_id)
);
```

### Step 9: Create Sync Management Tables

```sql
-- Sync jobs
CREATE TABLE sync_jobs (
    id                  BIGSERIAL PRIMARY KEY,
    source_id           INTEGER NOT NULL REFERENCES data_sources(id),
    job_type            sync_job_type NOT NULL DEFAULT 'incremental',
    status              sync_job_status NOT NULL DEFAULT 'pending',
    sync_from           TIMESTAMPTZ,
    sync_to             TIMESTAMPTZ,
    started_at          TIMESTAMPTZ,
    completed_at        TIMESTAMPTZ,
    records_fetched     INTEGER DEFAULT 0,
    records_created     INTEGER DEFAULT 0,
    records_updated     INTEGER DEFAULT 0,
    records_skipped     INTEGER DEFAULT 0,
    records_failed      INTEGER DEFAULT 0,
    error_message       TEXT,
    error_details       JSONB,
    retry_count         INTEGER DEFAULT 0,
    triggered_by        VARCHAR(100),
    config              JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ DEFAULT NOW(),
    updated_at          TIMESTAMPTZ DEFAULT NOW()
);

-- Sync logs
CREATE TABLE sync_job_logs (
    id              BIGSERIAL PRIMARY KEY,
    job_id          BIGINT NOT NULL REFERENCES sync_jobs(id),
    level           log_level NOT NULL DEFAULT 'info',
    message         TEXT NOT NULL,
    details         JSONB,
    record_id       VARCHAR(255),
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Quick Start

### 1. Run Migrations
```bash
psql -d permit_db -f migrations/001_create_enums.sql
psql -d permit_db -f migrations/002_create_geographic.sql
# ... etc
```

### 2. Seed Initial Cities
```sql
INSERT INTO states (code, name) VALUES ('IL', 'Illinois'), ('WA', 'Washington'), ('CA', 'California');
INSERT INTO cities (state_id, name, slug) VALUES 
  (1, 'Chicago', 'chicago'),
  (2, 'Seattle', 'seattle'),
  (3, 'Los Angeles', 'los_angeles');
```

### 3. Configure Data Source
```sql
INSERT INTO data_sources (city_id, name, source_type, base_url, dataset_id, field_mappings)
VALUES (1, 'Chicago Building Permits', 'socrata', 'https://data.cityofchicago.org', 'ydr8-5enu',
  '{"permit_number": "permit_", "issued_date": "issue_date", "description": "work_description"}'
);
```

### 4. Create Partition & Start Syncing
```sql
SELECT create_city_partition(1, 'chicago');
-- Run ingestion pipeline
```

---

## Summary

This database design handles schema diversity through:

1. **Canonical schema** - Core fields common to all sources
2. **Field mappings** - Per-source configuration in `data_sources.field_mappings`
3. **JSONB columns** - `raw_data` preserves everything, `extra_fields` stores unmapped fields
4. **Value transformers** - Date, currency, and status normalization during ingestion
5. **Partitioning** - City-based partitions for scalability and maintenance
