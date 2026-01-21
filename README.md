# SQL Joins Workshop

This workshop demonstrates SQL joins using a normalized supermarket database. The database contains customer, cashier, product, and cart (transaction) tables.

## Database Structure

- **customer**: Customer information (customer_id, first_name, last_name, phone_number, date_of_birth)
- **cashier**: Cashier information (cashier_id, name)
- **product**: Product catalog (product_id, description, brand_name, category, unit_price)
- **cart**: Transaction records (purchase_id, transaction_id, customer_id, cashier_id, product_id, quantity, purchase_date, etc.)

---

## 1. Basic INNER JOIN - Two Tables

### Use Case 1: Display customer names with their purchase details

**Business Question**: Show all purchases with customer names to understand who bought what.

```sql
SELECT 
    c.purchase_id,
    c.purchase_date,
    cust.first_name,
    cust.last_name,
    c.quantity,
    c.payment_method
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
ORDER BY c.purchase_date DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_id,
    purchase_date,
    customer_first_name AS first_name,
    customer_last_name AS last_name,
    quantity,
    payment_method
FROM supermarket_transactions
ORDER BY purchase_date DESC;
```

### Use Case 2: Show cashier information with transaction details

**Business Question**: Display all transactions with the cashier who processed them for performance tracking.

```sql
SELECT 
    c.purchase_id,
    c.transaction_id,
    c.purchase_date,
    c.purchase_time,
    cash.name AS cashier_name,
    c.payment_method
FROM cart c
INNER JOIN cashier cash ON c.cashier_id = cash.cashier_id
ORDER BY c.purchase_date DESC, c.purchase_time;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_id,
    transaction_id,
    purchase_date,
    purchase_time,
    cashier_name,
    payment_method
FROM supermarket_transactions
ORDER BY purchase_date DESC, purchase_time;
```

---

## 2. INNER JOIN with Product Information

### Use Case 1: Show complete purchase details with product names

**Business Question**: Generate a detailed receipt showing what products were purchased.

```sql
SELECT 
    c.purchase_id,
    c.purchase_date,
    cust.first_name || ' ' || cust.last_name AS customer_name,
    p.description AS product_name,
    p.brand_name,
    c.quantity,
    p.unit_price,
    (c.quantity * p.unit_price) AS total_price
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
INNER JOIN product p ON c.product_id = p.product_id
ORDER BY c.purchase_date DESC, c.purchase_id;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_id,
    purchase_date,
    customer_first_name || ' ' || customer_last_name AS customer_name,
    product_description AS product_name,
    brand_name,
    quantity,
    unit_price,
    (quantity * unit_price) AS total_price
FROM supermarket_transactions
ORDER BY purchase_date DESC, purchase_id;
```

### Use Case 2: Display product catalog with category and brand information

**Business Question**: Show all products with their category and brand details for inventory management.

```sql
SELECT 
    p.product_id,
    p.description,
    p.brand_name,
    p.category,
    p.unit_price,
    COUNT(c.purchase_id) AS times_purchased
FROM product p
INNER JOIN cart c ON p.product_id = c.product_id
GROUP BY p.product_id, p.description, p.brand_name, p.category, p.unit_price
ORDER BY p.category, p.brand_name, p.description;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    product_description AS description,
    brand_name,
    category,
    unit_price,
    COUNT(purchase_id) AS times_purchased
FROM supermarket_transactions
GROUP BY product_description, brand_name, category, unit_price
ORDER BY category, brand_name, product_description;
```

---

## 3. LEFT JOIN - Include All Customers

### Use Case 1: Find customers who have never made a purchase

**Business Question**: Identify customers in the system who haven't made any purchases yet (useful for marketing campaigns).

```sql
SELECT 
    cust.customer_id,
    cust.first_name,
    cust.last_name,
    cust.phone_number
FROM customer cust
LEFT JOIN cart c ON cust.customer_id = c.customer_id
WHERE c.customer_id IS NULL;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

*Note: This query cannot be performed on the denormalized schema as customer data only exists for customers who made purchases. The denormalized table doesn't contain customers who never purchased.*

### Use Case 2: Show all customers with their purchase count (including zero)

**Business Question**: Get a complete list of all customers and their purchase activity for customer segmentation.

```sql
SELECT 
    cust.customer_id,
    cust.first_name || ' ' || cust.last_name AS customer_name,
    COUNT(c.purchase_id) AS total_purchases,
    COALESCE(SUM(c.quantity * p.unit_price), 0) AS total_spent
