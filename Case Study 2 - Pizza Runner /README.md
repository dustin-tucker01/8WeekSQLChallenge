# Case Study 2 - Pizza Runner
# Table of contents 
* [Data Cleaning](#Data-Cleaning) 
* [Questions](#Questions)
  * A. [Pizza Metrics](#A.-Pizza-Metrics)
  * B. [Runner and Customer Experience](#B.-Runner-and-Customer-Experience) 
  * C. [Ingredient Optimization](#C.-Ingredient-Optimization) (the most challenging)
  * D. [Pricing and Ratings](#D.-Pricing-and-Ratings)

## Problem Statement
Danny, inspired by the global love for pizza and a vision of combining 80s Retro Styling with Pizza, launched Pizza Runner. 
To expand his Pizza Empire, he infused his idea with an Uber-like model, birthing Pizza Runner. Recruiting "runners" to deliver fresh pizzas, he initiated operations from his house, while simultaneously developing a mobile app to streamline orders. 
Leveraging his expertise as a data scientist, Danny prioritized data collection, recognizing its pivotal role in business growth. Now, he seeks assistance in cleaning and analyzing data to optimize Pizza Runner's operations.

## Available Data
Within the pizza_runner database schema, Danny has provided the following tables for analysis:
1. runners
2. customer_orders
3. runner_orders
4. pizza_names
5. pizza_recipes
6. pizza_toppings

## ER Diagram
<img width="616" alt="Screenshot 2024-04-12 at 9 24 07 PM" src="https://github.com/dustin-tucker01/8WeekSQLChallenge/assets/148160789/b407817a-5b35-49c3-85d6-bee67f61ced1">

## Data Cleaning 
Before going into the data and answering the questions, I first needed to clean up the data to make sure it would be easy to properly aggregate and manipulate the data

### Updating Errors in Tables 
There were a few errors in the runner_orders data regarding the cancellation, duration, distance, and pickup_time columns.
```sql
UPDATE runner_orders
SET 
    cancellation = CASE 
                        WHEN cancellation = 'null' OR cancellation = '' THEN NULL 
                        ELSE cancellation 
                    END,
    duration = CASE 
                    WHEN LENGTH(REGEXP_REPLACE(duration, '[^0-9]', '', 'g')) = 0 THEN NULL 
                    ELSE REGEXP_REPLACE(duration, '[^0-9]', '', 'g') 
                END,
    distance = CASE 
                    WHEN LENGTH(REGEXP_REPLACE(distance, '[^0-9]', '', 'g')) = 0 THEN NULL 
                    ELSE REGEXP_REPLACE(distance, '[^0-9.]', '', 'g') 
                END, 
    pickup_time = CASE 
                        WHEN pickup_time = 'null' THEN NULL 
                        ELSE pickup_time 
                  END;
```
I then needed to fix errors in the customer_orders table within the exclusions and extras columns 
```sql
UPDATE customer_orders
SET 
    exclusions = CASE 
                    WHEN LENGTH(REGEXP_REPLACE(exclusions, '[^0-9,]', '', 'g')) = 0 THEN NULL 
                    ELSE REGEXP_REPLACE(exclusions, '[^0-9,]', '', 'g') 
                END, 
    extras = CASE 
                WHEN LENGTH(REGEXP_REPLACE(extras, '[^0-9,]', '', 'g')) = 0 THEN NULL 
                ELSE REGEXP_REPLACE(extras, '[^0-9,]', '', 'g') 
            END;
```

### Changing Data Types 
I also needed to go ahead and change datatypes to be able to properly work with aggregations and dates later on.
```sql
ALTER TABLE runner_orders
  ALTER COLUMN pickup_time TYPE TIMESTAMP USING pickup_time::TIMESTAMP,
  ALTER COLUMN distance TYPE DECIMAL(5,2) USING distance::DECIMAL(5,2),
  ALTER COLUMN duration TYPE DECIMAL(5,2) USING duration::DECIMAL(5,2); 
```
### Renaming Columns 
I then renamed some columns for clarity. 
```sql
ALTER TABLE runner_orders 
RENAME COLUMN distance TO distance_km;

ALTER TABLE runner_orders 
RENAME COLUMN duration TO duration_min;
```

# Questions
## A. Pizza Metrics

### How many pizzas were ordered?

```sql
SELECT COUNT(pizza_id) AS num_of_pizzas_ordered
FROM customer_orders;
```
*output* 

| num_of_pizzas_ordered |
|-----------------------|
|           14          |

### How many unique customer orders were made?

```sql
SELECT COUNT(*) AS total_customer_orders
FROM runner_orders;
```
*output*

| total_customer_orders |
|-----------------------|
|           10          |

### How many successful orders were delivered by each runner?

```sql
SELECT runner_id, COUNT(order_id) AS successful_deliveries 
FROM runner_orders 
WHERE cancellation IS NULL
GROUP BY runner_id;
```
*output*

| runner_id | successful_deliveries |
|-----------|-----------------------|
|     1     |           4           |
|     2     |           3           |
|     3     |           1           |


### How many of each type of pizza was delivered?

```sql
SELECT p.pizza_name, COUNT(c.pizza_id) AS delivered
FROM customer_orders c
JOIN pizza_names p
ON c.pizza_id = p.pizza_id
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE cancellation IS NULL
GROUP BY p.pizza_name;
```
*output*

| pizza_name  | delivered |
|-------------|-----------|
| Vegetarian  |     3     |
| Meatlovers  |     9     |

### How many Vegetarian and Meatlovers were ordered by each customer?

```sql
SELECT DISTINCT(cc.customer_id),x.pizza_name, (SELECT COUNT(c.pizza_id)
                                               FROM customer_orders c
                                               JOIN pizza_names p
                                               ON c.pizza_id = p.pizza_id
                                               WHERE c.customer_id = cc.customer_id
                                               AND p.pizza_name = x.pizza_name) AS num_of_orders
FROM customer_orders cc
CROSS JOIN 
(SELECT pizza_name
FROM pizza_names) x
ORDER BY cc.customer_id, x.pizza_name;
```
*output*

| customer_id | pizza_name  | num_of_orders |
|-------------|-------------|---------------|
|     101     | Meatlovers  |       2       |
|     101     | Vegetarian  |       1       |
|     102     | Meatlovers  |       2       |
|     102     | Vegetarian  |       1       |
|     103     | Meatlovers  |       3       |
|     103     | Vegetarian  |       1       |
|     104     | Meatlovers  |       3       |
|     104     | Vegetarian  |       0       |
|     105     | Meatlovers  |       0       |
|     105     | Vegetarian  |       1       |


### What was the maximum number of pizzas delivered in a single order?

```sql
SELECT COUNT(c.pizza_id) AS max_pizzas_delivered
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE r.cancellation IS NULL
GROUP BY c.order_id
ORDER BY max_pizzas_delivered DESC
LIMIT 1;
```
*output*

| max_pizzas_delivered |
|----------------------|
|           3          |

### For each customer, how many delivered pizzas had at least 1 change and how many had no changes?

```sql
SELECT c.customer_id, 
SUM(CASE WHEN c.exclusions IS NULL AND c.extras IS NULL THEN 1 ELSE 0 END) AS no_changes, 
SUM(CASE WHEN c.exclusions IS NOT NULL OR c.extras IS NOT NULL THEN 1 ELSE 0 END) AS atleast_1_change
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE r.cancellation IS NULL
GROUP BY c.customer_id;
```
*output*

| customer_id | no_changes | atleast_1_change |
|-------------|------------|------------------|
|     101     |     2      |         0        |
|     102     |     3      |         0        |
|     103     |     0      |         3        |
|     104     |     1      |         2        |
|     105     |     0      |         1        |

### How many pizzas were delivered that had both exclusions and extras?

```sql
SELECT SUM(CASE WHEN c.exclusions IS NOT NULL AND c.extras IS NOT NULL THEN 1 ELSE 0 END) AS both_exclusions_extras
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE r.cancellation IS NULL;
```
*output*

| both_exclusions_extras |
|------------------------|
|           1            |


### What was the total volume of pizzas ordered for each hour of the day?

```sql
WITH cte AS
(SELECT generate_series AS hour
FROM generate_series(1,24)),
main AS 
(SELECT EXTRACT(HOUR FROM order_time) AS hour, COUNT(pizza_id) AS orders
FROM customer_orders
GROUP BY EXTRACT(HOUR FROM order_time))
SELECT c.hour, COALESCE(x.orders,0) AS orders
FROM cte c
LEFT JOIN main x
ON x.hour = c.hour
ORDER BY c.hour;
```
<details>
    <summary>output</summary>
    <p>
      
| hour | orders |
|------|--------|
|   1  |   0    |
|   2  |   0    |
|   3  |   0    |
|   4  |   0    |
|   5  |   0    |
|   6  |   0    |
|   7  |   0    |
|   8  |   0    |
|   9  |   0    |
|  10  |   0    |
|  11  |   1    |
|  12  |   0    |
|  13  |   3    |
|  14  |   0    |
|  15  |   0    |
|  16  |   0    |
|  17  |   0    |
|  18  |   3    |
|  19  |   1    |
|  20  |   0    |
|  21  |   3    |
|  22  |   0    |
|  23  |   3    |
|  24  |   0    |

</p>
</details>

### What was the volume of orders for each day of the week?

```sql
WITH dow AS
(SELECT to_char(generate_series('2024-04-01', '2024-04-07', INTERVAL '1 Day' ), 'Day') AS d),
orders AS 
(SELECT to_char(order_time,'Day') AS dow, COUNT(to_char(order_time,'Day')) AS o
FROM customer_orders
GROUP BY to_char(order_time,'Day'))
SELECT dd.d AS day_of_week, COALESCE(oo.o,0) AS orders
FROM dow dd
LEFT JOIN orders oo
ON dd.d = oo.dow;
```
*output*

| day_of_week | orders |
|-------------|--------|
|  Monday     |    0   |
|  Tuesday    |    0   |
|  Wednesday  |    5   |
|  Thursday   |    3   |
|  Friday     |    1   |
|  Saturday   |    5   |
|  Sunday     |    0   |


## B. Runner and Customer Experience

### How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)

```sql
WITH cte AS
(SELECT generate_series('2021-01-01', '2021-12-31', INTERVAL '7 Day') as week_start, 
generate_series(1,53,1) AS week_num),
leadd AS
(SELECT *, LEAD(week_start) OVER() - INTERVAL '1 Day' AS week_end
FROM cte)
SELECT EXTRACT(YEAR FROM l.week_start) AS year,l.week_num , COUNT(r.runner_id) AS runner_signups
FROM runners r
RIGHT JOIN leadd l
ON r.registration_date BETWEEN l.week_start AND l.week_end
GROUP BY EXTRACT(YEAR FROM l.week_start), l.week_num
ORDER BY l.week_num;
```

<details>
    <summary>output</summary>
    <p>

| year | week_num | runner_signups |
|------|----------|----------------|
| 2021 |    1     |       2        |
| 2021 |    2     |       1        |
| 2021 |    3     |       1        |
| 2021 |    4     |       0        |
| 2021 |    5     |       0        |
| 2021 |    6     |       0        |
| 2021 |    7     |       0        |
| 2021 |    8     |       0        |
| 2021 |    9     |       0        |
| 2021 |   10     |       0        |
| 2021 |   11     |       0        |
| 2021 |   12     |       0        |
| 2021 |   13     |       0        |
| 2021 |   14     |       0        |
| 2021 |   15     |       0        |
| 2021 |   16     |       0        |
| 2021 |   17     |       0        |
| 2021 |   18     |       0        |
| 2021 |   19     |       0        |
| 2021 |   20     |       0        |
| 2021 |   21     |       0        |
| 2021 |   22     |       0        |
| 2021 |   23     |       0        |
| 2021 |   24     |       0        |
| 2021 |   25     |       0        |
| 2021 |   26     |       0        |
| 2021 |   27     |       0        |
| 2021 |   28     |       0        |
| 2021 |   29     |       0        |
| 2021 |   30     |       0        |
| 2021 |   31     |       0        |
| 2021 |   32     |       0        |
| 2021 |   33     |       0        |
| 2021 |   34     |       0        |
| 2021 |   35     |       0        |
| 2021 |   36     |       0        |
| 2021 |   37     |       0        |
| 2021 |   38     |       0        |
| 2021 |   39     |       0        |
| 2021 |   40     |       0        |
| 2021 |   41     |       0        |
| 2021 |   42     |       0        |
| 2021 |   43     |       0        |
| 2021 |   44     |       0        |
| 2021 |   45     |       0        |
| 2021 |   46     |       0        |
| 2021 |   47     |       0        |
| 2021 |   48     |       0        |
| 2021 |   49     |       0        |
| 2021 |   50     |       0        |
| 2021 |   51     |       0        |
| 2021 |   52     |       0        |
| 2021 |   53     |       0        |

</p>
</details>

### What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

```sql
WITH cte AS
(SELECT order_id, MAX(order_time) AS order_time
 FROM customer_orders
 GROUP BY order_id )
SELECT r.runner_id, ROUND(AVG(EXTRACT(MINUTE from r.pickup_time - x.order_time)),2) AS avg_minutes_for_pickup 
FROM runner_orders r
JOIN cte x 
ON x.order_id = r.order_id
WHERE pickup_time IS NOT NULL
GROUP BY r.runner_id
ORDER BY runner_id;          
```
*output*

| runner_id | avg_minutes_for_pickup |
|-----------|------------------------|
|     1     |          14.00         |
|     2     |          19.67         |
|     3     |          10.00         |


### Is there any relationship between the number of pizzas and how long the order takes to prepare? yes 

```sql
WITH cte AS
(SELECT order_id, COUNT(pizza_id) AS num_of_pizzas, MAX(order_time) AS order_time
FROM customer_orders
GROUP BY order_id
ORDER BY num_of_pizzas)
SELECT x.num_of_pizzas, 
ROUND(AVG(EXTRACT(MINUTE from r.pickup_time - x.order_time)),2) AS avg_minutes_for_prep 
FROM runner_orders r 
JOIN cte x
ON x.order_id = r.order_id  
WHERE pickup_time IS NOT NULL 
GROUP BY x.num_of_pizzas 
ORDER BY x.num_of_pizzas;
```
*output*

| num_of_pizzas | avg_minutes_for_prep |
|---------------|----------------------|
|       1       |         12.00        |
|       2       |         18.00        |
|       3       |         29.00        |


### What was the average distance traveled for each customer?

```sql
WITH cte AS
(SELECT order_id, customer_id 
FROM customer_orders
GROUP BY order_id, customer_id)
SELECT c.customer_id, ROUND(AVG(r.distance_km),2) AS avg_distance_traveled_km
JOIN runner_orders r
ON c.order_id = r.order_id
WHERE r.distance_km IS NOT NULL
GROUP BY c.customer_id
ORDER BY customer_id;
```
*output*

| customer_id | avg_distance_traveled_km |
|-------------|-----------------------|
|     101     |         20.00         |
|     102     |         18.40         |
|     103     |         23.40         |
|     104     |         10.00         |
|     105     |         25.00         |

### What was the difference between the longest and shortest delivery times for all orders?
```sql
SELECT MAX(duration_min) - MIN(duration_min) AS delivery_range
FROM runner_orders
WHERE duration_min IS NOT NULL;
```
*output*

| delivery_range |
|----------------|
|      30.00     |


### What was the average speed for each runner for each delivery? 
* do you notice any trend for these values? *runner 2 is probably a liability*

```sql
SELECT runner_id, order_id, ROUND(distance_km/(duration_min/60),0) AS avg_speed_km_per_hr
FROM runner_orders
WHERE cancellation IS NULL
ORDER BY runner_id;
```

*output*

| runner_id | order_id | avg_speed_km_per_hr |
|-----------|----------|---------------------|
|     1     |    1     |         38          |
|     1     |    2     |         44          |
|     1     |    3     |         40          |
|     1     |   10     |         60          |
|     2     |    7     |         60          |
|     2     |    8     |         94          |
|     2     |    4     |         35          |
|     3     |    5     |         40          |


### What is the successful delivery percentage for each runner?

```sql
WITH cte AS
(SELECT rr.runner_id, ROUND(SUM(CASE WHEN rr.cancellation IS NULL THEN 1 ELSE 0 END)*1.00/COUNT(*),2)*100 AS delivery_percentage
FROM runner_orders rr
GROUP BY rr.runner_id
ORDER BY rr.runner_id)
SELECT r.runner_id, c.d_percent
FROM cte c
RIGHT JOIN runners r
ON c.runner_id = r.runner_id;
```

*output*

| runner_id | delivery_percentage |
|-----------|---------------------|
|     1     |       100.00        |
|     2     |        75.00        |
|     3     |        50.00        |
|     4     |        null         |


## C. Ingredient Optimization 

### What are the standard ingredients for each pizza?

```sql
WITH cte AS
(SELECT pizza_id, UNNEST(string_to_array(toppings,','))::INT AS ingredient
FROM pizza_recipes) 
SELECT pizza_id, ARRAY_TO_STRING(ARRAY_AGG(topping_name),', ') AS standard_ingredients
FROM cte c
JOIN pizza_toppings p
ON c.ingredient = p.topping_id
GROUP BY pizza_id
ORDER BY pizza_id;
```

*output*

| pizza_id |      standard_ingredients       |
|----------|---------------------------------|
|    1     | Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |
|    2     | Cheese, Mushrooms, Onions, Peppers, Tomatoes, Tomato Sauce |

### What was the most commonly added extra?

```sql
WITH cte AS 
(SELECT *, UNNEST(STRING_TO_ARRAY(extras,','))::INT AS extrass
FROM customer_orders)
SELECT p.topping_name, COUNT(p.topping_name) AS extra_count
FROM cte c
JOIN pizza_toppings p
ON c.extrass = p.topping_id
GROUP BY p.topping_name
ORDER BY extra_count DESC
LIMIT 1;
```
*output*

| topping_name | extra_count |
|--------------|-------------|
|    Bacon     |      4      |


### What was the most common exclusion?

```sql
WITH cte AS 
(SELECT *, UNNEST(STRING_TO_ARRAY(exclusions,','))::INT AS exclusionss
FROM customer_orders)
SELECT p.topping_name, COUNT(p.topping_name) AS exclusion_count
FROM cte c
JOIN pizza_toppings p
ON c.exclusionss = p.topping_id
GROUP BY p.topping_name
ORDER BY exclusion_count DESC
LIMIT 1;
```
*output*

| topping_name | exclusion_count |
|--------------|-----------------|
|    Cheese    |        4        |


### Generate an order item for each record in the customers_orders table in the format of one of the following:
* Meat Lovers
* Meat Lovers - Exclude Beef
* Meat Lovers - Extra Bacon
* Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Pepper

```sql
WITH cte AS
(SELECT co.order_id, co.customer_id, co.pizza_id, co.exclusions, co.extras, pn.pizza_name, ROW_NUMBER() OVER() AS rr
FROM customer_orders co
JOIN pizza_names pn 
ON co.pizza_id = pn.pizza_id)
SELECT
CASE 
  WHEN c.exclusions IS NOT NULL AND c.extras IS NULL THEN CONCAT(pizza_name, ' - Exclude ', exclude_toppings.topping_list)
  WHEN c.exclusions IS NULL AND c.extras IS NOT NULL THEN CONCAT(pizza_name, ' - Extra ', extra_toppings.topping_list)
  WHEN c.exclusions IS NOT NULL AND c.extras IS NOT NULL THEN CONCAT(pizza_name, ' - Exclude ', exclude_toppings.topping_list, ' - Extra ', extra_toppings.topping_list)
  ELSE pizza_name 
END AS order_item , c.order_id, c.pizza_id, c.exclusions, c.extras
FROM cte c 
LEFT JOIN LATERAL 
  (SELECT ARRAY_TO_STRING(ARRAY_AGG(pt.topping_name), ', ') AS topping_list
  FROM pizza_toppings pt
  WHERE pt.topping_id IN (
    SELECT unnest(string_to_array (cc.exclusions, ','))::INT AS topping_id
    FROM cte cc
    WHERE cc.rr = c.rr)) AS exclude_toppings ON TRUE
LEFT JOIN LATERAL
  (SELECT ARRAY_TO_STRING(ARRAY_AGG(pt.topping_name), ', ') AS topping_list
  FROM pizza_toppings pt
  WHERE pt.topping_id IN (
    SELECT unnest(string_to_array(cc.extras, ','))::INT AS topping_id
    FROM cte cc
    WHERE cc.rr = c.rr)) AS extra_toppings ON TRUE
ORDER BY c.order_id;
```
*output*

|         order_item         | order_id | pizza_id | exclusions |  extras   |
|---------------------------|----------|----------|------------|-----------|
|        Meatlovers         |    1     |    1     |    null    |    null   |
|        Meatlovers         |    2     |    1     |    null    |    null   |
|        Meatlovers         |    3     |    1     |    null    |    null   |
|        Vegetarian         |    3     |    2     |    null    |    null   |
| Vegetarian - Exclude Cheese |    4     |    2     |     4      |    null   |
| Meatlovers - Exclude Cheese |    4     |    1     |     4      |    null   |
| Meatlovers - Exclude Cheese |    4     |    1     |     4      |    null   |
|   Meatlovers - Extra Bacon |    5     |    1     |    null    |     1     |
|        Vegetarian         |    6     |    2     |    null    |    null   |
| Vegetarian - Extra Bacon  |    7     |    2     |    null    |     1     |
|        Meatlovers         |    8     |    1     |    null    |    null   |
| Meatlovers - Exclude Cheese - Extra Bacon, Chicken |    9     |    1     |     4      |    1,5    |
| Meatlovers - Exclude BBQ Sauce, Mushrooms - Extra Bacon, Cheese |   10    |    1     |    2,6     |   1,4     |
|        Meatlovers         |   10     |    1     |    null    |    null   |


### Generate an alphabetically ordered comma separated ingredient list for each pizza order from the customer_orders table and add a 2x in front of any relevant ingredients
* For example: "Meat Lovers: 2xBacon, Beef, ... , Salami"
 
```sql
WITH cte AS 
(SELECT order_id, customer_id, c.pizza_id, exclusions, extras, p.pizza_name, pp.toppings, row_number() OVER() AS rr
FROM customer_orders c 
JOIN pizza_names p 
ON c.pizza_id = p.pizza_id
JOIN pizza_recipes pp
ON c.pizza_id = pp.pizza_id)
SELECT c.*,  CONCAT(pizza_name, ': ', xxx.namee)		   
FROM cte c
JOIN LATERAL
(SELECT ARRAY_TO_STRING(ARRAY_AGG(namee),', ') AS namee
FROM
	(SELECT CASE WHEN num > 1 THEN CONCAT(num,'x', namee) ELSE namee END AS namee
	FROM 
		(SELECT ii.namee, COUNT(ii.namee)AS num
		 FROM
			(SELECT (topping_name) AS namee
			FROM pizza_toppings p 
			WHERE topping_id IN
				(SELECT topping_id
				 FROM 
					(SELECT UNNEST(string_to_array(toppings,','))::INT AS topping_id
					 FROM cte cc
					 WHERE cc.rr = c.rr)
				 WHERE topping_id NOT IN 
						(SELECT unnest(string_to_array(exclusions,','))::INT AS topping_id
						 FROM cte cc
						 WHERE cc.rr = c.rr))
				 UNION ALL
				(SELECT topping_name AS namee 
				 FROM pizza_toppings
				 WHERE topping_id  IN 
					(SELECT unnest(string_to_array(extras,','))::INT AS topping_id
					 FROM cte cc
					 WHERE cc.rr = c.rr)))ii
		  GROUP BY ii.namee
ORDER BY ii.namee)
	)) AS xxx ON TRUE;
```
*output*

| ingredient_list                                                        | order_id | pizza_id | exclusions |  extras   |
|------------------------------------------------------------------------|----------|----------|------------|-----------|
| Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |    1     |    1     |    null    |    null   |
| Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |    2     |    1     |    null    |    null   |
| Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |    3     |    1     |    null    |    null   |
| Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes |    3     |    2     |    null    |    null   |
|  Vegetarian: Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes         |    4     |    2     |     4      |    null   |
|  Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami  |    4     |    1     |     4      |    null   |
|  Meatlovers: Bacon, BBQ Sauce, Beef, Chicken, Mushrooms, Pepperoni, Salami  |    4     |    1     |     4      |    null   |
|  Meatlovers: 2xBacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |    5     |    1     |    null    |     1     |
|  Vegetarian: Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes |    6     |    2     |    null    |    null   |
|  Vegetarian: Bacon, Cheese, Mushrooms, Onions, Peppers, Tomato Sauce, Tomatoes  |    7     |    2     |    null    |     1     |
|  Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |    8     |    1     |    null    |    null   |
|  Meatlovers: 2xBacon, BBQ Sauce, Beef, 2xChicken, Mushrooms, Pepperoni, Salami   |    9     |    1     |     4      |    1,5    |
|  Meatlovers: 2xBacon, Beef, 2xCheese, Chicken, Pepperoni, Salami  |   10     |    1     |    2,6     |   1,4     |
|  Meatlovers: Bacon, BBQ Sauce, Beef, Cheese, Chicken, Mushrooms, Pepperoni, Salami |   10     |    1     |    null    |    null   |

### What is the total quantity of each ingredient used in all delivered pizzas sorted by most frequent first?

```sql
WITH cte AS
(SELECT c.order_id, c.customer_id, c.pizza_id, c.exclusions, c.extras, cancellation, p.toppings, ROW_NUMBER() OVER() rr
FROM customer_orders c
JOIN pizza_recipes p 
ON c.pizza_id = p.pizza_id
JOIN runner_orders r 
ON r.order_id = c.order_id) 
SELECT pp.topping_name, COUNT(ii.ingredients_used) AS times_used
FROM cte c
JOIN LATERAL 
	((SELECT topping_id AS ingredients_used
	  FROM 
	       (SELECT UNNEST(string_to_array(toppings,','))::INT AS topping_id
			FROM cte cc
			WHERE cc.rr = c.rr) 
	 WHERE topping_id NOT IN 
			(SELECT unnest(string_to_array(exclusions,','))::INT AS topping_id
			FROM cte cc
			WHERE cc.rr = c.rr))
     UNION ALL
	(SELECT unnest(string_to_array(extras,','))::INT AS topping_id
	 FROM cte cc
	 WHERE cc.rr = c.rr)) ii 
ON TRUE
RIGHT JOIN pizza_toppings pp
ON pp.topping_id = ii.ingredients_used
WHERE c.cancellation IS NULL
GROUP BY pp.topping_name
ORDER BY times_used DESC;
```
*output*

| topping_name | times_used |
|--------------|------------|
|    Bacon     |     12     |
|  Mushrooms   |     11     |
|    Cheese    |     10     |
|  Pepperoni   |     9      |
|    Salami    |     9      |
|   Chicken    |     9      |
|     Beef     |     9      |
|  BBQ Sauce   |     8      |
|   Tomatoes   |     3      |
|    Onions    |     3      |
|   Peppers    |     3      |
| Tomato Sauce |     3      |


## D. Pricing and Ratings

### If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes 
* how much money has Pizza Runner made so far if there are no delivery fees?

```sql
SELECT SUM(CASE WHEN p.pizza_name = 'Meatlovers' THEN 12::MONEY ELSE 10::MONEY END) AS total_revenue
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
JOIN pizza_names p
ON p.pizza_id = c.pizza_id
WHERE cancellation IS NULL;
```
*output*

| total_revenue |
|---------------|
|   $138.00     |


### What if there was an additional $1 charge for any pizza extras?
* Add cheese is $1 extra

```sql
WITH cte AS
(SELECT
CASE WHEN p.pizza_name = 'Meatlovers' THEN 12::MONEY
WHEN p.pizza_name = 'Vegetarian' THEN 10::MONEY END pizza_cost,
CASE WHEN c.extras IS NOT NULL THEN 1::MONEY END AS extra_charge,
CASE WHEN extras IS NOT NULL AND extras LIKE '%4%' THEN 1::MONEY END AS added_cheese
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
JOIN pizza_names p
ON p.pizza_id = c.pizza_id
WHERE r.cancellation IS NULL) 
SELECT SUM(pizza_cost) + SUM(extra_charge) + SUM(added_cheese) AS total_revenue
FROM cte;
```
*output*

| total_revenue |
|---------------|
|   $142.00     |

### The Pizza Runner team now wants to add an additional ratings system that allows customers to rate their runner, how would you design an additional table for this new dataset?
* generate a schema for this new table and insert your own data for ratings for each successful customer order between 1 to 5.
```sql
CREATE TABLE runner_rating (
	order_id INT,
	runner_id INT,
	rating INT);

INSERT INTO runner_rating
(SELECT order_id, runner_id, floor(random() * 5) + 1 
FROM runner_orders ro
WHERE cancellation IS NULL
AND NOT EXISTS
	(SELECT 1 
	FROM runner_rating r  
	WHERE r.order_id = ro.order_id));

SELECT * 
FROM runner_rating
ORDER BY order_id;
```
*output*

| order_id | runner_id | rating |
|----------|-----------|--------|
|    1     |     1     |   2    |
|    2     |     1     |   2    |
|    3     |     1     |   1    |
|    4     |     2     |   3    |
|    5     |     3     |   2    |
|    7     |     2     |   1    |
|    8     |     2     |   3    |
|   10     |     1     |   3    |


### Using your newly generated table - can you join all of the information together to form a table which has the following information for successful deliveries?
* customer_id
* order_id
* runner_id
* rating
* order_time
* pickup_time
* Time between order and pickup
* Delivery duration
* Average speed
* Total number of pizzas

```sql
SELECT c.customer_id, r.order_id, r.runner_id, rr.rating, c.order_time, r.pickup_time,
EXTRACT (MINUTE FROM (r.pickup_time - c.order_time)) AS time_between_op_min,
r.duration_min, ROUND((distance_km /(duration_min/60)),2) AS avg_speed_km_per_hour, c.num_of_pizzas
FROM runner_orders r
JOIN
(SELECT order_id, customer_id, COUNT(pizza_id) AS num_of_pizzas, MAX(order_time) AS order_time
FROM customer_orders 
GROUP BY order_id, customer_id) c
ON c.order_id = r.order_id
JOIN runner_rating rr
ON rr.order_id = r.order_id
WHERE cancellation IS NULL;
```
*output*

| customer_id | order_id | runner_id | rating |      order_time      |     pickup_time      | time_between_op_min | duration_min | avg_speed_km_per_hour | num_of_pizzas |
|-------------|----------|-----------|--------|----------------------|----------------------|----------------------|--------------|-----------------------|---------------|
|    101      |    1     |     1     |   2    | 2020-01-01 18:05:02  | 2020-01-01 18:15:34  |          10          |     32.00    |         37.50         |       1       |
|    101      |    2     |     1     |   2    | 2020-01-01 19:00:52  | 2020-01-01 19:10:54  |          10          |     27.00    |         44.44         |       1       |
|    102      |    3     |     1     |   2    | 2020-01-02 23:51:23  | 2020-01-03 00:12:37  |          21          |     20.00    |         40.20         |       2       |
|    103      |    4     |     2     |   5    | 2020-01-04 13:23:46  | 2020-01-04 13:53:03  |          29          |     40.00    |         35.10         |       3       |
|    104      |    5     |     3     |   4    | 2020-01-08 21:00:29  | 2020-01-08 21:10:57  |          10          |     15.00    |         40.00         |       1       |
|    105      |    7     |     2     |   1    | 2020-01-08 21:20:29  | 2020-01-08 21:30:45  |          10          |     25.00    |         60.00         |       1       |
|    102      |    8     |     2     |   3    | 2020-01-09 23:54:33  | 2020-01-10 00:15:02  |          20          |     15.00    |         93.60         |       1       |
|    104      |    10    |     1     |   2    | 2020-01-11 18:34:49  | 2020-01-11 18:50:20  |          15          |     10.00    |         60.00         |       2       |

### If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometer traveled 
* how much money does Pizza Runner have left over after these deliveries?
```sql
WITH cte AS
(SELECT *, distance_km*0.30::MONEY AS runner_wage, CASE WHEN p.pizza_name = 'Meatlovers' THEN 12::MONEY
														 WHEN p.pizza_name = 'Vegetarian' THEN 10::MONEY END pizza_revenue
FROM customer_orders c
JOIN runner_orders r
ON c.order_id = r.order_id
JOIN pizza_names p
ON p.pizza_id = c.pizza_id
WHERE r.cancellation IS NULL) 
SELECT SUM(pizza_revenue) - SUM(runner_wage) AS total_yield
FROM cte;
```
*output*

| total_yield |
|-------------|
|   $73.38    |


## Bonus Question
### If Danny wants to expand his range of pizzas how would this impact the existing data design? 
* Write an INSERT statement to demonstrate what would happen if a new Supreme pizza with all the toppings was added to the Pizza Runner menu?

*output*
```sql
INSERT INTO pizza_names
	VALUES(3, 'Supreme');

INSERT INTO pizza_recipes
 VALUES (3, (SELECT ARRAY_TO_STRING(ARRAY_AGG(topping_id),', ')
		FROM pizza_toppings));

SELECT * 
FROM pizza_names;

SELECT * 
FROM pizza_recipes;

```
*output*

| pizza_id |  pizza_name  |
|----------|--------------|
|    1     |  Meatlovers  |
|    2     |  Vegetarian  |
|    3     |   Supreme    |


| pizza_id |            toppings             |
|----------|--------------------------------|
|    1     |    1, 2, 3, 4, 5, 6, 8, 10     |
|    2     |         4, 6, 7, 9, 11, 12     |
|    3     | 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12 |









