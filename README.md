# Домашнее задание 2. Основные операторы PostgreSQL.

### 0. Загрузка и проверка набора данных
#### Создание обьектов базы данных и загрузка при помощи tsql
```
create database hw2;

\c hw2

CREATE TABLE customer (
    customer_id INT,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    gender VARCHAR(50),
    DOB DATE,
    job_title VARCHAR(100),
    job_industry_category VARCHAR(100),
    wealth_segment VARCHAR(100),
    deceased_indicator VARCHAR(10),
    owns_car BOOLEAN,
    address VARCHAR(200),
    postcode VARCHAR(50),
    state VARCHAR(100),
    country VARCHAR(100),
    property_valuation INT
);

\copy customer FROM '/home/george/projects/db/hw02/customer.csv' WITH (FORMAT CSV,DELIMITER ';', HEADER TRUE);

CREATE TABLE product (
    product_id INT PRIMARY KEY,
    brand VARCHAR(100),
    product_line VARCHAR(50),
    product_class VARCHAR(50),
    product_size VARCHAR(
    list_price DECIMAL(10, 2),
    standard_cost DECIMAL(10, 2)
);

-- duplictaing key in raw
CREATE TABLE product (                                                                                      
    product_id INT,
    brand VARCHAR(100),
    product_line VARCHAR(50),
    product_class VARCHAR(50),
    product_size VARCHAR(50),
    list_price DECIMAL(10, 2),
    standard_cost DECIMAL(10, 2)
);

\copy product FROM '/home/george/projects/db/hw02/product.csv' WITH (FORMAT CSV,DELIMITER ',', HEADER TRUE);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    online_order BOOLEAN,
    order_status VARCHAR(50)
);

-- orders содержит "" в поле online_order, по этому не можем использовать BOOLEAN
CREATE TABLE orders (                                                                                     
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    online_order VARCHAR(50),
    order_status VARCHAR(50)
);

\copy orders FROM '/home/george/projects/db/hw02/orders.csv' WITH (FORMAT CSV,DELIMITER ',', HEADER TRUE);

-- order_items
CREATE TABLE order_items (
    order_item_id INT,
    order_id INT,
    product_id INT,
    quantity DECIMAL(10, 2),
    item_list_price_at_sale DECIMAL(10, 2),
    item_standard_cost_at_sale DECIMAL(10, 2)
);

\copy order_items FROM '/home/george/projects/db/hw02/order_items.csv' WITH (FORMAT CSV,DELIMITER ',', HEADER TRUE);
```
#### Модель набора данных
<img width="712" height="635" alt="hw2" src="https://github.com/user-attachments/assets/d2b54d5a-db12-4541-ab93-bdb8a00a81f1" />

#### Аномальные/дублирующиеся значения полей
```
-- аномальные/дублирующиеся значения первичного ключа в таблице product:
WITH anomaly AS (
	SELECT count(p.product_id)
  FROM product p
	GROUP BY p.product_id
	HAVING count(p.product_id) > 1)
SELECT count(*)
FROM anomaly;

-- аномальные значения бренда
SELECT count(*)
FROM product p 
WHERE length(p.brand) = 0;
```

### 1. Вывести все уникальные бренды, у которых есть хотя бы один продукт со стандартной стоимостью выше 1500 долларов, и суммарными продажами не менее 1000 единиц.
```
/*
 1. Вывести все уникальные бренды, у которых есть хотя бы один продукт  
    со стандартной стоимостью выше 1500 долларов, 
    и суммарными продажами не менее 1000 единиц.  
*/
WITH product_sale_quantity AS (
    SELECT oi.product_id, SUM(oi.quantity)
    FROM order_items oi
    GROUP BY oi.product_id 
    HAVING SUM(oi.quantity) >= 1000
)
SELECT DISTINCT p.brand
FROM product p
INNER JOIN product_sale_quantity psq ON p.product_id = psq.product_id 
    AND p.standard_cost > 1500;
```

<img width="1004" height="595" alt="1-1" src="https://github.com/user-attachments/assets/d0833a2a-d367-4a93-a774-752a9ce480d4" />