FROM customer cust
LEFT JOIN cart c ON cust.customer_id = c.customer_id
LEFT JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name
ORDER BY total_purchases DESC, customer_name;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    customer_first_name || ' ' || customer_last_name AS customer_name,
    COUNT(purchase_id) AS total_purchases,
    SUM(quantity * unit_price) AS total_spent
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name
ORDER BY total_purchases DESC, customer_name;
```

---

## 4. LEFT JOIN - Include All Products

### Use Case 1: Show all products and their sales count (including unsold products)

**Business Question**: Identify products that haven't been sold yet to manage inventory.

```sql
SELECT 
    p.product_id,
    p.description,
    p.category,
    COUNT(c.purchase_id) AS times_sold
FROM product p
LEFT JOIN cart c ON p.product_id = c.product_id
GROUP BY p.product_id, p.description, p.category
ORDER BY times_sold ASC, p.description;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    product_description AS description,
    category,
    COUNT(purchase_id) AS times_sold
FROM supermarket_transactions
GROUP BY product_description, category
ORDER BY times_sold ASC, product_description;
```

### Use Case 2: Display all products with average sale price and total revenue

**Business Question**: Analyze product performance including products with no sales to identify slow-moving inventory.

```sql
SELECT 
    p.product_id,
    p.description,
    p.category,
    p.unit_price,
    COUNT(c.purchase_id) AS sales_count,
    COALESCE(AVG(c.quantity * p.unit_price), 0) AS avg_sale_amount,
    COALESCE(SUM(c.quantity * p.unit_price), 0) AS total_revenue
FROM product p
LEFT JOIN cart c ON p.product_id = c.product_id
GROUP BY p.product_id, p.description, p.category, p.unit_price
ORDER BY total_revenue DESC, p.description;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    product_description AS description,
    category,
    unit_price,
    COUNT(purchase_id) AS sales_count,
    AVG(quantity * unit_price) AS avg_sale_amount,
    SUM(quantity * unit_price) AS total_revenue
FROM supermarket_transactions
GROUP BY product_description, category, unit_price
ORDER BY total_revenue DESC, product_description;
```

---

## 5. WHERE Clause with JOINs

### Use Case 1: Find purchases by a specific customer

**Business Question**: Retrieve all purchase history for customer "Maria Garcia".

```sql
SELECT 
    c.purchase_date,
    p.description AS product,
    c.quantity,
    p.unit_price,
    (c.quantity * p.unit_price) AS line_total
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
INNER JOIN product p ON c.product_id = p.product_id
WHERE cust.first_name = 'Maria' AND cust.last_name = 'Garcia'
ORDER BY c.purchase_date DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_date,
    product_description AS product,
    quantity,
    unit_price,
    (quantity * unit_price) AS line_total
FROM supermarket_transactions
WHERE customer_first_name = 'Maria' AND customer_last_name = 'Garcia'
ORDER BY purchase_date DESC;
```

### Use Case 2: Filter transactions by payment method

**Business Question**: Analyze all credit card transactions to understand payment preferences.

```sql
SELECT 
    c.purchase_id,
    c.purchase_date,
    cust.first_name || ' ' || cust.last_name AS customer,
    p.description AS product,
    c.payment_method,
    (c.quantity * p.unit_price) AS amount
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
INNER JOIN product p ON c.product_id = p.product_id
WHERE c.payment_method = 'Credit Card'
ORDER BY c.purchase_date DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_id,
    purchase_date,
    customer_first_name || ' ' || customer_last_name AS customer,
    product_description AS product,
    payment_method,
    (quantity * unit_price) AS amount
