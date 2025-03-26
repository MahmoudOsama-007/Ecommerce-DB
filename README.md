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

