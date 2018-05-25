# BigQuery-ETL
A script and guide on how to move data from data sources to staging and build a complete data warehouse on BigQuery

# Supported data sources
RDBMS in general
MongoDB

# Tools to build data pipeline
Apache NiFi: https://nifi.apache.org
Python3 and libs
Google Cloud Storage (staging source to store data)
Google BigQuery (data warehouse)
Jenkins (to schedule and manage job scheduling)

That's it! So cheap and efficient

# How the data pipeline works

## RDBMS data sources:

```mermaid
graph LR;
    SourceDatabases-->|Apache NiFi|GoogleCloudStorage;
    GoogleCloudStorage-->|python MERGE script|BigQuery;
    BigQuery-->BItools;
```

## MongoDB data sources:

```mermaid
graph LR;
    MongoDB-->|python load script|GoogleCloudStorage;
    GoogleCloudStorage-->|python MERGE script|BigQuery;
    BigQuery-->BItools;
```

The MERGE script do the following things:
1. Get ETL metadata from Metadata Dataset in BigQuery
2. Convert string to timestamp and string to float64 for specified columns
3. Run MERGE (upsert) query to perform insert / update with the data loaded in to Google Cloud Storage

Data on Google Cloud Storage is set to be self-deleted after 7 days from the loaded time. This approach reduces the timestamp management phase that normally sees in traditional ETLs.

# Metadata table

The stucture of this table should looks like this (written in JSON, but on BigQuery it is a normal table):
```json
{
    "job_name": "merge_sale_order_line",
    "project_name": "<your_project_name>",
    "staging_dataset_name": "<>",
    "staging_table_name": "<>",
    "destination_dataset_name": "<>",
    "destination_table_name": "<>",
    "timestamp_column": "create_date, write_date, created_at, updated_at",
    "float_column": "qty_delivered, price_total",
    "id_column_name": "id",
    "max_column_name": "updated_at",
    "last_run_time": "2018-05-11 02:41:33.994 UTC",
    "contains_record_type": 1 // Does the job need to transform table with RECORD field
  }
```

# How to set up Jenkins

Follow the instruction on https://www.digitalocean.com/community/tutorials/how-to-install-jenkins-on-ubuntu-16-04

If `No valid crumb was included` issue pops up when save changes in Jenkins, fix by following this: https://stackoverflow.com/questions/44062737/no-valid-crumb-was-included-in-the-request-jenkins-in-windows

Add jenkins user to Docker group: `sudo usermod -aG docker jenkins`

If the server does not sync with GMT+7, use the following command: `sudo timedatectl set-timezone Asia/Ho_Chi_Minh`

Restart jenkins service `sudo service jenkins restart`

Jenkins can be access from URL: `http://159.89.200.42:8081/`

# How to run docker set up

From the src root: `cd etl/bigquery_staging_to_warehouse`

1. Build docker container: `docker build -t bigquery_etl .`
2. Run docker container: `docker run --restart=always -d -v $PWD:/src --name bigquery_etl bigquery_etl`

# Insert new table to transform to ETL metadata table on BigQuery

Run the following query in BigQuery

```sql
INSERT INTO
  `metadata.etl_metadata` ( job_name,
    project_name,
    staging_dataset_name,
    staging_table_name,
    destination_dataset_name,
    destination_table_name,
    timestamp_column,
    float_column,
    id_column_name,
    max_column_name,
    last_run_time,
    contains_record_type)
VALUES
  ( 'merge_sale_order', 
  'duyluan-data-platform', 
  'staging', 
  'sale_order', 
  'dwh', 
  'sale_order', 
  'create_date, write_date, date_order', 
  'amount_total, cod_amount, amount_tax', 
  'id', 
  'write_date', 
  NULL,
  0)
  ```
  
# ETL commands

## BigQuery staging table merge and load to Data warehouse

`docker exec -t bigquery_etl python3 /src/src/bigquery_merge.py <job_name>`

## Load data from MongoDB to Google Cloud Storage and merge / load data to Data warehouse

`docker exec -t bigquery_etl python3 /src/src/mongodb_to_gcs_run.py '<job_object_json>'`

The `job_object_json` is a `1-line JSON string` contain information to run the job. A sample is included below.
Please note that the JSON must be flatten without any \n (new line)

```json
{"main_dataset_name":"tenren_dwh","staging_dataset_name":"tenren_staging","table_name":"sale_order","last_load_column_
name":"updated_at","mongodb_db_name":"Sale","mongodb_collection_name":"SaleOrder","merge_job_name":"merge_tenren_sale_order","config":{"is_tenren_sale_order":1}}
```

JSON Explanation

```json
{
  "main_dataset_name": "tenren_dwh",
  "staging_dataset_name": "tenren_staging",
  "table_name": "sale_order",
  "last_load_column_name": "updated_at",
  "mongodb_db_name": "Sale",
  "mongodb_collection_name": "SaleOrder",
  "merge_job_name": "merge_tenren_sale_order",
  "config": #optional {
    "is_tenren_sale_order": 1 #for sale order only
  }
}

```

You can use this site to compose your JSON https://jsoneditoronline.org