FROM supermarket_transactions
WHERE payment_method = 'Credit Card'
ORDER BY purchase_date DESC;
```

---

## 6. WHERE Clause - Filter by Date Range

### Use Case 1: Sales report for a specific month

**Business Question**: Generate a sales report for October 2025.

```sql
SELECT 
    c.purchase_date,
    cust.first_name || ' ' || cust.last_name AS customer,
    p.description AS product,
    c.quantity,
    (c.quantity * p.unit_price) AS amount
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
INNER JOIN product p ON c.product_id = p.product_id
WHERE c.purchase_date >= '2025-10-01' 
  AND c.purchase_date < '2025-11-01'
ORDER BY c.purchase_date, c.purchase_id;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_date,
    customer_first_name || ' ' || customer_last_name AS customer,
    product_description AS product,
    quantity,
    (quantity * unit_price) AS amount
FROM supermarket_transactions
WHERE purchase_date >= '2025-10-01' 
  AND purchase_date < '2025-11-01'
ORDER BY purchase_date, purchase_id;
```

### Use Case 2: Filter sales by store location

**Business Question**: Compare sales performance across different store locations.

```sql
SELECT 
    c.store_location,
    c.purchase_date,
    cust.first_name || ' ' || cust.last_name AS customer,
    p.description AS product,
    (c.quantity * p.unit_price) AS amount
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
INNER JOIN product p ON c.product_id = p.product_id
WHERE c.store_location = 'Downtown'
ORDER BY c.purchase_date DESC, c.purchase_id;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    store_location,
    purchase_date,
    customer_first_name || ' ' || customer_last_name AS customer,
    product_description AS product,
    (quantity * unit_price) AS amount
FROM supermarket_transactions
WHERE store_location = 'Downtown'
ORDER BY purchase_date DESC, purchase_id;
```

---

## 7. COUNT() with GROUP BY on Joined Tables

### Use Case 1: Count purchases per customer

**Business Question**: Identify the most active customers by purchase count.

```sql
SELECT 
    cust.customer_id,
    cust.first_name || ' ' || cust.last_name AS customer_name,
    COUNT(c.purchase_id) AS total_purchases
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name
ORDER BY total_purchases DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    customer_first_name || ' ' || customer_last_name AS customer_name,
    COUNT(purchase_id) AS total_purchases
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name
ORDER BY total_purchases DESC;
```

### Use Case 2: Count products sold per category

**Business Question**: Understand product distribution across categories for inventory planning.

```sql
SELECT 
    p.category,
    COUNT(DISTINCT p.product_id) AS unique_products,
    COUNT(c.purchase_id) AS total_sales
FROM product p
INNER JOIN cart c ON p.product_id = c.product_id
GROUP BY p.category
ORDER BY total_sales DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    category,
    COUNT(DISTINCT product_description) AS unique_products,
    COUNT(purchase_id) AS total_sales
FROM supermarket_transactions
GROUP BY category
ORDER BY total_sales DESC;
```

---

## 8. COUNT() - Products Sold by Category

### Use Case 1: Analyze sales by product category

**Business Question**: Which product categories have the most sales?

```sql
SELECT 
    p.category,
    COUNT(c.purchase_id) AS number_of_sales,
    SUM(c.quantity) AS total_quantity_sold
FROM product p
INNER JOIN cart c ON p.product_id = c.product_id
GROUP BY p.category
ORDER BY number_of_sales DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    category,
    COUNT(purchase_id) AS number_of_sales,
    SUM(quantity) AS total_quantity_sold
FROM supermarket_transactions
GROUP BY category
ORDER BY number_of_sales DESC;
```

### Use Case 2: Count transactions processed by each cashier

**Business Question**: Evaluate cashier workload and transaction processing efficiency.

```sql
SELECT 
    cash.cashier_id,
    cash.name AS cashier_name,
    COUNT(DISTINCT c.transaction_id) AS unique_transactions,
    COUNT(c.purchase_id) AS total_items_processed
FROM cashier cash
INNER JOIN cart c ON cash.cashier_id = c.cashier_id
GROUP BY cash.cashier_id, cash.name
ORDER BY total_items_processed DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    cashier_id,
    cashier_name,
    COUNT(DISTINCT transaction_id) AS unique_transactions,
    COUNT(purchase_id) AS total_items_processed
