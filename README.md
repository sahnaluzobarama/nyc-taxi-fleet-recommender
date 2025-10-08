# NYC Taxi Demand Forecasting & Fleet Analytics Project

## Project Overview

This project implements a comprehensive urban mobility analytics system for New York City taxi services using the TLC (Taxi & Limousine Commission) dataset. It features demand forecasting, anomaly detection, and fleet recommendations through a cloud-native big data pipeline on Google Cloud Platform.

## Architecture & Components

### Data Pipeline Structure (Medallion Architecture)
- **RawBronze**: Raw ingested data from TLC
- **CleanSilver**: Cleaned and validated data
- **PreMLGold**: Feature-engineered data ready for ML
- **PostMLGold**: Model predictions and anomaly results

### Key Notebooks Workflow

1. **00_AutoIngestTLCFiles.ipynb**
   - Downloads monthly TLC data (Yellow, Green, FHV, HVFHV)
   - Stores raw parquet files in Google Cloud Storage
   - Logs ingestion status

2. **01_GCStoBronzeIngestion.ipynb**
   - Loads parquet files from GCS to BigQuery RawBronze dataset
   - Handles schema normalization across taxi types
   - Uses PySpark on Dataproc for distributed processing

3. **Data Cleaning Notebooks** (Taxi-Type Specific):
   - **02aYellowRawBronzeToCleanSilver.ipynb**: Yellow taxi cleaning pipeline
   - **02bGreenRawBronzeToCleanSilver.ipynb**: Green taxi cleaning pipeline
   - **02cFHVRawBronzeToCleanSilver.ipynb**: For-Hire Vehicle cleaning pipeline
   - **02dFHVHVRawBronzeToCleanSilver.ipynb**: High-Volume For-Hire Vehicle cleaning pipeline
   
   Each notebook performs:
   - Type casting (timestamps, categorical variables)
   - Null passenger count removal
   - Invalid trip filtering (pickup < dropoff time)
   - Zero/negative distance and fare removal
   - Outlier removal using IQR method
   - Trip duration filtering (5 seconds to 2 hours)
   - Year/month validation
   - Outputs to CleanSilver dataset

4. **03CleanSilverToPreML.ipynb**
   - Aggregates cleaned data into ML-ready features
   - Creates passenger buckets (single, small, medium, large)
   - Generates three aggregation types:
     - Daily aggregations (_daily tables)
     - Hourly aggregations (_hourly tables)  
     - Hotspot/zone aggregations (_hotspot tables)
   - Outputs to PreMLGold dataset

5. **Model Implementation Notebooks**:
   
   **04aXGBoostFleetRecommender.ipynb**
   - Zone-level demand forecasting for 260+ NYC zones
   - Implements two models:
     - XGBoost (primary model)
     - KNN Regressor (comparison model)
   - Predicts demand for 4 passenger buckets:
     - Single (1 passenger)
     - Small (2-3 passengers)
     - Medium (4-5 passengers)
     - Large (6+ passengers)
   - Features:
     - 15-day forecast horizon
     - Hourly granularity
     - Confidence intervals using residual-based method
     - Interactive zone selection
     - Geographic heatmap visualization
   
   **04bAnomalies.ipynb**
   - Implements anomaly detection using Autoencoder
   - Architecture:
     - Input layer → 64 → 32 → 16 → 32 → 64 → Output
     - Reconstruction error threshold for anomaly flagging
   - Detects unusual demand patterns:
     - Holiday effects (Thanksgiving, Christmas, New Year)
     - Special events
     - System disruptions
   - Stores anomaly flags in PostMLGold
   - Note: Isolation Forest was tested but removed due to excessive false positives

## Technology Stack

### Google Cloud Platform Services
- **BigQuery**: Data warehouse for all datasets
- **Cloud Storage (GCS)**: Raw file storage, logs, model artifacts
- **Dataproc (Apache Spark)**: Distributed data processing

### Machine Learning
- **XGBoost**: Primary forecasting model (best performance)
- **TensorFlow/Keras**: Autoencoder for anomaly detection
- **Scikit-learn**: Model evaluation and preprocessing
- **KNeighborsRegressor**: Alternative zone-level model

### Visualization
- **Gradio**: Interactive dashboard
- **Plotly**: Maps and time series visualizations

## Dataset Details

- **Source**: NYC TLC Trip Record Data
- **Coverage**: January 2023 - July 2025 (configurable)
- **Taxi Types**: 
  - Yellow: Traditional NYC taxis
  - Green: Boro taxis (outer boroughs)
  - FHV: For-Hire Vehicles
  - HVFHV: High-Volume For-Hire Vehicles (Uber, Lyft)
- **Scale**: Billions of trip records
- **Zones**: 260+ geographic zones across NYC

## Replication Guide

### Prerequisites
1. Google Cloud Platform account with billing enabled
2. Enable APIs: BigQuery, Dataproc, Cloud Storage
3. Python 3.8+ environment
4. Jupyter or Google Colab

### Setup Steps