```
/* Примечание:
   Из описания таблицы orders не понятно являются ли все записи
   таблицы orders совершенными продажами или признак продажи есть 
   статус заказа Approved.
   Если признак продаж есть статус заказа Approved, то в запросе
   учитываем статус заказа как дополнительное условие.
*/
WITH product_sale_quantity AS (
    SELECT oi.product_id, SUM(oi.quantity)
    FROM order_items oi
    INNER JOIN orders o ON o.order_id = oi.order_id
        AND o.order_status = 'Approved'
    GROUP BY oi.product_id 
    HAVING SUM(oi.quantity) >= 1000
)
SELECT DISTINCT p.brand
FROM product p
INNER JOIN product_sale_quantity psq ON p.product_id = psq.product_id 
    AND p.standard_cost > 1500;
```
<img width="1004" height="614" alt="1-2" src="https://github.com/user-attachments/assets/5c2ce145-36d1-47d2-97de-81b247e915ae" />

### 2. Для каждого дня в диапазоне с 2017-04-01 по 2017-04-09 включительно вывести количество подтвержденных онлайн-заказов и количество уникальных клиентов, совершивших эти заказы.

```
/*
 2. Для каждого дня в диапазоне с 2017-04-01 по 2017-04-09 включительно вывести 
    количество подтвержденных онлайн-заказов и количество уникальных клиентов, 
    совершивших эти заказы.
   
   Примечание:
   o.online_order может содержать пустые '' значения, интерпретируем их как FALSE
*/
WITH order_date AS (
    SELECT o.order_date, count(DISTINCT o.order_id) AS order_num
    FROM orders o
    WHERE (o.order_date BETWEEN '2017-04-01' AND '2017-04-09')
        AND COALESCE(NULLIF(o.online_order, '')::BOOLEAN, FALSE) = TRUE 
        AND o.order_status = 'Approved'
    GROUP BY o.order_date
),
customer_date AS (
    SELECT o.order_date, count(DISTINCT o.customer_id) AS customer_num
    FROM orders o
    WHERE o.order_date BETWEEN '2017-04-01' AND '2017-04-09'
        AND COALESCE(NULLIF(o.online_order, '')::BOOLEAN, FALSE) = TRUE 
        AND o.order_status = 'Approved'
    GROUP BY o.order_date
)
SELECT od.order_date, od.order_num, cd.customer_num 
FROM order_date od
INNER JOIN customer_date cd ON cd.order_date = od.order_date
ORDER BY od.order_date;
```
<img width="1004" height="899" alt="2" src="https://github.com/user-attachments/assets/ffa7206c-2758-4587-9273-85a2ad3c807e" />

### 3. Вывести профессии клиентов:
- из сферы IT, чья профессия начинается с Senior;
- из сферы Financial Services, чья профессия начинается с Lead.
- Для обеих групп учитывать только клиентов старше 35 лет. Объединить выборки с помощью UNION ALL.

```
/*
3. Вывести профессии клиентов:
   из сферы IT, чья профессия начинается с Senior;
   из сферы Financial Services, чья профессия начинается с Lead.
   Для обеих групп учитывать только клиентов старше 35 лет.
   Объединить выборки с помощью UNION ALL.
*/
SELECT job_title
FROM customer c 
WHERE c.job_title LIKE 'Senior%'
    AND c.job_industry_category = 'IT'
    AND EXTRACT(YEAR FROM age(NOW(), c.dob)) > 35
UNION ALL 
SELECT c.job_title
FROM customer c 
WHERE c.job_title LIKE 'Lead%'
    AND c.job_industry_category = 'Financial Services'
    AND EXTRACT(YEAR FROM age(NOW(), c.dob)) > 35;
```
<img width="1004" height="633" alt="3" src="https://github.com/user-attachments/assets/77b2d564-7fe1-4ae5-84d0-50bc5d4b29f9" />

### 4. Вывести бренды, которые были куплены клиентами из сферы Financial Services, но не были куплены клиентами из сферы IT.

