# Ecommerce-DB
## Table of Contents
- [Overview](#overview)
- [Data Model](#data-model)
- [Inserting Data into E-commerce Database](#inserting-data-into-e-commerce-database)
- [SQL Queries](#sql-queries)
- [Query Performance Optimization](#query-performance-optimization)
---
## Overview
This project focuses on optimizing SQL queries for a simulated e-commerce database system containing large datasets across multiple tables including `products`, `customers`, `categories`, `orders`, and `orderItems`
. The main 
objective is to analyze and improve the performance of key business queries using MySQL's EXPLAIN ANALYZE tool.

---
## Data Model
### ERD
![erd](https://github.com/user-attachments/assets/f1f368a0-c1de-40da-bea0-a3bcd878b753)

### Tables Schema
```sql
CREATE TABLE Customer (
    customer_ID INT PRIMARY KEY AUTO_INCREMENT,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100) UNIQUE,
    Password VARCHAR(200)
);

CREATE TABLE Category (
    category_id INT PRIMARY KEY AUTO_INCREMENT,
    category_name VARCHAR(100)
);

CREATE TABLE Product (
    Product_ID INT PRIMARY KEY AUTO_INCREMENT,
    category_ID INT NOT NULL,
    name VARCHAR(100),
    description VARCHAR(200),
    price DECIMAL(15,2),
    stock_quantity INT,
    FOREIGN KEY (category_ID) REFERENCES Category(category_id)
);

CREATE TABLE Orders (
    order_ID INT PRIMARY KEY AUTO_INCREMENT,
    Customer_ID INT NOT NULL,
    order_date DATE,
    total_amount DECIMAL(6,2),
    FOREIGN KEY (Customer_ID) REFERENCES Customer(customer_ID)
);

CREATE TABLE OrderItems (
    order_item_id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT,
    unit_price DECIMAL(6,2),
    FOREIGN KEY (order_id) REFERENCES Orders(order_ID),
    FOREIGN KEY (product_id) REFERENCES Product(Product_ID)
);

```
---
## Inserting Data into E-commerce Database

1. [Insert Categories](#insert-categories)
2. [Insert Products](#insert-products)
3. [Insert Customers](#insert-customers)
4. [Insert Orders and OrderItems](#insert-orders-and-orderitems)

### Insert Categories
This procedure inserts 100 categories into the `Category` table
```sql
DELIMITER $$

CREATE PROCEDURE InsertCategories()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 100 DO
        INSERT INTO Category (category_name) 
        VALUES (CONCAT('Category ', i));
        SET i = i + 1;
    END WHILE;
END $$

DELIMITER ;
```
### Insert Products
This procedure inserts 100k products into the `Product` table
```sql

DELIMITER $$
CREATE PROCEDURE InsertProducts()
BEGIN
    DECLARE i INT DEFAULT 0;
    
    WHILE i <= 100000 DO
        INSERT INTO Product (category_ID, name, description, price, stock_quantity)
        VALUES (
            FLOOR(1 + RAND() * 100), -- Assuming 100 categories exist
            CONCAT('Product ', i), 
            CONCAT('Description for product ', i), 
            ROUND(RAND() * 1000, 2), -- Random price between 0 and 1000
            FLOOR(RAND() * 500) -- Random stock between 0 and 500
        );
        
        SET i = i + 1;
    END WHILE;
    
END $$

DELIMITER ;

```
### Insert Customers
This procedure inserts 1m Customers into the `Customer` table
```sql

DELIMITER $$

CREATE PROCEDURE InsertCustomers()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 1000000 DO
        INSERT INTO Customer (first_name, last_name, email, Password)
        VALUES (
            CONCAT('FirstName', i),                    -- First Name
            CONCAT('LastName', i),                     -- Last Name
            CONCAT('customer', i, '@example.com'),     -- Email
            LEFT(SHA1(CONCAT('Pass', i)), 10)
        );
        SET i = i + 1;
    END WHILE;
END $$

DELIMITER ;


```
### Insert Orders and OrderItems
This procedure inserts 3.5m orders and random orderItems, based on customers and products, and calculating the total order amount.
```sql
DELIMITER $$

CREATE PROCEDURE InsertOrdersAndItems()
BEGIN
    DECLARE orderIndex INT DEFAULT 1;
    DECLARE customerID INT;
    DECLARE orderID INT;
    DECLARE productCount INT;
    DECLARE productIndex INT;
    DECLARE productID INT;
    DECLARE productPrice DECIMAL(6,2);


    -- create 1000000 orders 
    WHILE orderIndex <= 1000000 DO
        -- Get Random customer_id
        SELECT customer_ID INTO customerID
        FROM Customer
        ORDER BY RAND()
        LIMIT 1;

        -- Insert an order for this customer
        INSERT INTO Orders (Customer_ID, order_date, total_amount)
        VALUES (customerID, DATE_ADD('2023-01-01', INTERVAL FLOOR(RAND() * 1095) DAY), 0.00);

        -- Get the last inserted order ID
        SET orderID = orderIndex;

        -- Get the total number of products
        SELECT COUNT(*) INTO productCount FROM Product;

        -- Insert 1 to 5 random items for this order
        SET productIndex = 1;
        WHILE productIndex <= FLOOR(1 + RAND() * 5) DO
            -- Select a random product
            SELECT Product_ID, price INTO productID, productPrice
            FROM Product
            ORDER BY RAND()
            LIMIT 1;

            -- Insert order item
            INSERT INTO OrderItems (order_id, product_id, quantity, unit_price)
            VALUES (orderID, productID, FLOOR(1 + RAND() * 5), productPrice);

            -- Update order's total amount
            UPDATE Orders
            SET total_amount = total_amount + (FLOOR(1 + RAND() * 5) * productPrice)
            WHERE order_ID = orderID;

            SET productIndex = productIndex + 1;
        END WHILE;

        SET orderIndex = orderIndex + 1;
    END WHILE;
END $$

DELIMITER ;

```
### Explanation:
1. **Loop to create orders** and pick random `customer_id`
    - Inserts a new order with the random date and a `total_amount` initialized to `0.00`.
    - Random date from 2023 to 2025
    `DATE_ADD('2023-01-01', INTERVAL FLOOR(RAND() * 1095) DAY)`
        - `'2023-01-01'` is the starting date.
        - `RAND() * 1095` generates a random number of days (1095 = 3 years × 365 days).
        - `FLOOR()` ensures we get a whole number of days.
        - `DATE_ADD()` shifts the date forward by that many days.
2. **Insert Order Items**: For each order:
    - Inserts **1 to 5** random order items. (loop)
        - Selects a random product for each order item.
        - Randomly generates quantity between **1 and 5**.
        - Updates the order’s `total_amount` based on the inserted items.
### OR you can use py script to make it faster
```python
import mysql.connector
import random
from datetime import datetime, timedelta

# Connect to MySQL
conn = mysql.connector.connect(
    host="localhost", user="root", password="mahmoud", database="ecommercedb"
)
cursor = conn.cursor()

# Fetch all customer IDs and product info in memory
cursor.execute("SELECT customer_ID FROM Customer")
customer_ids = [row[0] for row in cursor.fetchall()]

cursor.execute("SELECT product_ID, price FROM Product")
products = cursor.fetchall()

# Batch size
ORDER_BATCH = 10000
TOTAL_ORDERS = 3500000

print("Starting insert...")

order_id = 1
for batch_start in range(0, TOTAL_ORDERS, ORDER_BATCH):
    orders = []
    order_items = []

    for _ in range(ORDER_BATCH):
        customer_id = random.choice(customer_ids)
        order_date = datetime(2023, 1, 1) + timedelta(days=random.randint(0, 1095))
        order_total = 0

        # Random number of products per order
        for _ in range(random.randint(1, 5)):
            product_id, price = random.choice(products)
            quantity = random.randint(1, 5)
            order_items.append((order_id, product_id, quantity, price))
            totalprice = price * quantity
            order_total += totalprice

        orders.append(
            (order_id, customer_id, order_date.strftime("%Y-%m-%d"), order_total)
        )

        order_id += 1

    # Insert orders
    cursor.executemany(
        """
        INSERT INTO Orders (order_ID, customer_ID, order_date, total_amount)
        VALUES (%s, %s, %s, %s)
    """,
        orders,
    )

    # Insert order items
    cursor.executemany(
        """
        INSERT INTO OrderItems (order_ID, product_ID, quantity, unit_price)
        VALUES (%s, %s, %s, %s)
    """,
        order_items,
    )

    conn.commit()
    print(f"Inserted orders {batch_start + 1} to {batch_start + ORDER_BATCH}")
print("All done!")
cursor.close()
conn.close()


```
---
## SQL Queries
1. [Retrieve the Total Number of Products in Each Category](#1-retrieve-the-total-number-of-products-in-each-category)
2. [Find the Top Customers by Total Spending](#2-find-the-top-customers-by-total-spending)
3. [Retrieve the Most Recent Orders with Customer Information (1000 Orders)](#3-retrieve-the-most-recent-orders-with-customer-information-1000-orders)
4. [List Products with Low Stock Quantities](#4-list-products-with-low-stock-quantities)
5. [Calculate the Revenue Generated from Each Product Category](#5-calculate-the-revenue-generated-from-each-product-category)

## 1. Retrieve the Total Number of Products in Each Category

### initial Query
```sql
SELECT c.category_name,
       count(p.product_ID) AS total_products
FROM category c
LEFT JOIN product p ON p.category_ID=c.category_id
GROUP BY c.category_name;
```
### Problem
![image](https://github.com/user-attachments/assets/b55383a7-9c9f-40a1-a83d-99b06d79f320)
- grouping by `c.category_name` but does not have index on it
- JOIN operation before aggregation

### Optimization 
 Aggregate before join	
```sql
SELECT
    c.category_id,
    p.product_count AS total_products
FROM
    category c
INNER JOIN (
    SELECT
        category_id,
        COUNT(*) as product_count
    FROM
        product
    GROUP BY
        category_id
) p ON c.category_id = p.category_id;
```
![image](https://github.com/user-attachments/assets/adf6d217-fa65-4919-a383-d54a3d232e59)

## 2. Find the Top Customers by Total Spending
### initial Query
```sql
SELECT concat(c.first_name, ' ', c.last_name)AS customer_name,
       sum(total_amount) AS total_sum,
       c.Customer_ID
FROM orders o
JOIN customer c ON o.customer_id=c.customer_id
GROUP BY Customer_ID
ORDER BY total_sum DESC
LIMIT 10;
```
![image](https://github.com/user-attachments/assets/ed44b381-108f-49d2-9c05-26018a9d7f75)

### Problem

- using `CONCAT` in the SELECT statement might add slight overhead if used repeatedly. This can be avoided by 
    - returning first_name and last_name separately and combining them later in your application layer.
    - OR add column `fullName`
- JOIN operation before aggregation
- table scan on `orders` because you will need to sort based on total_sum -> sum(total_amount)
    - solve it by creating cover index
### Optimization
- create cover index
  ```sql
  CREATE index idx_orders_customer_amount ON orders (Customer_ID,total_amount);
  ```
CTE
```sql 
WITH TOTAL_SPENDING AS (
SELECT Customer_ID, SUM(total_amount) AS TOTAL_SPENDING 
     FROM orders 
     GROUP BY Customer_ID 
     ORDER BY TOTAL_SPENDING DESC 
     LIMIT 10
)
SELECT c.Customer_ID, 
    c.first_name, 
    c.last_name,  TS.TOTAL_SPENDING
FROM CUSTOMER C
JOIN TOTAL_SPENDING TS ON C.CUSTOMER_ID = TS.CUSTOMER_ID;
```
![image](https://github.com/user-attachments/assets/0ac284d8-d9c3-4755-81ab-37858a47741f)

## 3. Retrieve the Most Recent Orders with Customer Information (1000 Orders)
### initial Query
```sql
SELECT c.customer_ID,
       concat(c.first_name, ' ', c.last_name) AS customer_name,
       o.order_ID,
       o.order_date
FROM orders o
JOIN customer c ON c.customer_ID=o.customer_ID
ORDER BY o.order_date DESC
LIMIT 1000;
```
### Problem
![image](https://github.com/user-attachments/assets/f3d62df4-43eb-4a31-865c-eb82e3613bbb)
- Sorting on `order_date` Without an Index
- Sorting large dataset after join

### Optimization 
- create index on order_data
    ```sql
        CREATE Index idx_orders_data ON orders(order_date);
    ```
CTE 
```sql
with recent_orders as (
	SELECT Customer_ID,
          order_date
   FROM orders
   ORDER BY order_date DESC
   LIMIT 1000
)SELECT concat(c.first_name, ' ', c.last_name) AS customer_name,
       recent_orders .order_date
FROM
  recent_orders 
JOIN customer c ON c.customer_ID=recent_orders .customer_ID;
```
![image](https://github.com/user-attachments/assets/f5b17b49-7209-4fc9-b473-7c69d9cae620)

## 4. List Products with Low Stock Quantities

### initial Query
```sql
SELECT name,
       Product_ID,
       stock_quantity
FROM product
WHERE stock_quantity<10;
```
### Problem
![image](https://github.com/user-attachments/assets/56fa1ad7-f9c7-40a8-87ba-2e71b7f7d16e)
- table scan stock quantity
### Optimization
Create index on 
```sql
create INDEX idx_stock_quantity on product(stock_quantity)
```
![image](https://github.com/user-attachments/assets/3d813535-c72c-47c4-9c80-bd67347dd6cf)

## 5. Calculate the Revenue Generated from Each Product Category
### initial Query
```sql
SELECT p.category_ID,
       sum(sub.total_price) AS Revenue
FROM product p
JOIN
  (SELECT product_id,
          sum(quantity) * max(unit_price) AS total_price
   FROM orderitems
   GROUP BY product_id) AS sub ON sub.product_id = p.product_id
GROUP BY p.category_ID;
```
### Problem
![image](https://github.com/user-attachments/assets/65c77b0b-0e0b-4781-b4e8-935c4586aa0b)
### 1. Nested Loop Join
- nested loop (inner join), which is highly inefficient for large datasets

### 2. Subquery Performance
```sql 
SELECT product_id,
       sum(quantity) * max(unit_price) AS total_price
FROM orderitems
GROUP BY product_id

```
- performs an aggregation (GROUP BY) on `orderitems`, which can be slow if `orderitems` contains many rows.

- Inefficient aggregation strategy: The optimizer might not be able to use indexes effectively for both `SUM(quantity)` and `MAX(unit_price)`.
- No pre-aggregated data

### 3. Index Scans on Large Tables
- The query plan shows an index scan on orderitems using `product_id` and `orderitems` table likely has millions    of rows, making this scan expensive.
- Solution: A composite index on (`product_id`, `unit_price`, `quantity`) could speed this up.

### Optimization

### Approche 1: CTE + CoverIndex
#### Use an Indexed Approach for Aggregation
- Instead of computing `SUM(quantity) * MAX(unit_price)`, create an index on `orderitems(product_id, quantity, unit_price)`.
```sql
CREATE INDEX idx_orderitems1 ON orderitems(product_id, unit_price, quantity);
CREATE INDEX idx_product ON product(product_id, category_ID);

```
#### More Efficient JOIN Strategy by using CTE  instead of aggregation in a subquery
 ```sql 
 WITH orderitems_agg AS (
    SELECT product_id, 
           SUM(quantity) AS total_quantity,
           MAX(unit_price) AS max_price
    FROM orderitems
    GROUP BY product_id
)
SELECT p.category_ID, 
       SUM(o.total_quantity * o.max_price) AS Revenue
FROM product p
JOIN orderitems_agg o ON p.product_id = o.product_id
GROUP BY p.category_ID;

 ```
![image](https://github.com/user-attachments/assets/75f53661-9591-463f-a924-1c75fb8d17a1)
### Approche 2: 
---
## Query Performance Optimization
| Query Description                    | Original Execution Time | Optimized Execution Time |
| ------------------------------------ | ----------------------- | ------------------------ |
| Products per Category                | 102 ms                  | 23.4 ms                  |
| Top Customers by Spending            | grater than 30000 ms    | 1670 ms                  |
| Recent Orders with Customer Info     | 3047 ms                 | 413 ms                   |
| Low Stock Products                   | 78 ms              	 | 8 ms      		    |
| Category Revenue                     | -	                 | -                        |