1. **Create GCP Resources**
   ```bash
   # Create GCS bucket
   gsutil mb gs://nyc_raw_data_bucket
   
   # Create BigQuery datasets
   bq mk --dataset --location=US RawBronze
   bq mk --dataset --location=US CleanSilver
   bq mk --dataset --location=US PreMlGold
   bq mk --dataset --location=US PostMlGold
   ```

2. **Configure Dataproc**
   - Create a Dataproc cluster or use serverless Spark
   - Install required Python packages on cluster

3. **Update Configuration**
   - Modify `PROJECT_ID` in all notebooks
   - Update `BUCKET_NAME` to your GCS bucket
   - Adjust `START_DATE` and `END_DATE` in ingestion notebook

4. **Run Pipeline Sequentially**
   ```
   00_AutoIngestTLCFiles.ipynb
   ↓
   01_GCStoBronzeIngestion.ipynb
   ↓
   02a_YellowRawBronzeToCleanSilver.ipynb
   02b_GreenRawBronzeToCleanSilver.ipynb
   02c_FHVRawBronzeToCleanSilver.ipynb
   02d_FHVHVRawBronzeToCleanSilver.ipynb
   ↓
   03CleanSilverToPreML.ipynb
   ↓
   04a_XGBoostFleetRecommender.ipynb
   04b_Anomalies.ipynb
   ```

### Key Parameters

- **Passenger Buckets**:
  - Single: 1 passenger
  - Small: 2-3 passengers
  - Medium: 4-5 passengers
  - Large: 6+ passengers

- **Forecasting Models Evaluated**:
  - SARIMAX
  - Prophet
  - Holt-Winters
  - LSTM
  - XGBoost (selected for production)

- **Cleaning Thresholds**:
  - Trip duration: 5 seconds to 2 hours
  - Fare amount: > 0
  - Trip distance: > 0
  - Outlier removal: 1.5 * IQR

## Dashboard Features

The Gradio dashboard provides three main tabs:

1. **Data Explorer**

2. **Forecasting**: 
   - 2a)- Global city-wide predictions (trips and Revenue)
   - 2b)- Zone-level granular forecasts (trips - Passenger bucket predictions)
   - 15-day horizon with confidence intervals

3. **Anomaly Detection**: 
   - Flagged unusual demand patterns
   - Holiday and event detection
   - Reconstruction error visualization


## Storage Structure

### GCS Organization
```
gs://nyc_raw_data_bucket/
├── yellow/
│   └── 2023-2025/
├── green/
│   └── 2023-2025/
├── fhv/
│   └── 2023-2025/
├── fhvhv/
│   └── 2023-2025/
└── logs/
    └── pipeline_logs/
```

### Public Data Access

#### Direct Bucket Access

The NYC taxi data is publicly available through Google Cloud Storage. You can explore the bucket contents and download files directly:

- **Bucket API Endpoint**: https://storage.googleapis.com/storage/v1/b/nyc_raw_data_bucket/o
  - Returns JSON listing of all files in the bucket
  - Paginated results (1000 items per page)
  - Use `?prefix=yellow/` to filter by taxi type

- **Example Direct Download URL**:
  ```
  https://storage.googleapis.com/nyc_raw_data_bucket/fhv/2023/fhv_tripdata_2023-01.parquet
  ```
  
#### Sample Data Files

For testing or exploration, you can download individual parquet files directly:

- **FHV January 2023**: [fhv_tripdata_2023-01.parquet](https://storage.googleapis.com/download/storage/v1/b/nyc_raw_data_bucket/o/fhv%2F2023%2Ffhv_tripdata_2023-01.parquet?generation=1755746442538125&alt=media)
- **File Structure**: `{taxi_type}/{year}/{taxi_type}_tripdata_{yyyy-mm}.parquet`

#### Programmatic Access

```python
# Download a specific file
import requests

url = "https://storage.googleapis.com/nyc_raw_data_bucket/yellow/2023/yellow_tripdata_2023-01.parquet"
response = requests.get(url)
with open("yellow_tripdata_2023-01.parquet", "wb") as f:
    f.write(response.content)
```

#### Available Data Range
- **Time Period**: January 2023 - July 2025
- **Taxi Types**: yellow, green, fhv, fhvhv
- **File Size**: Varies from ~50MB to ~500MB per month depending on taxi type

### BigQuery Tables
```
RawBronze.{taxi_type}_{yyyy_mm}
CleanSilver.{taxi_type}_{yyyy_mm}
PreMlGold.{taxi_type}_{yyyy_mm}_daily
PreMlGold.{taxi_type}_{yyyy_mm}_hourly
PreMlGold.{taxi_type}_{yyyy_mm}_hotspot
PostMlGold.zone_{model(xgboost/knn)}predictions_{model}_{date}
PostMlGold.anomalies_{date}
```

## Monitoring & Logs

- Pipeline logs stored in GCS under `/logs/`
- Each notebook generates timestamped log files
- BigQuery audit logs track data lineage
- Spark session management includes retry logic with quota handling