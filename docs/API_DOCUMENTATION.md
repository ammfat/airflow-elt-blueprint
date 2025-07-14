# Weather & Climate Data Pipeline - API Documentation

## Overview

This project is a comprehensive data pipeline built with Apache Airflow that ingests, processes, and visualizes weather and climate data. The pipeline uses MinIO for object storage, DuckDB for data warehousing, and Streamlit for interactive reporting.

## Table of Contents

1. [Global Variables and Configuration](#global-variables-and-configuration)
2. [DAGs (Data Pipeline Workflows)](#dags-data-pipeline-workflows)
3. [Custom Operators](#custom-operators)
4. [Custom Task Groups](#custom-task-groups)
5. [Utility Functions](#utility-functions)
6. [Streamlit Application](#streamlit-application)
7. [Usage Examples](#usage-examples)

---

## Global Variables and Configuration

### Module: `include.global_variables.global_variables`

This module contains all configuration variables, connection settings, and utility functions used across the pipeline.

#### Configuration Variables

```python
# User Configuration
MY_NAME = "ammfat"           # Owner name for DAGs
MY_CITY = "Karawang"         # Default city for weather data

# MinIO Configuration
MINIO_ACCESS_KEY = "minioadmin"
MINIO_SECRET_KEY = "minioadmin"
MINIO_IP = "host.docker.internal:9000"
WEATHER_BUCKET_NAME = "weather"
CLIMATE_BUCKET_NAME = "climate"
ARCHIVE_BUCKET_NAME = "archive"

# DuckDB Configuration
DUCKDB_INSTANCE_NAME = "dwh"
WEATHER_IN_TABLE_NAME = "in_weather"
CLIMATE_TABLE_NAME = "temp_global_table"
REPORTING_TABLE_NAME = "reporting_table"
CONN_ID_DUCKDB = "duckdb_default"
```

#### Datasets

Airflow datasets for data lineage and scheduling:

```python
DS_CLIMATE_DATA_MINIO = Dataset(f"minio://{CLIMATE_BUCKET_NAME}")
DS_WEATHER_DATA_MINIO = Dataset(f"minio://{WEATHER_BUCKET_NAME}")
DS_DUCKDB_IN_WEATHER = Dataset("duckdb://in_weather")
DS_DUCKDB_IN_CLIMATE = Dataset("duckdb://in_climate")
DS_DUCKDB_REPORTING = Dataset("duckdb://reporting")
DS_START = Dataset("start")
```

#### Utility Functions

##### `get_minio_client()`

Returns a configured MinIO client instance.

**Returns:** `Minio` - Configured MinIO client

**Example:**
```python
from include.global_variables import global_variables as gv

client = gv.get_minio_client()
buckets = client.list_buckets()
```

---

## DAGs (Data Pipeline Workflows)

### 1. Start DAG (`dags/start.py`)

**Purpose:** Initializes the pipeline by creating DuckDB connection pool and triggering downstream workflows.

**Schedule:** `@once` (manual trigger after initial setup)

**Key Tasks:**
- `create_duckdb_pool`: Creates Airflow connection pool for DuckDB

**Usage:**
```bash
# Trigger the start DAG to begin the pipeline
airflow dags trigger start
```

### 2. Climate Data Ingestion (`dags/ingestion/in_climate_data.py`)

**Purpose:** Ingests climate data from local CSV files to MinIO storage.

**Schedule:** Triggered by `DS_START` dataset

**Key Tasks:**
- `create_climate_bucket`: Creates MinIO bucket for climate data
- `ingest_climate_data`: Uploads CSV files to MinIO

**File Processing:**
- Processes `temp_global.csv` containing global temperature data

### 3. Local Weather Ingestion (`dags/ingestion/in_local_weather.py`)

**Purpose:** Fetches current weather data from API and stores in MinIO.

**Schedule:** Triggered by `DS_START` dataset

**Key Tasks:**
- `get_lat_long_for_city`: Converts city name to coordinates
- `get_current_weather`: Fetches weather data from Open-Meteo API
- `save_weather_data`: Stores weather data in MinIO

**Example API Response:**
```json
{
    "city": "Karawang",
    "lat": -6.3015,
    "long": 107.3025,
    "current_weather": {
        "temperature": 28.5,
        "windspeed": 5.2,
        "winddirection": 180,
        "weathercode": 0,
        "time": "2023-01-01T12:00"
    },
    "API_response": 200
}
```

### 4. Data Loading (`dags/load/load_data.py`)

**Purpose:** Loads data from MinIO to DuckDB for analysis.

**Schedule:** Triggered by both climate and weather data availability

**Key Tasks:**
- `load_climate_data`: Loads climate CSV into DuckDB table
- `load_weather_data`: Loads weather JSON into DuckDB table
- `archive_files`: Moves processed files to archive bucket

**Table Schemas:**

**Weather Table (`in_weather`):**
```sql
CREATE TABLE in_weather (
    city VARCHAR,
    lat DOUBLE,
    long DOUBLE,
    temperature DOUBLE,
    windspeed DOUBLE,
    winddirection DOUBLE,
    weathercode INTEGER,
    timestamp TIMESTAMP
);
```

**Climate Table (`temp_global_table`):**
```sql
CREATE TABLE temp_global_table (
    dt DATE,
    LandAverageTemperature DOUBLE,
    LandAverageTemperatureUncertainty DOUBLE,
    Country VARCHAR
);
```

### 5. Data Transformation (`dags/transform/create_reporting_table.py`)

**Purpose:** Creates aggregated reporting table using Astro SDK.

**Schedule:** Triggered by both weather and climate data in DuckDB

**Key Transformation:**
```sql
SELECT CAST(dt AS DATE) AS date, 
    AVG(LandAverageTemperature) OVER(PARTITION BY YEAR(CAST(dt AS DATE))/10*10) AS decade_average_temp,
    AVG(LandAverageTemperature) OVER(PARTITION BY YEAR(CAST(dt AS DATE))) AS year_average_temp,
    AVG(LandAverageTemperature) OVER(PARTITION BY MONTH(CAST(dt AS DATE))) AS month_average_temp,
    AVG(LandAverageTemperature) OVER(PARTITION BY CAST(dt AS DATE)) AS day_average_temp
FROM in_climate
```

### 6. Streamlit Report (`dags/report/run_streamlit_app.py`)

**Purpose:** Launches interactive Streamlit dashboard.

**Schedule:** Triggered by reporting table availability

**Key Features:**
- Runs Streamlit app with custom environment variables
- Sets up city coordinates from Airflow Variables
- Configures app for container environment

---

## Custom Operators

### MinIO Operators (`include/custom_operators/minio.py`)

#### MinIOHook

Base hook for MinIO operations.

**Parameters:**
- `minio_conn_id` (str): Airflow connection ID (default: "minio_default")
- `secure_connection` (bool): Use HTTPS (default: False)

**Methods:**

##### `put_object(bucket_name, object_name, data, length, part_size)`
Uploads data to MinIO bucket.

##### `list_objects(bucket_name, prefix="")`
Lists objects in a bucket with optional prefix filter.

##### `copy_object(source_bucket_name, source_object_name, dest_bucket_name, dest_object_name)`
Copies object between buckets.

##### `delete_objects(bucket_name, object_names, bypass_governance_mode=False)`
Deletes one or more objects from bucket.

#### LocalFilesystemToMinIOOperator

Uploads local files or JSON data to MinIO.

**Parameters:**
- `bucket_name` (str): Target bucket name
- `object_name` (str): Object key in bucket
- `local_file_path` (str, optional): Path to local file
- `json_serializeable_information` (dict, optional): JSON data to upload
- `minio_conn_id` (str): MinIO connection ID
- `length` (int): Data size (-1 for unknown)
- `part_size` (int): Multipart upload chunk size

**Example:**
```python
upload_task = LocalFilesystemToMinIOOperator(
    task_id="upload_weather_data",
    bucket_name="weather",
    object_name="current_weather.json",
    json_serializeable_information=weather_data,
    minio_conn_id="minio_default"
)
```

#### MinIOListOperator

Lists objects in a MinIO bucket.

**Parameters:**
- `bucket_name` (str): Bucket to list
- `prefix` (str): Object prefix filter
- `minio_conn_id` (str): MinIO connection ID

**Example:**
```python
list_task = MinIOListOperator(
    task_id="list_weather_files",
    bucket_name="weather",
    prefix="2023/",
    minio_conn_id="minio_default"
)
```

#### MinIOCopyObjectOperator

Copies objects between MinIO buckets.

**Parameters:**
- `source_bucket_name` (str): Source bucket
- `source_object_names` (str|list): Source object name(s)
- `dest_bucket_name` (str): Destination bucket
- `dest_object_names` (str|list): Destination object name(s)
- `minio_conn_id` (str): MinIO connection ID

#### MinIODeleteObjectsOperator

Deletes objects from MinIO bucket.

**Parameters:**
- `bucket_name` (str): Target bucket
- `object_names` (str|list): Object name(s) to delete
- `minio_conn_id` (str): MinIO connection ID
- `bypass_governance_mode` (bool): Force deletion

---

## Custom Task Groups

### CreateBucket (`include/custom_task_groups/create_bucket.py`)

Reusable task group that creates MinIO bucket if it doesn't exist.

**Parameters:**
- `task_id` (str): Task group identifier
- `bucket_name` (str): Name of bucket to create
- `task_group` (TaskGroup, optional): Parent task group

**Internal Tasks:**
1. `list_buckets_minio`: Lists existing buckets
2. `decide_whether_to_create_bucket`: Branching logic
3. `create_bucket`: Creates bucket if needed
4. `bucket_already_exists`: No-op if bucket exists
5. `bucket_exists`: Final synchronization task

**Example:**
```python
bucket_tg = CreateBucket(
    task_id="create_weather_bucket",
    bucket_name="weather"
)
```

**Task Flow:**
```
list_buckets_minio >> decide_whether_to_create_bucket >> [create_bucket | bucket_already_exists] >> bucket_exists
```

---

## Utility Functions

### Meteorology Utils (`include/meterology_utils.py`)

#### `get_lat_long_for_cityname(city: str)`

Converts city name to geographical coordinates using Nominatim geocoding.

**Parameters:**
- `city` (str): City name to geocode

**Returns:**
- `dict`: Contains 'city', 'lat', 'long' keys

**Example:**
```python
from include.meterology_utils import get_lat_long_for_cityname

coordinates = get_lat_long_for_cityname("Karawang")
# Returns: {"city": "Karawang", "lat": -6.3015, "long": 107.3025}
```

**Error Handling:**
- Returns 'NA' for lat/long if geocoding fails
- Logs warnings for failed geocoding attempts

#### `get_current_weather_from_city_coordinates(coordinates, timestamp)`

Fetches current weather data from Open-Meteo API.

**Parameters:**
- `coordinates` (dict): Dictionary with 'lat', 'long', 'city' keys
- `timestamp` (str): Timestamp for the request

**Returns:**
- `dict`: Weather data with API response status

**Example:**
```python
from include.meterology_utils import get_current_weather_from_city_coordinates

coordinates = {"city": "Karawang", "lat": -6.3015, "long": 107.3025}
weather = get_current_weather_from_city_coordinates(coordinates, "2023-01-01")

# Returns:
{
    "city": "Karawang",
    "lat": -6.3015,
    "long": 107.3025,
    "current_weather": {
        "temperature": 28.5,
        "windspeed": 5.2,
        "winddirection": 180,
        "weathercode": 0,
        "time": "2023-01-01T12:00"
    },
    "API_response": 200
}
```

**Error Handling:**
- Returns NULL values for weather data if API fails
- Logs warnings with error details and status codes

---

## Streamlit Application

### Weather vs Climate App (`include/streamlit_app/weather_v_climate_app.py`)

Interactive dashboard comparing local weather with global climate trends.

#### Key Functions

##### `get_city_data(city, db)`

Retrieves local weather data for a specific city from DuckDB.

**Parameters:**
- `city` (str): City name to query
- `db` (str): Path to DuckDB database

**Returns:**
- `pandas.DataFrame`: Weather data for the city

##### `get_global_surface_temp_data(db)`

Retrieves global surface temperature data from the reporting table.

**Parameters:**
- `db` (str): Path to DuckDB database

**Returns:**
- `pandas.DataFrame`: Global temperature trends

##### `get_chart(data, grain)`

Creates Altair charts for temperature visualization.

**Parameters:**
- `data` (pandas.DataFrame): Temperature data
- `grain` (str): Time granularity ('decade', 'year', 'month')

**Returns:**
- `altair.Chart`: Interactive chart object

#### App Features

1. **City Weather Display**: Shows current weather for configured city
2. **Global Temperature Trends**: Displays long-term climate data
3. **Interactive Charts**: Multiple time granularities (decade, year, month)
4. **Responsive Design**: Optimized for web display

#### Configuration

The app uses environment variables:
- `my_city`: Target city name
- `my_name`: User identifier
- `city_coordinates`: JSON string with coordinates

---

## Usage Examples

### 1. Setting Up the Pipeline

```bash
# 1. Start the pipeline
airflow dags trigger start

# 2. Monitor DAG execution
airflow dags list
airflow tasks list start

# 3. Check dataset status
airflow datasets list
```

### 2. Custom Weather Data Ingestion

```python
# Create a custom weather ingestion task
from include.meterology_utils import get_lat_long_for_cityname, get_current_weather_from_city_coordinates
from include.custom_operators.minio import LocalFilesystemToMinIOOperator

@task
def fetch_custom_weather(city_name):
    # Get coordinates
    coords = get_lat_long_for_cityname(city_name)
    
    # Fetch weather
    weather = get_current_weather_from_city_coordinates(coords, "2023-01-01")
    
    return weather

# Upload to MinIO
upload_weather = LocalFilesystemToMinIOOperator(
    task_id="upload_custom_weather",
    bucket_name="weather",
    object_name=f"{city_name}_weather.json",
    json_serializeable_information="{{ ti.xcom_pull(task_ids='fetch_custom_weather') }}"
)

fetch_custom_weather("Jakarta") >> upload_weather
```

### 3. Custom Data Transformation

```python
from astro import sql as aql
from astro.sql.table import Table

@aql.transform
def custom_climate_analysis(climate_table: Table):
    return """
        SELECT 
            YEAR(CAST(dt AS DATE)) as year,
            AVG(LandAverageTemperature) as avg_temp,
            MAX(LandAverageTemperature) as max_temp,
            MIN(LandAverageTemperature) as min_temp
        FROM {{ climate_table }}
        WHERE YEAR(CAST(dt AS DATE)) >= 2000
        GROUP BY YEAR(CAST(dt AS DATE))
        ORDER BY year
    """

# Use in DAG
climate_analysis = custom_climate_analysis(
    climate_table=Table(name="temp_global_table", conn_id="duckdb_default"),
    output_table=Table(name="climate_analysis", conn_id="duckdb_default")
)
```

### 4. Running Streamlit App Locally

```bash
# Navigate to app directory
cd include/streamlit_app

# Set environment variables
export my_city="YourCity"
export my_name="YourName"
export city_coordinates='{"lat": 0.0, "long": 0.0}'

# Run the app
streamlit run weather_v_climate_app.py
```

### 5. Testing MinIO Operations

```python
from include.global_variables import global_variables as gv

# Test MinIO connection
client = gv.get_minio_client()

# List buckets
buckets = client.list_buckets()
print([bucket.name for bucket in buckets])

# List objects in bucket
objects = client.list_objects("weather")
for obj in objects:
    print(f"Object: {obj.object_name}, Size: {obj.size}")
```

### 6. Custom Bucket Creation

```python
from include.custom_task_groups.create_bucket import CreateBucket

# Create custom bucket task group
create_my_bucket = CreateBucket(
    task_id="create_my_custom_bucket",
    bucket_name="my-custom-data"
)

# Use in DAG
@dag(...)
def my_custom_dag():
    bucket_creation = create_my_bucket
    
    # Follow with other tasks
    bucket_creation >> other_tasks
```

---

## Error Handling and Logging

### Logging

All components use Airflow's task logger:

```python
from include.global_variables import global_variables as gv

# Log information
gv.task_log.info("Processing completed successfully")

# Log warnings
gv.task_log.warn("API returned non-200 status code")

# Log errors
gv.task_log.error("Failed to connect to database")
```

### Common Error Patterns

1. **MinIO Connection Issues**: Check `MINIO_IP` and credentials
2. **API Rate Limiting**: Open-Meteo API may throttle requests
3. **DuckDB Lock Issues**: Use connection pools for concurrent access
4. **Geocoding Failures**: Handle cases where city names aren't found

### Retry Configuration

DAGs are configured with automatic retries:

```python
default_args = {
    "owner": MY_NAME,
    "depends_on_past": False,
    "retries": 2,
    "retry_delay": duration(minutes=5),
}
```

---

## Dependencies and Requirements

### Python Packages

```txt
duckdb==0.7.0
streamlit==1.18.0
geopy==2.3.0
minio==7.1.13
pandas==1.5.3
airflow-provider-duckdb==0.0.2
astro-sdk-python==1.5.0
```

### External Services

1. **MinIO**: Object storage server
2. **DuckDB**: Analytical database
3. **Open-Meteo API**: Weather data source
4. **Nominatim**: Geocoding service

### Docker Configuration

The project includes Docker setup with:
- Airflow with custom operators
- MinIO server
- DuckDB database
- Streamlit application

---

This documentation covers all public APIs, functions, and components in the Weather & Climate Data Pipeline. Each component includes usage examples, parameter descriptions, and error handling guidance.