# Module 3 Homework: Data Warehousing & BigQuery

This repository contains the solution for Module 3 of the Data Engineering Zoomcamp. It details the process of loading Yellow Taxi Trip Records (Jan–June 2024) into Google Cloud Storage (GCS) and analyzing them using BigQuery.

## 1. Data Loading (Python)

### Prerequisites
* Google Cloud SDK installed and authenticated.
* Python 3.x installed.
* `google-cloud-storage` library installed (`pip install google-cloud-storage`).

### The Script
Save the following code as `load_yellow_taxi_data.py`. This script downloads the Parquet files and uploads them to your GCS bucket.

```python
import os
import sys
import urllib.request
from concurrent.futures import ThreadPoolExecutor
from google.cloud import storage
import time

# UPDATE THIS TO YOUR BUCKET NAME
BUCKET_NAME = "yellow-taxi-2024-joy-hw3"

BASE_URL = "[https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-](https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-)"
MONTHS = [f"{i:02d}" for i in range(1, 7)]
DOWNLOAD_DIR = "."
CHUNK_SIZE = 8 * 1024 * 1024

os.makedirs(DOWNLOAD_DIR, exist_ok=True)

# Authenticate via: gcloud auth application-default login
client = storage.Client()

def download_file(month):
    url = f"{BASE_URL}{month}.parquet"
    file_path = os.path.join(DOWNLOAD_DIR, f"yellow_tripdata_2024-{month}.parquet")

    try:
        print(f"Downloading {url}...")
        urllib.request.urlretrieve(url, file_path)
        print(f"Downloaded: {file_path}")
        return file_path
    except Exception as e:
        print(f"Failed to download {url}: {e}")
        return None

def upload_to_gcs(file_path):
    bucket = client.bucket(BUCKET_NAME)
    blob_name = os.path.basename(file_path)
    blob = bucket.blob(blob_name)
    blob.chunk_size = CHUNK_SIZE
    
    print(f"Uploading {file_path}...")
    blob.upload_from_filename(file_path)
    print(f"Uploaded: {blob_name}")

if __name__ == "__main__":
    # 1. Download files
    with ThreadPoolExecutor(max_workers=4) as executor:
        file_paths = list(executor.map(download_file, MONTHS))

    # 2. Upload to GCS
    with ThreadPoolExecutor(max_workers=4) as executor:
        executor.map(upload_to_gcs, filter(None, file_paths))

    print("✅ All files processed and uploaded.")

# 2.  BigQuery Setup
Run the following SQL queries in the BigQuery Console to prepare your environment.

 Create External Table 
Creates a definition pointing to the files in GCS (no data movement).

```sql
CREATE OR REPLACE EXTERNAL TABLE `de-zoomcamp-hw3-486917.ny_taxi.external_yellow_tripdata`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://yellow-taxi-2024-joy-hw3/yellow_tripdata_2024-*.parquet']
);

Create Materialized Table
Creates a standard, non-partitioned table in BigQuery storage.

```sql
CREATE OR REPLACE TABLE `de-zoomcamp-hw3-486917.ny_taxi.yellow_tripdata_non_partitioned` AS
SELECT * FROM `de-zoomcamp-hw3-486917.ny_taxi.external_yellow_tripdata`;
3. Homework Questions & Answers
Question 1: What is count of records for the 2024 Yellow Taxi Data?
Answer: 20,332,093

Query:

SQL
SELECT COUNT(*) FROM `de-zoomcamp-hw3-486917.ny_taxi.yellow_tripdata_non_partitioned`;
Question 2: Estimated amount of data read (External vs. Materialized)
Answer: 0 MB for the External Table and 155.12 MB for the Materialized Table

External Table: BigQuery cannot estimate the size for external Parquet files (shows 0 MB).

Materialized Table: BigQuery knows the exact size of the PULocationID column (~155 MB).

Query for External Table:

```sql
SELECT COUNT(DISTINCT PULocationID) FROM `de-zoomcamp-hw3-486917.ny_taxi.external_yellow_tripdata`;
Query for Materialized Table:

```sql
SELECT COUNT(DISTINCT PULocationID) FROM `de-zoomcamp-hw3-486917.ny_taxi.yellow_tripdata_non_partitioned`;
Question 3: Why are the estimated number of Bytes different?
Answer: BigQuery is a columnar database, and it only scans the specific columns requested in the query. Querying two columns (PULocationID, DOLocationID) requires reading more data than querying one column (PULocationID), leading to a higher estimated number of bytes processed.

Question 4: How many records have a fare_amount of 0?
Answer: 8,333

Query:

```sql
SELECT COUNT(*) FROM `de-zoomcamp-hw3-486917.ny_taxi.yellow_tripdata_non_partitioned`
WHERE fare_amount = 0;
Question 5: Best strategy for optimized table
Scenario: Query always filters by tpep_dropoff_datetime and orders by VendorID.

Answer: Partition by tpep_dropoff_datetime and Cluster on VendorID

SQL to create this table:

```sql
CREATE OR REPLACE TABLE `de-zoomcamp-hw3-486917.ny_taxi.yellow_tripdata_partitioned_clustered`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID AS
SELECT * FROM `de-zoomcamp-hw3-486917.ny_taxi.external_yellow_tripdata`;
Question 6: Estimated bytes for distinct VendorID (March 1-15, 2024)
Answer: 310.24 MB for non-partitioned table and 26.84 MB for the partitioned table

Query (Non-Partitioned):

```sql
SELECT DISTINCT VendorID
FROM `de-zoomcamp-hw3-486917.ny_taxi.yellow_tripdata_non_partitioned`
WHERE tpep_dropoff_datetime >= '2024-03-01' AND tpep_dropoff_datetime <= '2024-03-15';
Query (Partitioned):

```sql
SELECT DISTINCT VendorID
FROM `de-zoomcamp-hw3-486917.ny_taxi.yellow_tripdata_partitioned_clustered`
WHERE tpep_dropoff_datetime >= '2024-03-01' AND tpep_dropoff_datetime <= '2024-03-15';


Question 7: Where is the data stored in the External Table?
Answer: GCP Bucket (External tables store metadata in BigQuery, but the actual data remains in Google Cloud Storage).


Question 8: Always create partitioned tables from External Tables?
Answer: False (It is often more efficient to perform heavy transformations on data already inside BigQuery, or to use a staging table first).