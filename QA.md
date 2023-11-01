### QA.md
---
&nbsp;

**Risks to explore:**
1. Revenue data from session_transactions may differ from session_analytics
2. Product quantities from session_transactions, session_analytics
3. Product prices from session_transactions may differ from session_analytics
4. Sales quantities from session_transactions may differ from product_sales

&nbsp;

## Risk 1 : Revenue values from session_transactions may differ from session_analytics

### SQL Queries:
- explore how session_transactions.totaltransactionrevenue differs from session_analytics.revenue
```SQL
-- create a table that displays all overlapping id-pairs
-- and their associated revenue and transaction data
CREATE OR REPLACE VIEW qa_merged_transaction_data AS (
	SELECT
		COALESCE(st.sessionid, sa.sessionid) AS sessionid,
		st.transactionoccured,
		st.totaltransactionrevenue st_revenue,
		st.productprice st_price,
		st.productquantity AS st_quantity,
		sa.revenue AS sa_revenue,
		sa.unitssold AS sa_quantity,
		sa.unitprice AS sa_price
	FROM
		session_transactions st
	JOIN
		session_analytics sa USING (sessionid)
);
```
```SQL
-- display where revenue values differ
CREATE OR REPLACE VIEW qa1_revenue_variance AS (
	SELECT
		sessionid,
		st_revenue,
		sa_revenue
	FROM
		qa_merged_transaction_data
	WHERE
		st_revenue != sa_revenue
		AND transactionoccured
		AND sa_revenue IS NOT NULL
);

SELECT * FROM qa1_revenue_variance;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=107QU0RHBQ_-jDd2ox8QQxcmrzgaLDDme)

- (60) entries where st_revenue differs from sa_revenue from st_revenue for the same session sa_revenue has multiple values for a given st_revenue in the same session

&nbsp;

```SQL
-- compare aggregated sa_revenue to st_revenue
CREATE OR REPLACE VIEW qa1_aggregated_revenue_variance AS (
	SELECT
		sessionid,
		st_revenue,
		SUM(sa_revenue)
	FROM
		qa_merged_transaction_data
	GROUP BY
		sessionid,
		st_revenue
	HAVING
		SUM(sa_revenue) IS NOT NULL
);
SELECT * FROM qa1_aggregated_revenue_variance;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=10CW-fx5zUm5BSC4nYiLt2tQ5Ho9V-hgI)

### FINDINGS:
- revenue variance with st_revenue is resolved by aggregating sa_revenue

&nbsp;

## Risk 2 : Product quantities from session_transactions may differ from session_analytics

### SQL Queries:
```SQL
-- sa_quantity can have multiple values for a given sessionid - st_quantity match
-- assume that sa_quantity must be aggregated similar to sa_revenue based on FINDINGS from 'Risk 1'
CREATE OR REPLACE VIEW qa2_productquantity_variance AS (
	SELECT
		sessionid,
		st_quantity,
		SUM(sa_quantity) AS sa_quantity
	FROM
		qa_merged_transaction_data
	GROUP BY
		sessionid,
		st_quantity
	HAVING
		SUM(sa_quantity) IS NOT NULL
		AND SUM(sa_quantity) != st_quantity
);

SELECT * FROM qa2_productquantity_variance;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=10GkcyrHVXNSr2zswFznQz9MoTH66Tp8J)

### FINDINGS:
- (**29**) rows where st_quantity and aggregated sa_quantity have difference quantities
- either quantity value can be larger

&nbsp;

## Risk 3 : Product prices from session_transactions may differ from session_analytics

### SQL Queries:
```SQL
-- sa_price can have multiple values for a given sessionid - st_price match
-- assume that sa_price must be approriately aggregated similar to sa_revenue based on FINDINGS from 'Risk 1'
CREATE OR REPLACE VIEW qa3_productprice_variance AS (
	SELECT
		sessionid,
		st_price,
		ROUND(AVG(sa_price), 2) AS sa_price
	FROM
		qa_merged_transaction_data
	GROUP BY
		sessionid,
		st_price
	HAVING
		SUM(sa_price) IS NOT NULL
		AND SUM(sa_price) != st_price
);

SELECT * FROM qa3_productprice_variance;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=10JcJ4QaS1m0kR4hpMBHVq9kbCISP-IYC)

### FINDINGS:
- (**3673**) rows where st_price and averaged sa_price have different quantities
- either st_price or avg sa_price can be higher for a given sessionid
- requires further investigation into pattern to determine what this variance represents

&nbsp;

## Risk 4 : Sales quantities from session_transactions may differ from product_sales

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW qa4_product_sales_variance AS (
	SELECT
		st.sku,
		SUM(st.productquantity) AS st_orderedquantity,
		ps.orderedquantity,
		ps.totalordered 
	FROM
		session_transactions st
	JOIN
		product_sales ps
		ON st.sku = ps.sku
		AND transactionoccured
	GROUP BY
		st.sku,
		ps.orderedquantity,
		ps.totalordered
	HAVING
		SUM(st.productquantity) != ps.orderedquantity
		OR SUM(st.productquantity) != ps.totalordered
);


SELECT * FROM qa4_product_sales_variance;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=10WgCrQcIe2ND2itKoCMSoHo90G3ga3Bw)

### FINDINGS:
- (**46**) productskus where aggregated product sales from product_transactions do not match either recorded sales quantity in product_sales
- more exploration needed to determine if productquantity in session_transactions (AKA st_quantity) was interpolated incorrectly
- more exploration needed to determine the difference between orderquantity and totalordered columns in product_sales
- more exploration needed to determine if quantities in session_analytics align with product_sales values