FROM supermarket_transactions
GROUP BY cashier_id, cashier_name
ORDER BY total_items_processed DESC;
```

---

## 9. SUM() with GROUP BY - Revenue by Customer

### Use Case 1: Calculate total spending per customer

**Business Question**: Who are our top-spending customers?

```sql
SELECT 
    cust.customer_id,
    cust.first_name || ' ' || cust.last_name AS customer_name,
    COUNT(c.purchase_id) AS purchase_count,
    SUM(c.quantity * p.unit_price) AS total_spent
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name
ORDER BY total_spent DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    customer_first_name || ' ' || customer_last_name AS customer_name,
    COUNT(purchase_id) AS purchase_count,
    SUM(quantity * unit_price) AS total_spent
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name
ORDER BY total_spent DESC;
```

### Use Case 2: Calculate revenue by product category

**Business Question**: Determine which product categories generate the most revenue for business strategy.

```sql
SELECT 
    p.category,
    COUNT(c.purchase_id) AS sales_count,
    SUM(c.quantity) AS total_quantity,
    SUM(c.quantity * p.unit_price) AS category_revenue
FROM product p
INNER JOIN cart c ON p.product_id = c.product_id
GROUP BY p.category
ORDER BY category_revenue DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    category,
    COUNT(purchase_id) AS sales_count,
    SUM(quantity) AS total_quantity,
    SUM(quantity * unit_price) AS category_revenue
FROM supermarket_transactions
GROUP BY category
ORDER BY category_revenue DESC;
```

---

## 10. SUM() - Revenue by Cashier

### Use Case 1: Performance analysis by cashier

**Business Question**: Which cashier processed the most revenue?

```sql
SELECT 
    cash.cashier_id,
    cash.name AS cashier_name,
    COUNT(DISTINCT c.transaction_id) AS transactions_processed,
    SUM(c.quantity * p.unit_price) AS total_revenue
FROM cashier cash
INNER JOIN cart c ON cash.cashier_id = c.cashier_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cash.cashier_id, cash.name
ORDER BY total_revenue DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    cashier_id,
    cashier_name,
    COUNT(DISTINCT transaction_id) AS transactions_processed,
    SUM(quantity * unit_price) AS total_revenue
FROM supermarket_transactions
GROUP BY cashier_id, cashier_name
ORDER BY total_revenue DESC;
```

### Use Case 2: Calculate revenue by store location

**Business Question**: Compare revenue performance across different store locations to identify best-performing branches.

```sql
SELECT 
    c.store_location,
    COUNT(DISTINCT c.customer_id) AS unique_customers,
    COUNT(c.purchase_id) AS total_sales,
    SUM(c.quantity * p.unit_price) AS location_revenue
FROM cart c
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY c.store_location
ORDER BY location_revenue DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    store_location,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(purchase_id) AS total_sales,
    SUM(quantity * unit_price) AS location_revenue
FROM supermarket_transactions
GROUP BY store_location
ORDER BY location_revenue DESC;
```

---

## 11. Multiple JOINs with Aggregation

### Use Case 1: Daily sales summary with customer and cashier details

**Business Question**: Generate a daily sales report showing customer activity and cashier performance.

```sql
SELECT 
    c.purchase_date,
    COUNT(DISTINCT c.customer_id) AS unique_customers,
    COUNT(DISTINCT c.cashier_id) AS active_cashiers,
    COUNT(c.purchase_id) AS total_items_sold,
    SUM(c.quantity * p.unit_price) AS daily_revenue
FROM cart c
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY c.purchase_date
ORDER BY c.purchase_date DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_date,
    COUNT(DISTINCT customer_id) AS unique_customers,
    COUNT(DISTINCT cashier_id) AS active_cashiers,
    COUNT(purchase_id) AS total_items_sold,
    SUM(quantity * unit_price) AS daily_revenue
