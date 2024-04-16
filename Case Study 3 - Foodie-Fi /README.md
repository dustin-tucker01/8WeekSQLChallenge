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
```
*output*
## Data Analysis Questions
### How many customers has Foodie-Fi ever had?

```sql
```
*output*
### What is the monthly distribution of trial plan start_date values for our dataset? 
* Use the start of the month as the group by value.

```sql
```
*output*

### What plan start_date values occur after the year 2020 for our dataset? 
* Show the breakdown by count of events for each plan_name.

```sql
```
*output*

### What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

```sql
```
*output*

### How many customers have churned straight after their initial free trial?  
* What percentage is this rounded to the nearest whole number?

```sql
```
*output*

### What is the number and percentage of customer plans after their initial free trial?

```sql
```
*output*

### What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

```sql
```
*output*

### How many customers have upgraded to an annual plan in 2020?

```sql
```
*output*

### How many days on average does it take for a customer to buy an annual plan from the day they join Foodie-Fi?

```sql
```
*output*

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
