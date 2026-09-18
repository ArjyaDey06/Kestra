# E-commerce Data Engineering Pipeline with Kestra

This project is a beginner-friendly, hands-on example of building an ETL
pipeline with [Kestra](https://kestra.io/). It demonstrates how a workflow can
ingest data from multiple sources, clean and enrich it with Python and Pandas,
and produce useful sales analytics.

## Project overview

The pipeline combines:

- Order data created as a CSV file inside Kestra.
- Product data fetched from the [DummyJSON Products API](https://dummyjson.com/products).

The order source contains only the order ID, product ID, and quantity. Product
details such as the title, category, and price are retrieved from the API and
joined to the orders during the transformation step.

```text
Create orders (CSV) ───────────────┐
                                   │
Fetch products (REST API) ────────┤
                                   ▼
                         Clean and validate orders
                                   │
                                   ▼
                         Join and enrich the data
                                   │
                                   ▼
                         Calculate order revenue
                                   │
                                   ▼
                         Write clean_orders.csv
                                   │
                                   ▼
                           Generate analytics
```

## Numbered workflow progression

The project was built incrementally. The following checkpoints preserve the
workflow YAML in the same order as the original learning journey.

### 5. First Kestra flow: log tasks

This first flow was used to learn how a flow, tasks, executions, and logs work.

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

### 6. Create the initial orders CSV

The log-only `orders_received` task was replaced with a storage task that
creates actual CSV data.

```yaml
- id: create_orders
  type: io.kestra.plugin.core.storage.Write
  content: |
    order_id,product,quantity,price
    1001,Laptop,1,65000
    1002,Mouse,2,800
    1002,Mouse,2,800
    1003,Keyboard,1,2500
    1004,Monitor,2,15000
    1005,Webcam,-1,3500
  extension: .csv
```

This version deliberately contains a duplicate order and an invalid negative
quantity so that the cleaning step has real data-quality problems to solve.

### 9. Add Python and Pandas cleaning

The next version passed the storage output to a Python task. It removed
duplicates and invalid quantities, calculated revenue, and exposed the
resulting CSV as a task output.

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

    df = df.drop_duplicates(subset=["order_id"])
    df = df[df["quantity"] > 0]
    df["revenue"] = df["quantity"] * df["price"]
    df.to_csv("clean_orders.csv", index=False)

    print("\nCLEAN DATA")
    print(df)
    print("\nTOTAL REVENUE:", df["revenue"].sum())
    print("\nCleaned CSV successfully created!")
  outputFiles:
    - "clean_orders.csv"
```

### 15. Add sales analytics

The analytics task consumed the output file from `clean_orders`.

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

At this checkpoint, the sample result was four valid orders, total revenue of
`99,100`, and an average order value of `24,775`.

### 17. Update the completion message

After the analytics task was added, the final log task was updated to make the
successful end state explicit:

```yaml
- id: processing_complete
  type: io.kestra.plugin.core.log.Log
  message: "E-commerce data pipeline completed successfully."
```

### 18–19. Ingest and inspect product data from a REST API

The pipeline then added the DummyJSON Products API as a second source.

```yaml
- id: fetch_product_data
  type: io.kestra.plugin.core.http.Request
  uri: https://dummyjson.com/products
  method: GET
```

The first inspection attempt used the following YAML and failed because
Kestra does not provide the `json()` function used in the message:

```yaml
- id: inspect_product_data
  type: io.kestra.plugin.core.log.Log
  message: |
    Product API successfully fetched.
    First product: {{ json(outputs.fetch_product_data.body).products[0].title }}
    Category: {{ json(outputs.fetch_product_data.body).products[0].category }}
    Price: ${{ json(outputs.fetch_product_data.body).products[0].price }}
```

The response was inspected with a separate log task:

```yaml
- id: inspect_product_data
  type: io.kestra.plugin.core.log.Log
  message: |
    Product API successfully fetched.
    First product: {{ outputs.fetch_product_data.body | jq('.products[0].title') | first }}
    Category: {{ outputs.fetch_product_data.body | jq('.products[0].category') | first }}
    Price: ${{ outputs.fetch_product_data.body | jq('.products[0].price') | first }}
```

The initial attempt used `json(...)` in the Kestra expression and failed with
`Function or Macro [json] does not exist`. The working version uses `jq(...)`.

### 23. Redesign the order source for enrichment

To make the second source useful, the order data was changed to contain only
the order ID, product ID, and quantity:

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

Product names, categories, and prices now come from the API rather than the
orders source.

### 25–31. Join and enrich the two sources

The cleaning task was updated to parse the API response, align the key names,
join the two datasets, calculate revenue using the API price, and create the
enriched CSV.

```yaml
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

    orders = orders.drop_duplicates(subset=["order_id"])
    orders = orders[orders["quantity"] > 0]

    products = products[["id", "title", "category", "price"]]
    products = products.rename(columns={"id": "product_id"})

    enriched = orders.merge(
        products,
        on="product_id",
        how="left"
    )
    enriched["revenue"] = enriched["quantity"] * enriched["price"]

    print("\nENRICHED ORDERS")
    print(enriched)
    enriched.to_csv("clean_orders.csv", index=False)
  outputFiles:
    - "clean_orders.csv"
```

### 32–33. Update analytics for the enriched schema

The API calls the product-name field `title`, so the analytics task was
updated accordingly:

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
    top_product = df.loc[df["revenue"].idxmax(), "title"]

    print("===== SALES ANALYTICS =====")
    print("Total Orders:", total_orders)
    print("Total Revenue:", total_revenue)
    print("Average Order Value:", average_order_value)
    print("Highest Revenue Product:", top_product)
```

This change fixed the `KeyError: 'product'` caused by the upstream schema
changing from `product` to `title`.

## What the pipeline does

The current flow runs these tasks in order:

1. `pipeline_started` logs the beginning of the workflow.
2. `create_orders` creates sample order data with a deliberate duplicate and
   an invalid negative quantity.
3. `fetch_product_data` makes a `GET` request to the DummyJSON API.
4. `inspect_product_data` extracts and logs sample fields from the API
   response.
5. `clean_orders`:
   - removes duplicate orders;
   - removes orders with a quantity less than or equal to zero;
   - selects the required product fields;
   - joins orders with product data using `product_id`;
   - calculates `revenue`;
   - writes `clean_orders.csv`.
6. `analyze_orders` calculates total orders, total revenue, average order
   value, and the product with the highest revenue.
7. `processing_complete` logs a successful completion message.

The resulting dataset has the following columns:

```text
order_id, product_id, quantity, title, category, price, revenue
```

## Prerequisites

Only a small local setup is required:

- Windows with Docker Desktop
- Docker Desktop configured to use the WSL 2 backend
- Internet access when the flow calls the DummyJSON API

A Docker account is not required for this local setup.

## Run Kestra locally

1. Install and start [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/).
2. Open PowerShell.
3. Start Kestra with:

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

4. Open [http://localhost:8080](http://localhost:8080).
5. Create the local administrator account when prompted.

You can confirm that Docker is available from PowerShell with:

```powershell
docker --version
```

## 36. Current complete working flow

This is the complete checkpoint flow after the previous incremental steps.
Use this version for execution.

In the Kestra user interface:

1. Create or open the namespace `dsa.dataengineering`.
2. Create a flow with the ID `ecommerce_pipeline`.
3. Paste the YAML below into the flow editor.
4. Save the flow.
5. Select **Execute**.
6. Open the execution and inspect task logs, outputs, and the Gantt view.

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

      orders = orders.drop_duplicates(subset=["order_id"])
      orders = orders[orders["quantity"] > 0]

      products = products[["id", "title", "category", "price"]]
      products = products.rename(columns={"id": "product_id"})

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

## Expected data quality behavior

The sample input intentionally includes:

- A duplicate record for order `1002`.
- An invalid record for order `1005` with a quantity of `-2`.

After cleaning, the duplicate is reduced to one record and the invalid order
is removed. The final output therefore contains four valid orders. Revenue is
calculated using the price returned by the product API, so the exact totals
depend on the API response at execution time.

## Kestra concepts demonstrated

### Flows, tasks, and executions

- A **flow** defines the complete workflow.
- A **task** performs one operation in the flow.
- An **execution** is one run of the flow.
- **Logs** and the Gantt view show what ran, what succeeded, and where a
  failure occurred.

### Passing task outputs

Kestra makes task results available to downstream tasks through expressions:

```text
{{ outputs.create_orders.uri }}
{{ outputs.clean_orders.outputFiles['clean_orders.csv'] }}
```

This connects the tasks without requiring a separate local file-management
step.

### Multi-source enrichment

The pipeline uses `product_id` as the common key between the orders and API
data. The API's `id` field is renamed to `product_id` before the Pandas
left-join.

### ETL

- **Extract:** create order data and fetch product data from the REST API.
- **Transform:** validate, deduplicate, join, enrich, and calculate revenue.
- **Load:** currently write the transformed result to `clean_orders.csv`.

## Troubleshooting lessons

### Use `jq` to inspect JSON in Kestra expressions

Kestra expression syntax does not provide a generic `json()` function in this
flow. To read fields from the HTTP response, use `jq`, for example:

```text
{{ outputs.fetch_product_data.body | jq('.products[0].title') | first }}
```

### Keep downstream schemas synchronized

The API calls the product-name field `title`, not `product`. If downstream
analytics code still references `df["product"]`, Pandas raises:

```text
KeyError: 'product'
```

When an upstream task changes its schema, review every downstream task that
consumes its output.

## Current limitations and next steps

The current version is an educational local pipeline. It does not yet include:

- Loading data into a SQL database.
- Scheduled executions.
- Retries and structured failure handling.
- Notifications.
- A dashboard or BI layer.
- Production deployment.

The next stage is to add a database load step, followed by scheduling,
retries, failure handling, and operational notifications.

> This README will continue to grow as more pipeline features, examples, and
> lessons are added.
