# Kestra E-Commerce Data Engineering Pipeline

A beginner-friendly, hands-on project for learning [Kestra](https://kestra.io/) by building an e-commerce data pipeline from ingestion to SQL analytics.

The project demonstrates:

- CSV and REST API ingestion
- Data cleaning and validation
- Multi-source joins and enrichment
- Python and Pandas transformations
- SQLite database loading
- SQL analytics
- Scheduled executions
- Retries and failure handling
- Kestra outputs, logs, executions, and observability

> **Current checkpoint:** Extract, Transform, Load, SQLite, SQL analytics, scheduling, retries, failure handling, artifact management, and observability are complete. Data quality checks are the next planned feature.

## Table of contents

- [Project overview](#project-overview)
- [Architecture](#architecture)
- [Learning path](#learning-path)
- [Prerequisites](#prerequisites)
- [Run Kestra locally](#run-kestra-locally)
- [Part 1: Build the enriched CSV pipeline](#part-1-build-the-enriched-csv-pipeline)
- [Part 2: Load SQLite and add reliability](#part-2-load-sqlite-and-add-reliability)
- [Current working flow](#current-working-flow)
- [Expected results](#expected-results)
- [Kestra concepts](#kestra-concepts)
- [Troubleshooting lessons](#troubleshooting-lessons)
- [Limitations and next steps](#limitations-and-next-steps)
- [Author](#author)

## Project overview

We act as data engineers for an e-commerce company with two sources.

### Orders CSV

```text
order_id,product_id,quantity
```

### Product REST API

Product details come from the [DummyJSON Products API](https://dummyjson.com/products):

```text
id, title, category, price
```

The pipeline matches `orders.product_id` with `products.id`, then produces enriched order records containing:

```text
order_id, product_id, quantity, title, category, price, revenue
```

## Architecture

```text
Orders CSV ───────────────┐
                          ▼
                    create_orders
                          │
                          │
DummyJSON API ──► fetch_product_data
                          │
                   inspect_product_data
                          │
                          ▼
                    clean_orders
              clean, validate, join,
              enrich, calculate revenue
                          │
                          ▼
                   clean_orders.csv
                          │
                          ▼
                  load_to_database
                          │
                          ▼
                    ecommerce.db
                   orders table
                          │
                          ▼
                    sql_analytics
                          │
                          ▼
                   Business insights
                          │
                          ▼
                   processing_complete

If the API or another task fails:

             retries → pipeline_failed → alert log

The complete flow can also be started automatically by a two-minute schedule.
```

## Learning path

The project was built incrementally so each stage introduces one concept:

1. Create a simple Kestra flow with log tasks.
2. Generate an orders CSV with `storage.Write`.
3. Add duplicate and invalid records deliberately.
4. Use Python and Pandas to clean the data.
5. Pass files between tasks with Kestra output expressions.
6. Fetch product data from a REST API.
7. Inspect JSON with `jq`.
8. Join orders with product metadata.
9. Enrich orders and calculate revenue.
10. Write the result to `clean_orders.csv`.
11. Load the enriched CSV into SQLite.
12. Query the SQLite table using SQL.
13. Schedule the workflow with a cron trigger.
14. Add retries for temporary failures.
15. Add a flow-level error handler for unrecoverable failures.
16. Expose `ecommerce.db` as an output artifact for downstream tasks.

## Prerequisites

- Windows
- Docker Desktop
- Docker Desktop configured to use the WSL 2 backend
- Internet access for the DummyJSON API

A Docker account is not required for this local setup.

## Run Kestra locally

1. Install and start [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/).
2. Wait until the Docker Engine is running.
3. Open PowerShell and verify Docker:

   ```powershell
   docker --version
   ```

4. Start Kestra:

   ```powershell
   docker run --pull=always --rm -it -p 8080:8080 `
     --user=root `
     --name kestra `
     -v kestra_data:/app/storage `
     -v kestra_db:/app/data `
     -v /var/run/docker.sock:/var/run/docker.sock `
     -v /tmp:/tmp `
     kestra/kestra:latest-slim server local
   ```

5. Open [http://localhost:8080](http://localhost:8080).
6. Create the local administrator account when prompted.

## Part 1: Build the enriched CSV pipeline

Part 1 builds the Extract and Transform stages:

```text
Orders CSV + Product REST API
            ↓
     Clean and validate
            ↓
       Join and enrich
            ↓
    Calculate order revenue
            ↓
      clean_orders.csv
            ↓
        Sales analytics
```

The sample orders intentionally include:

- Duplicate order `1002`.
- Invalid order `1005` with a negative quantity.

The `clean_orders` task removes duplicate order IDs, filters out quantities less than or equal to zero, joins product details using `product_id`, and calculates:

```text
revenue = quantity × price
```

At the end of Part 1, the transformed data is available as `clean_orders.csv`. Part 2 loads that artifact into SQLite and adds operational reliability.

## Part 2: Load SQLite and add reliability

### 1. Test SQLite first

Before loading the real dataset, create a small test task. This confirms that Python can create a database, execute SQL, insert rows, and read them back.

```yaml
- id: test_database
  type: io.kestra.plugin.scripts.python.Script
  script: |
    import sqlite3

    connection = sqlite3.connect("ecommerce.db")
    cursor = connection.cursor()

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS test_table (
            id INTEGER,
            message TEXT
        )
    """)

    cursor.execute("""
        INSERT INTO test_table
        VALUES (1, 'Database connection successful')
    """)

    connection.commit()

    rows = cursor.execute(
        "SELECT * FROM test_table"
    ).fetchall()

    print("DATABASE RESULTS:")
    print(rows)
    connection.close()
```

Add this task after `clean_orders`, save the flow, execute it, and inspect the task logs. Once it succeeds, remove it before using the complete flow below.

### 2. Load the transformed data into SQLite

The `load_to_database` task reads the CSV produced by `clean_orders` and writes it to a SQLite table named `orders`.

```yaml
- id: load_to_database
  type: io.kestra.plugin.scripts.python.Script
  beforeCommands:
    - pip install pandas
  script: |
    import pandas as pd
    import sqlite3

    df = pd.read_csv(
        "{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}"
    )

    print("DATA TO BE LOADED:")
    print(df)

    connection = sqlite3.connect("ecommerce.db")

    df.to_sql(
        "orders",
        connection,
        if_exists="replace",
        index=False
    )

    print("\nData successfully loaded into SQL database!")

    result = pd.read_sql("SELECT * FROM orders", connection)
    print("\nDATA FROM DATABASE:")
    print(result)

    connection.close()
  outputFiles:
    - "ecommerce.db"
```

The expression below passes the CSV artifact from the previous task:

```text
{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}
```

`if_exists: replace` is useful while learning because every run creates a predictable table from the latest transformed data. Production pipelines may instead use append, upsert, merge, incremental loading, or change-data-capture patterns.

The `outputFiles` declaration is important. It tells Kestra to store the database file as a task artifact so that downstream tasks can consume it explicitly.

```text
load_to_database
        ↓
creates ecommerce.db
        ↓
outputFiles
        ↓
Kestra-managed artifact
        ↓
sql_analytics
```

### 3. Query the database with SQL

Add `sql_analytics` after `load_to_database`:

```yaml
- id: sql_analytics
  type: io.kestra.plugin.scripts.python.Script
  script: |
    import sqlite3

    connection = sqlite3.connect(
        "{{ outputs.load_to_database.outputFiles['ecommerce.db'] }}"
    )
    cursor = connection.cursor()

    print("===== SQL ANALYTICS =====")

    cursor.execute("""
        SELECT SUM(revenue)
        FROM orders
    """)
    print("Total Revenue:", cursor.fetchone()[0])

    cursor.execute("""
        SELECT category, SUM(revenue) AS total_revenue
        FROM orders
        GROUP BY category
        ORDER BY total_revenue DESC
    """)

    print("\nRevenue by Category:")
    for row in cursor.fetchall():
        print(row)

    cursor.execute("""
        SELECT title, SUM(revenue) AS total_revenue
        FROM orders
        GROUP BY title
        ORDER BY total_revenue DESC
        LIMIT 1
    """)

    print("\nTop Product:")
    print(cursor.fetchone())
    connection.close()
```

This task answers three business questions:

- What is the total revenue?
- Which categories generate the most revenue?
- Which product generated the most revenue?

It introduces `SUM`, `GROUP BY`, `ORDER BY`, aliases, and `LIMIT` in a real pipeline context.

### 4. Schedule the flow

Add `triggers` at the root level of the flow. Do not place it inside `tasks`:

```yaml
triggers:
  - id: every_two_minutes
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "*/2 * * * *"
```

The expression `*/2 * * * *` means every two minutes. After saving the flow, open **Kestra → Executions** and watch for new executions to appear automatically.

For experimentation, you can comment out or remove the trigger and execute the flow manually. This is especially useful while testing intentional failures.

### 5. Add retries for temporary failures

Retries are useful for transient network, API, database, or service problems.

Add this configuration to `fetch_product_data`:

```yaml
- id: fetch_product_data
  type: io.kestra.plugin.core.http.Request
  uri: https://dummyjson.com/products
  method: GET
  retry:
    type: constant
    interval: PT5S
    maxAttempts: 3
```

This means Kestra waits five seconds between attempts and allows a maximum of three attempts.

To observe the behavior safely:

1. Disable the two-minute trigger temporarily.
2. Change the URI to `https://dummyjson.com/this-does-not-exist`.
3. Save and execute manually.
4. Watch the task fail, wait, retry, and eventually exhaust its attempts.
5. Restore `https://dummyjson.com/products` afterward.

> **Important:** The property is `maxAttempts`, not `maxAttempt`. The singular form causes a flow validation error before execution starts.

### 6. Add failure handling

Retries handle temporary failures. An error handler defines what happens after retries are exhausted.

Add `errors` at the root level, alongside `tasks` and `triggers`:

```yaml
errors:
  - id: pipeline_failed
    type: io.kestra.plugin.core.log.Log
    message: |
      ALERT: E-commerce pipeline failed.
      Execution ID: {{ execution.id }}
      Please check the Kestra execution logs.
```

The hierarchy should look like this:

```yaml
id: ecommerce_pipeline
namespace: dsa.dataengineering

tasks:
  # normal tasks

errors:
  # failure tasks

triggers:
  # schedules
```

Test the handler by temporarily using the invalid API URI again. The expected lifecycle is:

```text
Task fails → retry → retry → retries exhausted → pipeline_failed
```

In a production workflow, the error handler could send an email or Slack message, create an incident, record failure metadata, or trigger a recovery workflow. Here it writes an alert to the Kestra logs.

## Current working flow

Use this flow as the Part 2 checkpoint. It includes the complete pipeline, SQLite loading, SQL analytics, retries, failure handling, and scheduling.

```yaml
id: ecommerce_pipeline
namespace: dsa.dataengineering

tasks:
  - id: pipeline_started
    type: io.kestra.plugin.core.log.Log
    message: "E-commerce data pipeline started"

  - id: create_orders
    type: io.kestra.plugin.core.storage.Write
    content: |
      order_id,product_id,quantity
      1001,1,2
      1002,2,1
      1002,2,1
      1003,3,3
      1004,4,1
      1005,5,-2
    extension: .csv

  - id: fetch_product_data
    type: io.kestra.plugin.core.http.Request
    uri: https://dummyjson.com/products
    method: GET
    retry:
      type: constant
      interval: PT5S
      maxAttempts: 3

  - id: inspect_product_data
    type: io.kestra.plugin.core.log.Log
    message: |
      Product API successfully fetched.
      First product: {{ outputs.fetch_product_data.body | jq('.products[0].title') | first }}
      Category: {{ outputs.fetch_product_data.body | jq('.products[0].category') | first }}
      Price: ${{ outputs.fetch_product_data.body | jq('.products[0].price') | first }}

  - id: clean_orders
    type: io.kestra.plugin.scripts.python.Script
    beforeCommands:
      - pip install pandas
    script: |
      import json
      import pandas as pd

      orders = pd.read_csv("{{ outputs.create_orders.uri }}")
      products_json = json.loads(r'''{{ outputs.fetch_product_data.body }}''')
      products = pd.DataFrame(products_json["products"])

      orders = orders.drop_duplicates(subset=["order_id"])
      orders = orders[orders["quantity"] > 0]

      products = products[["id", "title", "category", "price"]]
      products = products.rename(columns={"id": "product_id"})

      enriched = orders.merge(products, on="product_id", how="left")
      enriched["revenue"] = enriched["quantity"] * enriched["price"]
      enriched.to_csv("clean_orders.csv", index=False)

      print("ENRICHED ORDERS")
      print(enriched)
    outputFiles:
      - "clean_orders.csv"

  - id: load_to_database
    type: io.kestra.plugin.scripts.python.Script
    beforeCommands:
      - pip install pandas
    script: |
      import pandas as pd
      import sqlite3

      df = pd.read_csv(
          "{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}"
      )

      connection = sqlite3.connect("ecommerce.db")
      df.to_sql("orders", connection, if_exists="replace", index=False)

      print("Data successfully loaded into SQL database!")
      print(pd.read_sql("SELECT * FROM orders", connection))
      connection.close()
    outputFiles:
      - "ecommerce.db"

  - id: sql_analytics
    type: io.kestra.plugin.scripts.python.Script
    script: |
      import sqlite3

      connection = sqlite3.connect(
          "{{ outputs.load_to_database.outputFiles['ecommerce.db'] }}"
      )
      cursor = connection.cursor()

      cursor.execute("SELECT SUM(revenue) FROM orders")
      print("Total Revenue:", cursor.fetchone()[0])

      cursor.execute("""
          SELECT category, SUM(revenue) AS total_revenue
          FROM orders
          GROUP BY category
          ORDER BY total_revenue DESC
      """)
      print("Revenue by Category:")
      for row in cursor.fetchall():
          print(row)

      cursor.execute("""
          SELECT title, SUM(revenue) AS total_revenue
          FROM orders
          GROUP BY title
          ORDER BY total_revenue DESC
          LIMIT 1
      """)
      print("Top Product:", cursor.fetchone())
      connection.close()

  - id: analyze_orders
    type: io.kestra.plugin.scripts.python.Script
    beforeCommands:
      - pip install pandas
    script: |
      import pandas as pd

      df = pd.read_csv(
          "{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}"
      )

      print("Total Orders:", len(df))
      print("Average Order Value:", df["revenue"].sum() / len(df))

  - id: processing_complete
    type: io.kestra.plugin.core.log.Log
    message: "E-commerce data pipeline completed successfully."

errors:
  - id: pipeline_failed
    type: io.kestra.plugin.core.log.Log
    message: |
      ALERT: E-commerce pipeline failed.
      Execution ID: {{ execution.id }}
      Please check the Kestra execution logs.

triggers:
  - id: every_two_minutes
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "*/2 * * * *"
```

To run it manually, create or open the namespace `dsa.dataengineering`, create the flow `ecommerce_pipeline`, paste the YAML, save it, and select **Execute**. To test scheduling, leave the trigger enabled and watch the executions page.

## Expected results

The input contains six rows. After cleaning:

- Duplicate order `1002` is reduced to one row.
- Invalid order `1005` is removed.
- Four valid orders remain.
- Product details are joined from the API.
- Revenue is calculated from quantity and API price.
- The enriched dataset is stored in the SQLite table `orders`.
- SQL analytics prints total revenue, revenue by category, and the top product.

Because product prices come from an external API, exact revenue values can change if the API data changes.

## Kestra concepts demonstrated

### Orchestration

Python performs cleaning and transformation. SQLite stores structured data. SQL produces analytics. Kestra coordinates the complete workflow:

- When the pipeline runs
- Which task runs first
- How task outputs move downstream
- Whether a failed task should retry
- What happens after retries are exhausted
- Where to inspect execution logs and task outputs

### Task outputs and artifacts

Use explicit output references rather than assuming that a local file created by one task is available in another task:

```text
{{ outputs.create_orders.uri }}
{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}
{{ outputs.load_to_database.outputFiles['ecommerce.db'] }}
```

### ETL

- **Extract:** create order data and fetch product data from the REST API.
- **Transform:** parse JSON, validate, deduplicate, join, enrich, and calculate revenue.
- **Load:** write the transformed records into SQLite as the `orders` table.

### Reliability and observability

- **Scheduling:** starts executions automatically.
- **Retries:** recover from temporary failures.
- **Error handling:** logs an alert after retries are exhausted.
- **Logs:** show task output and error details.
- **Executions:** show each run and its status.
- **Gantt view:** shows task timing and dependencies.

## Troubleshooting lessons

### `Function or Macro [json] does not exist`

Use `jq` to inspect fields in the API response:

```text
{{ outputs.fetch_product_data.body | jq('.products[0].title') | first }}
```

### `KeyError: 'product'`

The API uses `title` for the product name. Update downstream code from `df["product"]` to `df["title"]` when the upstream schema changes.

### `Unrecognized field "maxAttempt"`

Use the plural property:

```yaml
maxAttempts: 3
```

A configuration or validation error occurs before execution starts. A runtime error occurs after the flow has started. Checking which kind of error occurred helps narrow the investigation.

### SQLite file is not available downstream

A file created inside one script task is not a reliable downstream dependency by itself. Declare it explicitly:

```yaml
outputFiles:
  - "ecommerce.db"
```

Then reference it with:

```text
{{ outputs.load_to_database.outputFiles['ecommerce.db'] }}
```

## Limitations and next steps

The current version is an educational local pipeline. It does not yet include:

- Data quality checks that fail the flow when invalid data survives transformation.
- A production database or warehouse.
- Incremental loading, upserts, or CDC.
- Notifications through email, Slack, or Teams.
- A dashboard or BI layer.
- Production deployment and secrets management.

The next milestone is **data quality**. Planned checks include:

```text
duplicate order_id → invalid
quantity <= 0       → invalid
price IS NULL       → invalid
product_id IS NULL  → invalid
revenue < 0         → invalid
```

The key lesson for the next stage is:

```text
Pipeline technically succeeded ≠ Data is necessarily correct
```

## Author

**Arjya Dey**
