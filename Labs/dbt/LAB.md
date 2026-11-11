# Hands-On Lab: Getting Started with dbt (Data Build Tool)

This minimal lab will walk you through setting up a dbt project, loading raw data, and building your first data transformations. We will use **DuckDB** as the database because it runs locally and requires zero setup.

## Prerequisites

*   Python installed on your machine (Python 3.10+ recommended).
*   A terminal / command prompt.

---

## Step 1: Environment Setup

First, let's create an isolated Python environment and install dbt along with the DuckDB adapter.

1. Create and activate a virtual environment:
   ```bash
   python -m venv venv
   ```
   ```bash
   source venv/bin/activate
   ```

3. Install `dbt` utilities (`dbt-core`) and the DuckDB adapter (`dbt-duckdb`):
   ```bash
   pip install -r requirements.txt
   ```

---

## Step 2: Initialize Your dbt Project

Now we will create the scaffolding for our dbt project.

1. Run the initialization command:
   ```bash
   dbt init dbt_lab
   ```
2. When prompted for which database you would like to use, type the number corresponding to `duckdb` (usually `1`).
3. Navigate into your new project folder:
   ```bash
   cd dbt_lab
   ```

---

## Step 3: Configure Your Profile

dbt uses a `profiles.yml` file to know how to connect to your database. By default, dbt looks in the dbt project directory. If not found there, it looks in `~/.dbt/profiles.yml` file. For this lab, we'll keep it local (in the dbt project directory).

1. Create a file named `profiles.yml` **inside** your `dbt_lab/` directory with the following content:

   ```yaml
   dbt_lab:
     outputs:
       dev:
         type: duckdb
         path: local_database.duckdb
         schema: orders
     target: dev
   ```

2. Open the existing `dbt_project.yml` file in the root of your project and modify the models configuration:

    Current configuration:
    ```yaml
    models:
        dbt_lab:
            # Config indicated by + and applies to all files under models/example/
            example:
            +materialized: view
    ```

    New configuration:
    ```yaml
    models:
        dbt_lab:
            +materialized: table
            mart_monthly_revenue:
                +materialized: view
    ```

3. Test your connection:
   ```bash
   dbt debug
   ```

---

## Step 4: Add Raw Data (Seeds)

dbt can load CSV files into your database using a feature called "seeds".

1. Copy the file `raw_orders.csv` from the `data/` directory into your `dbt_lab/seeds/` directory.

2. Load this data into your DuckDB database:
   ```bash
   dbt seed
   ```

---

## Step 5: Build Your First Models

In dbt, models are simply SQL `SELECT` statements. We will create two models: a staging model to clean up the raw data, and a mart model to aggregate it.

1. Inside the `models` directory, delete the `example/` directory to keep things clean.
2. Create a new file: `models/stg_orders.sql`. Add this code:

   ```sql
   -- models/stg_orders.sql
   with source as (
       select * from {{ ref('raw_orders') }}
   ),

   renamed as (
       select
           order_id,
           customer_id,
           order_date::date as order_date,
           status as order_status,
           amount::decimal(10,2) as order_amount
       from source
   )

   select * from renamed
   ```

3. Create another file for aggregation: `models/mart_monthly_revenue.sql`. Add this code:

   ```sql
   -- models/mart_monthly_revenue.sql
   select
       date_trunc('month', order_date) as order_month,
       sum(order_amount) as total_revenue,
       count(order_id) as total_orders
   from {{ ref('stg_orders') }}
   where order_status = 'completed'
   group by 1
   ```

---

## Step 6: Run Your Transformations

Now we will tell dbt to execute the SQL and build the tables/views in the database.

Run the models:
   ```bash
   dbt run
   ```
   >*You should see green text indicating that `stg_orders` and `mart_monthly_revenue` were created successfully.*

---

## Step 7: Testing (Optional but Recommended)

Let's add a quick test to ensure our `order_id` is unique and never null.

1. Create a file named `models/schema.yml` and add:

   ```yaml
   version: 2

   models:
     - name: stg_orders
       columns:
         - name: order_id
           tests:
             - unique
             - not_null
   ```

2. Run the tests:
   ```bash
   dbt test
   ```

## Step 8: Documentation (Optional but Recommended)

In this section, we will generate a documentation which will describe our data and data pipeline.

1. Generate the documentation:
   ```bash
   dbt docs generate
   ```
   >*You should see green text indicating that `docs` were created successfully.*

2. Open the generated documentation in the browser:
   ```bash
   dbt docs serve
   ```


## Conclusion

Congratulations! You have successfully:
1. Configured a dbt project.
2. Seeded raw data into a local database.
3. Built a staging model (cleaning data).
4. Built a mart model (aggregating data).
5. Tested your data quality.
6. Generated documentation and observed the data lineage.