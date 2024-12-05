
# IKEA Retail Sales SQL Project

![Project Banner Placeholder](https://upload.wikimedia.org/wikipedia/commons/thumb/c/c5/Ikea_logo.svg/1920px-Ikea_logo.svg.png)

Welcome to the **IKEA Retail Sales SQL Project**! This project leverages a detailed dataset of millions of sales records, product inventory, and store information across IKEA's global operations. The analysis focuses on uncovering sales trends, product performance, and inventory management insights to assist in data-driven decision-making.

---

## Table of Contents
- [Introduction](#introduction)
- [Project Structure](#project-structure)
- [Database Schema](#database-schema)
- [Business Problems](#business-problems)
- [SQL Queries & Analysis](#sql-queries--analysis)
- [Getting Started](#getting-started)
- [Questions & Feedback](#questions--feedback)
- [Contact Me](#contact-me)
- [ERD (Entity-Relationship Diagram)](#erd-entity-relationship-diagram)

---

## Introduction

The IKEA Retail Sales SQL Project demonstrates the use of SQL to analyze retail data, including **sales records**, **store performance**, **product trends**, and **inventory status**. Using a robust schema, this project answers critical business questions and provides actionable insights to optimize IKEA's operational efficiency and profitability.

---

## Project Structure

1. **SQL Scripts**: Contains SQL queries to create the database schema, populate tables, and perform analyses.
2. **Dataset**: Includes sales data, product information, store details, and inventory records.
3. **Analysis**: SQL queries solve key business problems, leveraging advanced SQL techniques like joins, aggregations, and subqueries.

---

## Database Schema

### 1. **Products Table**
- **product_id**: Unique identifier for each product (Primary Key).
- **product_name**: Name of the product.
- **category**: Category to which the product belongs.
- **subcategory**: Subcategory of the product.
- **unit_price**: Price per unit of the product.

### 2. **Stores Table**
- **store_id**: Unique identifier for each store (Primary Key).
- **store_name**: Name of the store.
- **city**: City where the store is located.
- **country**: Country where the store operates.

### 3. **Sales Table**
- **order_id**: Unique identifier for each sales order (Primary Key).
- **order_date**: Date when the order was placed.
- **product_id**: Foreign key referencing the `products` table.
- **qty**: Quantity of the product sold.
- **discount_percentage**: Discount applied to the order.
- **unit_price**: Price per unit of the product at the time of sale.
- **store_id**: Foreign key referencing the `stores` table.

### 4. **Inventory Table**
- **inventory_id**: Unique identifier for each inventory record (Primary Key).
- **product_id**: Foreign key referencing the `products` table.
- **current_stock**: Current stock level of the product.
- **reorder_level**: Minimum stock level to trigger a reorder.

---

## Business Problems

This project tackles the following business problems:

### Easy-Medium Level Queries
1. Find the average discount and total revenue generated for each subcategory across all stores.
```sql
SELECT 
p.subcategory,
ROUND(AVG(s.discount_percentage * 100)::numeric, 2) AS avg_discount,
ROUND(SUM(s.net_sale)::numeric,2) AS total_revenue
FROM sales AS s
INNER JOIN products AS p
ON p.product_id = s.product_id
GROUP BY 1
```
2. Retrieve the top three products by total sales revenue in each store.
```sql
WITH t1
AS
(
SELECT
s.store_id,
p.product_id,
ROUND(SUM(s.net_sale)::NUMERIC, 2) AS total_revenue
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2
)
SELECT 
store_id,
        product_id,
        total_revenue,
		DENSE_RANK() OVER (PARTITION BY store_id ORDER BY total_revenue DESC) AS rank
FROM t1
limit 3;
```
3. Determine the product with the highest number of units sold in each category and store.
```sql
WITH t1
AS
(
SELECT 
st.store_id,
p.product_id,
st.store_name,
p.product_name,
p.category,
SUM(net_sale) AS highest_no_units
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
INNER JOIN stores AS st
ON s.store_id = st.store_id
GROUP BY 1,2,3,4,5
),
t2
AS
(SELECT
t1.*,
t1.highest_no_units,
DENSE_RANK() OVER (PARTITION BY t1.store_id,t1.category ORDER BY highest_no_units DESC) AS top_rank,
RANK() OVER (PARTITION BY t1.store_id,t1.category ORDER BY highest_no_units DESC) AS top_ranks
FROM t1
)
SELECT 
t2.category,
t2.store_name,
t2.product_name,
    t2.top_rank,
    t2.top_ranks
FROM t2
WHERE t2.top_rank =1
ORDER BY t2.store_id, t2.category;
```
4. Identify the best selling top 2 category of each month on the qty sold for 2023.
```sql
WITH category_sales
AS
(
SELECT 
TO_CHAR(order_date, 'Month') AS months,
p.category,
SUM(qty) AS total_quantity
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
WHERE 
     EXTRACT(YEAR FROM s.order_date) = 2023
GROUP BY 1,2
),
category_ranks
AS
(SELECT
*,
DENSE_RANK() OVER (PARTITION BY months ORDER BY total_quantity DESC) AS ranks
FROM category_sales)
SELECT 
* 
FROM category_ranks
WHERE ranks <= 3;
```
5. identify the store with decreasing revenue compared to last year.
```sql
WITH t1
AS
(
SELECT
store_id,
SUM(net_sale) AS last_year_revenue
FROM sales
WHERE EXTRACT(YEAR FROM order_date) =2022
GROUP BY 1),
t2
AS
(SELECT
store_id,
SUM(net_sale) AS current_year_revenue
FROM sales
WHERE EXTRACT(YEAR FROM order_date) =2023
GROUP BY 1),
t3
AS
(SELECT 
t1.store_id,
last_year_revenue,
current_year_revenue,
ROUND((last_year_revenue-current_year_revenue)::numeric/last_year_revenue::numeric*100, 2) AS percentage,
DENSE_RANK() OVER (ORDER BY t1.store_id) AS ranks
FROM t1
JOIN
t2
ON t1.store_id = t2.store_id
WHERE last_year_revenue > current_year_revenue
ORDER BY 4 DESC)
SELECT 
*
FROM t3
JOIN stores AS st
ON t3.store_id=st.store_id
WHERE ranks <= 5
ORDER BY percentage DESC;
```


### Medium to Hard-Level Queries
1. For each store, list the top three most frequently sold product categories and the total revenue generated by each.
```sql
WITH top_3
AS
(
SELECT 
s.store_id,
p.category,
SUM(net_sale) AS total_revenue
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2),
t2
AS
(SELECT 
*,
total_revenue AS revenue,
DENSE_RANK () OVER (PARTITION BY category ORDER BY total_revenue DESC) AS ranks
FROM top_3
)
SELECT 
store_id,
category,
revenue,
ranks
FROM t2
WHERE ranks <= 3;
```
2. Categorize stores based on sales performance as "High," "Medium," or "Low" using the total sales revenue.
```sql
WITH
t1
AS
(
SELECT 
store_id,
SUM(net_sale) AS total_sales_revenue
FROM sales 
GROUP BY 1
),
t2
AS
(
SELECT 
store_id,
---((total_sales_revenue*100)/100) AS percentage
MAX(total_sales_revenue) AS max_sales,
MIN(total_sales_revenue) AS min_sales
FROM t1
GROUP BY 1
)
SELECT 
t2.store_id,
t1.total_sales_revenue,
CASE 
    WHEN t1.total_sales_revenue >= 0.94 * t2.max_sales THEN 'High_sales'
	WHEN t1.total_sales_revenue >= 0.70 * t2.max_sales THEN 'Medium_sales'
	ELSE 'Low_sales'
	END AS Sales_performance
FROM t1
JOIN t2
ON t1.store_id= t2.store_id;
```
3. Create a column indicating if the product price is above or below the average price for its category.
```sql
WITH 
category_avg_price 
AS
(
SELECT
--p.product_id,
p.category,
AVG(s.net_sale) AS avg_price
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1
)
SELECT
--s.product_id,
p.product_name,
p.category,
s.unit_price,
c.avg_price,
CASE 
    WHEN s.unit_price > c.avg_price THEN 'Above Average'
    WHEN s.unit_price < c.avg_price THEN 'Below Average'
	ELSE 'Average'
	END AS price_comparision
FROM sales s
INNER JOIN products AS p 
ON s.product_id = p.product_id
INNER JOIN category_avg_price AS c 
ON p.category = c.category;
```
4. List cities with total sales greater than the average sales for their country.
```sql
WITH
country_sales
AS
(
SELECT 
st.country,
SUM(s.net_sale) AS total_sales,
AVG(s.net_sale) AS country_avg_sales
FROM sales AS s
INNER JOIN stores AS st
ON s.store_id = st.store_id
GROUP BY 1
),
city_sales
AS
(
SELECT 
st.city,
st.country,
SUM(s.net_sale) AS city_tatal_sales,
AVG(s.net_sale) AS city_avg_sales
FROM sales AS s
INNER JOIN stores AS st
ON s.store_id = st.store_id
GROUP BY 1,2
)
SELECT 
ct.country,
cs.city,
ct.country_avg_sales,
cs.city_avg_sales
FROM city_sales AS cs
JOIN country_sales AS ct
ON cs.country =ct.country
WHERE cs.city_avg_sales > ct.country_avg_sales
ORDER BY cs.city;
```
5. List the top five products by sales quantity within each store.
```sql
WITH t1
AS
(
SELECT
s.store_id,
p.product_id,
p.product_name,
SUM(s.qty) AS sales_qty
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2,3
),
t2
AS
(
SELECT 
t1.store_id,
t1.product_id,
t1.product_name,
t1.sales_qty,
DENSE_RANK () OVER (PARTITION BY store_id ORDER BY sales_qty DESC) AS ranks
FROM t1
)
SELECT 
* 
FROM t2
WHERE ranks <= 5;
```
6. Rank each product by quantity sold in each store.
```sql
WITH
stores_qty
AS
(
SELECT
s.store_id,
p.product_id,
p.product_name,
SUM(s.qty) AS total_qty
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2,3
)
SELECT 
sq.store_id,
sq.product_name,
sq.total_qty,
DENSE_RANK () OVER (PARTITION BY store_id ORDER BY total_qty ASC) AS ranks
FROM stores_qty AS sq
ORDER BY store_id;
```
7. Identify the store with the highest revenue in each country.
```sql
WITH
t1
AS
(
SELECT
st.store_id,
st.store_name,
st.country,
ROUND(SUM(s.net_sale)::numeric, 2) AS highest_revenue
FROM sales AS s
INNER JOIN stores AS st
ON s.store_id = st.store_id
GROUP BY 1,2,3
),
t2
AS
(
SELECT 
t1.store_id,
t1.store_name,
t1.country,
t1.highest_revenue,
DENSE_RANK() OVER (PARTITION BY country ORDER BY highest_revenue DESC) AS ranks
FROM t1
)
SELECT 
store_id,
store_name,
country,
highest_revenue,
ranks
FROM t2
WHERE ranks = 1;
```
8. For each store, list the top three most frequently sold product categories and the total revenue generated by each.
```sql
WITH
t1
AS
(
SELECT
s.store_id,
p.category,
COUNT(*) AS total_sales,
SUM(net_sale) AS total_revenue
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2
),
t2
AS
(
SELECT 
t1.store_id,
t1.category,
t1.total_sales,
t1.total_revenue,
DENSE_RANK() OVER (PARTITION BY t1.category ORDER BY total_revenue DESC) AS ranks
FROM t1
)
SELECT
*
FROM t2
WHERE ranks <= 3;
```
9. Find the top five stores by sales quantity for products in the "Kitchen" category, with rankings adjusted based on discount levels.
```sql
WITH
t1
AS
(
SELECT 
s.store_id,
p.category,
SUM(s.qty) AS sales_quantity,
ROUND(AVG(s.discount_percentage * 100)::numeric, 2) AS avg_discount
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
WHERE category = 'Kitchen'
GROUP BY 1,2
),
t2
AS
(
SELECT 
t1.store_id,
t1.category,
t1.sales_quantity,
t1.avg_discount,
RANK () OVER (PARTITION BY category ORDER BY avg_discount DESC) AS ranks
FROM t1
)
SELECT 
*
FROM t2
WHERE ranks <= 5
ORDER BY ranks;
```
10. identify the top-performing stores based on total sales revenue and assign a performance rank to each.
```sql
WITH
t1
AS
(
SELECT
s.store_id,
p.category,
ROUND(SUM(net_sale)::NUMERIC, 2) AS total_sales
FROM sales AS s
INNER JOIN products AS p
ON s.product_id = p.product_id
GROUP BY 1,2
)
SELECT 
t1.store_id,
t1.category,
t1.total_sales,
DENSE_RANK() OVER (PARTITION BY category ORDER BY total_sales DESC) AS performance_ranks
FROM t1
ORDER BY category, performance_ranks;
```
---

## SQL Queries & Analysis

All SQL queries developed for this project are available in the `queries.sql` file. The queries demonstrate advanced SQL skills, including:

- Aggregations with `GROUP BY`.
- Filtering data using `WHERE` and `HAVING`.
- Joining multiple tables to uncover insights.
- Using subqueries and window functions for complex analyses.

---

## Getting Started

### Prerequisites
- PostgreSQL (or any SQL-compatible database).
- Basic knowledge of SQL.

### Steps to Run
1. **Clone the Repository**:
   ```bash
   git clone https://github.com/yourusername/ikea-sales-sql-project.git
   ```
2. **Set Up the Database**:
   - Run `schema.sql` to create the database schema.
   - Populate tables with sample data using `data.sql`.

3. **Execute Queries**:
   - Open `queries.sql` and execute the queries for analysis.

---

## Questions & Feedback

Feel free to reach out with questions or suggestions. Here's an example query for reference:

### Example Query
**Question**: Retrieve the total sales revenue for each store in a specific country.
```sql
SELECT 
    s.store_name, 
    SUM(sales.qty * sales.unit_price) AS total_revenue
FROM 
    sales
JOIN 
    stores s ON sales.store_id = s.store_id
WHERE 
    s.country = 'USA'
GROUP BY 
    s.store_name
ORDER BY 
    total_revenue DESC;
```

---

## Contact Me

📧 **[Email](mailto:your.pravingiri2222@gmail.com)**  
💼 **[LinkedIn](https://www.linkedin.com/in/pravin-giri-966612212)**  

---

## ERD (Entity-Relationship Diagram)

Here’s the ERD for the IKEA Retail Sales SQL Project:

![ERD Placeholder](https://github.com/najirh/sql-b01-ikea/blob/main/IKEA.png)

---
