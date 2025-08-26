# 🌍 Free City Guide (Pilot) – POI Management System

## Overview

The **Free City Guide** is a global tourism initiative designed for deployment on interactive information desks in major tourist cities, this pilot project launches in **Phoenix** to help travelers discover Points of Interest (POIs) quickly and intuitively.

This system is backed by a normalized, relational **MS SQL Server (v15+)** database and supports dynamic spatial querying via GeoJSON and WKT standards.

---

## 🚀 Project Objectives

- Load and normalize POI data from a provided CSV source.
- Store POI and geographic metadata in a relational schema.
- Enable POI discovery using flexible, user-defined filters.
- Return filtered POI data in compressed **GeoJSON** format.

---

## 🧱 Data Model

ER diagram attached (pilot - dbo.png)

<details>
  <summary>Details here...</summary>

### 🔴 `fact_poi` – Core POI Table

| Column                | Description                                   |
|------------------------|-----------------------------------------------|
| `poi_id`              | Surrogate key for POI                          |
| `poi_external_id`     | External POI ID from source                    |
| `poi_location_name`   | Name of the location                           |
| `poi_geopoint`        | Coordinates (GeoPoint)                         |
| `parent_poi_id`       | Reference to parent POI                        |
| `brand_idfk`          | Foreign key to `dim_brand`                     |
| `subcategory_idfk`    | Foreign key to `dim_subcategory`               |
| `postal_code_idfk`    | Foreign key to `dim_postal_code`               |
| `polygon_idfk`        | Foreign key to `dim_polygon`                   |
| `operation_hours_idfk`| Foreign key to `dim_operation_hours`           |

---

### 🟩 Dimension Tables

#### `dim_country`
| Column         | Description                   |
|----------------|-------------------------------|
| `country_id`   | Surrogate key                 |
| `country_code` | ISO 2-letter country code     |

#### `dim_region`
| Column          | Description                 |
|------------------|-----------------------------|
| `region_id`      | Surrogate key               |
| `country_idfk`   | Foreign key to country      |
| `region_code`    | Region/state code (e.g., AZ)|

#### `dim_city`
| Column       | Description          |
|--------------|----------------------|
| `city_id`    | Surrogate key        |
| `region_idfk`| Foreign key to region|
| `city_name`  | City name            |

#### `dim_postal_code`
| Column           | Description                |
|------------------|----------------------------|
| `postal_code_id` | Surrogate key              |
| `city_idfk`      | Foreign key to city        |
| `postal_code`    | Postal or ZIP code         |

#### `dim_brand`
| Column             | Description               |
|--------------------|---------------------------|
| `brand_id`         | Surrogate key             |
| `brand_external_id`| Brand ID from source data |
| `brand_name`       | Name of the brand         |

#### `dim_category`
| Column           | Description           |
|------------------|-----------------------|
| `category_id`    | Surrogate key         |
| `category_name`  | Top-level category    |

#### `dim_subcategory`
| Column           | Description                 |
|------------------|-----------------------------|
| `subcategory_id` | Surrogate key               |
| `category_idfk`  | Foreign key to category     |
| `subcategory_name` | Optional subcategory name |

#### `dim_category_tag`
| Column            | Description     |
|-------------------|-----------------|
| `category_tag_id` | Surrogate key   |
| `category_tag_name` | Tag used in UI|

#### `dim_category_tag_lnk`
| Column               | Description              |
|----------------------|--------------------------|
| `category_tag_lnk_id`| Surrogate key            |
| `poi_idfk`           | Foreign key to POI       |
| `category_tag_idfk`  | Foreign key to tag       |

#### `dim_polygon`
| Column             | Description                |
|--------------------|----------------------------|
| `polygon_id`       | Surrogate key              |
| `polygon_value`    | WKT geometry               |
| `polygon`          | Native SQL geometry        |
| `polygon_value_hash` | Geometry hash value      |

#### `dim_operation_hours`
| Column                | Description                    |
|------------------------|--------------------------------|
| `operation_hours_id`   | Surrogate key                 |
| `operation_hours_value`| Hours stored as JSON          |

#### `vw_poi_geojson`
- View used to transform POI records into **compressed GeoJSON** format for client use.

