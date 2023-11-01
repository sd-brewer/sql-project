### starting_with_questions.md
---
&nbsp;
    
# Question 1: Which cities and countries have the highest level of transaction revenues on the site?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW q1_totaltransactionrevenue_ranked_by_country AS (
	SELECT
		si.country,
		SUM(st.totaltransactionrevenue) totaltransactionrevenue,
		RANK() OVER (ORDER BY SUM(st.totaltransactionrevenue) DESC
		) AS country_rank
	FROM
		session_index si
	JOIN
		session_transactions st
		ON si.sessionid = st.sessionid
		AND st.transactionoccured
		AND si.country IS NOT NULL	
	GROUP BY
		country
);

SELECT * FROM q1_totaltransactionrevenue_ranked_by_country;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-0m7wjB4nvTYGYjTdKBy27wQ4Knt9wnv)

&nbsp;

```SQL
CREATE OR REPLACE VIEW q1_totaltransactionrevenue_ranked_by_city AS (
	SELECT
		si.city,
		SUM(st.totaltransactionrevenue) totaltransactionrevenue,
		RANK() OVER (ORDER BY SUM(st.totaltransactionrevenue) DESC
		) AS city_rank
	FROM
		session_index si
	JOIN
		session_transactions st
		ON si.sessionid = st.sessionid
		AND st.transactionoccured
		AND si.city IS NOT NULL	
	GROUP BY
		city
);

SELECT * FROM q1_totaltransactionrevenue_ranked_by_city;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-2D8yUwaUYxbkpYGsBaEGRlLpf5JhaG9)

### Answer:
- Country with highest transaction revenue: **United States**
- City with highest transaction revenue: **San Francisco**

&nbsp;

# Question 2: What is the average number of products ordered from visitors in each city and country?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW q2_avg_orderquantity_by_country AS (
	SELECT
		si.country,
		AVG(st.productquantity)::INT AS avg_orderquantity_per_visitor
	FROM
		session_index si
	JOIN
		session_transactions st
		ON si.sessionid = st.sessionid
		AND st.transactionoccured
		AND si.country IS NOT NULL	
	GROUP BY
		si.country
	ORDER BY
		avg_orderquantity_per_visitor DESC
);

SELECT * FROM q2_avg_orderquantity_by_country;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-6t8U4WwlG5Kpw_n4SdrbEwSm7qO4T_J)

&nbsp;

```SQL
CREATE OR REPLACE VIEW q2_avg_orderquantity_by_city AS (
	SELECT
		si.city,
		AVG(st.productquantity)::INT AS avg_orderquantity_per_visitor
	FROM
		session_index si
	JOIN
		session_transactions st
		ON si.sessionid = st.sessionid
		AND st.transactionoccured
		AND si.city IS NOT NULL	
	GROUP BY
		si.city
	ORDER BY
		avg_orderquantity_per_visitor DESC
);

SELECT * FROM q2_avg_orderquantity_by_city;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-IQ2GzPlQPeh7xhIABtmv3sj8mYpyW39)

&nbsp;

### Answer:
- See query outputs

&nbsp;

# Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?


### SQL Queries:
```SQL
CREATE OR REPLACE VIEW q3_ranked_ordercategory_by_country AS (
	WITH ranked_country_categories AS (
		SELECT
			si.country,
			pi.category,
			COUNT(*) num_orders,
			DENSE_RANK() OVER (
						PARTITION BY si.country 
						ORDER BY COUNT(*) DESC
			) AS category_rank_by_country
		FROM
			session_index si
		JOIN
			session_transactions st
			ON si.sessionid = st.sessionid
			AND st.transactionoccured
			AND si.country IS NOT NULL
		JOIN
			product_index pi
			ON pi.sku = st.sku
		GROUP BY
			si.country, pi.category
	)
	
	SELECT
		country,
		category,
		num_orders
	FROM
		ranked_country_categories
	WHERE
		category_rank_by_country IN (1, 2, 3)
);

SELECT * FROM q3_ranked_ordercategory_by_country;
```

**Query output**


