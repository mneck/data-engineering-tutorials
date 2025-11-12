# Sample Data Files for AWS Data Lake Tutorial

This directory contains sample datasets for practicing AWS Data Lake concepts (S3 + Glue + Athena).

## Files Overview

### 1. sales_data.csv
**Use case:** E-commerce sales transactions
**Schema:**
- `id` (int): Unique transaction ID
- `customer` (string): Customer name
- `product` (string): Product purchased
- `amount` (int): Sale amount in USD
- `date` (string): Transaction date (YYYY-MM-DD)

**Sample queries:**
```sql
-- Top customers by revenue
SELECT customer, SUM(amount) as total_revenue
FROM sales_data
GROUP BY customer
ORDER BY total_revenue DESC;

-- Sales by product
SELECT product, COUNT(*) as order_count, SUM(amount) as revenue
FROM sales_data
GROUP BY product;
```

**Practice:** Upload to S3, create Glue crawler, query with Athena

---

### 2. customers.csv
**Use case:** Customer master data
**Schema:**
- `customer` (string): Customer name
- `email` (string): Email address
- `country` (string): Country of residence
- `signup_date` (string): Account creation date

**Sample queries:**
```sql
-- Join sales with customer data
SELECT s.customer, c.email, c.country, SUM(s.amount) as total_spent
FROM sales_data s
JOIN customers c ON s.customer = c.customer
GROUP BY s.customer, c.email, c.country
ORDER BY total_spent DESC;

-- Customers by country
SELECT country, COUNT(*) as customer_count
FROM customers
GROUP BY country;
```

**Practice:** Join two tables in Athena

---

### 3. products.csv
**Use case:** Product catalog
**Schema:**
- `product_id` (int): Unique product identifier
- `product_name` (string): Product name
- `category` (string): Product category
- `price` (int): Price in USD
- `stock_quantity` (int): Available inventory

**Sample queries:**
```sql
-- Products by category
SELECT category, COUNT(*) as product_count, AVG(price) as avg_price
FROM products
GROUP BY category;

-- Low stock alert
SELECT product_name, stock_quantity
FROM products
WHERE stock_quantity < 100
ORDER BY stock_quantity ASC;
```

---

### 4. clickstream_events.json
**Use case:** Web analytics / user behavior tracking
**Schema:**
- `event_id` (string): Unique event identifier
- `user_id` (string): User identifier
- `event_type` (string): Type of event (page_view, add_to_cart, purchase, search)
- `page` (string): Page URL (for page_view events)
- `product` (string): Product name (for cart/purchase events)
- `query` (string): Search query (for search events)
- `amount` (int): Purchase amount (for purchase events)
- `timestamp` (string): Event timestamp (ISO 8601)
- `session_id` (string): Session identifier
- `device` (string): Device type (desktop, mobile, tablet)
- `country` (string): User country

**Sample queries:**
```sql
-- Event funnel analysis
SELECT event_type, COUNT(*) as event_count
FROM clickstream_events
GROUP BY event_type
ORDER BY event_count DESC;

-- Conversion rate by device
SELECT device,
       SUM(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END) as views,
       SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) as purchases,
       CAST(SUM(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS DOUBLE) /
       SUM(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END) as conversion_rate
FROM clickstream_events
GROUP BY device;
```

**Practice:** Glue crawler with JSON, nested schema detection

---

### 5. server_logs.txt
**Use case:** Application logs / monitoring
**Format:** Space-delimited log format
**Fields:**
- Date (YYYY-MM-DD)
- Time (HH:MM:SS)
- Log level (INFO, WARNING, ERROR)
- IP address
- HTTP method (GET, POST)
- URL path
- HTTP status code
- Response time
- User ID
- Optional error message

**Sample queries:**
```sql
-- Error rate by hour
SELECT SUBSTR(timestamp, 1, 13) as hour,
       COUNT(*) as total_requests,
       SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) as errors
FROM server_logs
GROUP BY SUBSTR(timestamp, 1, 13)
ORDER BY hour;

-- Slowest endpoints
SELECT url_path, AVG(response_time) as avg_response_time
FROM server_logs
GROUP BY url_path
ORDER BY avg_response_time DESC;
```

**Practice:** Custom Glue classifier for log parsing, regex SerDe

