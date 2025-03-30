
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
  