---

### 🧾 `source_file` (Staging Table)

A flat staging table that mirrors the source CSV. Used for data import and transformation.

---
</details>

## 🧠 Stored Procedure: `sp_GetFilteredPOIsFromJson`

This procedure filters POIs based on user-defined criteria provided in a single **JSON input parameter**. The result is returned in **GeoJSON** format, compressed for client use.

### 🔍 Search Criteria (All Optional)

- `country` (ISO code, e.g., "US")
- `region` (e.g., "AZ")
- `city` (e.g., "Phoenix")
- `location` (lat/lon + radius in meters)
- `polygon_wkt` (user-defined polygon in WKT format)
- `category` (e.g., "Grocery Stores")
- `location_name` (partial text match)

If **no filters** are provided, the procedure returns all POIs within **200 meters** of a default location.

### 📤 Output Fields

| Field             | Description                          |
|------------------|--------------------------------------|
| `poi_id`         | POI ID                                |
| `parent_id`      | Parent POI (if any)                   |
| `country_code`   | ISO country code                      |
| `region_code`    | Region/state code                     |
| `city_name`      | City name                             |
| `latitude`       | Latitude of the POI                   |
| `longitude`      | Longitude of the POI                  |
| `category`       | Top-level category                    |
| `subcategory`    | Subcategory (if any)                  |
| `polygon_wkt`    | WKT polygon (if exists)               |
| `location_name`  | Human-readable name of POI            |
| `postal_code`    | Postal or ZIP code                    |
| `operation_hours`| JSON-formatted hours (if exists)      |

### 🧪 Usage Example

```sql
DECLARE @output VARCHAR(MAX);
DECLARE @input VARCHAR(MAX) = 
'{
  "category": "Grocery Stores",
  "city": "Phoenix",
  "country": "US",
  "location": {
    "lat": 33.5031433105469,
    "lon": -112.143409729004,
    "radius": 5000
  },
  "location_name": "Stop",
  "polygon_wkt": "POLYGON ((...))",
  "region": "AZ"
}';

EXEC dbo.sp_GetFilteredPOIsFromJson 
    @in_json = @input, 
    @out_json = @output OUTPUT;

SELECT LEN(@output) AS geoJson_Length;
SELECT @output AS geoJson;

-- Optional workaround if result is truncated:
SELECT CAST('<![CDATA[' + @output + ']]>' AS XML) geoJson;
```

## Deployment Instructions for the `pilot` Database

### Preparation

1. **Ensure Required Files Are in Place**  
   Make sure the following files and catalogs exist on the server. Adjust paths if needed, or replicate the folder structure with appropriate permissions:

   - Backup file: `C:\POI\pilot.bak`
   - MDF (data) and LDF(log) catalog: `C:\POI\DATA`
   
2. **SQL Server Requirements**  
   - Target SQL Server instance must be **SQL Server 2019 (15.x)** or higher.

---

### Step-by-Step Deployment

1. **Connect to the SQL Server Instance**  
   - Use SQL Server Management Studio (SSMS) or any compatible SQL client.

2. **Execute the Following Script**  
   This script restores the `pilot` database from the backup and places the MDF and LDF files in the specified locations:

   ```sql
   USE [master];

   RESTORE DATABASE [pilot] 
   FROM DISK = N'C:\POI\pilot.bak' 
   WITH 
       FILE = 1,
       MOVE N'pilot' TO N'C:\POI\DATA\pilot.mdf',
       MOVE N'pilot_log' TO N'C:\POI\DATA\pilot_log.ldf',
       NOUNLOAD,
       STATS = 5;
   GO

### Alternative Recovery (If Issues Occur)

If restoring from the `.bak` file fails, you can recreate the `pilot` database using the provided `pilot.sql` script.

#### Steps:

1. Unarchive pilot.zip and open the `pilot.sql` script using SQL Server Management Studio (SSMS) or any other SQL client.
2. Locate the file path definitions (typically in the `ON PRIMARY` and `LOG ON` clauses).
3. Update the file paths if they differ from your environment.
4. Execute the script to create the `pilot` database manually.

> 💡 **Note:** Ensure that the SQL Server service account has appropriate read/write permissions to the specified folder locations.