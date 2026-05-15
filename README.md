# Airbnb dbt + Snowflake Pipeline

An end-to-end analytics engineering project that ingests, transforms, and models Airbnb booking data using **dbt** on **Snowflake**, following the **medallion architecture** (bronze → silver → gold) with Kimball-style dimensional modeling.

Built to demonstrate production-grade data warehousing patterns: incremental processing, slowly-changing-dimension tracking, modular Jinja macros, and secure key-pair authentication.

---

## Architecture

    Raw Sources (Snowflake)
              |
              v
    +------------------+
    |      BRONZE      |   Raw ingestion, light typing & trimming
    |   (incremental)  |   bronze_bookings, bronze_hosts, bronze_listings
    +--------+---------+
              |
              v
    +------------------+
    |      SILVER      |   Cleansed, conformed, business rules applied
    |   (incremental)  |   silver_bookings, silver_hosts, silver_listings
    +--------+---------+
              |
              v
    +------------------+
    |       GOLD       |   Star schema + OBT for analytics consumption
    | (table + SCD-2)  |   fact, obt, dim_bookings, dim_hosts, dim_listings
    +------------------+

---

## Key Features

- **Medallion layering** — bronze (raw), silver (cleaned), gold (analytics-ready)
- **11 dbt models** processing 1.9M+ rows through the full DAG
- **Incremental materialization** on bronze + silver layers for efficient reprocessing
- **SCD Type-2 snapshots** on `dim_bookings`, `dim_hosts`, `dim_listings` for historical attribute tracking
- **Fact table + denormalized OBT** at the gold layer following Kimball methodology
- **Custom Jinja macros** for reusable transformations (`multiply`, `trimmer`, `tag`, `generate_schema_name`)
- **Ephemeral models** in the gold layer for CTE-style intermediate logic
- **RSA key-pair authentication** to Snowflake (production pattern, no password in config)
- **Environment-isolated profiles** kept outside version control via `.gitignore`

---

## Tech Stack

| Layer            | Tool                                |
|------------------|-------------------------------------|
| Warehouse        | Snowflake                           |
| Transformation   | dbt Core 1.11                       |
| Orchestration    | dbt CLI (local)                     |
| Language         | SQL + Jinja2                        |
| Auth             | RSA key-pair                        |
| Version Control  | Git / GitHub                        |

---

## Project Structure

    airbnb-dbt-snowflake-pipeline/
    |-- analyses/              # Ad-hoc exploratory SQL (compiled, not run)
    |-- macros/                # Reusable Jinja macros
    |   |-- generate_schema_name.sql
    |   |-- multiply.sql
    |   |-- tag.sql
    |   `-- trimmer.sql
    |-- models/
    |   |-- bronze/            # Raw ingestion layer (incremental)
    |   |-- silver/            # Cleansed business layer (incremental)
    |   |-- gold/
    |   |   |-- ephemeral/     # CTE-style intermediate models
    |   |   |-- fact.sql       # Central fact table
    |   |   `-- obt.sql        # Denormalized analytics table
    |   `-- sources/
    |       `-- sources.yml    # Source declarations & tests
    |-- snapshots/             # SCD-2 change tracking
    |   |-- dim_bookings.yml
    |   |-- dim_hosts.yml
    |   `-- dim_listings.yml
    |-- seeds/
    |-- tests/
    |-- dbt_project.yml
    `-- README.md

---

## Getting Started

### Prerequisites
- Python 3.12+
- A Snowflake account with `ACCOUNTADMIN` (or equivalent) role
- Source tables loaded in `AIRBNB.STAGING` schema

### Setup

1. Clone the repo:

       git clone https://github.com/Dheerajc15/airbnb-dbt-snowflake-pipeline.git
       cd airbnb-dbt-snowflake-pipeline

2. Create a virtual environment and install dbt:

       python -m venv .venv
       .venv\Scripts\activate
       pip install dbt-snowflake

3. Configure Snowflake authentication (RSA key-pair):

       openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out snowflake_key.p8 -nocrypt
       openssl rsa -in snowflake_key.p8 -pubout -out snowflake_key.pub

   Then register the public key in Snowflake:

       ALTER USER <YOUR_USER> SET RSA_PUBLIC_KEY='<paste_public_key_here>';

4. Create `profiles.yml` (kept out of version control):

       airbnb_dbt_snowflake:
         outputs:
           dev:
             account: <your_account>
             database: AIRBNB
             role: ACCOUNTADMIN
             schema: dbt_schema
             threads: 1
             type: snowflake
             user: <your_user>
             warehouse: COMPUTE_WH
             private_key_path: <path_to_your_p8_file>
         target: dev

5. Validate and build:

       dbt debug
       dbt build

---

## Sample Commands

    dbt run                          # build all models
    dbt run -s bronze                # build only bronze layer
    dbt run -s silver+               # build silver and everything downstream
    dbt snapshot                     # capture SCD-2 changes
    dbt test                         # run data quality tests
    dbt docs generate && dbt docs serve   # interactive lineage docs