```
/*
 4. Вывести бренды, которые были куплены клиентами из сферы Financial Services, 
    но не были куплены клиентами из сферы IT.
*/
SELECT DISTINCT p.brand 
FROM product p 
INNER JOIN order_items oi ON p.product_id = oi.product_id 
INNER JOIN orders o ON o.order_id = oi.order_id 
INNER JOIN customer c ON c.customer_id = o.customer_id 
WHERE c.job_industry_category = 'Financial Services'
EXCEPT
SELECT DISTINCT p.brand 
FROM product p 
INNER JOIN order_items oi ON p.product_id = oi.product_id 
INNER JOIN orders o ON o.order_id = oi.order_id 
INNER JOIN customer c ON c.customer_id = o.customer_id 
WHERE c.job_industry_category = 'IT';
```
<img width="1004" height="557" alt="4" src="https://github.com/user-attachments/assets/53f5ab43-1e24-4fd2-b7fc-6b4d5e2d3b3f" />

### 5. Вывести 10 клиентов (ID, имя, фамилия), которые совершили наибольшее количество онлайн-заказов (в штуках) брендов Giant Bicycles, Norco Bicycles, Trek Bicycles, при условии, что они активны и имеют оценку имущества (property_valuation) выше среднего среди клиентов из того же штата.

```
/*
 5. Вывести 10 клиентов (ID, имя, фамилия), которые совершили наибольшее количество 
    онлайн-заказов (в штуках) брендов Giant Bicycles, Norco Bicycles, Trek Bicycles, 
    при условии, что они активны и имеют оценку имущества (property_valuation) 
    выше среднего среди клиентов из того же штата.
*/
WITH state_avg_property_valuation AS (
    SELECT c.state, AVG(c.property_valuation )
    FROM customer c 
    GROUP BY c.state),
selected_brand AS (
    SELECT p.product_id
    FROM product p
    WHERE p.brand IN ('Giant Bicycles', 'Norco Bicycles', 'Trek Bicycles')
),
customer_order_count AS (
    SELECT c.customer_id, count(o.order_id) AS order_count
    FROM customer c
    INNER JOIN orders o
        ON o.customer_id  = c.customer_id
            AND COALESCE(NULLIF(o.online_order, '')::BOOLEAN, FALSE) = TRUE
            AND c.deceased_indicator = 'N'
    INNER JOIN order_items oi ON oi.order_id = o.order_id 
    INNER JOIN state_avg_property_valuation sapv
        ON sapv.state = c.state
            AND c.property_valuation > sapv.avg
    INNER JOIN selected_brand sb ON sb.product_id = oi.product_id
    GROUP BY c.customer_id  
    ORDER BY count(o.order_id) DESC 
    LIMIT 10
 )
 SELECT c.customer_id, c.first_name, c.last_name
 FROM customer c 
 INNER JOIN customer_order_count coc ON coc.customer_id = c.customer_id;
```
<img width="1004" height="1032" alt="5" src="https://github.com/user-attachments/assets/dd9e5829-90d0-4fc4-aaba-09caf249222b" />

### 6. Вывести всех клиентов (ID, имя, фамилия), у которых нет подтвержденных онлайн-заказов за последний год, но при этом они владеют автомобилем и их сегмент благосостояния не Mass Customer.

```
/*
 6. Вывести всех клиентов (ID, имя, фамилия),
    у которых нет подтвержденных онлайн-заказов за последний год,
    но при этом они владеют автомобилем и их сегмент благосостояния не Mass Customer.
    
    Или другими словами, выбрать таких клиентов которые
    имеют машину, в сегменте не 'Mass Customer'
    и у которых небыло заказов за прошедший год или были но в состояние не Approved
    
 Примечание:
 1) o.online_order может содержать пустые '' значения, интерпретируем их как FALSE
 2) последний год отсчитываем от CURRENT_DATE.
 
*/
WITH customer_last_year AS (
    SELECT o.customer_id, o.order_id
    FROM orders o 
    WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 year'
        AND o.order_status = 'Approved'
        AND COALESCE(NULLIF(o.online_order, '')::BOOLEAN, FALSE) = TRUE
)
SELECT DISTINCT c.customer_id, c.first_name, c.last_name 
FROM customer c
LEFT JOIN customer_last_year cly ON cly.customer_id = c.customer_id
WHERE c.owns_car = TRUE
    AND (NOT c.wealth_segment = 'Mass Customer')
    AND cly.order_id IS NULL 
```
<img width="1004" height="956" alt="6" src="https://github.com/user-attachments/assets/3601e261-a134-4705-beb5-6c32d2770308" />