FROM supermarket_transactions
GROUP BY purchase_date
ORDER BY purchase_date DESC;
```

### Use Case 2: Monthly revenue summary by customer

**Business Question**: Track monthly spending patterns per customer for loyalty program analysis.

```sql
SELECT 
    cust.customer_id,
    CONCAT(cust.first_name, ' ', cust.last_name) AS customer_name,
    DATE_FORMAT(c.purchase_date, '%Y-%m-01') AS month,
    COUNT(c.purchase_id) AS monthly_purchases,
    SUM(c.quantity * p.unit_price) AS monthly_spending
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name, DATE_FORMAT(c.purchase_date, '%Y-%m-01')
ORDER BY customer_name, month DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    CONCAT(customer_first_name, ' ', customer_last_name) AS customer_name,
    DATE_FORMAT(purchase_date, '%Y-%m-01') AS month,
    COUNT(purchase_id) AS monthly_purchases,
    SUM(quantity * unit_price) AS monthly_spending
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name, DATE_FORMAT(purchase_date, '%Y-%m-01')
ORDER BY customer_name, month DESC;
```

---

## 12. JOIN with ORDER BY - Top Products

### Use Case 1: Best-selling products

**Business Question**: What are our top 10 best-selling products by quantity?

```sql
SELECT 
    p.product_id,
    p.description,
    p.category,
    SUM(c.quantity) AS total_quantity_sold,
    COUNT(c.purchase_id) AS times_purchased,
    SUM(c.quantity * p.unit_price) AS total_revenue
FROM product p
INNER JOIN cart c ON p.product_id = c.product_id
GROUP BY p.product_id, p.description, p.category
ORDER BY total_quantity_sold DESC
LIMIT 10;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    product_description AS description,
    category,
    SUM(quantity) AS total_quantity_sold,
    COUNT(purchase_id) AS times_purchased,
    SUM(quantity * unit_price) AS total_revenue
FROM supermarket_transactions
GROUP BY product_description, category
ORDER BY total_quantity_sold DESC
LIMIT 10;
```

### Use Case 2: Top customers by total spending

**Business Question**: Identify the top 5 customers by total spending for VIP program eligibility.

```sql
SELECT 
    cust.customer_id,
    cust.first_name || ' ' || cust.last_name AS customer_name,
    COUNT(c.purchase_id) AS purchase_count,
    SUM(c.quantity * p.unit_price) AS total_spent,
    AVG(c.quantity * p.unit_price) AS avg_purchase_value
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name
ORDER BY total_spent DESC
LIMIT 5;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    customer_first_name || ' ' || customer_last_name AS customer_name,
    COUNT(purchase_id) AS purchase_count,
    SUM(quantity * unit_price) AS total_spent,
    AVG(quantity * unit_price) AS avg_purchase_value
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name
ORDER BY total_spent DESC
LIMIT 5;
```

---

## 13. Complex JOIN with Multiple Conditions

### Use Case 1: Discounted purchases analysis

**Business Question**: Show all discounted purchases with customer and product details, ordered by discount amount.

```sql
SELECT 
    c.purchase_id,
    cust.first_name || ' ' || cust.last_name AS customer,
    p.description AS product,
    c.quantity,
    p.unit_price AS original_price,
    c.discount_percent,
    (p.unit_price * (1 - c.discount_percent)) AS discounted_price,
    (p.unit_price * c.discount_percent * c.quantity) AS savings
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
INNER JOIN product p ON c.product_id = p.product_id
WHERE c.is_discounted = TRUE
ORDER BY savings DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_id,
    customer_first_name || ' ' || customer_last_name AS customer,
    product_description AS product,
    quantity,
    unit_price AS original_price,
    discount_percent,
    (unit_price * (1 - discount_percent)) AS discounted_price,
    (unit_price * discount_percent * quantity) AS savings
FROM supermarket_transactions
WHERE is_discounted = TRUE
ORDER BY savings DESC;
```

### Use Case 2: High-value purchases with customer details

**Business Question**: Identify purchases above $20 to understand high-value transaction patterns.

```sql
SELECT 
    c.purchase_id,
    c.purchase_date,
    cust.first_name || ' ' || cust.last_name AS customer,
    p.description AS product,
    c.quantity,
    p.unit_price,
    (c.quantity * p.unit_price) AS purchase_amount,
    cash.name AS cashier_name
FROM cart c
INNER JOIN customer cust ON c.customer_id = cust.customer_id
INNER JOIN product p ON c.product_id = p.product_id
INNER JOIN cashier cash ON c.cashier_id = cash.cashier_id
WHERE (c.quantity * p.unit_price) > 20
ORDER BY purchase_amount DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_id,
    purchase_date,
    customer_first_name || ' ' || customer_last_name AS customer,
    product_description AS product,
    quantity,
    unit_price,
    (quantity * unit_price) AS purchase_amount,
    cashier_name
