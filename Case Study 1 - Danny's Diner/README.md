# Case Study 1 - Danny's Diner 

## Problem Statement
Danny wants to analyze customer data to understand their visiting patterns, spending habits, and favorite menu items. This will help him provide a better experience for loyal customers. He also wants to use insights to decide on expanding the loyalty program and needs help generating basic datasets for easy data inspection. He has provided three key datasets: 
* sales
* menu
* members

## ER Diagram
<img width="632" alt="Screenshot 2024-04-12 at 9 13 46 PM" src="https://github.com/dustin-tucker01/8WeekSQLChallenge/assets/148160789/7a5f4668-f889-4136-8e47-17acf101a1bd">


## Questions 
### What is the total amount each customer spent at the restaurant?
```sql
SELECT s.customer_id, SUM(m.price) as total_amount_spent
FROM sales s
JOIN menu m 
ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id;
```
*Output*

| customer_id | total_amount_spent |
|-------------|--------------------|
| A           | 76                 |
| B           | 74                 |
| C           | 36                 |


### How many days has each customer visited the restaurant?
```sql
SELECT customer_id, COUNT(DISTINCT order_date)
FROM sales 
GROUP BY customer_id;
```
*Output*
| customer_id | days_visited |
|-------------|--------------|
| A           | 4            |
| B           | 6            |
| C           | 2            |

### What was the first item from the menu purchased by each customer?
```sql
SELECT DISTINCT s.customer_id, m.product_name 
FROM sales s
JOIN menu m
ON s.product_id = m.product_id 
WHERE order_date = (SELECT MIN(order_date)
                    FROM sales ss 
                    WHERE s.customer_id = ss.customer_id);
```
*Output*
| customer_id | product_name |
|-------------|--------------|
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

### What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT product_name, COUNT(m.product_name) AS purchases
FROM sales s 
JOIN menu m 
ON s.product_id = m.product_id 
GROUP BY product_name
ORDER BY purchases DESC 
LIMIT 1;
```
*Output*
| product_name | purchases |
|--------------|-----------|
| ramen        | 8         |

### Which item was the most popular for each customer?
```sql
WITH purchase_data AS 
(SELECT s.customer_id, m.product_name, COUNT(m.product_name) AS purchases
FROM sales s 
JOIN menu m 
ON s.product_id = m.product_id
GROUP BY s.customer_id, m.product_name
ORDER BY customer_id, purchases DESC) 
SELECT customer_id, product_name AS most_popular_product, purchases
FROM purchase_data s
WHERE purchases = (SELECT MAX(purchases) 
                   FROM purchase_data ss
                   WHERE s.customer_id = ss.customer_id);
