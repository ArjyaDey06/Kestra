# 🚀 Kestra E-Commerce Data Engineering Pipeline

A beginner-friendly, hands-on project for learning [Kestra](https://kestra.io/) by building an e-commerce data pipeline from ingestion to SQL analytics.

The project covers CSV and REST API ingestion, data cleaning, enrichment, task dependencies, SQLite loading, SQL analytics, scheduling, retries, failure handling, observability, and debugging.

> **Current checkpoint:** ETL, SQLite, SQL analytics, scheduling, retries, failure handling, artifact management, and observability are complete. Data quality checks are the next milestone.

## Table of contents

- [Project overview](#project-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Run Kestra locally](#run-kestra-locally)
- [Part 1: Build the enriched CSV pipeline](#part-1-build-the-enriched-csv-pipeline)
- [Part 2: Load SQLite and add reliability](#part-2-load-sqlite-and-add-reliability)
- [Current complete flow](#current-complete-flow)
- [Expected results](#expected-results)
- [Kestra concepts](#kestra-concepts)
- [Troubleshooting](#troubleshooting)
- [Next steps](#next-steps)
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
id,title,category,price
```

The pipeline joins `orders.product_id` with `products.id` and produces:

```text
order_id,product_id,quantity,title,category,price,revenue
```

## Architecture

```text
Orders CSV ───────────────┐
                          ▼
                    create_orders
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

Failures: retries → pipeline_failed → alert log
```

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

5. Open [http://localhost:8080](http://localhost:8080) and create the local administrator account.

# Part 1: Build the enriched CSV pipeline

Part 1 builds the Extract and Transform stages. Follow these steps in order if you want to reproduce the learning journey.

## 1. Create a first flow with log tasks

Before working with data, create a flow named `ecommerce_pipeline` in the namespace `dsa.dataengineering`:

```yaml
id: ecommerce_pipeline
namespace: dsa.dataengineering

tasks:
  - id: pipeline_started
    type: io.kestra.plugin.core.log.Log
    message: "E-commerce data pipeline started"

  - id: orders_received
    type: io.kestra.plugin.core.log.Log
    message: "500 new orders received for processing"

  - id: processing_complete
    type: io.kestra.plugin.core.log.Log
    message: "Order processing completed successfully"
```

Save the flow and select **Execute**. Open the execution and inspect the Gantt view and logs. This introduces the basic Kestra concepts: flow, task, execution, and logs.

## 2. Create the orders CSV

Replace the demonstration message with a real storage task:

```yaml
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
```

The input intentionally contains:

- Duplicate order `1002`.
- Invalid order `1005` with a negative quantity.

The task creates a CSV artifact that downstream tasks can consume through its output URI.

## 3. Clean orders with Python and Pandas

Add this task after `create_orders`:

```yaml
- id: clean_orders
  type: io.kestra.plugin.scripts.python.Script
  beforeCommands:
    - pip install pandas
  script: |
    import pandas as pd

    df = pd.read_csv("{{ outputs.create_orders.uri }}")

    print("RAW DATA")
    print(df)

    # Keep one record per order.
    df = df.drop_duplicates(subset=["order_id"])

    # Keep only valid positive quantities.
    df = df[df["quantity"] > 0]

    # Calculate order revenue.
    df["revenue"] = df["quantity"] * df["price"]

    df.to_csv("clean_orders.csv", index=False)

    print("\nCLEAN DATA")
    print(df)
    print("\nTOTAL REVENUE:", df["revenue"].sum())
  outputFiles:
    - "clean_orders.csv"
```

`beforeCommands` installs Pandas inside the script task. The expression below passes the file created by `create_orders` to Python:

```text
{{ outputs.create_orders.uri }}
```

The `outputFiles` declaration exposes `clean_orders.csv` as a Kestra-managed artifact.

## 4. Add analytics for the initial CSV schema

Before introducing the API, the CSV contains a `product` column. Add this task after `clean_orders`:

```yaml
- id: analyze_orders
  type: io.kestra.plugin.scripts.python.Script
  beforeCommands:
    - pip install pandas
  script: |
    import pandas as pd

    df = pd.read_csv(
        "{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}"
    )

    total_revenue = df["revenue"].sum()
    total_orders = len(df)
    average_order_value = total_revenue / total_orders
    top_product = df.loc[df["revenue"].idxmax(), "product"]

    print("===== SALES ANALYTICS =====")
    print("Total Orders:", total_orders)
    print("Total Revenue:", total_revenue)
    print("Average Order Value:", average_order_value)
    print("Highest Revenue Product:", top_product)
```

At this checkpoint, four valid orders remain. With the original manually supplied prices, the total revenue is `99,100` and the average order value is `24,775`.

## 5. Fetch product data from the REST API

The order source should not contain product details if we want to demonstrate enrichment. Add the API task:

```yaml
- id: fetch_product_data
  type: io.kestra.plugin.core.http.Request
  uri: https://dummyjson.com/products
  method: GET
```

This performs an HTTP `GET` request and stores the JSON response as a task output.

## 6. Inspect the API response with `jq`

Add a log task after `fetch_product_data`:

```yaml
- id: inspect_product_data
  type: io.kestra.plugin.core.log.Log
  message: |
    Product API successfully fetched.
    First product: {{ outputs.fetch_product_data.body | jq('.products[0].title') | first }}
    Category: {{ outputs.fetch_product_data.body | jq('.products[0].category') | first }}
    Price: ${{ outputs.fetch_product_data.body | jq('.products[0].price') | first }}
```

The response has a structure similar to:

```json
{
  "products": [
    {
      "id": 1,
      "title": "Essence Mascara Lash Princess",
      "category": "beauty",
      "price": 9.99
    }
  ]
}
```

The expression `.products[0].title` selects the title of the first product. Kestra's `jq` filter is used to extract the value.

## 7. Redesign the order source for enrichment

Replace the earlier order data with this version:

```yaml
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
```

The order system now knows only the order ID, product ID, and quantity. Product names, categories, and prices come from the API.

The two datasets share a join key:

```text
Orders:   product_id
Products: id
```

## 8. Join and enrich the two sources

Replace the earlier `clean_orders` task with this version:

```yaml
- id: clean_orders
  type: io.kestra.plugin.scripts.python.Script
  beforeCommands:
    - pip install pandas
  script: |
    import json
    import pandas as pd

    # Read orders from the Kestra output of create_orders.
    orders = pd.read_csv("{{ outputs.create_orders.uri }}")

    # Parse the API response and create a DataFrame.
    products_json = json.loads(r'''{{ outputs.fetch_product_data.body }}''')
    products = pd.DataFrame(products_json["products"])

    print("RAW ORDERS")
    print(orders)

    # Clean orders.
    orders = orders.drop_duplicates(subset=["order_id"])
    orders = orders[orders["quantity"] > 0]

    # Keep only the product fields required by this pipeline.
    products = products[["id", "title", "category", "price"]]

    # Rename the API key so it matches the orders key.
    products = products.rename(columns={"id": "product_id"})

    # Join orders with product data.
    enriched = orders.merge(
        products,
        on="product_id",
        how="left"
    )

    # Calculate revenue using the API price.
    enriched["revenue"] = (
        enriched["quantity"] * enriched["price"]
    )

    print("\nENRICHED ORDERS")
    print(enriched)

    enriched.to_csv("clean_orders.csv", index=False)
  outputFiles:
    - "clean_orders.csv"
```

This task performs parsing, cleaning, column selection, column renaming, joining, enrichment, revenue calculation, and artifact creation.

## 9. Update analytics for the enriched schema

The API calls the product-name field `title`, not `product`. Update the analytics task accordingly:

```yaml
- id: analyze_orders
  type: io.kestra.plugin.scripts.python.Script
  beforeCommands:
    - pip install pandas
  script: |
    import pandas as pd

    df = pd.read_csv(
        "{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}"
    )

    total_revenue = df["revenue"].sum()
    total_orders = len(df)
    average_order_value = total_revenue / total_orders

    top_product = df.loc[
        df["revenue"].idxmax(),
        "title"
    ]

    print("===== SALES ANALYTICS =====")
    print("Total Orders:", total_orders)
    print("Total Revenue:", total_revenue)
    print("Average Order Value:", average_order_value)
    print("Highest Revenue Product:", top_product)
```

The output dependency is:

```text
{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}
```

This explicitly tells Kestra to provide the file produced by `clean_orders` to `analyze_orders`.

## Part 1 troubleshooting lessons

### `Function or Macro [json] does not exist`

An initial inspection attempt used:

```text
{{ json(outputs.fetch_product_data.body).products[0].title }}
```

That failed because `json()` was not available in this expression context. Use:

```text
{{ outputs.fetch_product_data.body | jq('.products[0].title') | first }}
```

### `KeyError: 'product'`

After enrichment, the schema changed from `product` to `title`. Any downstream task still using `df["product"]` fails. Update it to `df["title"]` or standardize the schema before downstream processing.

# Part 2: Load SQLite and add reliability

Part 2 completes the Load phase and adds SQL analytics, scheduling, retries, failure handling, and artifact management.

## 1. Test SQLite first

Add this temporary task after `clean_orders`:

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
    rows = cursor.execute("SELECT * FROM test_table").fetchall()

    print("DATABASE RESULTS:")
    print(rows)
    connection.close()
```

Execute it and inspect the logs. Once SQLite works, remove this temporary task.

## 2. Load the enriched CSV into SQLite

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

`if_exists="replace"` is convenient during learning because every run recreates the table from the latest transformed data. Production pipelines may use append, upsert, merge, incremental loading, or CDC instead.

The database is declared as an output artifact so that later tasks can consume it:

```text
{{ outputs.load_to_database.outputFiles['ecommerce.db'] }}
```

## 3. Run SQL analytics

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

    cursor.execute("SELECT SUM(revenue) FROM orders")
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

This answers total revenue, revenue by category, and top product using `SUM`, `GROUP BY`, `ORDER BY`, aliases, and `LIMIT`.

## 4. Schedule the flow

Add `triggers` at the root level, alongside `tasks`:

```yaml
triggers:
  - id: every_two_minutes
    type: io.kestra.plugin.core.trigger.Schedule
    cron: "*/2 * * * *"
```

`*/2 * * * *` runs the flow every two minutes. Watch **Kestra → Executions** for automatic executions. Remove or comment out the trigger when testing intentional failures.

## 5. Add retries

Add a retry policy to the HTTP task:

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

This waits five seconds between attempts and allows three attempts. To test it, disable the schedule, temporarily use `https://dummyjson.com/this-does-not-exist`, execute manually, and watch the retries. Restore the real URL afterward.

> Use `maxAttempts`, not `maxAttempt`. The singular property causes a validation error before execution starts.

## 6. Add failure handling

Add `errors` at the root level alongside `tasks` and `triggers`:

```yaml
errors:
  - id: pipeline_failed
    type: io.kestra.plugin.core.log.Log
    message: |
      ALERT: E-commerce pipeline failed.
      Execution ID: {{ execution.id }}
      Please check the Kestra execution logs.
```

The failure lifecycle is:

```text
Task fails → retry → retry → retries exhausted → pipeline_failed
```

In production, this handler could send a notification, create an incident, or trigger a recovery workflow.

# Current complete flow

Use the following as the latest Part 2 checkpoint. It includes the Part 1 ingestion and transformation code plus SQLite, SQL analytics, retries, failure handling, and scheduling.

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

To run it, open the `dsa.dataengineering` namespace, create `ecommerce_pipeline`, paste the YAML, save it, and select **Execute**. Leave the trigger enabled to test automatic execution.

## Expected results

After cleaning:

- Duplicate order `1002` is reduced to one row.
- Invalid order `1005` is removed.
- Four valid orders remain.
- Product details are joined from the API.
- Revenue is calculated using quantity and API price.
- The enriched records are stored in SQLite as the `orders` table.
- SQL analytics prints total revenue, revenue by category, and the top product.

Exact revenue values may change because product prices come from an external API.

## Kestra concepts demonstrated

- **Flow:** the complete workflow.
- **Task:** one operation in the workflow.
- **Execution:** one run of the flow.
- **Outputs:** artifacts passed between tasks.
- **Orchestration:** task order, dependencies, schedules, and failure behavior.
- **Observability:** logs, execution status, outputs, and the Gantt view.
- **ETL:** extract from CSV/API, transform with Python/Pandas, and load into SQLite.

Important output references used in the flow:

```text
{{ outputs.create_orders.uri }}
{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}
{{ outputs.load_to_database.outputFiles['ecommerce.db'] }}
```

## Troubleshooting

### `Function or Macro [json] does not exist`

Use `jq` in Kestra expressions instead of the unavailable `json()` function.

### `KeyError: 'product'`

The enriched schema uses `title`, not `product`. Update downstream code when upstream schemas change.

### `Unrecognized field "maxAttempt"`

Use the plural property:

```yaml
maxAttempts: 3
```

A validation error happens before execution. A runtime error happens after execution starts.

### SQLite file is unavailable downstream

Declare the database as an output:

```yaml
outputFiles:
  - "ecommerce.db"
```

Then reference it with:

```text
{{ outputs.load_to_database.outputFiles['ecommerce.db'] }}
```

## Next steps

The next milestone is **data quality**. Planned checks include:

```text
duplicate order_id → invalid
quantity <= 0       → invalid
price IS NULL       → invalid
product_id IS NULL  → invalid
revenue < 0         → invalid
```

The next lesson is:

```text
Pipeline technically succeeded ≠ Data is necessarily correct
```

Future improvements include incremental loading, notifications, a dashboard or BI layer, secrets management, and production deployment.

## Author

**Arjya Dey**
