# Order ETL and Data Quality Pipeline

A batch ETL pipeline for e-commerce order data, built with Python, Pandas and SQLite in Google Colab. It validates the source data before loading, so bad records are caught and quarantined instead of silently reaching the final tables.

## What it does

1. **Extract:** reads three tables from the Olist Brazilian E-Commerce dataset (orders, order items, customers)
2. **Validate:** runs 11 data quality checks across the three tables and saves a PASS/FAIL report
3. **Transform:** quarantines problem records and builds a `fact_orders` table (order + customer location + item totals + delivery time in days)
4. **Load:** writes `fact_orders` and `orders_rejected` into SQLite

## Data quality checks

| Check | What it looks for | Why it matters |
|---|---|---|
| Null key columns | Missing order_id, customer_id, product_id, price | Missing keys break joins |
| Duplicate keys | Repeated order_id, customer_id, or (order_id, order_item_id) | Inflates counts and revenue |
| Orphan records | Orders without a customer, items without an order | Rows silently disappear in joins |
| Negative amounts | Negative price or freight value | Corrupts revenue totals |
| Delivered before purchase | Delivery date earlier than purchase date | Breaks delivery-time metrics |
| Delivered without date | Status "delivered" but no delivery date | Delivery-time metrics silently skip these rows |

Each check records how many problem rows it found and saves those rows to a reports folder for inspection.

## Results

- 11 checks run: 10 passed, 1 failed (8 issues in total)
- 8 orders have the status "delivered" but no delivery date. They are kept in an `orders_rejected` table instead of being dropped, so they can be reviewed or sent back to the source system.
- 99,433 clean orders were loaded into `fact_orders` (99,441 source orders minus 8 quarantined)
- Sanity query on the loaded data: states in the north of Brazil show the longest average delivery times

## Design notes

- **Quarantine, don't delete:** problem records are separated and kept, not removed silently
- **Row-count assertions:** the row count is checked after the join and after the load to catch silent data loss or duplicated rows
- **Re-runnable load:** target tables are replaced on each run, so running the pipeline again does not create duplicates
- **Aggregate before join:** order items are summed per order before joining, so orders with several items are not duplicated

## Tech stack

Python, Pandas, SQLite, Google Colab

## How to run

1. Download the dataset from Kaggle: [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
2. Put these files in a `data` folder in your Google Drive:
   - `olist_orders_dataset.csv`
   - `olist_order_items_dataset.csv`
   - `olist_customers_dataset.csv`
3. Open `order_etl_pipeline.ipynb` in Google Colab
4. Set `PROJECT_DIR` to your Drive folder and run all cells in order

The dataset is not included in this repository. See the Kaggle page for its license and terms.

## Next steps

- Add pytest tests for the data quality checks
- Schedule the pipeline with Apache Airflow
