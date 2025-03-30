
# databank

## Project overview
There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches like Kudabank.

Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!

Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!

Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts. There are a few interesting caveats that go with this business model, and this is where I come in!


## Problem statement
The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.

### About dataset
Just like popular cryptocurrency platforms - Data Bank is also run off a network of nodes where both money and data is stored across the globe. In a traditional banking sense - you can think of these nodes as bank branches or stores that exist around the world.
The regions table contains the region_id and their respective region_name values.Customers are randomly distributed across the nodes according to their region - this also specifies exactly which node contains both their cash and data. This random distribution changes frequently to reduce the risk of hackers getting into Data Bank’s system and stealing customer’s money and data. The customer transactions  table stores all customer deposits, withdrawals and purchases made using their Data Bank debit card.

### Tools used
- SQL sever for data exploration
- Power BI for data visualization

### Business Questioons
This is divided into three sections;
- Customer node exploration
- Customer transactions
- Data allocation challenge

### Customer node exploration
- The number of unique nodes(branches) neobank has  on the Data Bank system?
```
SELECT COUNT(DISTINCT node_id) AS no_of_nodes
  FROM customer_nodes;
```
- The number of nodes per region?
```
 SELECT COUNT(node_id) AS no_of_nodes,n.region_id,region_name
  FROM customer_nodes n
  LEFT JOIN regions r ON n.region_id = r.region_id
  GROUP BY n.region_id,region_name
  ORDER BY n.region_id;
  -- Australia has the highest number of nodes while Europe has the least number of nodes
```
-  How many customers are allocated to each region
```
  SELECT COUNT(DISTINCT customer_id) AS no_of_customers,n.region_id,region_name
  FROM customer_nodes n
  LEFT JOIN regions r ON n.region_id = r.region_id
  GROUP BY n.region_id,region_name
  ORDER BY n.region_id;
```
- How many days on average are customers reallocated to a different node?
```
 -- how many days on average are customers reallocated to a different node
  -- firstly, how many days it takes for them to the next node
  -- then add the number of days for each customer
  -- then the average number of days 
  WITH DAYS_NO AS (
  SELECT  customer_id,node_id,SUM(DATEDIFF(DAY,start_date,end_date)) AS no_of_days
  FROM customer_nodes
  WHERE end_date <> '9999-12-31'
  -- added this condition because thats the last end endate showing that after this customers did not move again
  -- we want to check the next node for each customer
  GROUP BY customer_id,node_id
  )
  SELECT customer_id,no_of_days,AVG(no_of_days)  OVER()  AS avg_no_before_reallocation 
  FROM 
  DAYS_NO
  GROUP BY customer_id,no_of_days
  ;
```
- What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
```
 WITH DAYS_NO AS (
  SELECT customer_id,region_name,node_id,SUM(DATEDIFF(DAY,start_date,end_date)) AS no_of_days
  FROM customer_nodes n
  INNER JOIN regions r ON n.region_id = r.region_id
  -- to get the region name from regions table
  WHERE end_date <> '9999-12-31'
  -- added this condition because thats the last end endate showing that after this customers did not move again
  -- we want to check the next node for each customer
  GROUP BY region_name,customer_id,node_id
  ),
  ROW_TABLE AS (
  SELECT region_name,no_of_days,ROW_NUMBER() OVER(PARTITION BY region_name ORDER BY no_of_days) AS rnk
  -- using the row function because we want to know how many in each region
  -- and with the row number, we can get the maximum distribution for each region
  -- this would help us in getting the median and 80th percentile
  FROM DAYS_NO 
  ), MAX_ROW AS (
  SELECT region_name,CAST(MAX(rnk) AS FLOAT) AS max_rn
  FROM ROW_TABLE
  GROUP BY region_name
  )
  SELECT region_name,max_rn,max_rn/2 AS median,max_rn*0.80 AS eighty_percentile,max_rn*0.95 AS ninety_five_percentile
  FROM MAX_ROW;
```
### Customer transactions
- The unique count and total amount for each transaction type?
```
 SELECT COUNT(DISTINCT customer_id) AS unique_customers,SUM(txn_amount) AS total_amount,txn_type
 FROM customer_transactions
 GROUP BY txn_type
 ORDER BY 2 DESC;
 -- we have 439 withdrawals,500 deposits,448 purchases
```
- The average total historical deposit counts and amounts for all customers?
```
-- firstly we get the average deposits for each customer then we get the average total deposit counts for all customers
  WITH deposits AS (
 SELECT customer_id,COUNT(*) AS no_deposits,CAST(AVG(txn_amount) AS FLOAT) AS avg_amount
 -- changing the data type to float when using measures of central tendency because i want to get the exact value
 FROM customer_transactions
 WHERE txn_type = 'deposit'
 GROUP BY customer_id
 )
 SELECT AVG(avg_amount) AS average_total_amount,AVG(no_deposits) AS avg_total_deposit_count
 FROM deposits;
 -- we have an average of 5 deposits for each customer and an average deposit amount of 508.248

```
- For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
```
WITH monthly_transactions AS (
SELECT customer_id,
FORMAT(txn_date,'yyyy-MM') AS Month,
SUM(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS depositcount,
SUM(CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) AS purchasecount,
SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS withdrawalcount
FROM customer_transactions
GROUP BY customer_id,FORMAT(txn_date,'yyyy-MM')
)
SELECT Month, COUNT(DISTINCT customer_id) AS qualifiedcustomers
FROM monthly_transactions
WHERE depositcount > 1 AND (purchasecount >= 1 OR withdrawalcount >= 1)
GROUP BY Month
ORDER BY Month;

-- 192 customers in the month of march, 181 for the month of february,168 for january and 70 for April
```
- What is the closing balance for each customer at the end of the month?
```
WITH monthlytransactions AS (
SELECT  customer_id,txn_date,
FORMAT(txn_date,'yyyy-MM') AS month,
SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
	WHEN txn_type IN ('withdrawal','purchase') THEN -(txn_amount)
	ELSE 0 
	END) AS daily_transactions
	-- to calculate the total monthly balance at the end of each transaction day
FROM customer_transactions
GROUP BY customer_id,FORMAT(txn_date,'yyyy-MM'),txn_date
),
running_balance AS (SELECT mt.customer_id,mt.txn_date,mt.month,mt.daily_transactions,SUM(mt.daily_transactions)
OVER(PARTITION BY mt.customer_id ORDER BY mt.txn_date ) AS closing_balance
FROM monthlytransactions mt
GROUP BY mt.customer_id,mt.daily_transactions,mt.txn_date,mt.month
-- to calculate the closing balance after all deductions has been made
),
final_balance AS (SELECT r.customer_id,r.txn_date,r.month,r.daily_transactions,SUM(r.closing_balance)
OVER (PARTITION BY r.customer_id,r.month) AS end_balance,ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY txn_date DESC) AS rn 
FROM running_balance r
) -- i used the row number function so we can get the final balance for each customer at the last date
SELECT *
FROM final_balance
WHERE rn = 1
;
-- therefore the end balance is the closing balance for each customer in the data bank

```
- What is the percentage of customers who increase their closing balance by more than 5%?
```
WITH monthlytransactions AS (
SELECT  customer_id,txn_date,
FORMAT(txn_date,'yyyy-MM') AS month,
SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
	WHEN txn_type IN ('withdrawal','purchase') THEN -(txn_amount)
	ELSE 0 
	END) AS Monthly_net_change
	-- to calculate the monthly balance at the end of each transaction day
FROM customer_transactions
GROUP BY customer_id,FORMAT(txn_date,'yyyy-MM'),txn_date
),
running_balance AS (SELECT mt.customer_id,mt.txn_date,mt.month,mt.Monthly_net_change,SUM(mt.Monthly_net_change)
OVER(PARTITION BY mt.customer_id ORDER BY mt.txn_date ) AS closing_balance
FROM monthlytransactions mt
GROUP BY mt.customer_id,mt.month,mt.Monthly_net_change,mt.txn_date
-- to calculate the closing balance at the end of each month for each customer
),
final_balance AS (SELECT r.customer_id,r.txn_date,r.month,r.Monthly_net_change,r.closing_balance,SUM(r.closing_balance)
OVER (PARTITION BY r.customer_id,r.month) AS end_balance 
FROM running_balance r
-- the final balance for each customer after their purchases and withdrawals
),
final_balance_percent AS (
SELECT customer_id,month,end_balance,LEAD(end_balance) OVER (PARTITION BY customer_id ORDER BY month) AS next_end_balance,
CASE WHEN  LEAD(end_balance) OVER (PARTITION BY customer_id ORDER BY month)  IS NOT NULL THEN
(end_balance - LEAD(end_balance) OVER (PARTITION BY customer_id ORDER BY month))*100/
LEAD(end_balance) OVER (PARTITION BY customer_id ORDER BY month) 
ELSE 0
END AS percentage_change
FROM final_balance
GROUP BY customer_id,month,end_balance)
-- we want to see the percent difference in the end balance monthly
-- this will help us to calculate how many customers increased their balance by more than 5%
SELECT (COUNT(DISTINCT CASE WHEN percentage_change > 5 THEN customer_id END)*100)/COUNT(DISTINCT customer_id) AS percentage_of_customers
FROM final_balance_percent;
-- 55 percent of customers increased by 5% in their transactions
```





  