![Query Output](https://drive.google.com/uc?export=view&id=1-LdafakL9wXJy3A9CNUNMjSOGc9TheMP)

&nbsp;

```SQL
CREATE OR REPLACE VIEW q3_top_ordercategories_by_city AS (
	WITH ranked_city_categories AS (
		SELECT
			si.city,
			pi.category,
			COUNT(*) num_orders,
			DENSE_RANK() OVER (
						PARTITION BY si.city
						ORDER BY COUNT(*) DESC
			) AS category_rank_by_city
		FROM
			session_index si
		JOIN
			session_transactions st
			ON si.sessionid = st.sessionid
			AND st.transactionoccured
			AND si.country IS NOT NULL
			AND si.city IS NOT NULL
		JOIN
			product_index pi
			ON pi.sku = st.sku
		GROUP BY
			si.city, pi.category
	)

	SELECT
		city,
		category,
		num_orders
	FROM
		ranked_city_categories
	WHERE
		category_rank_by_city = 1
);

SELECT * FROM q3_top_ordercategories_by_city;
```

**Query output**


![Query Output](https://drive.google.com/uc?export=view&id=1-N0kGmfRzl5btdD0Q_9SRvKAOIIJT0KG)


### Answer:
- **clothing** and **electronics** are the top selling categories when compared by both country and city

&nbsp;

# Question 4: What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?


### SQL Queries:
```SQL
CREATE OR REPLACE VIEW q4_top_selling_products_by_country AS (
	WITH ranked_product_sales_by_country AS (
		SELECT
			si.country,
			pi.product,
			SUM(st.productquantity) AS quantity_sold_per_country,
			DENSE_RANK() OVER (
						PARTITION BY si.country
						ORDER BY SUM(st.productquantity) DESC
			) AS product_sales_rank_by_country
		FROM
			session_index si
		JOIN
			session_transactions st
			ON si.sessionid = st.sessionid
			AND st.transactionoccured
			AND si.country IS NOT NULL
		JOIN
			product_index pi
			ON st.sku = pi.sku
		GROUP BY
			si.country,
			pi.product
	)

	SELECT
		country,
		product,
		quantity_sold_per_country
	FROM
		ranked_product_sales_by_country
	WHERE
		product_sales_rank_by_country = 1
    ORDER BY
		quantity_sold_per_country DESC
);

SELECT * FROM q4_top_selling_products_by_country;
```

**Query output**


![Query Output](https://drive.google.com/uc?export=view&id=1-RNycNF6my6BwIiXrnSXteDKxeI5QtG-)

&nbsp;

```SQL
CREATE OR REPLACE VIEW q4_top_selling_products_by_city AS (
	WITH ranked_product_sales_by_city AS (
		SELECT
			si.city,
			pi.product,
			SUM(st.productquantity) AS quantity_sold_per_city,
			DENSE_RANK() OVER (
						PARTITION BY si.city
						ORDER BY SUM(st.productquantity) DESC
			) AS product_sales_rank_by_city
		FROM
			session_index si
		JOIN
			session_transactions st
			ON si.sessionid = st.sessionid
			AND st.transactionoccured
			AND si.city IS NOT NULL
		JOIN
			product_index pi
			ON st.sku = pi.sku
		GROUP BY
			si.city,
			pi.product
	)

	SELECT
		city,
		product,
		quantity_sold_per_city
	FROM
		ranked_product_sales_by_city
	WHERE
		product_sales_rank_by_city = 1
    ORDER BY
		quantity_sold_per_city DESC
);

SELECT * FROM q4_top_selling_products_by_city;
```

**Query output**


![Query Output](https://drive.google.com/uc?export=view&id=1-SGNNBM3cH9R9UQwaoh7mD3KP3pNauB4)


### Answer:
- products with the highest quantities sold appear to be inexpensive accessories (lip balm, stickers, shopping bag, etc)
- hoodies and t-shirts are the highest sold clothing items

&nbsp;

# Question 5: Can we summarize the impact of revenue generated from each city/country?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW q5_revenue_impact_by_country AS (
	SELECT
		si.country,
		SUM(st.totaltransactionrevenue) AS total_revenue,
		AVG(st.totaltransactionrevenue)::INT AS avg_revenue_per_order,
		AVG(st.productquantity)::INT AS avg_order_quantity,
		COUNT(DISTINCT st.sku) AS product_sales_diversity,
		tsp.product AS top_selling_product
	FROM
		session_index si
	JOIN
		session_transactions st
		ON si.sessionid = st.sessionid
		AND st.transactionoccured
		AND si.country IS NOT NULL
	JOIN
		q4_top_selling_products_by_country tsp
		ON tsp.country = si.country
	GROUP BY
		si.country, tsp.product
	ORDER BY
		total_revenue DESC
);

SELECT * FROM q5_revenue_impact_by_country;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-UMimUysIPCtK9JhgkAUv4jfmou-pcML)

&nbsp;

```SQL
CREATE OR REPLACE VIEW q5_revenue_impact_by_city AS (
	WITH city_aggregates AS (
		SELECT
			si.city,
			SUM(st.totaltransactionrevenue) AS total_revenue,
			AVG(st.totaltransactionrevenue)::INT AS avg_revenue_per_order,
			AVG(st.productquantity)::INT AS avg_order_quantity,
			COUNT(DISTINCT st.sku) AS product_sales_diversity
		FROM
			session_index si
		JOIN
			session_transactions st
			ON si.sessionid = st.sessionid
			AND st.transactionoccured
			AND si.city IS NOT NULL
		GROUP BY
			si.city
	)

	SELECT
		ca.city,
		ca.total_revenue,
		ca.avg_revenue_per_order,
		ca.avg_order_quantity,
		ca.product_sales_diversity,
		STRING_AGG(tsp.product, ' | ') AS top_selling_products
	FROM
		city_aggregates ca
	JOIN
		q4_top_selling_products_by_city tsp
		ON tsp.city = ca.city
	GROUP BY
		ca.city, ca.total_revenue,
		ca.avg_revenue_per_order,
		ca.avg_order_quantity,
		ca.product_sales_diversity
	ORDER BY
		ca.total_revenue DESC;
);

SELECT * FROM q5_revenue_impact_by_city;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-Zs7HxJrJiO04wTtanCmTLYl94gb9bsU)

&nbsp;

### Answer:
- see query outputs





