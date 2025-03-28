# Ecommerce-DB
## Table of Contents
- [Overview](#overview)
- [Data Model](#data-model)
- [Inserting Data into E-commerce Database](#inserting-data-into-e-commerce-database)
- [SQL Queries](##sql-queries-initial-query)
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
        INSERT INTO Customer (firsr_name, last_name, email, Password)
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
    host='localhost',
    user='root',
    password='pass',
    database='ecommercedb'
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
        orders.append((order_id, customer_id, order_date.strftime('%Y-%m-%d'), 0.00))

        # Random number of products per order
        for _ in range(random.randint(1, 5)):
            product_id, price = random.choice(products)
            quantity = random.randint(1, 5)
            order_items.append((order_id, product_id, quantity, price))

        order_id += 1

    # Insert orders
    cursor.executemany("""
        INSERT INTO Orders (order_ID, customer_ID, order_date, total_amount)
        VALUES (%s, %s, %s, %s)
    """, orders)

    # Insert order items
    cursor.executemany("""
        INSERT INTO OrderItems (order_ID, product_ID, quantity, unit_price)
        VALUES (%s, %s, %s, %s)
    """, order_items)

    conn.commit()
    print(f"Inserted orders {batch_start + 1} to {batch_start + ORDER_BATCH}")

# Update total_amounts in one query
cursor.execute("""
    UPDATE Orders o
    JOIN (
        SELECT order_ID, SUM(quantity * unit_price) AS total
        FROM OrderItems
        GROUP BY order_ID
    ) oi ON o.order_ID = oi.order_ID
    SET o.total_amount = oi.total;
""")
conn.commit()

print("All done!")
cursor.close()
conn.close()

```
---
## SQL Queries (initial Query)
1. [Retrieve the Total Number of Products in Each Category](#1-retrieve-the-total-number-of-products-in-each-category)
2. [Find the Top Customers by Total Spending](#2-find-the-top-customers-by-total-spending)
3. [Retrieve the Most Recent Orders with Customer Information (1000 Orders)](#3-retrieve-the-most-recent-orders-with-customer-information-1000-orders)
4. [List Products with Low Stock Quantities](#4-list-products-with-low-stock-quantities)
5. [Calculate the Revenue Generated from Each Product Category](#5-calculate-the-revenue-generated-from-each-product-category)

## 1. Retrieve the Total Number of Products in Each Category

This query retrieves the total number of products available in each category.

```sql
SELECT c.category_name,
       count(p.product_ID) AS total_products
FROM category c
LEFT JOIN product p ON p.category_ID=c.category_id
GROUP BY c.category_ID;
```
