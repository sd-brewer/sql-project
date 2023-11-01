### starting_with_data.md
---
&nbsp;

# Question 1 : what percentage of unique visitors to the site make a purchase?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW qd1_visitor_conversion_rate AS (
	WITH transacting_visitor_count AS (
		SELECT
			COUNT(DISTINCT si.fullvisitorid)::FLOAT transacting_visitors
		FROM
			session_transactions st
		JOIN
			session_index si USING (sessionid)
		WHERE
			st.transactionoccured
	),

	all_visitor_count AS (
		SELECT
			COUNT(DISTINCT si.fullvisitorid)::FLOAT all_visitors
		FROM
			session_transactions st
		JOIN
			session_index si USING (sessionid)
	)

	SELECT
		((tvc.transacting_visitors / avc.all_visitors) * 100)::NUMERIC(10,2)
		AS conversion_rate_percentage
	FROM
		transacting_visitor_count tvc
	CROSS JOIN
		all_visitor_count avc
);

SELECT * FROM qd1_visitor_conversion_rate;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-pwW0O0SuI3xyhUl3zr90FLsDDwXaV2T)

### FINDINGS:
- **0.56%** of unique visitors (by fullvisitorid) end up making a purchase

&nbsp;

# Question 2 : What are the top channelgroupings for each country?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW qd2_top_channelgroupings_by_country AS (
	WITH ranked_channelgrouping_by_country AS (
		SELECT
			country,
			channelgrouping,
			COUNT(*) num_visits,
			RANK() OVER (
						PARTITION BY country
						ORDER BY COUNT(*) DESC
			) AS channelgrouping_rank
		FROM
			session_index si
		WHERE
			country IS NOT NULL
			AND channelgrouping IS NOT NULL
		GROUP BY
			country, channelgrouping
	)
	
	SELECT
		country,
		channelgrouping,
		num_visits
	FROM
		ranked_channelgrouping_by_country
	WHERE
		channelgrouping_rank = 1
	ORDER BY 
		num_visits DESC
);

SELECT * FROM qd2_top_channelgroupings_by_country;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-ljnsFmmlUJ3jAiaM22Yd245mYXDgjwa) ... 
![Query Output](https://drive.google.com/uc?export=view&id=1-eDssKyGIo9uu39psRTRZ2N0JMZVGaUx)

### FINDINGS:
- Pattern: top channelgroupings per country are most frequently 'Organic Search' or 'Direct'
- Countries with the highest volume of visits had 'Organic Search' as the top channelgrouping

&nbsp;

# Question 3 : What are the top 10 products based on unique visitor viewings?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW qd3_top_products_by_unique_viewers AS (
	SELECT
		pi.product,
		COUNT(DISTINCT si.fullvisitorid) AS num_unique_viewers
	FROM
		product_index pi
	JOIN
		session_transactions st USING (sku)
	JOIN
		session_index si USING (sessionid)
	GROUP BY
		pi.product
	ORDER BY
		num_unique_viewers DESC
	LIMIT 10
);

SELECT * FROM qd3_top_products_by_unique_viewers;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-r2tik6a52887ussJCntc5hWDgVlZI1Q)

### FINDINGS:
- Pattern: clothing category (hats and tops) make up the majority of the top viewed products

&nbsp;

# Question 4 : How many visits did the top 10 unique site visitors make?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW qd4_top_visitors_by_frequency AS (
	SELECT
		fullvisitorid,
		COUNT(*) AS num_visits
	FROM
		session_index
	GROUP BY
		fullvisitorid
	ORDER BY
		num_visits DESC
	LIMIT 10
);

SELECT * FROM qd4_top_visitors_by_frequency;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=1-sv4eLFcLZxKUwWz2teVI_HRV5AhY7Jb)

### FINDINGS:
- the top unique visitor visited the site 71 times

&nbsp;

# Question 5 : Do visitors who end up making a purchase spend more time on site?

### SQL Queries:
```SQL
CREATE OR REPLACE VIEW qd5_timeonsite_by_transactionoccurence AS (
	(
	SELECT
		AVG(si.timeonsite)::INT AS avg_timeonsite,
		'made purchase' AS visitor_type
	FROM
		session_index si
	JOIN
		session_transactions st
		ON si.sessionid = st.sessionid
		AND transactionoccured
	)
	UNION
	(
	SELECT
		AVG(si.timeonsite)::INT,
		'no purchase' AS visitor_type
	FROM
		session_index si
	JOIN
		session_transactions st
		ON si.sessionid = st.sessionid
		AND transactionoccured IS FALSE
	)
);

SELECT * FROM qd5_timeonsite_by_transactionoccurence;
```

**Query output**

![Query Output](https://drive.google.com/uc?export=view&id=105EpnS3J-w8DdPdlyeWU5N1QYqFtttQq)

### FINDINGS:
- visitors who ended up making a purchase spent, on average, more than **3x** the timeonsite compared to those who made no purchase

&nbsp;