FROM supermarket_transactions
WHERE (quantity * unit_price) > 20
ORDER BY purchase_amount DESC;
```

---

## 14. JOIN with HAVING Clause

### Use Case 1: Customers who spent more than $100

**Business Question**: Identify high-value customers for loyalty programs.

```sql
SELECT 
    cust.customer_id,
    cust.first_name || ' ' || cust.last_name AS customer_name,
    COUNT(c.purchase_id) AS purchase_count,
    SUM(c.quantity * p.unit_price) AS total_spent
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name
HAVING SUM(c.quantity * p.unit_price) > 100
ORDER BY total_spent DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    customer_first_name || ' ' || customer_last_name AS customer_name,
    COUNT(purchase_id) AS purchase_count,
    SUM(quantity * unit_price) AS total_spent
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name
HAVING SUM(quantity * unit_price) > 100
ORDER BY total_spent DESC;
```

### Use Case 2: Products sold more than 5 times

**Business Question**: Identify popular products that have been purchased multiple times for restocking decisions.

```sql
SELECT 
    p.product_id,
    p.description,
    p.category,
    COUNT(c.purchase_id) AS times_sold,
    SUM(c.quantity) AS total_quantity_sold,
    SUM(c.quantity * p.unit_price) AS total_revenue
FROM product p
INNER JOIN cart c ON p.product_id = c.product_id
GROUP BY p.product_id, p.description, p.category
HAVING COUNT(c.purchase_id) > 5
ORDER BY times_sold DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    product_description AS description,
    category,
    COUNT(purchase_id) AS times_sold,
    SUM(quantity) AS total_quantity_sold,
    SUM(quantity * unit_price) AS total_revenue
FROM supermarket_transactions
GROUP BY product_description, category
HAVING COUNT(purchase_id) > 5
ORDER BY times_sold DESC;
```

---

## 15. Advanced: Multiple JOINs with Date Filtering

### Use Case 1: Monthly sales by category and store location

**Business Question**: Analyze sales performance by product category across different store locations for a specific month.

```sql
SELECT 
    c.store_location,
    p.category,
    COUNT(c.purchase_id) AS sales_count,
    SUM(c.quantity) AS total_quantity,
    SUM(c.quantity * p.unit_price) AS category_revenue
FROM cart c
INNER JOIN product p ON c.product_id = p.product_id
WHERE c.purchase_date >= '2025-11-01' 
  AND c.purchase_date < '2025-12-01'
GROUP BY c.store_location, p.category
ORDER BY c.store_location, category_revenue DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    store_location,
    category,
    COUNT(purchase_id) AS sales_count,
    SUM(quantity) AS total_quantity,
    SUM(quantity * unit_price) AS category_revenue
FROM supermarket_transactions
WHERE purchase_date >= '2025-11-01' 
  AND purchase_date < '2025-12-01'
GROUP BY store_location, category
ORDER BY store_location, category_revenue DESC;
```

### Use Case 2: Daily revenue by cashier for a specific period

**Business Question**: Track daily cashier performance during a busy period to identify top performers.

```sql
SELECT 
    c.purchase_date,
    cash.cashier_id,
    cash.name AS cashier_name,
    COUNT(c.purchase_id) AS items_processed,
    SUM(c.quantity * p.unit_price) AS daily_revenue
FROM cart c
INNER JOIN cashier cash ON c.cashier_id = cash.cashier_id
INNER JOIN product p ON c.product_id = p.product_id
WHERE c.purchase_date >= '2025-10-01' 
  AND c.purchase_date < '2025-11-01'
GROUP BY c.purchase_date, cash.cashier_id, cash.name
ORDER BY c.purchase_date DESC, daily_revenue DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    purchase_date,
    cashier_id,
    cashier_name,
    COUNT(purchase_id) AS items_processed,
    SUM(quantity * unit_price) AS daily_revenue
FROM supermarket_transactions
WHERE purchase_date >= '2025-10-01' 
  AND purchase_date < '2025-11-01'
