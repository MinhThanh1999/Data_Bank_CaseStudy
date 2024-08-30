# B. Customer Transactions

1. What is the unique count and total amount for each transaction type?

```sql
SELECT  txn_type
, COUNT(DISTINCT customer_id) as unique_transaction
, SUM(txn_amount) as total_amount
FROM data_bank.customer_transactions
GROUP BY txn_type;
```

Output:

| txn_type | unique_transaction | total_amount |
| --- | --- | --- |
| deposit | 500 | 1359168 |
| purchase | 448 | 806537 |
| withdrawal | 439 | 793003 |

1. What is the average total historical deposit counts and amounts for all customers?

Query:

```sql
SELECT SUM(txn_amount)/ count(distinct customer_id) as avg_amount
, SUM(txn_amount) AS total_amount
FROM data_bank.customer_transactions
GROUP BY txn_type = 'deposit';
```

Output:

| avg_amount | total_amount |
| --- | --- |
| 3298 | 1599540 |
| 2718 | 1359168 |

⇒ The output above shows the average and total amounts for all customers by type of deposit.

1. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

Query:

```sql
WITH count_type as (
  SELECT customer_id 
  		, EXTRACT (MONTH FROM txn_date) AS months
  		, SUM(CASE WHEN txn_type = 'deposit' THEN 1 ELSE 0 END) AS total_deposit
  		, SUM(CASE WHEN txn_type = 'purchase' THEN 1 ELSE 0 END) AS total_purchase
  		, SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS total_withdrawal
  FROM data_bank.customer_transactions
  GROUP BY customer_id, months
  )
SELECT months
	, COUNT(customer_id)
FROM count_type
WHERE total_deposit > 1 AND (total_purchase = 1 or total_withdrawal = 1)
GROUP BY months
```

Output:

| months | count |
| --- | --- |
| 1 | 115 |
| 2 | 108 |
| 3 | 113 |
| 4 | 50 |

⇒ For our data bank, we have 168 customers in January, 181 customers in February, 192 customers in March, and 70 customers in April who qualified based on the mentioned conditions

1. What is the closing balance for each customer at the end of the month?

STEP:

1. **Create a CTE table that shows each customer’s total amount per month:**
    - Define the amount for each type of transaction, such as: “deposit” as positive, “purchase” and “withdrawal” as negative.
    - Then calculate the total amount of transactions per month for each customer.
2. **Use a window function to calculate the closing balance:**
    - The closing balance at the end of each month = the previous closing balance + the cash flow transactions in that month.
    - The window function allows us to perform the above calculation without excessive complexity.

```sql
WITH raw_table as (
SELECT customer_id
	, EXTRACT (MONTH FROM txn_date) AS months
	, SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE (-1)*txn_amount END) AS total_amount
FROM data_bank.customer_transactions
GROUP BY customer_id, months
ORDER BY customer_id, months
)
SELECT customer_id
		, months
		, SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY months) AS closing_balance
FROM raw_table

```

Output:

- This result is just a part of the output

| customer_id | months | closing_balance |
| --- | --- | --- |
| 1 | 1 | 312 |
| 1 | 3 | -640 |
| 2 | 1 | 549 |
| 2 | 3 | 610 |
| 3 | 1 | 144 |
| 3 | 2 | -821 |
| 3 | 3 | -1222 |
| 3 | 4 | -729 |
| 4 | 1 | 848 |
| 4 | 3 | 655 |
| 5 | 1 | 954 |
| 5 | 3 | -1923 |
| 5 | 4 | -2413 |

1. What is the percentage of customers who increase their closing balance by more than 5%?

ASSUMPTION: 

- I assume that we will calculate the customers who have had at least one instance where their next closing balance increased by more than 5% compared to their current balance in a positive way

### Steps:

1. **First step:**
    - This step will be the same as in question number 4.
2. **Create a Cash Flow CTE:**
    - In this step, create a CTE that includes the ‘closing balance’ and ‘next closing balance’.

```sql
cash_flow_table as (
SELECT customer_id
, months
, closing_balance
, COALESCE(LEAD(closing_balance) OVER (PARTITION BY customer_id ORDER BY months), 0) AS next_closing
FROM balance_table)
```

1. **Calculate the Growth Percentage:**
    - Use the following conditions:
        - next closing balance > closing balance to exclude accounts that did not increase positively.
        - (next closing balance - closing balance) > closing balance * 0.05 to ensure that the closing balance increased by more than 5%, and to avoid division by zero.

```sql
SELECT
ROUND (100.0 * COUNT(DISTINCT 
			(CASE WHEN (closing_balance < next_closing 
				AND (next_closing - closing_balance) > ABS(closing_balance*0.05)) 
				THEN customer_id ELSE null END)) / COUNT(DISTINCT customer_id),1) AS percent_growth
FROM cte
```

Full query:

```sql
WITH raw_table as (
SELECT customer_id
, EXTRACT (MONTH FROM txn_date) AS months
, SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE (-1)*txn_amount END) AS total_amount
FROM data_bank.customer_transactions
GROUP BY customer_id, months
ORDER BY customer_id, months
)
,
balance_table as (
SELECT customer_id
, months
, SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY months) AS closing_balance
FROM raw_table)
,	
cash_flow_table as (
```

```sql
SELECT customer_id
, months
, closing_balance
, COALESCE(LEAD(closing_balance) OVER (PARTITION BY customer_id ORDER BY months), 0) AS next_closing
FROM balance_table)
```

```sql
SELECT
ROUND (100.0 * COUNT(DISTINCT 
			(CASE WHEN (closing_balance < next_closing 
					AND (next_closing - closing_balance) > ABS(closing_balance*0.05)) 
					THEN customer_id ELSE null END)) / COUNT(DISTINCT customer_id),1)  AS percent_growth
FROM cte
```

Output:

| percent_growth |
| --- |
| 95.8 |

⇒ We have 95.8% of customers who have had at least one instance of increasing their closing balance by more than 5%.