---

## How to Use These Files

### Step 1: Upload to S3

**Option 1: AWS Console**
1. Open S3 Console
2. Create bucket: `my-datalake-raw-<your-name>`
3. Create folders: `sales/`, `customers/`, `products/`, `events/`, `logs/`
4. Upload respective files to each folder

**Option 2: AWS CLI**
```bash
# Create bucket
aws s3 mb s3://my-datalake-raw-yourname

# Upload all files
aws s3 cp sales_data.csv s3://my-datalake-raw-yourname/sales/
aws s3 cp customers.csv s3://my-datalake-raw-yourname/customers/
aws s3 cp products.csv s3://my-datalake-raw-yourname/products/
aws s3 cp clickstream_events.json s3://my-datalake-raw-yourname/events/
aws s3 cp server_logs.txt s3://my-datalake-raw-yourname/logs/
```

### Step 2: Create Glue Database

```sql
-- In Glue Console → Databases
CREATE DATABASE ecommerce_datalake;
```

### Step 3: Run Glue Crawlers

Create separate crawlers for each data source:
- `sales-crawler` → points to `s3://bucket/sales/`
- `customers-crawler` → points to `s3://bucket/customers/`
- `products-crawler` → points to `s3://bucket/products/`
- `events-crawler` → points to `s3://bucket/events/`
- `logs-crawler` → points to `s3://bucket/logs/`

**Why separate crawlers?** Different formats (CSV vs JSON vs logs) may need different classifiers.

### Step 4: Query with Athena

```sql
-- Set database
USE ecommerce_datalake;

-- Show tables
SHOW TABLES;

-- Query sales
SELECT * FROM sales LIMIT 10;

-- Join sales + customers
SELECT s.customer, c.email, SUM(s.amount) as total
FROM sales s
JOIN customers c ON s.customer = c.customer
GROUP BY s.customer, c.email;
```

---

## Practice Exercises

### Beginner
1. Upload `sales_data.csv` to S3
2. Create Glue crawler and run it
3. Query with Athena: Find total revenue per customer
4. Export results to CSV

### Intermediate
1. Upload all CSV files
2. Create crawlers for each dataset
3. Write a join query combining sales + customers + products
4. Partition sales data by month (reorganize S3 structure)

### Advanced
1. Upload JSON clickstream data
2. Configure Glue crawler to detect nested schema
3. Write funnel analysis query (page view → add to cart → purchase)
4. Convert JSON to Parquet using Glue ETL job
5. Compare query performance (JSON vs Parquet)

---

## Tips for Learning

### CSV Best Practices
- Always include header row
- Use consistent delimiters (comma for CSV)
- Avoid special characters in column names
- Quote fields containing delimiters

### JSON Best Practices
- Use JSON Lines format for large datasets (one JSON object per line)
- Keep nesting depth < 3 levels for better Athena performance
- Compress large JSON files (GZIP)

### Partitioning Strategy
Reorganize data by date for cost savings:
```
s3://bucket/sales/year=2025/month=01/day=15/data.csv
s3://bucket/sales/year=2025/month=01/day=16/data.csv
```

Query only specific partition:
```sql
SELECT * FROM sales
WHERE year=2025 AND month=01 AND day=15;
```

**Result:** Scan 1 day of data instead of entire dataset (huge cost savings).

---

## Data Generation

These sample files were created for educational purposes. To generate more data:

**Python script to generate sales data:**
```python
import csv
import random
from datetime import datetime, timedelta

customers = ['Alice', 'Bob', 'Charlie', 'David', 'Eve']
products = [('Laptop', 1200), ('Mouse', 25), ('Keyboard', 75), ('Monitor', 300)]

with open('sales_data.csv', 'w', newline='') as f:
    writer = csv.writer(f)
    writer.writerow(['id', 'customer', 'product', 'amount', 'date'])

    start_date = datetime(2025, 1, 1)
    for i in range(1, 101):  # Generate 100 rows
        customer = random.choice(customers)
        product, amount = random.choice(products)
        date = start_date + timedelta(days=random.randint(0, 60))
        writer.writerow([i, customer, product, amount, date.strftime('%Y-%m-%d')])
```

---

## License
These sample datasets are provided for educational use only. Feel free to modify and extend them for your learning.
