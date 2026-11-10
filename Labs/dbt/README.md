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
   source venv/bin/activate  # On Windows, use: venv\Scripts\activate
   ```

3. Install `dbt-duckdb`:
   ```bash
   pip install -r requirements.txt dbt-duckdb
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

dbt uses a `profiles.yml` file to know how to connect to your database. By default, dbt looks in `~/.dbt/profiles.yml`. However, for this lab, we'll keep it local.

1. Create a file named `profiles.yml` **inside** your `my_first_project` directory with the following content:

   ```yaml
   dbt_lab:
     outputs:
       dev:
         type: duckdb
         path: local_database.duckdb
     target: dev
   ```

2. Open the existing `dbt_project.yml` file in the root of your project and tell dbt to look for the profile in the current directory. Add/modify these lines at the top:

   ```yaml
   name: 'dbt_lab'
   version: '1.0.0'
   config-version: 2
   profile: 'dbt_lab'
   ```

3. Test your connection:
   ```bash
   DBT_PROFILES_DIR=. dbt debug
   ```
   *Note: Prepending `DBT_PROFILES_DIR=.` tells dbt to use the local profile we just made. All tests should pass!*

---

## Step 4: Add Raw Data (Seeds)

dbt can load CSV files into your database using a feature called "seeds".

1. Inside the `seeds` folder, create a file named `raw_orders.csv`.
2. Add the following sample data:

   ```csv
   order_id,customer_id,order_date,status,amount
   1,101,2023-01-15,completed,50.00
   2,102,2023-01-16,completed,125.50
   3,101,2023-01-18,returned,50.00
   4,103,2023-02-02,completed,200.00
   5,102,2023-02-10,completed,75.25
   ```

3. Load this data into your DuckDB database:
   ```bash
   DBT_PROFILES_DIR=. dbt seed
   ```

---

## Step 5: Build Your First Models

In dbt, models are simply SQL `SELECT` statements. We will create two models: a staging model to clean up the raw data, and a mart model to aggregate it.

1. Inside the `models` directory, delete the `example` folder to keep things clean.
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

1. Run the models:
   ```bash
   DBT_PROFILES_DIR=. dbt run
   ```
   *You should see green text indicating that `stg_orders` and `mart_monthly_revenue` were created successfully.*

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
   DBT_PROFILES_DIR=. dbt test
   ```

## Conclusion

Congratulations! You have successfully:
1. Configured a dbt project.
2. Seeded raw data into a local database.
3. Built a staging model (cleaning data).
4. Built a mart model (aggregating data).
5. Tested your data quality.