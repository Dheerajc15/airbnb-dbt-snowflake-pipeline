# Airbnb Analytics Engineering Pipeline

**dbt + Snowflake | Medallion Architecture | Kimball Dimensional Modeling**

An end-to-end analytics engineering pipeline that transforms raw Airbnb booking, host, and listing data into a governed, query-ready warehouse layer. The project simulates a real analytics engineering workflow inside a company's data platform team: land raw data, clean and conform it, then model it into a dimensional schema that business intelligence and analytics teams can query directly.

---

## Objective

Raw operational data is rarely fit for direct reporting — it's ungoverned, inconsistently typed, and lacks historical context. This project builds the transformation layer that turns raw Airbnb `LISTINGS`, `BOOKINGS`, and `HOSTS` tables into trustworthy, analysis-ready datasets by applying:

- A **medallion architecture** (bronze → silver → gold) to progressively clean and enrich data
- **Kimball-style dimensional modeling** (fact table + conformed dimensions) so downstream consumers get a single, well-understood grain
- **Incremental processing** so the pipeline scales without full-table reprocessing on every run
- **Historical tracking (SCD Type 2)** so changes to host and listing attributes over time aren't silently overwritten
- **Secure, production-style authentication** to the warehouse instead of hardcoded credentials

The result is a warehouse layer that answers questions like *"how has a host's response-rate quality trended over time?"* or *"what's the revenue mix by property type and city?"* without analysts needing to touch raw source tables.

---

## Architecture

    Raw Sources (Snowflake: AIRBNB.STAGING)
    LISTINGS · BOOKINGS · HOSTS
                  |
                  v
    +--------------------------+
    |          BRONZE          |   Raw ingestion, incremental load
    |  bronze_listings/        |   Filters on CREATED_AT watermark
    |  bookings/hosts          |   to avoid reprocessing full history
    +-------------+------------+
                  |
                  v
    +--------------------------+
    |          SILVER          |   Cleansed & business-rule layer
    |  silver_listings/        |   Derived fields, quality tagging,
    |  bookings/hosts          |   standardized text, incremental
    +-------------+------------+
                  |
                  v
    +--------------------------+
    |           GOLD           |   Analytics-ready consumption layer
    |  obt (wide table)        |   Denormalized OBT for fast BI queries
    |  fact + dimensions       |   Star schema for governed reporting
    |  dim_bookings/hosts/     |   SCD Type-2 snapshots preserve history
    |  listings (SCD-2)        |
    +--------------------------+

**Design rationale:** bronze and silver are incremental to keep run times proportional to new data, not total history. Gold materializes as full tables since it's the consumption layer and needs to be fast and predictable for BI tools. Snapshots run separately from the model DAG so historical state is captured deliberately, not as a side effect of a `dbt run`.

---

## Data Model

The gold layer exposes two complementary shapes over the same data, which is a deliberate modeling choice rather than duplication:

| Model | Grain | Purpose |
|---|---|---|
| `obt` | One row per booking, joined out to its listing and host | Fast, flat table for BI tools and ad-hoc analysis — no joins required |
| `fact` (via `ephemeral/bookings`) | One row per booking | Narrow fact table for governed, join-based reporting |
| `dim_hosts` (SCD-2) | One row per host per validity window | Tracks changes to superhost status, response-rate quality over time |
| `dim_listings` (SCD-2) | One row per listing per validity window | Tracks changes to pricing tier, property attributes over time |
| `dim_bookings` (SCD-2) | One row per booking per validity window | Tracks booking status changes over time |

Snapshots use `dbt`'s timestamp strategy against each source's `CREATED_AT` field, with `dbt_valid_to_current` set to `9999-12-31` so current-state queries don't need a `NULL` check.

---

## Tech Stack

| Layer | Tool |
|---|---|
| Cloud Data Warehouse | Snowflake |
| Transformation Framework | dbt Core 1.11 |
| Modeling Approach | Kimball dimensional modeling + medallion architecture |
| Language | SQL, Jinja2 |
| Authentication | RSA key-pair (no password/secret in config) |
| Version Control | Git / GitHub |

---

## Implementation Highlights

- **Incremental models with watermarking** — bronze and silver models filter on `CREATED_AT > MAX(CREATED_AT)` from the target table, so re-running the pipeline only processes new rows instead of the full source.
- **Reusable Jinja macros** — business logic like fee calculation (`multiply`) and price-tier bucketing (`tag`) is centralized in macros rather than duplicated across models, so a rule change happens in one place.
- **Custom schema-naming macro** (`generate_schema_name.sql`) — keeps environment schema naming consistent instead of dbt's default `<target_schema>_<custom_schema>` concatenation.
- **Ephemeral intermediate models** — the gold layer decomposes the wide `obt` table back into subject-area CTEs (`ephemeral/bookings`, `hosts`, `listings`) that feed the snapshots, without materializing extra tables in the warehouse.
- **SCD Type-2 history** — host, listing, and booking dimensions are snapshotted independently of the model run, so attribute changes are preserved for point-in-time analysis rather than overwritten in place.
- **Key-pair authentication** — connects to Snowflake using an RSA private key rather than a username/password, matching how production dbt deployments typically authenticate.
- **Secrets kept out of version control** — `profiles.yml`, `.p8`/`.pem` keys, and other environment-specific config are excluded via `.gitignore` rather than committed with placeholder values.

---

## Repository Structure

    airbnb_dbt_snowflake/
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
    |   |   |-- ephemeral/     # Subject-area CTEs feeding snapshots
    |   |   |-- fact.sql       # Booking fact table
    |   |   `-- obt.sql        # Denormalized wide table for BI
    |   `-- sources/
    |       `-- sources.yml    # Source declarations
    |-- snapshots/             # SCD-2 change tracking (dim_bookings/hosts/listings)
    |-- seeds/
    |-- tests/
    |-- dbt_project.yml
    `-- README.md

---

## Getting Started

### Prerequisites
- Python 3.12+
- A Snowflake account with `ACCOUNTADMIN` (or equivalent) role
- Source tables loaded in the `AIRBNB.STAGING` schema

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

---

## Skills Demonstrated

`Analytics Engineering` `Data Modeling (Kimball / Star Schema)` `dbt Core` `Snowflake` `SQL` `Jinja Templating` `Incremental ELT Design` `Slowly Changing Dimensions (SCD-2)` `Medallion Architecture` `Data Warehouse Security` `Git Version Control`

---

## Future Enhancements

- Add schema tests (`not_null`, `unique`, `relationships`) across bronze/silver/gold models and wire them into CI
- Automate runs with a scheduler (GitHub Actions, Airflow, or dbt Cloud) instead of manual `dbt build`
- Add seed-based reference data and documented column-level descriptions via `schema.yml`
- Connect the gold layer to a BI tool (e.g., Tableau, Power BI, Looker) to close the loop from raw data to dashboard
