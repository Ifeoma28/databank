
# DATA BANK
![dashboard](https://github.com/Ifeoma28/databank/blob/4dbcf1cc4939386ca0351e738e7439a5d60b0db5/Neo%20dashboard.png)


## PROJECT OVERVIEW
There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches like Kudabank.

Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!

Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!

Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts.

There are a few interesting caveats that go with this business model, and this is where I come in!


## PROBLEM STATEMENT
The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.

### ABOUT DATASET
We have three tables; customer nodes, customer transactions and regions table.
Just like popular cryptocurrency platforms - Data Bank is also run off a network of nodes where both money and data is stored across the globe. 

In a traditional banking sense - you can think of these nodes as bank branches or stores that exist around the world. The customer nodes table contain the customer ID, region ID,nodes ID,start_date and end_date.

Customers are randomly distributed across the nodes according to their region - this also specifies exactly which node contains both their cash and data. 
This random distribution changes frequently to reduce the risk of hackers getting into Data Bank’s system and stealing customer’s money and data.

![database relationship](https://github.com/Ifeoma28/databank/blob/657bdaf747a3c3e2e4ec054bac9231dcd3b07a18/relationship%20databank.png)
The regions table contains the region_id and their respective region_name values. The customer transactions  table stores all customer deposits, withdrawals and purchases made using their Data Bank debit card.


### TOOLS USED
- SQL sever for data exploration
- Power BI for data visualization

### BUSINESS QUESTIONS
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
![median reallocation](https://github.com/Ifeoma28/databank/blob/e286cf644f7d404f0f97b73558292fc5ec99db46/median%20information.png)
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
SELECT  customer_id,
EOMONTH(txn_date) AS end_of_month,
SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount
	WHEN txn_type IN ('withdrawal','purchase') THEN -(txn_amount)
	ELSE 0 
	END) AS daily_transactions
	-- to calculate the total monthly balance at the end of each transaction day
FROM customer_transactions
GROUP BY customer_id,EOMONTH(txn_date)
),
running_balance AS (SELECT mt.customer_id,mt.end_of_month,mt.daily_transactions,SUM(mt.daily_transactions)
OVER(PARTITION BY mt.customer_id ORDER BY end_of_month ) AS closing_balance
FROM monthlytransactions mt
GROUP BY mt.customer_id,mt.daily_transactions,mt.end_of_month
),
prev_closing AS (
SELECT customer_id,end_of_month,closing_balance,LAG(closing_balance) OVER 
(PARTITION BY customer_id ORDER BY end_of_month DESC ) AS prev_closing_balance
FROM running_balance
GROUP BY customer_id,end_of_month,closing_balance
)
-- we want to see the percent difference in the end balance monthly
-- this will help us to calculate how many customers increased their balance by more than 5%
SELECT CASE 
	WHEN COUNT(DISTINCT customer_id) = 0 THEN 0
	-- prevents divide by zero
	ELSE
		(COUNT(DISTINCT CASE WHEN prev_closing_balance IS NOT NULL
		AND prev_closing_balance > 0
		AND (closing_balance - prev_closing_balance)/prev_closing_balance > 0.05 
		THEN customer_id 
		END)*100)/ COUNT(DISTINCT customer_id) 
	END AS percentage_of_customers
FROM prev_closing;
-- 23 percent of customers increased their closing balance by more than 5% in their transactions
```
### Data allocation challenge
To test out a few different hypotheses - the Data Bank team wants to run an experiment where different groups of customers would be allocated data using 2 different options:

- Option 1: data is allocated based off the amount of money at the end of the previous month
- Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days

- Option 1
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
OVER(PARTITION BY mt.customer_id ORDER BY mt.txn_date ) AS running_closing_balance
FROM monthlytransactions mt
GROUP BY mt.customer_id,mt.daily_transactions,mt.txn_date,mt.month
-- to calculate the closing balance after all deductions has been made
),
final_balance AS (SELECT r.customer_id,r.txn_date,r.month,r.daily_transactions,r.running_closing_balance
FROM running_balance r
),
monthly AS (
SELECT customer_id,month,SUM(running_closing_balance) AS monthly_balance
FROM final_balance
GROUP BY month,customer_id
)
SELECT  *,LEAD(monthly_balance) OVER(PARTITION BY customer_id ORDER BY month DESC) AS previous_monthly_balance
FROM monthly;
-- with this method more customers would be allocated.
-- customers that had little money from the previous month would be allocated data even 
-- though their balance might be in red at the end of the month
```
- Option 2
```
-- we are trying to understand how data is being allocated, firstly lets see the average amount
--  of money kept in the account in the previous 30 days
-- or if its allocated based off the amount of money at the previous month
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
OVER(PARTITION BY mt.customer_id ORDER BY mt.txn_date ) AS running_closing_balance
FROM monthlytransactions mt
GROUP BY mt.customer_id,mt.daily_transactions,mt.txn_date,mt.month
-- to calculate the closing balance after all deductions has been made
),
final_balance AS (SELECT r.customer_id,r.txn_date,r.month,r.daily_transactions,r.running_closing_balance
FROM running_balance r
),
monthly AS (
SELECT customer_id,month,SUM(running_closing_balance) AS monthly_balance
FROM final_balance
GROUP BY month,customer_id
)
SELECT  *,AVG(monthly_balance) OVER(PARTITION BY customer_id) AS avg_monthly_balance
FROM monthly;
-- so this is the average amount of money kept in the account in the previous 30 days
-- we can see some customers wont be allocated cloud storage because they are in debt (-)

```
## ANALYSIS
Let's look at the average closing balance by Region.
![average closing balance](https://github.com/Ifeoma28/databank/blob/f7377134bc15b0c2417242df6e50cdb26fc5c6b1/Average%20closing%20balance.png)

Customers in America are the most financially stable with the highest average balance.

Europe has a lower but still positive balance, indicating some financial stability but not as strong as America.

Australia & Africa have the most financially struggling customers (deep negative balances).

Asia is slightly negative, but not as severe. Also they may just be few customers who deposit more. Let us look at the median.

![median closing balance](https://github.com/Ifeoma28/databank/blob/a1d2596c7fe5f8920436399d3e21d6b7c41a4cc8/median%20closing%20balance.png)
This means that at least 50% of customers in every region are in debt or have overdrafted their accounts.

This suggests higher debt levels or low savings behavior in these regions.

Let us look at the deposit rate, withdrawal and purchase rate in these regions 
![withdraw and purchase](https://github.com/Ifeoma28/databank/blob/7ebdb330b92c2aa3365709c3dedd7d35a36e002a/purchase%20and%20withdrawal.png)

  ![deposit rate](https://github.com/Ifeoma28/databank/blob/fe0eb84de516239f05efcb94ad87aa37b1eb75cf/deposit%20rate.png)
  Customers in Australia spend more than they earn and that's why they have the highest number of financially struggling customers despite the high deposit rate.
  
  ![declining balance](https://github.com/Ifeoma28/databank/blob/a8e307adf766a348a4523cf98f715eefaf1e0fc3/declining%20balances.png)
To identify financially struggling customers, i used the declining balance trend.
I created a DAX measure that checks if a customer's closing balance has decreased for 3 or more consecutive months.  

I created another metric to calculate active customers (customers that have made more than 7 transactions)


### KEY INSIGHTS

- Customer Movement & Node Reallocation:
The median time to reallocate in Australia is 206 days while in Europe is 166 days.

- The fact that Europeans change nodes more frequently could suggest more flexible digital banking habits and is another reason for having the lowest number of Active customers.

- Monthly Trends: A lot of customers had negative balances by the end of February till April.
![monthly closing balance](https://github.com/Ifeoma28/databank/blob/5c7f885701e01e42aef72b72353f9ba8ad2a1b87/closing%20balance%20monthly.png)

- Africa having the lowest average closing balance aligns with financial struggles observed earlier.
  
- Activity & Transaction Behavior:
361 customers made more than 7 transactions → highly active users.

- Australia & America having more active customers → Could indicate stronger engagement with digital banking.

![Active customers](https://github.com/Ifeoma28/databank/blob/da8a114af848a6396956f36edc019742eeb1c581/Active%20customers.png)

- 460 customers have had a declining balance for the past three months.

## RECOMMENDATION 
