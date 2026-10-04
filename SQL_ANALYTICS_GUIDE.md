# SQL Analytics Guide

This guide demonstrates SQL queries for business insights from the e-commerce pipeline.

## Essential SQL Patterns for Data Analysis

### 1. Revenue Analysis

```sql
-- Total revenue across all orders
SELECT SUM(revenue) AS total_revenue FROM orders;

-- Revenue by category (GROUP BY)
SELECT category, SUM(revenue) AS category_revenue
FROM orders
GROUP BY category
ORDER BY category_revenue DESC;

-- Revenue by product (AGGREGATION + SORTING)
SELECT title, COUNT(*) AS order_count, SUM(revenue) AS product_revenue
FROM orders
GROUP BY title
ORDER BY product_revenue DESC
LIMIT 10;
```

### 2. Data Quality Checks

```sql
-- Detect duplicate orders
SELECT order_id, COUNT(*) AS occurrence_count
FROM orders
GROUP BY order_id
HAVING COUNT(*) > 1;

-- Check for NULL values
SELECT * FROM orders
WHERE product_id IS NULL OR quantity IS NULL OR price IS NULL;

-- Validate quantity and revenue
SELECT * FROM orders
WHERE quantity <= 0 OR revenue < 0;
```

### 3. Customer Behavior Analysis

```sql
-- Orders per category
SELECT category, COUNT(*) AS total_orders
FROM orders
GROUP BY category;

-- Average order value by category
SELECT category, AVG(revenue) AS avg_order_value
FROM orders
GROUP BY category
ORDER BY avg_order_value DESC;

-- Top 5 products by revenue
SELECT title, SUM(revenue) AS total_revenue, SUM(quantity) AS units_sold
FROM orders
GROUP BY title
ORDER BY total_revenue DESC
LIMIT 5;
```

### 4. Filtering & Conditions (WHERE clause)

```sql
-- High-value orders (>$100)
SELECT order_id, title, revenue
FROM orders
WHERE revenue > 100
ORDER BY revenue DESC;

-- Category-specific analysis
SELECT title, SUM(revenue) AS category_revenue
FROM orders
WHERE category = 'beauty'
GROUP BY title
ORDER BY category_revenue DESC;
```

### 5. Date-based Analysis (when timestamps are added)

```sql
-- Daily revenue trend
SELECT DATE(order_date) AS date, SUM(revenue) AS daily_revenue
FROM orders
GROUP BY DATE(order_date)
ORDER BY date DESC;

-- Week-over-week comparison
SELECT 
  STRFTIME('%Y-W%W', order_date) AS week,
  SUM(revenue) AS weekly_revenue
FROM orders
GROUP BY week
ORDER BY week DESC;
```

## Business Metrics Derived from SQL

| Metric | SQL Query | Business Value |
|--------|-----------|-----------------|
| Total Revenue | `SUM(revenue)` | Overall business performance |
| Avg Order Value | `AVG(revenue)` | Customer spending patterns |
| Units Sold | `SUM(quantity)` | Volume insights |
| Order Count | `COUNT(*)` | Transaction frequency |
| Top Products | `ORDER BY revenue DESC LIMIT 5` | Focus inventory & marketing |

## Common Interview Patterns

### Pattern 1: Aggregation + Filtering
```sql
SELECT category, SUM(revenue) 
FROM orders 
WHERE quantity > 0 
GROUP BY category
ORDER BY SUM(revenue) DESC;
```

### Pattern 2: Multiple Joins (when normalized)
```sql
-- When schema includes separate tables
SELECT o.order_id, p.title, o.quantity, p.price, (o.quantity * p.price) AS revenue
FROM orders o
JOIN products p ON o.product_id = p.id
WHERE o.quantity > 0;
```

### Pattern 3: Window Functions (advanced)
```sql
-- Rank products by revenue
SELECT 
  title, 
  SUM(revenue) AS product_revenue,
  ROW_NUMBER() OVER (ORDER BY SUM(revenue) DESC) AS rank
FROM orders
GROUP BY title;
```

## Key SQL Concepts Demonstrated

✓ **Aggregation**: SUM(), COUNT(), AVG()  
✓ **Grouping**: GROUP BY, HAVING  
✓ **Filtering**: WHERE clauses  
✓ **Sorting**: ORDER BY, DESC/ASC  
✓ **Subqueries**: Nested SELECT statements  
✓ **Data Validation**: NULL checks, range checks  

---

Run these queries against `ecommerce.db` to practice SQL fundamentals.
