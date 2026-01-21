# City Permits API Guide - Fetching Available Building Permits

This guide explains how to use the Socrata Open Data API endpoints from `domain.json` to fetch building permit data from various cities across the US and Canada.

---

## Table of Contents

1. [Overview](#overview)
2. [Available Cities & Endpoints](#available-cities--endpoints)
3. [How to Fetch Permit Data](#how-to-fetch-permit-data)
4. [API Query Examples](#api-query-examples)
5. [Common Query Parameters](#common-query-parameters)
6. [Field Reference by City](#field-reference-by-city)
7. [Pagination](#pagination)
8. [Rate Limits & Best Practices](#rate-limits--best-practices)

---

## Overview

The `domain.json` file contains metadata for **4,410 permit-related datasets** from the Socrata Open Data platform. These datasets cover building permits, development permits, film permits, and more across multiple municipalities.

### Base API Format

```
https://{domain}/resource/{dataset_id}.json
```

---

## Available Cities & Endpoints

### Primary Building Permit Datasets

| City/Region | Domain | Dataset ID | Dataset Name | Update Frequency |
|-------------|--------|------------|--------------|------------------|
| **Chicago, IL** | `data.cityofchicago.org` | `ydr8-5enu` | Building Permits | Daily |
| **Calgary, AB** | `data.calgary.ca` | `c2es-76ed` | Building Permits | Daily |
| **New York City, NY** | `data.cityofnewyork.us` | `tg4x-b46p` | Film Permits | Daily |
| **Fort Worth, TX** | Data Portal | `quz7-xnsy` | Development Permits | Monthly |
| **Seattle, WA** | `cos-data.seattle.gov` | `76t5-zqzr` | Building Permits | Daily |
| **Los Angeles, CA** | `data.lacity.org` | `cpkv-aajs` | Building Permits: New Housing Units | Weekly |
| **San Diego County, CA** | `internal-sandiegocounty.data.socrata.com` | `dyzh-7eat` | Building Permits | Weekly |
| **Marin County, CA** | `data.marincounty.gov` | `nits-hbvx` | Building Permits Report | Daily |
| **Gainesville, FL** | `data.cityofgainesville.org` | `p798-x3nx` | Building Permits | - |
| **Framingham, MA** | `data.framinghamma.gov` | `2vzw-yean` | Building Permits | - |
| **Mesa, AZ** | `data.mesaaz.gov` | `2gkz-7z4f` | Building Permits (RETIRED) | - |
| **Cook County, IL** | `datacatalog.cookcountyil.gov` | `6yjf-dfxs` | Assessor - Permits | Monthly |
| **Winnipeg, MB** | `data.winnipeg.ca` | `it4w-cpf4` | Detailed Building Permit Data | - |
| **Delaware (State)** | `data.delaware.gov` | `2655-qn8j` | Well Permits | Daily |
| **Delaware (State)** | `data.delaware.gov` | `mv7j-tx3u` | Permitted Septic Systems | Monthly |
| **Montgomery County, MD** | `data.montgomerycountymd.gov` | `djk9-h36c` | Antenna/Wireless Permits | Daily |

---

## How to Fetch Permit Data

### Method 1: Direct API Request (JSON)

```bash
# Basic request to fetch permits from Chicago
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json"

# Fetch with limit
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json?$limit=100"
```

### Method 2: Using JavaScript/Fetch

```javascript
async function fetchPermits(domain, datasetId, params = {}) {
  const baseUrl = `https://${domain}/resource/${datasetId}.json`;
  const queryString = new URLSearchParams(params).toString();
  const url = queryString ? `${baseUrl}?${queryString}` : baseUrl;
  
  const response = await fetch(url);
  return await response.json();
}

// Example: Fetch Chicago Building Permits
const chicagoPermits = await fetchPermits(
  'data.cityofchicago.org', 
  'ydr8-5enu',
  { '$limit': 100 }
);
```

### Method 3: Using Python/Requests

```python
import requests

def fetch_permits(domain, dataset_id, params=None):
    """Fetch permit data from Socrata API."""
    url = f"https://{domain}/resource/{dataset_id}.json"
    response = requests.get(url, params=params)
    response.raise_for_status()
    return response.json()

# Example: Fetch Seattle Building Permits
seattle_permits = fetch_permits(
    domain='cos-data.seattle.gov',
    dataset_id='76t5-zqzr',
    params={'$limit': 100}
)
```

### Method 4: Using Sodapy (Official Socrata Python Client)

```python
from sodapy import Socrata

# No authentication needed for public datasets
client = Socrata("data.cityofchicago.org", None)

# Fetch first 2000 results (Sodapy default is 1000)
results = client.get("ydr8-5enu", limit=2000)

# Convert to DataFrame
import pandas as pd
df = pd.DataFrame.from_records(results)
```

---

## API Query Examples

### 1. Fetch All Permits from a City

```bash
# Chicago - All Building Permits
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json"

# Seattle - All Building Permits
curl "https://cos-data.seattle.gov/resource/76t5-zqzr.json"

# Calgary - All Building Permits
curl "https://data.calgary.ca/resource/c2es-76ed.json"
```

### 2. Filter by Date Range

```bash
# Chicago - Permits issued after Jan 1, 2025
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json?$where=issue_date>'2025-01-01'"

# Seattle - Permits applied after Jan 1, 2024
curl "https://cos-data.seattle.gov/resource/76t5-zqzr.json?$where=applieddate>'2024-01-01'"
```

### 3. Filter by Permit Type

```bash
# Chicago - Only PERMIT_NEW permits
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json?permit_type=PERMIT_NEW"

# Calgary - Only Commercial permits
curl "https://data.calgary.ca/resource/c2es-76ed.json?permitclassgroup=Commercial"
```

### 4. Search by Address/Location

```bash
# Chicago - Permits on a specific street
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json?$where=street_name='MICHIGAN'"

# Seattle - Permits in a specific zip code
curl "https://cos-data.seattle.gov/resource/76t5-zqzr.json?originalzip=98101"
```

### 5. Select Specific Fields

```bash
# Chicago - Only get permit number, issue date, and address
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json?$select=permit_,issue_date,street_number,street_name"

# Seattle - Get permit number, description, and cost
curl "https://cos-data.seattle.gov/resource/76t5-zqzr.json?$select=permitnum,description,estprojectcost"
```

### 6. Sort Results

```bash
# Chicago - Sort by issue date (newest first)
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json?$order=issue_date DESC"

# Calgary - Sort by estimated project cost
curl "https://data.calgary.ca/resource/c2es-76ed.json?$order=estprojectcost DESC"
```

### 7. Combined Query Example

```bash
# Chicago - New permits in 2025, sorted by date, with pagination
curl "https://data.cityofchicago.org/resource/ydr8-5enu.json?\
$where=issue_date>'2025-01-01' AND permit_type='PERMIT_NEW'\
&$order=issue_date DESC\
&$limit=50\
&$offset=0"
```

---

## Common Query Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `$limit` | Maximum rows to return (default: 1000) | `$limit=500` |
| `$offset` | Skip first N rows (for pagination) | `$offset=100` |
| `$where` | Filter with SoQL conditions | `$where=issue_date>'2024-01-01'` |
| `$order` | Sort by column(s) | `$order=issue_date DESC` |
| `$select` | Return only specific columns | `$select=permit_,issue_date` |
| `$q` | Full-text search | `$q=renovation` |
| `$group` | Group results by column | `$group=permit_type` |

---

## Field Reference by City

### Chicago Building Permits (`ydr8-5enu`)

| Field | Type | Description |
|-------|------|-------------|
| `id` | Text | Unique database record identifier |
| `permit_` | Text | Permit tracking number |
| `permit_type` | Text | Type of permit |
| `issue_date` | Date | Date permit was issued |
| `street_number` | Number | Address number |
| `street_direction` | Text | Street direction (N/S/E/W) |
| `street_name` | Text | Street name |
| `work_description` | Text | Description of work authorized |
| `work_type` | Text | Classification of activity |
| `reported_cost` | Number | Estimated cost of work |
| `total_fee` | Number | Total permit fees |
| `community_area` | Number | Chicago community area |
| `ward` | Number | City council ward |
| `latitude` | Number | Property latitude |
| `longitude` | Number | Property longitude |

### Seattle Building Permits (`76t5-zqzr`)

| Field | Type | Description |
|-------|------|-------------|
| `permitnum` | Text | Permit tracking number |
| `permitclass` | Text | Type of project |
| `permitclassmapped` | Text | Residential or non-residential |
| `permittypemapped` | Text | Permit category |
| `description` | Text | Work description |
| `estprojectcost` | Number | Estimated project cost |
| `applieddate` | Date | Application date |
| `issueddate` | Date | Issue date |
| `completeddate` | Date | Completion date |
| `statuscurrent` | Text | Current status |
| `originaladdress1` | Text | Street address |
| `originalzip` | Text | Zip code |
| `housingunits` | Number | Number of housing units |
| `latitude` | Number | Property latitude |
| `longitude` | Number | Property longitude |

### Calgary Building Permits (`c2es-76ed`)

| Field | Type | Description |
|-------|------|-------------|
| `permitnum` | Text | Permit number |
| `permittype` | Text | Type of permit |
| `permitclass` | Text | Permit class |
| `workclass` | Text | Type of work |
| `description` | Text | Work description |
| `estprojectcost` | Number | Estimated project cost |
| `applieddate` | Date | Application date |
| `issueddate` | Date | Issue date |
| `completeddate` | Date | Completion date |
| `statuscurrent` | Text | Current status |
| `communityname` | Text | Calgary community |
| `originaladdress` | Text | Property address |
| `housingunits` | Number | Number of housing units |
| `latitude` | Number | Property latitude |
| `longitude` | Number | Property longitude |

---

## Pagination

### Fetching All Records

To fetch all records, use pagination with `$limit` and `$offset`:

```python
import requests

def fetch_all_permits(domain, dataset_id, batch_size=1000):
    """Fetch all records using pagination."""
    all_records = []
    offset = 0
    
    while True:
        url = f"https://{domain}/resource/{dataset_id}.json"
        params = {
            '$limit': batch_size,
            '$offset': offset,
            '$order': ':id'  # Consistent ordering for pagination
        }
        
        response = requests.get(url, params=params)
        data = response.json()
        
        if not data:
            break
            
        all_records.extend(data)
        offset += batch_size
        
        print(f"Fetched {len(all_records)} records...")
    
    return all_records

# Example: Fetch all Seattle permits
all_seattle = fetch_all_permits('cos-data.seattle.gov', '76t5-zqzr')
```

---

## Rate Limits & Best Practices

### Rate Limits
- **Without App Token**: 1,000 requests/hour per IP
- **With App Token**: 10,000+ requests/hour (varies by dataset owner)

### Registering an App Token (Recommended)

1. Create account at [Socrata Developer Portal](https://dev.socrata.com/)
2. Register an application
3. Add to requests: `?$$app_token=YOUR_TOKEN`

```python
params = {
    '$$app_token': 'YOUR_APP_TOKEN',
    '$limit': 1000
}
response = requests.get(url, params=params)
```

### Best Practices

1. **Use `$select`** to only fetch fields you need
2. **Use `$where`** to filter at the API level, not client-side
3. **Paginate** large requests to avoid timeouts
4. **Cache** responses when appropriate
5. **Handle errors** gracefully (429, 500, etc.)

### Error Handling Example

```python
import requests
import time

def fetch_with_retry(url, params, max_retries=3):
    """Fetch with exponential backoff retry."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, params=params, timeout=30)
            
            if response.status_code == 429:  # Rate limited
                wait_time = 2 ** attempt
                print(f"Rate limited. Waiting {wait_time}s...")
                time.sleep(wait_time)
                continue
                
            response.raise_for_status()
            return response.json()
            
        except requests.exceptions.Timeout:
            print(f"Timeout on attempt {attempt + 1}")
            continue
            
    raise Exception("Max retries exceeded")
```

---

## Quick Reference - All City Endpoints

```python
CITY_ENDPOINTS = {
    'chicago': {
        'domain': 'data.cityofchicago.org',
        'dataset_id': 'ydr8-5enu',
        'name': 'Building Permits'
    },
    'seattle': {
        'domain': 'cos-data.seattle.gov',
        'dataset_id': '76t5-zqzr',
        'name': 'Building Permits'
    },
    'calgary': {
        'domain': 'data.calgary.ca',
        'dataset_id': 'c2es-76ed',
        'name': 'Building Permits'
    },
    'los_angeles': {
        'domain': 'data.lacity.org',
        'dataset_id': 'cpkv-aajs',
        'name': 'Building Permits: New Housing Units'
    },
    'new_york': {
        'domain': 'data.cityofnewyork.us',
        'dataset_id': 'tg4x-b46p',
        'name': 'Film Permits'
    },
    'marin_county': {
        'domain': 'data.marincounty.gov',
        'dataset_id': 'nits-hbvx',
        'name': 'Building Permits Report'
    },
    'gainesville': {
        'domain': 'data.cityofgainesville.org',
        'dataset_id': 'p798-x3nx',
        'name': 'Building Permits'
    },
    'cook_county': {
        'domain': 'datacatalog.cookcountyil.gov',
        'dataset_id': '6yjf-dfxs',
        'name': 'Assessor - Permits'
    },
    'winnipeg': {
        'domain': 'data.winnipeg.ca',
        'dataset_id': 'it4w-cpf4',
        'name': 'Detailed Building Permit Data'
    },
    'delaware_wells': {
        'domain': 'data.delaware.gov',
        'dataset_id': '2655-qn8j',
        'name': 'Well Permits'
    },
    'montgomery_county': {
        'domain': 'data.montgomerycountymd.gov',
        'dataset_id': 'djk9-h36c',
        'name': 'Antenna/Wireless Permits'
    }
}

def fetch_city_permits(city_key, params=None):
    """Fetch permits for a specific city."""
    if city_key not in CITY_ENDPOINTS:
        raise ValueError(f"Unknown city: {city_key}")
    
    config = CITY_ENDPOINTS[city_key]
    url = f"https://{config['domain']}/resource/{config['dataset_id']}.json"
    
    response = requests.get(url, params=params or {})
    response.raise_for_status()
    return response.json()

# Usage
permits = fetch_city_permits('chicago', {'$limit': 100})
```

---

## Need Help?

- **Socrata API Documentation**: [dev.socrata.com](https://dev.socrata.com/)
- **SoQL Query Language**: [dev.socrata.com/docs/queries/](https://dev.socrata.com/docs/queries/)
- **Dataset Discovery**: [api.us.socrata.com/api/catalog/v1](https://api.us.socrata.com/api/catalog/v1)
