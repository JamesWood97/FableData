# FableData
Databricks task for Fable Data

This repository contains a Databricks batch ETL pipeline for processing
daily transaction files.

The solution uses a medallion architecture:

- Bronze: raw ingestion and audit metadata
- Silver: cleansing, type conversion and business transformations
- Gold: aggregated reporting tables
- Dashboard: Databricks AI/BI dashboard using the Gold tables

### Bronze layer

The Bronze layer reads the source CSV without applying sny
transformations. It adds ingestion metadata, including the file date (at present derived from ingestion date) and
ingestion timestamp. It then writes the data to a table

The purpose of this layer is to preserve the source representation and
support reprocessing and auditing.

### Silver layer

The Silver layer cleans, standardises and enriches the data. The transformations are applied in the following order:

First, the source column names are renamed to match the target database schema and naming convention. For example, `record_id` becomes `RECORD_ID`, `Txn_Description` becomes `ORIGINAL_DESCRIPTION_TEXT`, and `customer_postcode` becomes `POSTCODE`.

The transaction and posting date fields are then converted from strings into Spark date values. Numeric fields are also cast to their expected types. `RECORD_ID` is converted to an integer and `TRANSACTION_AMOUNT` is converted to a decimal value. `try_cast` is used so that invalid values become null instead of causing the entire pipeline to fail.

Malformed rows are removed where `RECORD_ID` contains a non-numeric value and the remaining business fields are empty. This handles corrupted records found within the data.

The required business rules are then applied:

- `CUSTOMER_TYPE` is set to `Unspecified` when the customer's age band is `>=70` or their gender is unknown. Otherwise, the original customer type is retained.
- `POSTING_DATE` is set to the later of the original posting date and transaction date.
- `DESCRIPTION_TEXT` is created from the original transaction description. Any occurrence of `#####` is removed, and a trailing country code is removed only where it matches `COUNTRY_CODE`.
- `COUNTRY_CODE` is trimmed and converted to uppercase.
- `EU_FLAG` is derived using the supplied list of EU country codes. Transactions from Great Britain dated on or before 31 January 2020 are also classified as EU transactions.
- `GENDER_CODE` is set to `M` for male, `F` for female and `X` for any other or unknown value.
- UK-format postcodes have whitespace removed and their final two characters replaced with `**`. Values that do not match the expected UK postcode format are retained unchanged.
- Newline and hidden control characters are removed from the notes field.

The final cleaned transaction-level dataset is written as the Delta table:

`workspace.fable_data.silver_transactions`

The Silver table retains the detailed transaction records so that different Gold-layer reporting aggregates can be produced without rereading or reprocessing the original .csv file. Records with missing reporting-critical fields are retained in Silver for auditability but are excluded iwthin Gold.

### Gold layer

The Gold layer prepares reporting-ready datasets for use in the Databricks dashboard. It reads the cleaned transaction-level data from the Silver table and produces smaller aggregated Delta tables for specific purposes.

Before calculating the aggregates, records are filtered to ensure that `TRANSACTION_AMOUNT`, `TRANSACTION_DATE` and `CUSTOMER_KEY` are present. These fields are required for calculating transaction values, date-based trends and customer metrics.

A common set of measures is calculated for each Gold table:

- `TRANSACTION_COUNT`: the number of transactions
- `TOTAL_TRANSACTION_AMOUNT`: the sum of transaction amounts, rounded to two decimal places
- `AVERAGE_TRANSACTION_AMOUNT`: the average transaction amount, rounded to two decimal places
- `UNIQUE_CUSTOMERS`: the number of distinct customer keys within each group

The following Gold tables are created:

- `gold_daily_metrics` contains one row per transaction date and supports analysis of transaction value, volume and customer activity over time.
- `gold_country_metrics` contains one row per transaction date and country code and supports comparison of transaction activity between countries.
- `gold_age_band_metrics` contains one row per transaction date and age band and supports analysis of customer demographics.
- `gold_gender_metrics` contains one row per transaction date and gender and further supports analysis of customer demographics.

The resulting datasets are written as managed Delta tables in the `workspace.fable_data` schema:

- `workspace.fable_data.gold_daily_metrics`
- `workspace.fable_data.gold_country_metrics`
- `workspace.fable_data.gold_age_band_metrics`
- `workspace.fable_data.gold_gender_metrics`

Separating these aggregates into Gold tables reduces the amount of processing required by the dashboard and provides clear reporting datasets. The detailed Silver records remain available for other analysis.

## Key design decisions

### Medallion architecture

The pipeline is separated into Bronze, Silver and Gold layers to keep raw
ingestion, transformations and reporting logic independent.

### Delta format

Delta tables were used to provide schema enforcement, transactional writes
and support for future incremental processing.

### Gold aggregate tables

Dashboard calculations are prepared in the Gold layer to reduce repeated
processing and provide stable reporting datasets.