### 7. Вывести всех клиентов из сферы 'IT' (ID, имя, фамилия), которые купили 2 из 5 продуктов с самой высокой list_price в продуктовой линейке Road.

```
/*
 7. Вывести всех клиентов из сферы 'IT' (ID, имя, фамилия),
    которые купили 2 из 5 продуктов с самой высокой list_price в продуктовой линейке Road.
*/
SELECT c.customer_id, c.first_name, c.last_name 
FROM customer c 
WHERE c.job_industry_category = 'IT'
    AND c.customer_id IN (
        SELECT o.customer_id
        FROM orders o 
        INNER JOIN order_items oi ON oi.order_id = o.order_id 
            AND oi.product_id IN (
                SELECT p.product_id 
                FROM product p 
                WHERE p.product_line = 'Road'
                ORDER BY p.list_price DESC
                LIMIT 5)
        GROUP BY customer_id 
        HAVING count(customer_id) = 2)
```
<img width="1004" height="747" alt="7" src="https://github.com/user-attachments/assets/78542a39-ac7b-433c-956c-95e742c36d56" />

### 8. Вывести клиентов (ID, имя, фамилия, сфера деятельности) из сфер IT или Health, которые совершили не менее 3 подтвержденных заказов в период 2017-01-01 по 2017-03-01, и при этом их общий доход от этих заказов превышает 10 000 долларов. Разделить вывод на две группы (IT и Health) с помощью UNION.

```
/*
 8. Вывести клиентов (ID, имя, фамилия, сфера деятельности) из сфер IT или Health,
    которые совершили не менее 3 подтвержденных заказов в период 2017-01-01 по 2017-03-01,
    и при этом их общий доход от этих заказов превышает 10 000 долларов.
    Разделить вывод на две группы (IT и Health) с помощью UNION.
*/
WITH customer_order AS (
	SELECT o.customer_id
	FROM orders o
	INNER JOIN order_items oi
	    ON oi.order_id = o.order_id
	        AND (o.order_date BETWEEN '2017-01-01' AND '2017-03-01')
	        AND o.order_status = 'Approved'
	GROUP BY o.customer_id 
	HAVING  count(o.order_id ) >= 3
	    AND SUM(oi.item_list_price_at_sale * oi.quantity) > 10000
)
SELECT c.customer_id, c.first_name, c.last_name, c.job_industry_category
FROM customer c
INNER JOIN customer_order co ON co.customer_id = c.customer_id 
WHERE c.job_industry_category = 'IT'
UNION 
SELECT c.customer_id, c.first_name, c.last_name, c.job_industry_category
FROM customer c
INNER JOIN customer_order co ON co.customer_id = c.customer_id 
WHERE c.job_industry_category = 'Health';
```

<img width="1004" height="937" alt="8" src="https://github.com/user-attachments/assets/333d6205-57a5-48bb-baed-f6cf846070e0" />

```
--- Вариант без использования UNION
WITH customer_order AS (
	SELECT o.customer_id
	FROM orders o
	INNER JOIN order_items oi
	    ON oi.order_id = o.order_id
	        AND (o.order_date BETWEEN '2017-01-01' AND '2017-03-01')
	        AND o.order_status = 'Approved'
	GROUP BY o.customer_id 
	HAVING  count(o.order_id ) >= 3
	    AND SUM(oi.item_list_price_at_sale  * oi.quantity) > 10000
)
SELECT c.customer_id, c.first_name, c.last_name, c.job_industry_category
FROM customer c
INNER JOIN customer_order co ON co.customer_id = c.customer_id 
WHERE c.job_industry_category IN ('IT', 'Health')
ORDER BY c.job_industry_category;
```
<img width="1004" height="766" alt="8-1" src="https://github.com/user-attachments/assets/ebf0720e-1a3e-40ce-adb9-9b4a6a6cec6c" />

