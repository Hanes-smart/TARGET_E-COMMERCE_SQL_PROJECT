# DATA ANALYSIS ON TAGET E-COMMERCE SQL PROJECT

![logo](https://github.com/Hanes-smart/TARGET_E-COMMERCE_SQL_PROJECT/blob/main/downloadtaget.png)

## 1.Import the dataset and do usual exploratory analysis steps like checking the structure & characteristics of the dataset:

** 1.Get the time range between which the orders were placed.
** 2.display the Cities & States of customers who ordered during the given period. **

### 1.Get the time range between which the orders were placed.

```sql
select
min(order_purchase_timestamp) as start_time,
max(order_purchase_timestamp) as end_time
from `business-analysis-using-sql.Target_id.orders`;
```
### 2.display the Cities & States of customers who ordered during the given period.

```sql
select 
customer_city,customer_state
from `business-analysis-using-sql.Target_id.orders` as o
join `business-analysis-using-sql.Target_id.customers` as c
on o.customer_id = c.customer_id
where EXTRACT(year from order_purchase_timestamp) = 2018 and EXTRACT(month from order_purchase_timestamp) between 1 and 3;
```
## 2. In-depth Exploration:

** 1. Is there a growing trend in the no. of orders placed over the past years? **
** 2. Can we see some kind of monthly seasonality in terms of the no. of orders being placed? **
** 3. During what time of the day, do the Brazilian customers mostly place their orders? (Dawn, Morning, Afternoon or Night) **
- 0-6 hrs : Dawn
- 7-12 hrs : Mornings
- 13-18 hrs : Afternoon
- 19-23 hrs : Night 

### 1. Is there a growing trend in the no. of orders placed over the past years? (Monthly peaks across different years)

```sql
select 
extract(month from order_purchase_timestamp) as month,
count(order_id) as order_num
from `business-analysis-using-sql.Target_id.orders`
group by extract(month from order_purchase_timestamp)
order by order_num;
```

### 3. During what time of the day, do the Brazilian customers mostly place their orders? (Dawn, Morning, Afternoon or Night) 
- 0-6 hrs : Dawn
- 7-12 hrs : Mornings
- 13-18 hrs : Afternoon
- 19-23 hrs : Night */

```sql
with season as(
select *,
case 
    when EXTRACT(HOUR from order_purchase_timestamp)< 7 then "Dawn"
    when EXTRACT(HOUR from order_purchase_timestamp) between 7 and 13
    then "Mornings"
    when EXTRACT(HOUR from order_purchase_timestamp) between 13 and 19 then "Afternoon"
    when EXTRACT(HOUR from order_purchase_timestamp) between 19 and 23 then "Night"
    end as noon
from `business-analysis-using-sql.Target_id.orders`
)

select noon,count(*) as no_of_orders
from season
group by noon
order by no_of_orders
limit 1;
```

## 3. Evolution of E-commerce orders in the Brazil region:

** 1. Get the month on month no. of orders. **
** 2. How are the customers distributed across all the states of brazil? **
      
### 1. Get the month on month no. of orders.

```sql
select 
EXTRACT(year from order_purchase_timestamp) as year,
EXTRACT(month from order_purchase_timestamp) as month,
count(*) as no_of_orders
from `business-analysis-using-sql.Target_id.orders` 
group by year,month
order by year,month;
```

### 2. How are the customers distributed across all the states of brazil?
** DISTINCT customer_id because i need yo know unique customers not repeated. **

```sql
select customer_state,
count(DISTINCT customer_id) as distributed
from `business-analysis-using-sql.Target_id.customers`
group by customer_state
order by distributed;
```

## 4. Impact on Economy: Analyze the money movement by e-commerce by looking at order prices, freight and others.

** 1. Get the % increase in the cost of orders from year 2017 to 2018 (include months between Jan to Aug only).You can use the "payment_value" column in the payments table to get the cost of orders.**
** 2. Calculate the Total & Average price and frieght value for each state. **
      
### 1. Get the % increase in the cost of orders from year 2017 to 2018(include months between Jan to Aug only).You can use the "payment_value" column in the payments table to get the cost of orders.

```sql
** STEP1: Calculate total payments per year. **

with yearly_totals as(
select
EXTRACT(year from o.order_purchase_timestamp) as year,
sum(p.payment_value) as total_payment
from `business-analysis-using-sql.Target_id.payments` as p
join `business-analysis-using-sql.Target_id.orders` as o
on p.order_id = o.order_id
where EXTRACT(year from o.order_purchase_timestamp) in (2017,2018) and EXTRACT(month from o.order_purchase_timestamp) between 1 and 8
group by year),

** STEP2.Use LEAD window fn to compare each year's payments with the previous year. ** 

yearly_comparison as(select
year,
total_payment,
lead(total_payment) over (order by year desc) as prev_year_payment
from yearly_totals
)

** STEP3.percentage increase. ** 

select round(((total_payment-prev_year_payment)/prev_year_payment)*100,2) as percentage_cost
from yearly_comparison;
```

### 2. Calculate the Total & Average price and frieght value for each state.
** in schema order_item table is not connected to customers table(no common col) ** 
** the orders table and customers table are connected ** 
** freight value means the cost required to deliver from purchase to customer ** 

```sql
select 
customer_state,
sum(price)as sum_price,
AVG(price) as avg_price,
AVG(freight_value) as avg_freight
from `business-analysis-using-sql.Target_id.orders` as o
join `business-analysis-using-sql.Target_id.order_items` as oi
on o.order_id = oi.order_id
join `business-analysis-using-sql.Target_id.customers` as c
on o.customer_id = c.customer_id
group by c.customer_state;
```

## 5. Analysis based on sales, freight and delivery time.

** 1. Find the no. of days taken to deliver each order from the order’s purchase date as delivery time. **
        - days_to_deliver = order_delivered_customer_date - order_purchase_timestamp
        - diff_estimated_delivery = order_delivered_customer_date - order_estimated_delivery_date
** 2. Find out the top 5 states with the highest & lowest average freight value. **
** 3. Find out the top 5 states with the highest & lowest average delivery time. **
** 4. Find out the top 5 states where the order delivery is really fast as compared to the estimated date of delivery.You can use the difference between the averages of actual & estimated delivery date to **           ** figure out how fast the delivery was for each state **
        
### 1. calculating days between purchasing,delivering and estimated delivery.

```sql
select
DATE_DIFF(DATE(order_delivered_customer_date),DATE(order_purchase_timestamp),DAY) as days_to_deliver,
DATE_DIFF(DATE(order_delivered_customer_date),DATE(order_estimated_delivery_date),DAY) as Diffence_est_del
from `business-analysis-using-sql.Target_id.orders` ;

```sql

### 2. Find out the top 5 states with the highest & lowest average freight value.

```sql
WITH high_5_low_freight AS (
  SELECT
    c.customer_state,
    AVG(oi.freight_value) AS avg_freight
  FROM `business-analysis-using-sql.Target_id.orders` AS o
  JOIN `business-analysis-using-sql.Target_id.order_items` AS oi
    ON o.order_id = oi.order_id
  JOIN `business-analysis-using-sql.Target_id.customers` AS c
    ON o.customer_id = c.customer_id
  GROUP BY c.customer_state
)

(
  SELECT
    customer_state,
    avg_freight,
    'Top 5' AS category
  FROM high_5_low_freight
  ORDER BY avg_freight DESC
  LIMIT 5
)

UNION ALL

(
  SELECT
    customer_state,
    avg_freight,
    'Bottom 5' AS category
  FROM high_5_low_freight
  ORDER BY avg_freight ASC
  LIMIT 5
);
```

### 3. Find out the top 5 states with the highest & lowest average delivery time.

```sql
WITH avg_price_state AS (
  SELECT
    c.customer_state,
    AVG(EXTRACT (Date from o.order_delivered_customer_date)-EXTRACT (Date from o.order_purchase_timestamp)) AS avg_time_to_delivery
  FROM `business-analysis-using-sql.Target_id.orders` o
  JOIN `business-analysis-using-sql.Target_id.order_items` oi
    ON o.order_id = oi.order_id
  JOIN `business-analysis-using-sql.Target_id.customers` c
    ON o.customer_id = c.customer_id
  GROUP BY c.customer_state
),

ranked_states AS (
  SELECT
    customer_state,
    avg_time_to_delivery,
    ROW_NUMBER() OVER (ORDER BY avg_time_to_delivery DESC) AS rn_high,
    ROW_NUMBER() OVER (ORDER BY avg_time_to_delivery ASC) AS rn_low
  FROM avg_price_state
)

SELECT
  customer_state,
  avg_time_to_delivery,
  CASE
    WHEN rn_high <= 5 THEN 'Top 5'
    WHEN rn_low <= 5 THEN 'Bottom 5'
  END AS category
FROM ranked_states
WHERE rn_high <= 5 OR rn_low <= 5
ORDER BY category, avg_time_to_delivery DESC;
```

### 4. Find out the top 5 states where the order delivery is really fast as compared to the estimated date of delivery.You can use the difference between the averages of actual & estimated delivery date to figure out how fast the delivery was for each state

```sql
WITH avg_fast_del
 AS (
  SELECT
    c.customer_state,
    AVG(EXTRACT(Hour from o.order_delivered_customer_date)-EXTRACT (Hour from o.order_estimated_delivery_date)) AS avg_time_to_delivery
  FROM `business-analysis-using-sql.Target_id.orders` o
  JOIN `business-analysis-using-sql.Target_id.order_items` oi
    ON o.order_id = oi.order_id
  JOIN `business-analysis-using-sql.Target_id.customers` c
    ON o.customer_id = c.customer_id
  GROUP BY c.customer_state
),

ranked_states AS (
  SELECT
    customer_state,
    avg_time_to_delivery,
    ROW_NUMBER() OVER (ORDER BY avg_time_to_delivery DESC) AS rn_high,
    ROW_NUMBER() OVER (ORDER BY avg_time_to_delivery ASC) AS rn_low
  FROM avg_fast_del
)

SELECT
  customer_state,
  avg_time_to_delivery,
  CASE
    WHEN rn_high <= 5 THEN 'Top 5'
    WHEN rn_low <= 5 THEN 'Bottom 5'
  END AS category
FROM ranked_states
WHERE rn_high <= 5 OR rn_low <= 5
ORDER BY category, avg_time_to_delivery DESC;
```

## 6. Analysis based on the payments:

** 1. Find the no. of orders placed using different payment types on month. **
** 2. Find the no. of orders placed on the basis of the payment installments that have been paid. **

### 1. Find no. of orders placed using different payment types.()on month

```sql
select
payment_type,
EXTRACT (Year from order_purchase_timestamp) as year,
EXTRACT (month from order_purchase_timestamp) as month,
count(DISTINCT p.order_id) as order_count

from `business-analysis-using-sql.Target_id.payments` as p
JOIN `business-analysis-using-sql.Target_id.orders` as o
on p.order_id = o.order_id
group by payment_type,year,month
order by payment_type,year,month;
```

### 2. Find the no. of orders placed on the basis of the payment installments that have been paid.

```sql
select
payment_installments,
count(DISTINCT order_id) as num_orders
from `business-analysis-using-sql.Target_id.payments`
group by payment_installments;
```
