# A. Customer nodes exploration

1. **How many unique nodes are there on the Data Bank system?**

```sql
SELECT COUNT(DISTINCT node_id) as num_unique_nodes
FROM data_bank.customer_nodes;
```

Output:

| num_unique_nodes |
| --- |
| 5 |
1. **What is the number of nodes per region?**

```sql
SELECT region_name, 
	COUNT(node_id) as num_nodes
FROM data_bank.customer_nodes
LEFT JOIN data_bank.regions
USING (region_id)
GROUP BY region_name
ORDER BY num_nodes DESC
```

Output:

| region_name | num_nodes |
| --- | --- |
| Australia | 770 |
| America | 735 |
| Africa | 714 |
| Asia | 665 |
| Europe | 616 |
1. How many customers are allocated to each region?

```sql
SELECT region_name,
COUNT(DISTINCT customer_id) as num_customer
FROM data_bank.customer_nodes
LEFT JOIN data_bank.regions
USING (region_id)
GROUP BY region_name
ORDER BY num_customer DESC
```

Output:

| region_name | num_customer |
| --- | --- |
| Australia | 110 |
| America | 105 |
| Africa | 102 |
| Asia | 95 |
| Europe | 88 |

1. How many days on average are customers reallocated to a different node?
- Steps:
    - We have to check start_date and end_date columns for any misleading data

```sql
SELECT DISTINCT start_date
FROM customer_nodes
ORDER BY start_date DESC;
```

```sql
SELECT DISTINCT end_date
FROM customer_nodes
ORDER BY end_date DESC;
```

- then we see that an abnormal end_date is “9999-12-31”

⇒ if we do not remove this data it will be misleading for whole analysis so we have to exclude it from the query

```sql
WITH raw_table as(
SELECT customer_id
			, SUM(end_date - start_date) / count(distinct node_id) as avg_per_customer
FROM data_bank.customer_nodes
WHERE end_date <> '9999-12-31'
GROUP BY customer_id
)
SELECT ROUND(AVG(avg_per_customer),0) as avg_days
FROM raw_table
```

- Explain:
    - At first, calculate average days a customer reallocated to a different node
    - Then calculate average days for all customer

Output:

| avg_days |
| --- |
| 24 |

1. What is the median, 80th and 95th percentile for this same reallocation days metric for each region?
- Steps:
    - Create CTE table with date_diff column for finding median, 80th and 95th percentile. Exclude end_date = ‘9990-12-31’
    - Use PERCENTILE_CONT() to query

```sql
WITH filtered_table as (
SELECT region_name
, (end_date - start_date) AS avg_day
FROM data_bank.customer_nodes
LEFT JOIN data_bank.regions
USING (region_id)
WHERE end_date <> '9999-12-31'
)
```

```sql
SELECT region_name
, PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY avg_day) as median
, PERCENTILE_CONT(0.8) WITHIN GROUP (ORDER BY avg_day) as percentile_80
, PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY avg_day) as percentile_95
FROM filtered_table
GROUP BY region_name
```

Output:

| region_name | median | percentile_80 | percentile_95 |
| --- | --- | --- | --- |
| Africa | 15 | 24 | 28 |
| America | 15 | 23 | 28 |
| Asia | 15 | 23 | 28 |
| Australia | 15 | 23 | 28 |
| Europe | 15 | 24 | 28 |