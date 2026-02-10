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