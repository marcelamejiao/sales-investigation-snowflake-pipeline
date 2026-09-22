# Hamburg Weather and Sales Investigation

An end-to-end Snowflake data pipeline that investigates a Hamburg, Germany
sales anomaly and delivers an interactive weather-and-sales dashboard. The
project follows the **Ingestion–Transformation–Delivery (I-T-D)** framework.

## Business question

Tasty Bytes is a global food truck company. Analysts noticed that sales in
Hamburg dropped to `$0` for several days in February 2022. This project
combines daily sales with weather data to help explain the anomaly and provides
a repeatable data product that analysts can use to monitor Hamburg.

## Architecture

```text
Snowflake Marketplace             AWS S3
Pelmorex weather data       Tasty Bytes operational data
          │                              │
          └──────────────┬───────────────┘
                         ▼
                 Snowflake raw schemas
                         │
             SQL views and Python UDFs
                         │
              harmonized.weather_hamburg
                         │
          Streamlit in Snowflake + Altair
```

## Ingestion

### Snowflake Marketplace

The pipeline uses the **Pelmorex Weather Source** Marketplace dataset,
including daily temperature, precipitation, wind speed, postal-code, and city
attributes. `01_Transformation/Hamburg_Sales.sql` joins the weather history to
the Tasty Bytes country reference and creates
`TASTY_BYTES.HARMONIZED.DAILY_WEATHER_V`, filtered to supported Tasty Bytes
cities.

Before running the transformation scripts, subscribe to the Pelmorex listing
in the Snowflake Marketplace and make its shared database available in the
account. The shared database name must match the object references in the SQL
scripts, or those references must be updated for the target account.

### AWS S3

The Tasty Bytes source files are loaded from:

```text
s3://sfquickstarts/tasty-bytes-builder-education/
```

`00_Ingestion/Marketplace_data_tasty_bytes.sql` creates the Snowflake database
schemas, CSV file format, external stage, raw tables, harmonized views, and
analytics views. It then loads:

- Franchisees and trucks
- Locations and menus
- Order headers and order details
- Customer loyalty data

`00_Ingestion/AWS_countries_data.sql` loads the country and city reference
table. `00_Ingestion/AWS_tasty_bytes_data.sql` contains the focused stage and
country-table setup used for the S3 ingestion walkthrough.

The scripts are written for the sample Tasty Bytes environment and use
`ACCOUNTADMIN`, `COMPUTE_WH`, and `DEMO_BUILD_WH`. Review roles, warehouses,
database names, and privileges before running them in another account.

## Transformation

### SQL and views

The raw order tables are joined into
`TASTY_BYTES.HARMONIZED.ORDERS_V`, which provides order dates, cities,
countries, menu items, quantities, and prices. Analytics views expose the
harmonized order data and customer loyalty metrics.

`TASTY_BYTES.HARMONIZED.WEATHER_HAMBURG` combines February 2022 Hamburg
weather with daily sales. It calculates:

- Daily sales
- Average temperature in Fahrenheit
- Average temperature in Celsius
- Average precipitation in inches
- Average precipitation in millimeters
- Maximum wind speed in miles per hour

The sales investigation in `01_Transformation/Hamburg_Sales.sql` generates a
complete February date series and left joins sales. This is important because
dates with no matching orders remain in the result and are displayed as
`$0` instead of disappearing from the analysis.

### User-defined functions

`01_Transformation/UDFs.sql` creates two reusable SQL UDFs:

```sql
TASTY_BYTES.ANALYTICS.FAHRENHEIT_TO_CELSIUS(temp_f)
TASTY_BYTES.ANALYTICS.INCH_TO_MILLIMETER(inch)
```

The UDFs enrich the Hamburg weather view with analyst-friendly unit
conversions while keeping the aggregation and joins in SQL.

## Delivery: Streamlit in Snowflake

`03_Delivery/HAMBURG_GERMANY_TRENDS/streamlit_app.py` is a Python Streamlit
application that:

1. Reads `TASTY_BYTES.HARMONIZED.WEATHER_HAMBURG` with the active Snowpark
   session.
2. Converts sales to millions of dollars for chart readability.
3. Reshapes the weather metrics for visualization.
4. Renders an interactive Altair line chart with independent sales and weather
   axes.

The Streamlit deployment configuration is in:

```text
03_Delivery/HAMBURG_GERMANY_TRENDS/
├── pyproject.toml
├── snowflake.yml
├── streamlit_app.py
└── .streamlit/config.toml
```

The dashboard preview is available in
`assets/hamburg-weather-sales-dashboard.png`.

## Analytics result

The dashboard shows the following February 2022 pattern:

- Hamburg has regular daily sales before the anomaly, generally around
  `$27–$37 million`.
- Sales fall to `$0` for seven consecutive days, from **February 15 through
  February 21, 2022**.
- Sales resume on February 22 at approximately `$41 million`, followed by
  normal daily variation through the end of the month.
- Weather data remains populated during the zero-sales period. Temperature,
  precipitation, and wind-speed series continue across the same dates.
- The yellow **Max Wind Speed (mph)** series increases during the zero-sales
  period and reaches its highest level around February 19, before declining
  toward February 22.
The combined view makes the anomaly traceable: analysts can compare the
zero-sales interval with weather conditions and then inspect the underlying
order and load data for the affected dates.

![Weather and Sales Trends for Hamburg, Germany](assets/hamburg-weather-sales-dashboard.png)

## Repository layout

```text
00_Ingestion/
├── AWS_countries_data.sql
├── AWS_tasty_bytes_data.sql
└── Marketplace_data_tasty_bytes.sql

01_Transformation/
├── Hamburg_Sales.sql
└── UDFs.sql

03_Delivery/HAMBURG_GERMANY_TRENDS/
├── pyproject.toml
├── snowflake.yml
├── streamlit_app.py
└── .streamlit/config.toml

assets/
└── hamburg-weather-sales-dashboard.png
```

## Running the pipeline

1. Create or select a Snowflake database named `TASTY_BYTES`.
2. Confirm the required warehouse and role, or adjust the scripts for the
   target environment.
3. Subscribe to the Pelmorex Weather Source Marketplace listing.
4. Run `00_Ingestion/Marketplace_data_tasty_bytes.sql` to create and load the
   Tasty Bytes raw and analytics objects.
5. Run `00_Ingestion/AWS_countries_data.sql` if the country reference table is
   not already loaded.
6. Run `01_Transformation/UDFs.sql` to create the conversion functions.
7. Run `01_Transformation/Hamburg_Sales.sql` to create the weather views and
   inspect the February sales and weather results.
8. Deploy `03_Delivery/HAMBURG_GERMANY_TRENDS` as a Streamlit in Snowflake
   application using Snowflake CLI and the included `snowflake.yml`.

You can also run the Streamlit application directly in Snowflake Cloud from
the Workspace by opening `03_Delivery/HAMBURG_GERMANY_TRENDS/streamlit_app.py`
in the Streamlit editor and running the app there.

Do not commit credentials, storage integrations, account identifiers, or
private Marketplace configuration to the repository. Use environment-specific
Snowflake administration and deployment settings for those values.
