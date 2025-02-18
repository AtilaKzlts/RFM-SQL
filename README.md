
# RFM Analyis 
## Project Introduction

In this project, we use the RFM (Recency, Frequency, Monetary) model to analyze customer behavior using customer data from our PostgreSQL database. The goal is to make quick decisions regarding recommendations by segmenting customers based on three key metrics:

+ Recency (R): How recently a customer made a purchase.
+ Frequency (F): How often a customer makes a purchase.
+ Monetary (M): How much money a customer spends.

By evaluating these metrics, we aim to determine customer value segments. These segments will be used to manage marketing campaigns, increase customer loyalty, and optimize sales strategies.
## Executive summary

![image](https://github.com/AtilaKzlts/RFM-SQL/blob/main/assets/raports.png)

## Objective 
+ What are the dates with the most orders?
+ What are the dates with the highest quantity of items sold?
+ What are the dates with the highest revenue?
+ Who are the customers that generate the most revenue?
+ RFM Analysis
-----

## Analyis
### What are the dates with the most orders?
```sql
SELECT DISTINCT invoicedate,
       COUNT(invoicedate) OVER(PARTITION BY invoicedate) AS Number_Of_Orders
FROM online_retail or2 
ORDER BY Number_Of_Orders DESC;

```

| invoicedate          | number_of_orders |
|----------------------|------------------|
| 2024-10-31 14:41:00  | 1114             |
| 2024-12-08 09:28:00  | 749              |
| 2024-12-09 10:03:00  | 731              |
| 2024-12-05 17:24:00  | 721              |

### What are the dates with the highest quantity of items sold?
```sql
SELECT DISTINCT invoicedate,
            SUM(quantity) OVER(PARTITION BY invoicedate) as Total_Quantities_Per_Date
FROM online_retail  
ORDER BY Total_Quantities_Per_Date DESC;

```

| invoicedate          | total_quantities_per_date |
|----------------------|---------------------------|  
| 2024-12-09 09:15:00  | 80995                     |  
| 2024-01-18 10:01:00  | 74215                     |  
| 2024-06-15 13:37:00  | 15241                     |  
| 2024-08-11 16:12:00  | 14730                     |

### What are the dates with the highest revenue?
```sql
SELECT DISTINCT invoicedate,
            ROUND(SUM(unitprice::int *quantity) OVER(PARTITION BY invoicedate),0) AS Total_Price_Of_Orders
FROM online_retail or2 
ORDER BY Total_Price_Of_Orders DESC;

```


| invoicedate          | total_price_of_orders |  
|----------------------|-----------------------|  
| 2024-12-09 09:15:00  | 161990                |  
| 2024-01-18 10:01:00  | 74215                 |  
| 2024-11-07 17:42:00  | 53953                 |  
| 2024-11-14 17:55:00  | 51910                 |  
| 2024-06-10 15:28:00  | 39622                 |  



### Who are the customers that generate the most revenue?
```sql
SELECT DISTINCT customerId,
       invoicedate,
       ROUND(SUM(unitprice ::int *quantity) OVER(PARTITION BY customerId, invoicedate),0) AS Total_Price
FROM online_retail  
ORDER BY Total_Price DESC;

```
| customerid | invoicedate          | total_price |  
|------------|----------------------|-------------|  
| 16446      | 2024-12-09 09:15:00  | 161990      |  
| 12346      | 2024-01-18 10:01:00  | 74215       |  
| 50869      | 2024-11-07 17:42:00  | 53953       |  
| 98741      | 2024-11-14 17:55:00  | 51910       |  


### RFM

```sql
WITH customer_basic AS (
    SELECT 
        customerid,
        MAX(invoicedate::timestamp) AS last_date,
        COUNT(DISTINCT invoiceno) AS frequency,
        SUM(quantity * unitprice) AS monetary
    FROM 
        online_retail
    WHERE 
        customerid IS NOT NULL
    GROUP BY 
        customerid
),
customer_recency AS (
    SELECT 
        customerid,
        last_date,
        ROUND(EXTRACT(EPOCH FROM ('2024-12-09'::timestamp - last_date)) / (24 * 60 * 60)) AS recency,
        frequency,
        monetary
    FROM 
        customer_basic
),
customer_segment AS (
    SELECT 
        customerid,
        last_date,
        recency,
        frequency,
        monetary,
        NTILE(5) OVER(ORDER BY recency DESC) AS recency_score,
        NTILE(5) OVER(ORDER BY monetary) AS monetary_score,
        CASE
            WHEN CURRENT_DATE - last_date::date <= 30 THEN 'Very Recent'
            WHEN CURRENT_DATE - last_date::date <= 60 THEN 'Recent'
            ELSE 'Not Recent'
        END AS recency_segment,
        CASE
            WHEN frequency >= 5 THEN 'High Frequency'
            WHEN frequency >= 2 THEN 'Medium Frequency'
            ELSE 'Low Frequency'
        END AS frequency_segment,
        CASE
            WHEN monetary >= 500 THEN 'High Value'
            WHEN monetary >= 100 THEN 'Medium Value'
            ELSE 'Low Value'
        END AS monetary_segment
    FROM 
        customer_recency
)
SELECT 
    customerid,
    last_date,
    recency,
    frequency,
    monetary,
    recency_segment,
    frequency_segment,
    monetary_segment,
    CASE
        WHEN recency_score = 5 AND monetary_score = 5 THEN 'Champions'
        WHEN recency_score = 4 AND monetary_score = 5 THEN 'Champions'
        WHEN recency_score = 5 AND monetary_score = 4 THEN 'Champions'
        WHEN recency_score = 5 AND monetary_score = 2 THEN 'Potential Loyalists'
        WHEN recency_score = 4 AND monetary_score = 2 THEN 'Potential Loyalists'
        WHEN recency_score = 4 AND monetary_score = 3 THEN 'Potential Loyalists'
        WHEN recency_score = 3 AND monetary_score = 3 THEN 'Potential Loyalists'
        WHEN recency_score = 5 AND monetary_score = 3 THEN 'Loyal Customers'
        WHEN recency_score = 4 AND monetary_score = 4 THEN 'Loyal Customers'
        WHEN recency_score = 3 AND monetary_score = 5 THEN 'Loyal Customers'
        WHEN recency_score = 3 AND monetary_score = 4 THEN 'Loyal Customers'
        WHEN recency_score = 5 AND monetary_score = 1 THEN 'Recent Customers'
        WHEN recency_score = 4 AND monetary_score = 1 THEN 'Promising'
        WHEN recency_score = 3 AND monetary_score = 1 THEN 'Promising'
        WHEN recency_score = 3 AND monetary_score = 2 THEN 'Customers Needing Attention'
        WHEN recency_score = 2 AND monetary_score = 3 THEN 'Customers Needing Attention'
        WHEN recency_score = 2 AND monetary_score = 2 THEN 'Customers Needing Attention'
        WHEN recency_score = 2 AND monetary_score = 5 THEN 'At Risk'
        WHEN recency_score = 2 AND monetary_score = 4 THEN 'At Risk'
        WHEN recency_score = 1 AND monetary_score = 3 THEN 'At Risk'
        WHEN recency_score = 1 AND monetary_score = 5 THEN 'Cannot Lose Them'
        WHEN recency_score = 1 AND monetary_score = 4 THEN 'Cannot Lose Them'
        WHEN recency_score = 1 AND monetary_score = 2 THEN 'Hibernating'
        WHEN recency_score = 1 AND monetary_score = 1 THEN 'Lost'
    END AS customer_segment
FROM 
    customer_segment
ORDER BY 
    recency DESC, frequency DESC, monetary DESC;

```

| customerid | last_date              | recency | frequency | monetary | recency_segment | frequency_segment | monetary_segment | customer_segment |
|------------|------------------------|---------|-----------|----------|-----------------|-------------------|------------------|------------------|
| 18074.0    | 2024-12-01 09:53:00    | 373     | 1         | 489.60   | Not Recent      | Low Frequency     | Medium Value     | At Risk          |
| 17908.0    | 2024-12-01 11:45:00    | 373     | 1         | 243.28   | Not Recent      | Low Frequency     | Medium Value     | Hibernating      |
| 12791.0    | 2024-12-01 11:27:00    | 373     | 1         | 192.60   | Not Recent      | Low Frequency     | Medium Value     | Lost             |