GROUP BY purchase_date, cashier_id, cashier_name
ORDER BY purchase_date DESC, daily_revenue DESC;
```

---

## 16. Advanced: JOIN with Subquery

### Use Case 1: Customers who bought products from a specific category

**Business Question**: Find all customers who purchased items from the "Dairy" category.

```sql
SELECT DISTINCT
    cust.customer_id,
    cust.first_name,
    cust.last_name,
    CONCAT(cust.first_name, ' ', cust.last_name) AS customer_name,
    cust.phone_number
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
WHERE p.category = 'Dairy'
ORDER BY cust.last_name, cust.first_name;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT DISTINCT
    customer_id,
    customer_first_name,
    customer_last_name,
    CONCAT(customer_first_name, ' ', customer_last_name) AS customer_name,
    customer_phone_number AS phone_number
FROM supermarket_transactions
WHERE category = 'Dairy'
ORDER BY customer_last_name, customer_first_name;
```

### Use Case 2: Customers who purchased from multiple categories

**Business Question**: Identify customers with diverse purchasing habits who bought from at least 3 different product categories.

```sql
SELECT 
    cust.customer_id,
    CONCAT(cust.first_name, ' ', cust.last_name) AS customer_name,
    COUNT(DISTINCT p.category) AS categories_purchased,
    GROUP_CONCAT(DISTINCT p.category ORDER BY p.category SEPARATOR ', ') AS categories_list
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name
HAVING COUNT(DISTINCT p.category) >= 3
ORDER BY categories_purchased DESC, customer_name;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    CONCAT(customer_first_name, ' ', customer_last_name) AS customer_name,
    COUNT(DISTINCT category) AS categories_purchased,
    GROUP_CONCAT(DISTINCT category ORDER BY category SEPARATOR ', ') AS categories_list
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name
HAVING COUNT(DISTINCT category) >= 3
ORDER BY categories_purchased DESC, customer_name;
```

---

## 17. Advanced: Self-Join Concept (Transaction Analysis)

### Use Case 1: Find transactions with multiple items

**Business Question**: Show complete transaction details for multi-item purchases.

```sql
SELECT 
    c1.transaction_id,
    c1.purchase_date,
    cust.first_name || ' ' || cust.last_name AS customer,
    COUNT(c2.purchase_id) AS items_in_transaction,
    SUM(c2.quantity * p2.unit_price) AS transaction_total
FROM cart c1
INNER JOIN cart c2 ON c1.transaction_id = c2.transaction_id
INNER JOIN customer cust ON c1.customer_id = cust.customer_id
INNER JOIN product p2 ON c2.product_id = p2.product_id
GROUP BY c1.transaction_id, c1.purchase_date, cust.first_name, cust.last_name
HAVING COUNT(c2.purchase_id) > 1
ORDER BY items_in_transaction DESC, c1.purchase_date DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    t1.transaction_id,
    t1.purchase_date,
    t1.customer_first_name || ' ' || t1.customer_last_name AS customer,
    COUNT(t2.purchase_id) AS items_in_transaction,
    SUM(t2.quantity * t2.unit_price) AS transaction_total
FROM supermarket_transactions t1
INNER JOIN supermarket_transactions t2 ON t1.transaction_id = t2.transaction_id
GROUP BY t1.transaction_id, t1.purchase_date, t1.customer_first_name, t1.customer_last_name
HAVING COUNT(t2.purchase_id) > 1
ORDER BY items_in_transaction DESC, t1.purchase_date DESC;
```

### Use Case 2: Find customers who bought the same product multiple times

**Business Question**: Identify repeat purchases of the same product to understand customer loyalty to specific items.

```sql
SELECT 
    cust.customer_id,
    CONCAT(cust.first_name, ' ', cust.last_name) AS customer_name,
    p.product_id,
    p.description AS product,
    COUNT(c.purchase_id) AS times_purchased,
    SUM(c.quantity) AS total_quantity
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name, p.product_id, p.description
HAVING COUNT(c.purchase_id) > 1
ORDER BY times_purchased DESC, customer_name;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    CONCAT(customer_first_name, ' ', customer_last_name) AS customer_name,
    product_description AS product,
    COUNT(purchase_id) AS times_purchased,
    SUM(quantity) AS total_quantity
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name, product_description
HAVING COUNT(purchase_id) > 1
ORDER BY times_purchased DESC, customer_name;
```

---

## 18. Advanced: JOIN with CASE Statement

### Use Case 1: Categorize customers by spending level

**Business Question**: Classify customers into spending tiers for targeted marketing.

```sql
SELECT 
    cust.customer_id,
    cust.first_name || ' ' || cust.last_name AS customer_name,
    SUM(c.quantity * p.unit_price) AS total_spent,
    CASE 
        WHEN SUM(c.quantity * p.unit_price) > 200 THEN 'Premium'
        WHEN SUM(c.quantity * p.unit_price) > 100 THEN 'Regular'
        ELSE 'Casual'
    END AS customer_tier
