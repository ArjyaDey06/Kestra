# Kestra E-Commerce Data Engineering Pipeline

A beginner-friendly, hands-on project for learning [Kestra](https://kestra.io/) by building an end-to-end e-commerce ETL pipeline.

The project demonstrates workflow orchestration, CSV and REST API ingestion, data cleaning, multi-source joins, enrichment, analytics, observability, and debugging.

> **Current status:** The pipeline currently extracts data, transforms it with Python and Pandas, and writes an enriched CSV artifact. SQL loading, scheduling, retries, notifications, and production deployment are planned next.

## Table of contents

- [Project goal](#project-goal)
- [Architecture](#architecture)
- [Learning path](#learning-path)
- [Prerequisites](#prerequisites)
- [Run Kestra locally](#run-kestra-locally)
- [Run the pipeline](#run-the-pipeline)
- [Current working flow](#current-working-flow)
- [Expected behavior](#expected-behavior)
- [Kestra concepts](#kestra-concepts)
- [Troubleshooting lessons](#troubleshooting-lessons)
- [ETL perspective](#etl-perspective)
- [Limitations and next steps](#limitations-and-next-steps)

## Project goal

We are acting as data engineers for an e-commerce company with two data sources:

### Source 1: Order data

The order source contains:

```text
order_id, product_id, quantity
```

### Source 2: Product REST API

Product details come from the [DummyJSON Products API](https://dummyjson.com/products):

```text
id, title, category, price
```

The pipeline uses `product_id` from the orders and `id` from the API as the common join key.

## Architecture

```text
                         ┌─────────────────────┐
                         │  Orders CSV source   │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    create_orders    │
                         └──────────┬──────────┘
                                    │
                                    │
┌─────────────────────┐             │
│  DummyJSON REST API │             │
└──────────┬──────────┘             │
           ▼                        │
┌─────────────────────┐             │
│ fetch_product_data  │             │
└──────────┬──────────┘             │
           ▼                        │
┌─────────────────────┐             │
│ inspect_product_data│             │
└──────────┬──────────┘             │
           └──────────────┬─────────┘
                          ▼
                 ┌─────────────────┐
                 │   clean_orders  │
                 │                 │
                 │ • deduplicate   │
                 │ • validate      │
                 │ • join          │
                 │ • enrich        │
                 │ • calculate     │
                 │   revenue       │
                 └────────┬────────┘
                          ▼
                 ┌─────────────────┐
                 │ clean_orders.csv│
                 └────────┬────────┘
                          ▼
                 ┌─────────────────┐
                 │  analyze_orders │
                 └────────┬────────┘
                          ▼
                 ┌─────────────────┐
                 │processing_complete│
                 └─────────────────┘
```

## Learning path

The project was built incrementally. Each stage introduces one data-engineering or orchestration concept:

1. **Hello Kestra** — create a flow with simple log tasks.
2. **Create actual data** — generate an orders CSV with `storage.Write`.
3. **Introduce data-quality problems** — add a duplicate order and an invalid quantity.
4. **Use Python and Pandas** — read and transform task outputs.
5. **Clean the data** — remove duplicates and invalid records.
6. **Pass outputs between tasks** — use Kestra expressions to connect tasks.
7. **Create a data artifact** — write `clean_orders.csv` as an output file.
8. **Calculate analytics** — compute order count, revenue, average order value, and top product.
9. **Ingest a REST API** — fetch product details from DummyJSON.
10. **Inspect JSON** — extract API fields with `jq`.
11. **Join multiple sources** — match `product_id` to the API's `id`.
12. **Enrich and calculate** — add product details and calculate revenue using the API price.
13. **Debug failures** — fix an invalid Kestra expression and a Pandas schema mismatch.

The current checkpoint is an end-to-end enriched analytics pipeline.

## Prerequisites

Only a small local setup is required:

- Windows
- Docker Desktop
- Docker Desktop configured to use the WSL 2 backend
- Internet access for the DummyJSON API request

A Docker account/sign-in is not required for this local setup.

## Run Kestra locally

1. Install and start [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/).
2. Wait until the Docker Engine is running.
3. Confirm Docker is available from PowerShell:

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

## Run the pipeline

In the Kestra UI:

1. Create or open the namespace `dsa.dataengineering`.
2. Create a flow with the ID `ecommerce_pipeline`.
3. Paste the [complete flow](#current-working-flow) below into the editor.
4. Save the flow.
5. Select **Execute**.
6. Open the execution to inspect task status, logs, outputs, and the Gantt view.

## Current working flow

This is the current checkpoint version of the pipeline.

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

      print("RAW ORDERS")
      print(orders)

      # Remove duplicate orders and invalid quantities.
      orders = orders.drop_duplicates(subset=["order_id"])
      orders = orders[orders["quantity"] > 0]

      # Keep and standardize the product fields needed for the join.
      products = products[["id", "title", "category", "price"]]
      products = products.rename(columns={"id": "product_id"})

      # Enrich orders with product details.
      enriched = orders.merge(products, on="product_id", how="left")
      enriched["revenue"] = enriched["quantity"] * enriched["price"]

      print("\nENRICHED ORDERS")
      print(enriched)

      enriched.to_csv("clean_orders.csv", index=False)
    outputFiles:
      - "clean_orders.csv"

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
      top_product = df.loc[df["revenue"].idxmax(), "title"]

      print("===== SALES ANALYTICS =====")
      print("Total Orders:", total_orders)
      print("Total Revenue:", total_revenue)
      print("Average Order Value:", average_order_value)
      print("Highest Revenue Product:", top_product)

  - id: processing_complete
    type: io.kestra.plugin.core.log.Log
    message: "E-commerce data pipeline completed successfully."
```

## Expected behavior

The sample input intentionally includes:

- A duplicate record for order `1002`.
- An invalid record for order `1005` with a quantity of `-2`.

The `clean_orders` task:

1. Removes the duplicate order.
2. Removes the invalid order.
3. Joins the remaining orders to product data from the API.
4. Calculates `revenue` as `quantity * price`.
5. Writes the enriched data to `clean_orders.csv`.

The final dataset has this schema:

```text
order_id,product_id,quantity,title,category,price,revenue
```

The output contains four valid orders. Because product prices are retrieved from the API at execution time, exact revenue totals may change if the API data changes.

## Kestra concepts

### Flows, tasks, executions, and logs

- A **flow** defines the complete workflow.
- A **task** performs one operation in the flow.
- An **execution** is one run of the flow.
- **Logs** and the **Gantt view** show what ran, what succeeded, and where a failure occurred.

### Passing task outputs

Kestra makes task results available to downstream tasks through expressions:

```text
{{ outputs.create_orders.uri }}
{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}
```

This connects tasks without requiring separate local file-management steps.

### Multi-source enrichment

The orders and product API have different schemas:

```text
Orders:   order_id, product_id, quantity
Products: id, title, category, price
```

The pipeline renames the API's `id` column to `product_id`, then performs a Pandas left join.

### Data quality

The pipeline demonstrates two simple validation rules:

- Keep one record per `order_id`.
- Keep only rows where `quantity > 0`.

### ETL

- **Extract:** create order data and fetch product data from the REST API.
- **Transform:** parse JSON, validate, deduplicate, join, enrich, and calculate revenue.
- **Load:** write the transformed result to `clean_orders.csv`.

## Troubleshooting lessons

### Kestra expression error: `Function or Macro [json] does not exist`

The first API inspection attempt used:

```text
{{ json(outputs.fetch_product_data.body).products[0].title }}
```

That expression failed because `json()` was not available in this flow. The working expression uses `jq`:

```text
{{ outputs.fetch_product_data.body | jq('.products[0].title') | first }}
```

### Pandas error: `KeyError: 'product'`

The first version of the pipeline used a `product` column in the orders CSV. After moving product data to the API, the API provided the product name as `title` instead.

The analytics task still used `df["product"]`, so Pandas raised `KeyError: 'product'`. Updating it to `df["title"]` fixed the schema mismatch.

This illustrates an important production lesson: whenever an upstream schema changes, review every downstream task that consumes it.

## Limitations and next steps

The current version is an educational local pipeline. It does not yet include:

- Loading data into a SQL database.
- Scheduled automatic executions.
- Retries and structured failure handling.
- Notifications.
- A dashboard or BI layer.
- Production deployment.

Planned progression:

```text
Current:  Extract → Transform → CSV artifact

Next:     Extract → Transform → Load to SQL database

Later:    Scheduling → Retries → Failure handling → Notifications → BI
```

This README will grow as additional pipeline features and learning examples are added.