```
*Output*

| customer_id | most_popular_product | purchases |
|-------------|----------------------|-----------|
| A           | ramen                | 3         |
| B           | sushi                | 2         |
| B           | curry                | 2         |
| B           | ramen                | 2         |
| C           | ramen                | 3         |


### Which item was purchased first by the customer after they became a member?
```sql
WITH ordered_products AS
(SELECT s.customer_id, mm.product_name, DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date) AS rnk      
FROM sales s
JOIN members m
ON s.customer_id = m.customer_id
JOIN menu mm
ON s.product_id = mm.product_id
WHERE join_date < order_date)
SELECT customer_id, product_name AS first_purchase
FROM ordered_products
WHERE rnk = 1 ;
```
*Output*

| customer_id | first_purchase |
|-------------|----------------|
| A           | ramen          |
| B           | sushi          |

### Which item was purchased just before the customer became a member?
```sql
WITH ordered_products AS
(SELECT s.customer_id, mm.product_name, 
DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date DESC) AS rnk     
FROM sales s
JOIN members m
ON s.customer_id = m.customer_id
JOIN menu mm
ON s.product_id = mm.product_id
WHERE join_date > order_date)
SELECT customer_id, product_name AS last_purchase
FROM ordered_products
WHERE rnk = 1 ;
```
*Output*

| customer_id | last_purchase |
|-------------|---------------|
| A           | sushi         |
| A           | curry         |
| B           | sushi         |

### What is the total items and amount spent for each member before they became a member?
```sql
SELECT s.customer_id, COUNT(s.product_id) AS total_products, SUM(mm.price) AS amount_spent
FROM sales s
JOIN members m
ON s.customer_id = m.customer_id
JOIN menu mm
ON s.product_id = mm.product_id
WHERE s.order_date < m.join_date
GROUP BY s.customer_id
ORDER BY customer_id;
```
*Output*

| customer_id | total_products | amount_spent |
|-------------|----------------|--------------|
| A           | 2              | 25           |
| B           | 3              | 40           |

### If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
```sql
SELECT s.customer_id,
SUM(CASE WHEN m.product_name = 'sushi' THEN (m.price * 10)*2 ELSE price*10 END) AS points
FROM sales s
JOIN menu m 
ON s.product_id = m.product_id
GROUP BY customer_id
ORDER BY customer_id;
```
*Output*
| customer_id | points |
|-------------|--------|
| A           | 860    |
| B           | 940    |
| C           | 360    |

### In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
```sql
SELECT s.customer_id,
SUM(CASE WHEN s.order_date BETWEEN m.join_date AND m.join_date + INTERVAL '6 days' THEN (mm.price*10)*2 ELSE (mm.price * 10) END) AS points
FROM sales s
JOIN members m
ON s.customer_id = m.customer_id
JOIN menu mm  
ON mm.product_id = s.product_id 
WHERE s.order_date >= m.join_date AND EXTRACT(MONTH FROM s.order_date) = 1
GROUP BY s.customer_id
ORDER BY customer_id; 
```
*Output*
| customer_id | points |
|-------------|--------|
| A           | 1020   |
| B           | 320    |

## BONUS 
### Join All The Things
```sql
SELECT s.customer_id, s.order_date, m.product_name, m.price,  
CASE WHEN order_date < join_date OR join_date IS NULL THEN 'N' ELSE 'Y' END AS member
FROM sales s 
JOIN menu m
ON s.product_id = m.product_id
LEFT JOIN members mm 
ON mm.customer_id = s.customer_id
ORDER BY customer_id, order_date; 
```
*Output* 
| customer_id | order_date | product_name | price | member |
|-------------|------------|--------------|-------|--------|
| A           | 2021-01-01 | sushi        | 10    | N      |
| A           | 2021-01-01 | curry        | 15    | N      |
| A           | 2021-01-07 | curry        | 15    | Y      |
| A           | 2021-01-10 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| A           | 2021-01-11 | ramen        | 12    | Y      |
| B           | 2021-01-01 | curry        | 15    | N      |
| B           | 2021-01-02 | curry        | 15    | N      |
| B           | 2021-01-04 | sushi        | 10    | N      |
| B           | 2021-01-11 | sushi        | 10    | Y      |
| B           | 2021-01-16 | ramen        | 12    | Y      |
| B           | 2021-02-01 | ramen        | 12    | Y      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-01 | ramen        | 12    | N      |
| C           | 2021-01-07 | ramen        | 12    | N      |

### Rank All The Things
```sql
WITH joined_table AS
(SELECT s.customer_id, s.order_date, m.product_name, m.price,  
CASE WHEN order_date < join_date OR join_date IS NULL THEN 'N' ELSE 'Y' END AS member 
FROM sales s 
JOIN menu m
ON s.product_id = m.product_id
LEFT JOIN members mm 
ON mm.customer_id = s.customer_id
ORDER BY customer_id, order_date) 
SELECT *,
CASE WHEN member = 'N' THEN NULL ELSE DENSE_RANK() OVER (PARTITION BY customer_id, member ORDER BY order_date) END AS ranking
FROM joined_table;
```
*Output* 
| customer_id | order_date | product_name | price | member | ranking    |
|-------------|------------|--------------|-------|--------|------------|
| A           | 2021-01-01 | sushi        | 10    | N      |            |
| A           | 2021-01-01 | curry        | 15    | N      |            |
| A           | 2021-01-07 | curry        | 15    | Y      | 1          |
| A           | 2021-01-10 | ramen        | 12    | Y      | 2          |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3          |
| A           | 2021-01-11 | ramen        | 12    | Y      | 3          |
| B           | 2021-01-01 | curry        | 15    | N      |            |
| B           | 2021-01-02 | curry        | 15    | N      |            |
| B           | 2021-01-04 | sushi        | 10    | N      |            |
| B           | 2021-01-11 | sushi        | 10    | Y      | 1          |
| B           | 2021-01-16 | ramen        | 12    | Y      | 2          |
| B           | 2021-02-01 | ramen        | 12    | Y      | 3          |
| C           | 2021-01-01 | ramen        | 12    | N      |            |
| C           | 2021-01-01 | ramen        | 12    | N      |            |
| C           | 2021-01-07 | ramen        | 12    | N      |            |