FROM customer cust
INNER JOIN cart c ON cust.customer_id = c.customer_id
INNER JOIN product p ON c.product_id = p.product_id
GROUP BY cust.customer_id, cust.first_name, cust.last_name
ORDER BY total_spent DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    customer_id,
    customer_first_name || ' ' || customer_last_name AS customer_name,
    SUM(quantity * unit_price) AS total_spent,
    CASE 
        WHEN SUM(quantity * unit_price) > 200 THEN 'Premium'
        WHEN SUM(quantity * unit_price) > 100 THEN 'Regular'
        ELSE 'Casual'
    END AS customer_tier
FROM supermarket_transactions
GROUP BY customer_id, customer_first_name, customer_last_name
ORDER BY total_spent DESC;
```

### Use Case 2: Categorize products by price range

**Business Question**: Classify products into price tiers to understand pricing strategy and customer preferences.

```sql
SELECT 
    p.product_id,
    p.description,
    p.category,
    p.unit_price,
    COUNT(c.purchase_id) AS sales_count,
    CASE 
        WHEN p.unit_price > 15 THEN 'Premium'
        WHEN p.unit_price > 5 THEN 'Mid-Range'
        ELSE 'Budget'
    END AS price_tier
FROM product p
LEFT JOIN cart c ON p.product_id = c.product_id
GROUP BY p.product_id, p.description, p.category, p.unit_price
ORDER BY p.unit_price DESC;
```

**Equivalent Query for `initial.sql` (Denormalized Schema):**

```sql
SELECT 
    product_description AS description,
    category,
    unit_price,
    COUNT(purchase_id) AS sales_count,
    CASE 
        WHEN unit_price > 15 THEN 'Premium'
        WHEN unit_price > 5 THEN 'Mid-Range'
        ELSE 'Budget'
    END AS price_tier
FROM supermarket_transactions
GROUP BY product_description, category, unit_price
ORDER BY unit_price DESC;
```

---

## Key Concepts Demonstrated

1. **INNER JOIN**: Returns only matching records from both tables
2. **LEFT JOIN**: Returns all records from the left table, with matching records from the right table (NULL if no match)
3. **WHERE with JOINs**: Filtering results after joining tables
4. **GROUP BY**: Aggregating data across joined tables
5. **COUNT()**: Counting records in joined result sets
6. **SUM()**: Calculating totals across joined tables
7. **ORDER BY**: Sorting results from joined queries
8. **HAVING**: Filtering aggregated results
9. **Multiple JOINs**: Joining three or more tables
10. **Complex Queries**: Combining multiple SQL concepts

---

## Practice Exercises

1. Find all products that have never been sold.
2. Calculate the average transaction value per customer.
3. List cashiers who haven't processed any transactions.
4. Show the top 5 product categories by revenue.
5. Find customers who made purchases on multiple different dates.
6. Calculate the total discount amount given across all transactions.
7. Show the busiest day (by transaction count) in the database.
8. List all customers with their most purchased product category.

---

## Notes

- All queries are designed to work with the `normalized_db.sql` database structure
- Date formats follow ISO 8601 (YYYY-MM-DD)
- Some database systems may require different string concatenation syntax (e.g., `CONCAT()` instead of `||`)
- Adjust `LIMIT` syntax based on your database system (e.g., `TOP 10` for SQL Server)

