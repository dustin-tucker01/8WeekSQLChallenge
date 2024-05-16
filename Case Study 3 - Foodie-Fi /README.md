# Case Study 3 - Foodie-Fi 
# Table of contents 
* [Questions](#Questions)
  * A. [Customer Journey](#Customer-Journey)
  * B. [Data Analysis Questions](#Data-Analysis-Questions) 
  * C. [Challenge Payment Questions](#Challenge-Payment-Questions) 
  * D. [Outside The Box Questions](#Outside-The-Box-Questions)

## Problem Statement
Danny noticed a gap in the market for a food-focused streaming service similar to Netflix.
Teaming up with partners, he launched Foodie-Fi in 2020, offering monthly and annual subscriptions for exclusive culinary content. 
With a data-driven mindset, Danny ensured that all decisions at Foodie-Fi were guided by insights from subscription data, illustrating its importance in addressing key business questions.

## Available Data
Within the foodie_fi database schema, Danny has provided the following tables for analysis:
* Plans 
* Subscriptions

## ER Diagram
<img width="707" alt="Screenshot 2024-04-15 at 5 55 12 PM" src="https://github.com/dustin-tucker01/8WeekSQLChallenge/assets/148160789/56cc10d1-fb2f-4c43-8eec-5f728843125c">

# Questions
## Customer Journey
### Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.
* Try to keep it as short as possible - you may also want to run some sort of join to make your explanations a bit easier!
```sql
WITH cte AS
(SELECT 
	customer_id, 
	plan_name
FROM 
	sample_subscriptions s
JOIN 
	plans p ON s.plan_id = p.plan_id
ORDER BY 
	customer_id, 
	s.start_date)
SELECT 
	customer_id, 
	ARRAY_TO_STRING(ARRAY_AGG(plan_name),', ') AS customer_journey
FROM 
	cte
GROUP BY 
	customer_id;
-- Each customer's journey can be seen in this result table, with all customers starting with a trial 
-- and then each customer deciding to upgrade their subscription or churn
```
*output*

| customer_id | customer_journey                 |
|-------------|----------------------------------|
| 1           | trial, basic monthly             |
| 2           | trial, pro annual                |
| 11          | trial, churn                     |
| 13          | trial, basic monthly, pro monthly|
| 15          | trial, pro monthly, churn        |
| 16          | trial, basic monthly, pro annual |
| 18          | trial, pro monthly               |
| 19          | trial, pro monthly, pro annual   |

## Data Analysis Questions
### How many customers has Foodie-Fi ever had?
```sql
SELECT 
	COUNT(DISTINCT customer_id) AS total_customers
FROM 
	subscriptions;
```
*output*

| total_customers |
|-----------------|
| 1000            |

### What is the monthly distribution of trial plan start_date values for our dataset? 
* This query effectively includes the year, month, and the number of trial plan start dates within that month and year
* I did this query in this manner to allow for the use of this same query as the business gains trial plans into 2021. 
```sql
WITH months_years AS ( -- creating the base table for all 12 months for the years that are present in the dataset
    SELECT 
        y.year AS year, 
        TO_CHAR(x, 'Month') AS month 
    FROM 
        generate_series('2024-01-01'::DATE, '2024-12-01'::DATE, '1 Month') x
    CROSS JOIN 
        (SELECT DISTINCT EXTRACT(YEAR FROM start_date) AS year FROM subscriptions) y
    ORDER BY 
        y.year, x
), 
monthly_distributions AS ( -- getting the number of trial plan start dates grouped by year and month
    SELECT 
        EXTRACT(YEAR FROM start_date) AS year, 
        TO_CHAR(start_date, 'Month') AS month, 
        COUNT(start_date) AS trials
    FROM 
        subscriptions
    WHERE 
        plan_id = 0
    GROUP BY 
        year, 
        TO_CHAR(start_date, 'Month')
)
SELECT -- final table 
    o.year, 
    o.month, 
    m.trials
FROM 
    monthly_distributions m
RIGHT JOIN 
    months_years o ON m.year = o.year AND m.month = o.month;
-- joining the base table to the distributions present in the dataset to include months with no trial plan start dates

```
*output*
| year | month      | trials |
|------|------------|--------|
| 2020 | January    | 88     |
| 2020 | February   | 68     |
| 2020 | March      | 94     |
| 2020 | April      | 81     |
| 2020 | May        | 88     |
| 2020 | June       | 79     |
| 2020 | July       | 89     |
| 2020 | August     | 88     |
| 2020 | September  | 87     |
| 2020 | October    | 79     |
| 2020 | November   | 75     |
| 2020 | December   | 84     |
| 2021 | January    | null   |
| 2021 | February   | null   |
| 2021 | March      | null   |
| 2021 | April      | null   |
| 2021 | May        | null   |
| 2021 | June       | null   |
| 2021 | July       | null   |
| 2021 | August     | null   |
| 2021 | September  | null   |
| 2021 | October    | null   |
| 2021 | November   | null   |
| 2021 | December   | null   |

### What plan start_date values occur after the year 2020 for our dataset? 
* Show the breakdown by count of events for each plan_name.
```sql
WITH cte AS ( -- plan names and total subscriptions after 2020 
    SELECT 
        p.plan_name, 
        COUNT(*) AS total_subscriptions
    FROM 
        subscriptions s
    RIGHT JOIN 
        plans p ON p.plan_id = s.plan_id
    WHERE 
        EXTRACT(YEAR FROM s.start_date) > 2020 OR s.start_date IS NULL
    GROUP BY 
        p.plan_name
),
cte2 AS ( -- getting plans with no subscriptions 
    SELECT 
        plan_name, 
        0 AS total_subscriptions
    FROM 
        plans 
    WHERE 
        NOT EXISTS (SELECT 1 FROM cte WHERE cte.plan_name = plans.plan_name)
)
SELECT 
    * 
FROM 
    cte 
UNION 
SELECT 
    * 
FROM 
    cte2
ORDER BY 
    total_subscriptions DESC; 
```
*output*
| plan_name      | total_subscriptions |
|----------------|---------------------|
| churn          | 71                  |
| pro annual     | 63                  |
| pro monthly    | 60                  |
| basic monthly  | 8                   |
| trial          | 0                   |

### What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

```sql
SELECT 
	COUNT(DISTINCT(customer_id)) AS total_customers,
	ROUND((SUM(CASE WHEN plan_id = 4 THEN 1 ELSE 0 END)::DECIMAL/COUNT(DISTINCT customer_id))*100,1) AS churn_rate
FROM 
	subscriptions;
```
*output*

| total_customers | churn_rate |
|-----------------|------------|
| 1000            | 30.7       |

### How many customers have churned straight after their initial free trial?  
* What percentage is this rounded to the nearest whole number?
```sql
WITH cte AS (
    -- Numbering the journey per customer
    SELECT 
        *, 
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) trial_num
    FROM 
        subscriptions
),
cte2 AS ( -- getting total number of customers 
    SELECT 
        COUNT(DISTINCT customer_id) AS t
    FROM 
        subscriptions
)
SELECT 
    COUNT(*) AS churn_count, -- customers who churned after trial 
    ROUND((COUNT(*) / MAX(cte2.t)::numeric) * 100) AS churn_rate -- (churn rate after trial) 
FROM 
    cte 
CROSS JOIN 
    cte2
WHERE 
    trial_num = 2 
    AND plan_id = 4;
```
*output*

| churn_count | churn_rate |
|-------------|------------|
| 92          | 9          |

### What is the number and percentage of customer plans after their initial free trial?
```sql

WITH cte AS (
    -- Numbering the journey per customer
    SELECT 
        *, 
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date) trial_num
    FROM 
        subscriptions
),
cte2 AS ( -- number of plans after trial 
SELECT 
	COUNT(*) AS t
FROM 
	cte
WHERE 
	trial_num = 2
)
SELECT 
	p.plan_name, 
	COUNT(*), 
	ROUND(((COUNT(*)/MAX(cte2.t)::numeric)*100),2) AS plan_percentage
FROM 
	cte c
JOIN 
	plans p ON c.plan_id = p.plan_id
CROSS JOIN 
	cte2
WHERE 
	trial_num = 2
GROUP BY
	p.plan_name;
```
*output*

| plan_name      | count | plan_percentage |
|----------------|-------|-----------------|
| pro annual     | 37    | 3.70            |
| pro monthly    | 325   | 32.50           |
| churn          | 92    | 9.20            |
| basic monthly  | 546   | 54.60           |


### What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

```sql
WITH cte AS (
    SELECT 
        *, 
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY start_date DESC) trial_num
    FROM 
        subscriptions
    WHERE 
        start_date <= '2020-12-31'
),
cte2 AS (
    SELECT 
        *, 
        COUNT(*) OVER() AS total_customers
    FROM 
        cte
    WHERE 
        trial_num = 1
)
SELECT 
    p.plan_name, 
    COUNT(*) AS active_plans, 
    ROUND(100 * (COUNT(*) / MAX(c.total_customers)::numeric), 2) AS plan_percentage
FROM 
    cte2 c
JOIN 
    plans p ON c.plan_id = p.plan_id
GROUP BY 
    p.plan_name
ORDER BY 
    active_plans DESC;
```
*output*

| plan_name      | active_plans | percentage |
|----------------|--------------|------------|
| pro monthly    | 326          | 32.60      |
| churn          | 236          | 23.60      |
| basic monthly  | 224          | 22.40      |
| pro annual     | 195          | 19.50      |
| trial          | 19           | 1.90       |



### How many customers have upgraded to an annual plan in 2020?
```sql
SELECT 
    COUNT(*) AS total_annual
FROM 
    subscriptions
WHERE 
    plan_id = 3 
    AND EXTRACT(YEAR FROM start_date) = 2020;
```
*output*

| total_annual |
|--------------|
| 195          |


### How many days on average does it take for a customer to buy an annual plan from the day they join Foodie-Fi?

```sql
WITH cte AS (
    SELECT *
    FROM subscriptions
    WHERE plan_id = 0
),
cte2 AS (
    SELECT *
    FROM subscriptions 
    WHERE plan_id = 3
)
SELECT 
    CEIL(AVG(cc.start_date - c.start_date)) AS avg_days
FROM 
    cte c
JOIN 
    cte2 cc ON c.customer_id = cc.customer_id;
```
*output*

| avg_days |
|----------|
| 105      |

### Can you further breakdown this average value into 30 day periods? (i.e. 0-30 days, 31-60 days etc)

```sql
```
*output*

### How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

```sql
```
*output*

## Challenge Payment Questions
### The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:
* Monthly payments always occur on the same day of month as the original start_date of any monthly paid plan.
* Upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately.
* Upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period.
* Once a customer churns, they will no longer make payments.
* Example outputs for this table might look like the following:

<details>
    <summary>Example Output</summary>
    <p>
      
| customer_id | plan_id | plan_name     | payment_date | amount | payment_order |
|-------------|---------|---------------|--------------|--------|---------------|
| 1           | 1       | basic monthly | 2020-08-08   | 9.90   | 1             |
| 1           | 1       | basic monthly | 2020-09-08   | 9.90   | 2             |
| 1           | 1       | basic monthly | 2020-10-08   | 9.90   | 3             |
| 1           | 1       | basic monthly | 2020-11-08   | 9.90   | 4             |
| 1           | 1       | basic monthly | 2020-12-08   | 9.90   | 5             |
| 2           | 3       | pro annual    | 2020-09-27   | 199.00 | 1             |
| 13          | 1       | basic monthly | 2020-12-22   | 9.90   | 1             |
| 15          | 2       | pro monthly   | 2020-03-24   | 19.90  | 1             |
| 15          | 2       | pro monthly   | 2020-04-24   | 19.90  | 2             |
| 16          | 1       | basic monthly | 2020-06-07   | 9.90   | 1             |
| 16          | 1       | basic monthly | 2020-07-07   | 9.90   | 2             |
| 16          | 1       | basic monthly | 2020-08-07   | 9.90   | 3             |
| 16          | 1       | basic monthly | 2020-09-07   | 9.90   | 4             |
| 16          | 1       | basic monthly | 2020-10-07   | 9.90   | 5             |
| 16          | 3       | pro annual    | 2020-10-21   | 189.10 | 6             |
| 18          | 2       | pro monthly   | 2020-07-13   | 19.90  | 1             |
| 18          | 2       | pro monthly   | 2020-08-13   | 19.90  | 2             |
| 18          | 2       | pro monthly   | 2020-09-13   | 19.90  | 3             |
| 18          | 2       | pro monthly   | 2020-10-13   | 19.90  | 4             |
| 18          | 2       | pro monthly   | 2020-11-13   | 19.90  | 5             |
| 18          | 2       | pro monthly   | 2020-12-13   | 19.90  | 6             |
| 19          | 2       | pro monthly   | 2020-06-29   | 19.90  | 1             |
| 19          | 2       | pro monthly   | 2020-07-29   | 19.90  | 2             |
| 19          | 3       | pro annual    | 2020-08-29   | 199.00 | 3             |

</p>
</details>

```sql
```
*output*

## Outside The Box Questions
The following are open ended questions which might be asked during a technical interview for this case study - there are no right or wrong answers, but answers that make sense from both a technical and a business perspective make an amazing impression!
### How would you calculate the rate of growth for Foodie-Fi?
### What key metrics would you recommend Foodie-Fi management to track over time to assess performance of their overall business?
### What are some key customer journeys or experiences that you would analyze further to improve customer retention?
### If the Foodie-Fi team were to create an exit survey shown to customers who wish to cancel their subscription, what questions would you include in the survey?
### What business levers could the Foodie-Fi team use to reduce the customer churn rate?
* How would you validate the effectiveness of your ideas?
