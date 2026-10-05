# spotify-data-engineering-pipeline

## 📌 Project Overview

This project is an end-to-end **Data Engineering pipeline** that extracts music data from the Spotify Web API, stores the raw data in Amazon S3, transforms and cleans the data using AWS Lambda, and makes the processed data available for analytical querying through Amazon Athena.

The pipeline is designed using a serverless AWS architecture, where different stages of the data pipeline are automatically triggered based on events.

### High-Level Flow

```text
                Spotify Web API
                       │
                       ▼
              ┌─────────────────┐
              │ Extract Lambda   │
              └────────┬────────┘
                       │
                       ▼
                 Amazon S3
              ┌─────────────────┐
              │ to_beprocessed  │
              │   Raw JSON Data │
              └────────┬────────┘
                       │
                  S3 Event Trigger
                       │
                       ▼
              ┌─────────────────┐
              │ Transform Lambda│
              └────────┬────────┘
                       │
              ┌────────┴─────────┐
              │                  │
              ▼                  ▼
        processed/          Curated Data
        Raw Data            ┌──────────────┐
                            │ Albums       │
                            │ Artists      │
                            │ Songs        │
                            └──────┬───────┘
                                   │
                                   ▼
                              CSV Files
                                   │
                                   ▼
                         AWS Glue Crawler
                                   │
                                   ▼
                              AWS Glue
                             Data Catalog
                                   │
                                   ▼
                              Amazon Athena
                                   │
                                   ▼
                              SQL Queries
```

---

# 🏗️ Architecture

The project uses the following AWS services:

| AWS Service | Purpose |
|---|---|
| **AWS Lambda** | Runs extraction and transformation code |
| **Amazon S3** | Stores raw and processed data |
| **Amazon Event Notifications** | Triggers transformation when new files arrive |
| **AWS Glue Crawler** | Detects schema and creates metadata tables |
| **AWS Glue Data Catalog** | Stores table/schema metadata |
| **Amazon Athena** | Queries the processed data using SQL |

---

# 🔄 Data Pipeline

The pipeline consists of the following major stages:

## 1. Data Extraction

The first Lambda function is responsible for extracting data from the **Spotify Web API**.

The extraction process:

1. Authenticates with the Spotify API.
2. Sends API requests to retrieve Spotify data.
3. Receives the response in JSON format.
4. Stores the raw JSON response in Amazon S3.
5. Places the raw files inside the `to_beprocessed` folder.

Example:

```text
S3 Bucket
│
└── to_beprocessed/
      │
      ├── spotify_data_2026-10-01.json
      ├── spotify_data_2026-10-02.json
      └── spotify_data_2026-10-03.json
```

### Scheduling

The extraction Lambda function is configured with a scheduled trigger so that the extraction process runs **once every day**.

This allows the pipeline to automatically collect the latest Spotify data without manual execution.

---

# 2. Raw Data Storage

The extracted Spotify API response is initially stored in Amazon S3 in its original JSON format.

The `to_beprocessed` folder acts as the **landing/raw ingestion area**.

```text
S3
│
└── to_beprocessed/
       │
       └── raw Spotify JSON
```

Keeping the original API response provides several benefits:

- Preserves the original source data
- Allows the transformation process to be rerun
- Provides traceability
- Separates ingestion from transformation
- Makes debugging easier

---

# 3. Data Transformation

A second AWS Lambda function is responsible for transforming the raw Spotify data.

This Lambda is triggered when a new file is placed inside the `to_beprocessed` folder.

The transformation process includes:

- Reading the raw JSON file from S3
- Extracting required attributes
- Cleaning the data
- Structuring nested Spotify API responses
- Creating separate datasets
- Converting the transformed datasets into CSV format
- Writing the processed datasets back to S3

---

# 4. Data Modeling

The Spotify API returns nested JSON structures containing information about songs, albums, and artists.

The transformation process separates this information into different datasets.

For example:

### Albums

```text
album_id
album_name
album_release_date
album_total_tracks
```

### Artists

```text
artist_id
artist_name
```

### Songs

```text
song_id
song_name
song_duration
song_popularity
album_id
artist_id
```

The exact columns depend on the Spotify API response and the attributes selected during transformation.

---

# 5. Processed Data

After transformation, the original file is moved from the `to_beprocessed` location into the `processed` folder.

The cleaned datasets are stored separately as CSV files.

Example S3 structure:

```text
S3 Bucket
│
├── to_beprocessed/
│
├── processed/
│   └── raw/
│       └── spotify_data.json
│
└── transformed/
    │
    ├── albums/
    │     └── albums.csv
    │
    ├── artists/
    │     └── artists.csv
    │
    └── songs/
          └── songs.csv
```

This creates a separation between:

**Raw Data → Transformed/Cleaned Data**

---

# 6. AWS Glue Crawler

Once the transformed CSV files are available in S3, an **AWS Glue Crawler** is used to discover the structure of the data.

The crawler:

1. Scans the S3 locations.
2. Detects the files.
3. Identifies columns and data types.
4. Creates table metadata.
5. Stores the metadata in the AWS Glue Data Catalog.

For example:

```text
S3 CSV Files
      │
      ▼
AWS Glue Crawler
      │
      ▼
Glue Data Catalog
      │
      ├── albums
      ├── artists
      └── songs
```

---

# 7. Amazon Athena

The Glue Data Catalog tables are then queried using **Amazon Athena**.

Athena allows SQL queries to be executed directly against the data stored in Amazon S3 without requiring a traditional database server.

Example:

```sql
SELECT *
FROM songs
LIMIT 10;
```

Example analytical query:

```sql
SELECT
    artist_name,
    COUNT(*) AS total_songs
FROM songs
GROUP BY artist_name
ORDER BY total_songs DESC;
```

This provides a serverless analytical layer on top of the processed Spotify data.

---

# ⚙️ Lambda Functions

## Extract Lambda

Location:

```text
extract/spotify_extract.py
```

Responsibilities:

- Authenticate with Spotify API
- Extract data using API requests
- Handle API responses
- Store raw JSON data in S3
- Run on a scheduled basis

---

## Transform Lambda

Location:

```text
transform/spotify_transform.py
```

Responsibilities:

- Read raw JSON from S3
- Parse nested Spotify API response
- Extract album information
- Extract artist information
- Extract song information
- Clean and transform data
- Convert datasets to CSV
- Store processed data in S3
- Move the original raw file to the processed location

---

# 🔐 Security

Sensitive credentials are **not stored directly in the source code**.

The following should be stored using secure mechanisms such as Lambda environment variables or AWS Secrets Manager:

```text
Spotify Client ID
Spotify Client Secret
Spotify Access Token
Spotify Refresh Token
AWS credentials
```

The GitHub repository should never contain:

```text
client_secret
access_token
refresh_token
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

---

# 📂 Repository Structure

```text
spotify-data-engineering-pipeline/
│
├── README.md
│
├── extract/
│   └── spotify_extract.py
│
├── transform/
│   └── spotify_transform.py
│
├── sql/
│   └── athena_queries.sql
│
├── architecture/
│   └── architecture.png
│
└── .gitignore
```

---

# 🛠️ Technologies Used

### Programming

- Python
- JSON
- Pandas
- Requests

### API

- Spotify Web API

### AWS

- AWS Lambda
- Amazon S3
- AWS Glue
- AWS Glue Data Catalog
- Amazon Athena
- Amazon EventBridge / S3 Event Notifications

### Data Formats

- JSON
- CSV

### Query Language

- SQL

---

# 🎯 Key Data Engineering Concepts Demonstrated

This project demonstrates several practical Data Engineering concepts:

- REST API data extraction
- API authentication
- Serverless data pipelines
- Event-driven architecture
- Scheduled data ingestion
- Raw data ingestion
- Data transformation
- Data cleansing
- JSON parsing
- Nested JSON handling
- Data modeling
- Data partitioning/organization in S3
- CSV generation
- Schema discovery
- Metadata management
- Serverless SQL querying
- AWS Lambda
- Amazon S3
- AWS Glue
- Amazon Athena

---

# 🔁 End-to-End Pipeline

The complete workflow can be summarized as:

```text
Spotify API
     │
     ▼
Extract Lambda
     │
     │ Scheduled Daily
     ▼
S3 - to_beprocessed
     │
     │ S3 Event
     ▼
Transform Lambda
     │
     ├───────────────► Processed Raw Data
     │
     ▼
Clean & Transform
     │
     ├──────────────► Albums CSV
     ├──────────────► Artists CSV
     └──────────────► Songs CSV
                              │
                              ▼
                        Glue Crawler
                              │
                              ▼
                       Glue Data Catalog
                              │
                              ▼
                           Athena
                              │
                              ▼
                         SQL Analysis
```

---

# 🚀 Future Enhancements

The pipeline can be further enhanced by adding:

- Incremental data processing
- AWS Step Functions for workflow orchestration
- AWS Secrets Manager for Spotify credentials
- CloudWatch logging and monitoring
- Error handling and retry mechanisms
- Dead Letter Queue for failed Lambda executions
- Data quality validation
- Partitioning of S3 data
- Parquet instead of CSV for analytical workloads
- Athena views for reusable analytical queries
- CI/CD deployment using GitHub Actions
- Infrastructure as Code using Terraform or AWS CloudFormation

---

# 📌 Project Objective

The objective of this project is to build a **serverless, automated, and event-driven Data Engineering pipeline** that extracts data from an external REST API, stores raw data in Amazon S3, transforms and cleans the data using Python and AWS Lambda, catalogs the resulting datasets using AWS Glue, and provides a SQL-based analytical interface through Amazon Athena.

The project demonstrates how a modern cloud-based data pipeline can be built without maintaining traditional servers or database infrastructure.
