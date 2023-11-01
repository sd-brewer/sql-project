### cleaning_data.md
---
&nbsp;

# Ecommerce Database Cleaning Documentation

### Introduction
This document details the processes involved in cleaning and updating the ecommerce database originally constructed from .csv files. The project aims to address key issues in the database structure and content, thereby enhancing its efficiency and usability for ecommerce analytics.

#### Primary Issues Addressed in Database Cleaning
- **Product SKU Duplicates**: Identification and resolution of duplicate product SKUs.
- **Product Name Anomalies**: Rectification of irregularities in product names.
- **Key Constraints**: Establishment of primary and foreign key relationships.
- **Redundant Data**: Elimination of unnecessary columns and redundant data.
- **Aggregate Data**: Creation of new tables for aggregate product and session data.
- **Value Anomalies**: Correction of irregularities in price and revenue values.
- **Null Columns**: Handling of columns with null values.

#### Document Limitations
- **Non-chronological Process**: The cleaning steps in `clean_data.md` are compiled from various data explorations and might lack chronological flow.
- **Omitted Exploratory Analysis**: For brevity, the exploratory data analysis informing these decisions is not included, which may affect understanding.

### Cleaning Procss Outline
1. **SECTION 1 - Cleaning All Sessions Table**: Focus on deduplication and data integrity of session records. Normalize productsku and productname unique to table.
2. **SECTION 2 - Normalizing and Aggregating Product Data**: Techniques for streamlining and standardizing product information and aggregating key data.
3. **SECTION 3 - Normalizing ID-Pairs and Related Data**: Strategies for structuring and associating ID-pair data effectively.
4. **SECTION 4 - Finalizing Table Names and Variables**: Standardization of table naming conventions and variable consistency.
5. **SECTION 5 - Setting PK and FK Constraints**: Implementation of primary and foreign key constraints for database integrity.



&nbsp;

## SECTION 1 : Begin cleaning all_sessions (and a bit of analytics)
### Purpose
- all_sessions contains most columns, use to get acquinted with database variables


&nbsp;

### 1.1 : misc data cleaning
- address "low-hanging-fruit", obvious column value anomalies with straightforward fixes

```SQL
-- change revenue and price datatypes before performing corrective operations
ALTER TABLE all_sessions
	ALTER COLUMN totaltransactionrevenue TYPE NUMERIC,
	ALTER COLUMN productprice TYPE NUMERIC,
	ALTER COLUMN productrevenue TYPE NUMERIC,
	ALTER COLUMN transactionrevenue TYPE NUMERIC;
```
```SQL
-- do the same to analytics variables
ALTER TABLE analytics
	ALTER COLUMN revenue TYPE NUMERIC,
	ALTER COLUMN unitprice TYPE NUMERIC;
```
```SQL
-- correct value anomalies
-- leave null values null, update not null with decimal value
UPDATE all_sessions
	SET
		totaltransactionrevenue =
			CASE
				WHEN totaltransactionrevenue IS NULL THEN NULL
				ELSE (totaltransactionrevenue / 1000000.0)::NUMERIC(10,2)
			END,
		productprice = 
			CASE
				WHEN productprice IS NULL THEN NULL
				ELSE (productprice / 1000000.0)::NUMERIC(10,2)
			END,
		productrevenue =
			CASE
				WHEN productrevenue IS NULL THEN NULL
				ELSE (productrevenue / 1000000.0)::NUMERIC(10,2)
			END,
		transactionrevenue = 
			CASE
				WHEN transactionrevenue IS NULL THEN NULL
				ELSE (transactionrevenue / 1000000.0)::NUMERIC(10,2)
			END;
```
```SQL
-- do the same in analytics
UPDATE analytics
	SET
		revenue =
			CASE
				WHEN revenue IS NULL THEN NULL
				ELSE (revenue / 1000000.0)::NUMERIC(10,2)
			END,
		unitprice = 
			CASE
				WHEN unitprice IS NULL THEN NULL
				ELSE (unitprice / 1000000.0)::NUMERIC(10,2)
			END;
```
```SQL
-- normalize fullvisitorid length (varies from 16 to 19)
-- some have leading zeros, some don't
-- confirmed this doesn't compromise variable during EDA
UPDATE all_sessions
	SET fullvisitorid = LPAD(fullvisitorid, 19, '0');
```
```SQL
-- normalize fullvisitorid length (varies from 15 - 19 digits)
UPDATE analytics
	SET fullvisitorid = LPAD(fullvisitorid, 19, '0');
```
```SQL
-- standardize null values in city and country column
UPDATE all_sessions
	SET 
		city =
			CASE
				WHEN city = 'not available in demo dataset' THEN NULL
				WHEN city = '(not set)' THEN NULL
				ELSE city
			END,
		country = 
			CASE
				WHEN country = '(not set)' THEN NULL
				ELSE country
			END;
```
```SQL
-- remove spaces in productsku (3 entries have a space in them)
UPDATE all_sessions
	SET
		productsku = REPLACE(productsku, ' ', '')
	WHERE
		productsku LIKE '% %';
```
```SQL
-- normalize v2productname (some have double spaces)
UPDATE all_sessions
	SET
		v2productname = REPLACE(TRIM(v2productname), '  ', ' ')
	WHERE
		v2productname LIKE ' %'
		OR v2productname LIKE '%  %';
```

&nbsp;

### 1.2 : Addressing productsku - v2productname anomalies
- some skus have multiple productnames, some productnames have multiple skus
- forewarning: this is where the tablename iterations start to get messy :')

```SQL
-- *** RENAME 'all_sessions' as 's' ***
CREATE TABLE s AS (
    SELECT * FROM all_sessions
);
```
```SQL
-- resolve skus that have multiple productnames
-- these are due to string inconsistencies

-- determine replacement names by selecting the longer name
WITH replacement_names AS (
	SELECT
		productsku,
		MAX(LENGTH(v2productname)) as max_length
	FROM
		all_sessions
	GROUP BY
		productsku
)
-- update all_sessions with longer name, replacing shorter name
UPDATE s s1
	SET
		v2productname = 
			(
			SELECT
				v2productname
			FROM
				s s2
			JOIN
				replacement_names rn ON s2.productsku = rn.productsku
			WHERE
				LENGTH(s2.v2productname) = rn.max_length
				AND s1.productsku = s2.productsku
			LIMIT 1
			);
```
```SQL
-- isolate the duplicate skus in all_sessions (AKA s)where the duplicate is not present in external tables
-- these skus must be collapsed manually to remain connected to external skus

CREATE VIEW unmatched_session_sku_duplicates AS (
    -- productnames inside of all_sessions (s) that have multiple skus
    WITH multiple_skus AS (
        SELECT
            v2productname,
            COUNT(DISTINCT productsku) as num_skus
        FROM
            s
        GROUP BY
            v2productname
        HAVING COUNT(DISTINCT productsku) > 1
    )
    -- list of all skus that exist outside of all_sessions
    , external_skus AS (
        SELECT DISTINCT sku FROM sales_report_backup
        UNION
        SELECT DISTINCT sku FROM sales_by_sku_backup
        UNION
        SELECT DISTINCT sku FROM products_backup
    )
    -- list of products that have multiple skus that exist only in all_sessions
    -- if a productname only appears once on this list, that means only one of its duplicate skus is in external tables
    , unmatched AS (
        SELECT DISTINCT v2productname, productsku
        FROM s
        JOIN multiple_skus ms USING (v2productname)
        LEFT JOIN external_skus ext ON ext.sku = s.productsku
        WHERE ext.sku IS NULL
    )
    -- include on the unmatched pairs from above list
    , single_occurrences AS (
        SELECT v2productname
        FROM unmatched
        GROUP BY v2productname
        HAVING COUNT(v2productname) = 1
    )
    -- retrieve the productsku for the single occurences
    , final_data AS (
        SELECT unmatched.v2productname, unmatched.productsku
        FROM unmatched
        JOIN single_occurrences ON unmatched.v2productname = single_occurrences.v2productname
    )

    -- list productnames with duplicate skus, the productsku that is exists only in all_sessions (internal), 
    -- and any duplicate skus that are common to all_sessions and external tables
    SELECT 
        final_data.v2productname, 
        final_data.productsku AS internal_sku,
        STRING_AGG(DISTINCT s.productsku, ', ') AS external_sku
    FROM 
        final_data 
    LEFT JOIN s ON final_data.v2productname = s.v2productname
    INNER JOIN external_skus ON s.productsku = external_skus.sku
    GROUP BY final_data.v2productname, final_data.productsku
    HAVING final_data.productsku != ANY(ARRAY_AGG(s.productsku))
    ORDER BY final_data.v2productname
);
-- determine skus to manually update
```
```SQL
-- *** RENAME 's' as 'sessions' ***
CREATE TABLE sessions AS (
	SELECT * FROM s
);
```
```SQL
-- manually update the skus that have only one of the duplicate skus in external tables
UPDATE sessions
	SET 
		productsku =
			CASE
				WHEN v2productname = 'Android 17oz Stainless Steel Sport Bottle' THEN 'GGOEADHH073999'
				WHEN v2productname = 'Android Stretch Fit Hat Black' THEN 'GGOEGAAX0761'
				WHEN v2productname = 'Electronics Accessory Pouch' THEN 'GGOEGBFC018799'
				WHEN v2productname = 'Google 22 oz Water Bottle' THEN 'GGOEGDHC018299'
				WHEN v2productname = 'Google Collapsible Pet Bowl' THEN 'GGOEGAAX0231'
				WHEN v2productname = 'Google Laptop and Cell Phone Stickers' THEN 'GGOEGFKQ020399'
				WHEN v2productname = 'Google Men''s 100% Cotton Short Sleeve Hero Tee Red' THEN 'GGOEGAAX0107'
				WHEN v2productname = 'Google Men''s Vintage Badge Tee Black' THEN 'GGOEGAAX0338'
				WHEN v2productname = 'Google Men''s Vintage Tank' THEN 'GGOEGAAX0355'
				WHEN v2productname = 'Google Power Bank' THEN 'GGOEGETB023799'
				WHEN v2productname = 'Google Sunglasses' THEN 'GGOEGAAX0037'
				WHEN v2productname = 'Google Toddler 1/4 Zip Fleece Pewter' THEN 'GGOEGAAX0657'
				WHEN v2productname = 'Google Trucker Hat' THEN 'GGOEGHPA002910'
				WHEN v2productname = 'Google Wool Heather Cap Heather/Navy' THEN 'GGOEGHPA003010'
				WHEN v2productname = 'Google Youth Short Sleeve T-shirt Royal Blue' THEN 'GGOEGAAX0683'
				WHEN v2productname = 'Leatherette Journal' THEN 'GGOEGOCB017499'
				WHEN v2productname = 'Maze Pen' THEN 'GGOEGGOA017399'
				WHEN v2productname = 'Recycled Mouse Pad' THEN 'GGOEGODR017799'
				WHEN v2productname = 'Recycled Paper Journal Set' THEN 'GGOEGAAX0081'
				WHEN v2productname = 'Waze Women''s Typography Short Sleeve Tee' THEN 'GGOEWXXX0834'
				WHEN v2productname = 'YouTube Men''s 3/4 Sleeve Henley' THEN 'GGOEGAAX0314'
				WHEN v2productname = 'YouTube Men''s Vintage Tee' THEN 'GGOEGAAX0346'
				ELSE productsku
			END;
```
```SQL
-- *** RENAME sessions as sessions_2 ***
CREATE TABLE sessions_2 AS (
    SELECT * FROM sessions
);
```

&nbsp;

### 1.3 : Collapse remaining sku duplicates in sessions_2

```SQL
-- find products remaining that still have multiple skus
WITH multiple_skus AS (
    SELECT
        v2productname,
        COUNT(DISTINCT productsku) as num_skus
    FROM
        sessions_2
    GROUP BY
        v2productname
    HAVING COUNT(DISTINCT productsku) > 1
) -- choose the longer sku to replace the shorter one
, replacement_skus AS (
	SELECT DISTINCT ON (v2productname)
		v2productname,
		productsku
	FROM
		sessions_2
	JOIN
		multiple_skus USING (v2productname)
	ORDER BY
		v2productname, LENGTH(productsku) DESC
)
-- update sessions_2 collapsing the remaining skus 
UPDATE sessions_2
	SET 
		productsku = replacement_skus.productsku
	FROM 
		replacement_skus
	WHERE 
		sessions_2.v2productname = replacement_skus.v2productname;
```
&nbsp;
### CONCLUSION
- started cleaning process in all_sessions (now sessions_2)
- productsku, productname data have been normalized inside of sessions_2

&nbsp;
--

## SECTION 2 : normalizing and aggregating product data
### Purpose
- unifying product data from: all_sessions, products, sales_report

&nbsp;

### 2.1 : normalize product names in product and sales report

```SQL
-- products and sales_report both have unique skus and productnames not found in the other
-- sales_by_sku contains only redundant data and can be ignored
UPDATE products
	SET
		name = REPLACE(TRIM(name), '  ', ' ')
	WHERE
		name LIKE ' %'
		OR name LIKE '%  %';
```
```SQL
UPDATE sales_report
	SET
		name = REPLACE(TRIM(name), '  ', ' ')
	WHERE
		name LIKE ' %'
		OR name LIKE '%  %';
```

&nbsp;

### 2.2 : Aggregating all product data in database

```SQL
-- aggregate product data from products and sales_report
---***RENAME products to products_3***
CREATE TABLE products_3 AS
	SELECT 
		COALESCE(p.sku, s.sku) AS sku,
		COALESCE(p.name, s.name) AS name,
		p.orderedquantity,
		s.totalordered,
		COALESCE(p.stocklevel, s.stocklevel) AS stocklevel,
		COALESCE(p.restockingleadtime, s.restockingleadtime) AS restockingleadtime,
		COALESCE(p.sentimentscore, s.sentimentscore) AS sentimentscore,
		COALESCE(p.sentimentmagnitude, s.sentimentmagnitude) AS sentimentmagnitude,
		-- Set ratio to NULL initially, calculate after aggregating duplicates
		NULL::NUMERIC(10,5) AS ratio 
	FROM products p
	FULL JOIN sales_report s ON p.sku = s.sku;
```
```SQL
-- extract skus from sessions_2
CREATE TEMP TABLE sessions_skus AS 
    SELECT DISTINCT productsku, AS sku FROM sessions_2
```
```SQL
-- determine common skus between products_3 and sessions_2
CREATE TEMP TABLE common_skus AS (
    SELECT
        p.sku,
        p.name
    FROM
        products_3 p
    JOIN
        sessions_skus s ON s.sku = p.sku
);
```
```SQL
-- ***RENAME products_3 to products_4***
CREATE TABLE products_4 AS SELECT * FROM products_3;
```
```SQL
-- replace the skus in products_3 (where there are multiple skus per product name) with those
-- also found in sessions_2
UPDATE products_4 p
	SET sku = cs.sku
	FROM common_skus cs
	WHERE p.name = cs.name AND cs.sku IS NOT NULL;
```

&nbsp;

### 2.3 : Collapse remaining skus that have multiple names

```SQL
-- find remaining duplicates
CREATE TEMP TABLE remaining_duplicates AS (
    SELECT
        name,
        COUNT(DISTINCT sku)
    FROM
        products_4
    GROUP BY
        name
    HAVING
        COUNT(DISTINCT sku) > 1
);
```
```SQL
-- ***RENAME products_4 to products_5***
-- create another backup
CREATE TABLE products_5 AS SELECT * FROM products_4
```
```SQL
-- use isolated skus to update duplicates
UPDATE products_5
	SET
		sku = replacement_skus.sku
	FROM
		replacement_skus
	WHERE
		products_5.name = replacement_skus.name;
-- all skus in aggregated products table (products 5) now have single product name
```
```SQL
--***RENAME products_5 to products_6***
-- aggregate sales data based on sku and product name
CREATE TABLE products_6 AS (
    SELECT
        sku,
        name,
        SUM(orderedquantity) AS orderedquantity,
        SUM(totalordered) AS totalordered,
        SUM(stocklevel) AS stocklevel,
        AVG(restockingleadtime)::INT AS restockingleadtime,
        AVG(sentimentscore)::NUMERIC(10,1) AS sentimentscore,
        AVG(sentimentmagnitude)::NUMERIC(10,1) AS sentimentmagnitude,
        SUM(ratio)::NUMERIC(10,5) AS ratio
    FROM
        products_5
    GROUP BY
        sku, name
)
```
```SQL
-- repopulate ratio data based on aggregated sales data
UPDATE products_6 
	SET ratio =
		totalordered::FLOAT / stocklevel::FLOAT
	WHERE stocklevel > 0 AND stocklevel IS NOT NULL
```

&nbsp;

### 2.4 : Resolving variance between sessions_2.v2productname and products_6.name
- normalizing productnames and skus to create 1:1 relationship

```SQL
-- create table that combined all skus, and preferences v2productname
CREATE TEMP TABLE all_products AS (
    SELECT DISTINCT
        COALESCE(v2productname, name) AS name,
        COALESCE(productsku, sku) AS sku
    FROM
        sessions_2
    FULL JOIN
        products_6 ON productsku = sku
    ORDER BY
        COALESCE(v2productname, name)
);
```
```SQL
-- manually update a few name anomalies
UPDATE all_products
	SET
		name =
			CASE
				WHEN name = 'You Tube Toddler Short Sleeve Tee Red' THEN 'YouTube Toddler Short Sleeve Tee Red'
				WHEN name = '7&quot; Dog Frisbee' THEN '"7" Dog Frisbee'
				ELSE name
			END;
```
```SQL
-- v2productnames only differ from v1names due to brandname prefix
-- create table that extracts brand name 
-- all_products_2 contains list of all skus, and their name and brand
CREATE TABLE all_products_2 AS (
    SELECT
        sku,
        CASE
            WHEN 
                name LIKE 'Google%'
                OR name LIKE 'Android%'
                OR name LIKE 'YouTube%'
                OR name LIKE 'Waze%'
                OR name LIKE 'Nest®%'
            THEN 
                TRIM(SUBSTRING(name FROM POSITION(' ' IN name)))
            ELSE name
        END AS product,
        CASE
            WHEN name LIKE 'Google%' THEN 'Google'
            WHEN name LIKE 'Android%' THEN 'Android'
            WHEN name LIKE 'YouTube%' THEN 'YouTube'
            WHEN name LIKE 'Waze%' THEN 'Waze'
            WHEN name LIKE 'Nest®%' THEN 'Nest®'
            ELSE NULL
        END AS brand
    FROM
        all_products
    ORDER BY sku
)
```
```SQL
-- rename products_6 as product_sales
-- contains sales data with sku as foreign key
CREATE TABLE product_sales AS (
    SELECT * FROM products_6
);
```
```SQL
-- remove product name from table (now contained in all_products)
ALTER TABLE product_sales
	DROP COLUMN name;
```
```SQL
-- add skuss from all_sessions
INSERT INTO product_sales (sku)
	SELECT
		all_products.sku
	FROM
		all_products
	LEFT JOIN
		product_sales USING (sku)
	WHERE 
		product_sales.sku IS NULL;
```
```SQL
-- create table all_products_3 to inherit sku and name data
-- adds categories extracted from name and informed by pagetitle
CREATE TABLE all_products_3 AS (
    SELECT
        sku,
        brand,
        product,
        CASE
            WHEN
                LOWER(product) LIKE '%baby%'
                OR LOWER(product) LIKE '%bib%'
                OR LOWER(product) LIKE '%infant%'
                OR LOWER(product) LIKE '%kid%'
                OR LOWER(product) LIKE '%youth%'
                OR LOWER(product) LIKE '%toddler%'
            THEN 'kids'
            WHEN
                LOWER(product) LIKE '%dog%'
                OR LOWER(product) LIKE '%pet%'
            THEN 'pets'
            WHEN
                LOWER(product) LIKE '% tee%'
                OR LOWER(product) LIKE '%tank%'
                OR LOWER(product) LIKE '%sweat%'
                OR LOWER(product) LIKE '%onesie%'
                OR LOWER(product) LIKE '%cardigan%'
                OR LOWER(product) LIKE '%pant%'
                OR LOWER(product) LIKE '%hood%'
                OR LOWER(product) LIKE '%polo%'
                OR LOWER(product) LIKE '%shirt%'
                OR LOWER(product) LIKE '%sleeve%'
                OR LOWER(product) LIKE '%men%'
                OR LOWER(product) LIKE '%women%'
                OR LOWER(product) LIKE '%sock%'
                OR LOWER(product) LIKE '%henley%'
                OR LOWER(product) LIKE '%fleece%'
            THEN 'clothing'
            WHEN
                LOWER(product) LIKE '%cap%'
                OR LOWER(product) LIKE '%hat%'
                OR LOWER(product) LIKE '%glasses%'
            THEN 'headwear'
            WHEN
                LOWER(product) LIKE '%nest%'
                OR LOWER(product) LIKE '%charger%'
                OR LOWER(product) LIKE '%flashlight%'
                OR LOWER(product) LIKE '%power%'
                OR LOWER(product) LIKE '%earbud%'
                OR LOWER(product) LIKE '%speaker%'
                OR LOWER(product) LIKE '%bluetooth%'
                OR LOWER(product) LIKE '%sound%'
                OR LOWER(product) LIKE '%thermostat%'
                OR LOWER(product) LIKE '%camera%'
                OR LOWER(product) LIKE '%alarm%'
            THEN 'electronics'
            WHEN
                LOWER(product) LIKE '%backpack%'
                OR LOWER(product) LIKE '%rucksack%'
                OR LOWER(product) LIKE '%tote%'
                OR LOWER(product) LIKE '%bag%'
                OR LOWER(product) LIKE '%duffel%'
                OR LOWER(product) LIKE '%pouch%'
                OR LOWER(product) LIKE '%shopper%'
            THEN 'bags'
            WHEN
                LOWER(product) LIKE '%pen%'
                OR LOWER(product) LIKE '%journal%'
                OR LOWER(product) LIKE '%book%'
                OR LOWER(product) LIKE '%badge%'
                OR LOWER(product) LIKE '%crayon%'
            THEN 'office'
            WHEN
                LOWER(product) LIKE '%umbrella%'
                OR LOWER(product) LIKE '%holder%'
                OR LOWER(product) LIKE '%organize%'
                OR LOWER(product) LIKE '%packing%'
                OR LOWER(product) LIKE '%luggage%'
                OR LOWER(product) LIKE '%mat%'
                OR LOWER(product) LIKE '%lunch%'
                OR LOWER(product) LIKE '%yoga%'
            THEN 'lifestyle'
            WHEN
                LOWER(product) LIKE '%sticker%'
                OR LOWER(product) LIKE '%decal%'
                OR LOWER(product) LIKE '%windup%'
                OR LOWER(product) LIKE '%ball%'
            THEN 'fun'
            WHEN
                LOWER(product) LIKE '%mount%'
                OR LOWER(product) LIKE '%pad%'
                OR LOWER(product) LIKE '%selfie%'
                OR LOWER(product) LIKE '%clip%'
                OR LOWER(product) LIKE '%device%'
                OR LOWER(product) LIKE '%clean%'
            THEN 'electronics_accessories'
            WHEN
                LOWER(product) LIKE '%mug%'
                OR LOWER(product) LIKE '%bottle%'
                OR LOWER(product) LIKE '%cup%'
                OR LOWER(product) LIKE '%tumbler%'
                OR LOWER(product) LIKE '%kooler%'
            THEN 'drinkware'
            WHEN
                LOWER(product) LIKE '%balm%'
                OR LOWER(product) LIKE '%sanitize%'
            THEN 'personal_care'
            WHEN
                LOWER(product) LIKE '%gift card%'
            THEN 'giftcard'
            ELSE NULL
        END AS category
    FROM
        all_products_2
    ORDER BY
        sku, product, brand
);
```
&nbsp;

### CONCLUSION 
- sku can now be used as a primary key, has 1:1 relationship with standardized product names
- all_products_3 table contains exhaustive list of product skus, names, brand, category
- product_sales table connects via sku and contains sales data

&nbsp;
---
## SECTION 3 : Normalizing ids
### Purpose
-  merging and aggregating fullvisitorid and visit id from sessions_2 (AKA all_sessions) and analysis tables


```SQL
-- merge fullvisitorid - visitid combinations
CREATE TEMP TABLE distinct_ids AS (
SELECT DISTINCT
	COALESCE(a.visitid, s.visitid) AS visitid,
    COALESCE(a.fullvisitorid, s.fullvisitorid) AS fullvisitorid
FROM 
    analytics a
FULL JOIN 
    sessions_2 s ON s.visitid = a.visitid
);
```
```SQL
-- assign session id to fullvisitorid-visitid pair
CREATE TABLE sessions_3 AS (
	SELECT
		LPAD(ROW_NUMBER() OVER (ORDER BY visitid, fullvisitorid)::TEXT, 6, '0') 
		AS sessionid,
		visitid,
		fullvisitorid,
		to_timestamp(visitid)::DATE AS date,
		to_timestamp(visitid)::TIME AS time
	FROM
		distinct_ids
);
```

&nbsp;

### CONCLUSION
- sessions_3 contains all unique pairs of fullvisitorid-visitid from database
- can be references by a unique sessionid

&nbsp;
---
## SECTION 4 : Finalizing table names and columns

&nbsp;

### 4.1 : CREATE TABLE session_transactions
- split from sessions_2 AKA all_sessions

```SQL
-- isolate transaction columns from all_sessions into new table
CREATE TABLE session_transactions AS (
	SELECT
		s3.sessionid,
		s2.productsku AS sku,
		CASE  -- set up transactions col for conversion to boolean
			WHEN s2.transactions IS NULL THEN 0
			WHEN s2.transactions = 1 THEN 1
		END AS transactionoccured,
		s2.totaltransactionrevenue,
		s2.productprice,
		s2.productquantity,
		s2.currencycode
	FROM
		sessions_2 s2
	JOIN
		sessions_3 s3 
		ON s2.fullvisitorid = s3.fullvisitorid
		AND s2.visitid = s3.visitid	
	ORDER BY
		sessionid
);
```
```SQL
-- change transactions column to boolean flag for presence of transactions
ALTER TABLE session_transactions
	ALTER COLUMN transactionoccured TYPE BOOLEAN USING
		CASE
			WHEN transactionoccured = 1 THEN TRUE
			ELSE FALSE
		END;
```
```SQL
-- fill in missing productquantity 
WITH product_quantities AS (
	SELECT
		sessionid,
		totaltransactionrevenue / productprice AS inferred_quantity
	FROM
		session_transactions
	WHERE
		transactionoccured
),
-- use floor rounding to round up when inferred quantity is over 0.3 of an item
-- assume that customer payed less than listed price vs more
-- assume that below 0.3 is due to tax or other transaction costs
rounded_quantities AS (
	SELECT
		sessionid,
		CASE
			WHEN inferred_quantity - FLOOR(inferred_quantity) >= 0.3 
				THEN CEIL(inferred_quantity)
			ELSE FLOOR(inferred_quantity)
		END AS inferred_quantity
	FROM
		product_quantities
)

-- update session_transactions with inferred quantity	
UPDATE session_transactions
	SET 
		productquantity =
			CASE
				WHEN transactionoccured THEN inferred_quantity
				ELSE 0
			END,
		totaltransactionrevenue =
			CASE
				WHEN transactionoccured THEN totaltransactionrevenue
				ELSE 0
			END
	FROM
		rounded_quantities
	WHERE
		session_transactions.sessionid = rounded_quantities.sessionid;
```

&nbsp;

### 4.2 : CREATE TABLE session_ecommerceinfo
- split from sessions_2 AKA all_sessions

```SQL
-- create session_ecommerceinfo table with remaining info from all_sessions
CREATE TABLE session_ecommerceinfo AS (
	SELECT
		s3.sessionid,
		s2.channelgrouping,
		s2.type,
		s2.pagetitle,
		s2.pagepathlevel1,
		s2.ecommerceaction_type,
		s2.ecommerceaction_step,
		s2.ecommerceaction_option
	FROM
		sessions_2 s2
	JOIN
		sessions_3 s3 
		ON s2.fullvisitorid = s3.fullvisitorid
		AND s2.visitid = s3.visitid
);
```

&nbsp;

### 4.3 : CREATE TABLE session_index
- created from sessions_3 (aggregated id-pairs)

```SQL
-- create new session_index table from sessions_3 that includes
-- channelgrouping, pageviews, timeonsite from all_sessions from analytics
CREATE TABLE session_index AS (
	SELECT DISTINCT
		s3.sessionid,
		s3.visitid,
		s3.fullvisitorid,
		COALESCE(a.channelgrouping, s2.channelgrouping) AS channelgrouping,
		COALESCE(a.pageviews, s2.pageviews) AS pageviews,
		COALESCE(a.timeonsite, s2.timeonsite) AS timeonsite,
		s3.date,
		TO_TIMESTAMP(s3.visitid)::TIME AT TIME ZONE 'UTC' AS time
	FROM
		sessions_3 s3
	LEFT JOIN
		analytics a ON a.fullvisitorid = s3.fullvisitorid AND a.visitid = s3.visitid
	LEFT JOIN
		sessions_2 s2 ON s2.fullvisitorid = s3.fullvisitorid AND s2.visitid = s3.visitid
	ORDER BY
		s3.visitid, s3.fullvisitorid
);
```
```SQL
-- CORRECTING OVERSIGHT
-- add city and country from sessions_2 (aka all_sessions)
CREATE TABLE session_index_temp AS (
	SELECT DISTINCT
		si.sessionid,
		si.visitid,
		si.fullvisitorid,
		s2.country,
		s2.city,
		si.channelgrouping,
		si.pageviews,
		si.timeonsite,
		si.date,
		si.starttime
	FROM
		session_index si
	FULL JOIN
		sessions_2 s2 ON s2.fullvisitorid = si.fullvisitorid AND s2.visitid = si.visitid
);
```
```SQL
-- replace temp table with original table name
CREATE TABLE session_index AS SELECT * FROM session_index_temp ORDER BY sessionid
```
```SQL
-- CORRECTING OVERSIGHT
-- check to see if sessionid is still unique
CREATE TEMP TABLE duplicate_sessionid AS (
	SELECT
		sessionid,
		COUNT(*)
	FROM
		session_index
	GROUP BY
		sessionid
	HAVING
		COUNT(*) > 1
);
-- 213 sessionids have been duplicated
```
```SQL
-- investigate the nature of the duplication
SELECT DISTINCT
	*
FROM
	session_index
JOIN
	duplicate_sessionid
USING
	(sessionid)
ORDER BY
	sessionid;
-- duplicate sessionids have different pageviews and timeonsite values
-- ex) sessionid = '012695', pageviews = '13', '15', timeonsite = '550', '1527'
-- assume that these values have been seperated but refer to the same ecommerce session
```
```SQL
-- aggregate these metrics grouping by sessionid
-- create new table that sums pageviews and timeonsite for each session
CREATE TABLE session_index_temp AS (
	SELECT
		sessionid,
		visitid,
		fullvisitorid,
		channelgrouping,
		SUM(pageviews) AS pageviews,
		SUM(timeonsite) AS timeonsite,
		date,
		time AS starttime
	FROM
		session_index
	GROUP BY
		sessionid,
		visitid,
		fullvisitorid,
		channelgrouping,
		date,
		time
);
```
```SQL
-- replace temp table with original tablename
DROP TABLE session_index;

CREATE TABLE session_index AS (
	SELECT * FROM session_index_temp
);
```
```SQL
-- update null timeonsite values
-- determined that they had all.sessions.time (timeonsite in milliseconds) less than 1000
UPDATE session_index
	SET timeonsite =
 		CASE
			WHEN timeonsite IS NULL THEN 0
			ELSE timeonsite
		END;
```

&nbsp;

### 4.4 : CREATE TABLE product_index
- created from all_products_3 (contains aggregated normalized product skus, names, categories)

```SQL
-- create new table for all_products to better illustrate its contents
CREATE TABLE product_index AS SELECT * FROM all_products_3;
```

&nbsp;

### 4.5 : CREATE TABLE session_analytics
- split from analytics table
- removes null columns and redundant columns now found in session_index

```SQL
CREATE TABLE session_analytics AS (
	SELECT
		si.sessionid,
		a.visitnumber,
		a.revenue,
		a.unitssold,
		a.unitprice,
		a.socialengagementtype,
		a.bounces
	FROM
		analytics a
	JOIN
		session_index si ON a.fullvisitorid = si.fullvisitorid AND a.visitid = si.visitid
	ORDER BY
		sessionid
);
```
&nbsp;

### CONCLUSION
- tables have been set up in their final iterations and are ready to have PK-FK constraints applied
- all redundant columns have been merged into index tables 

&nbsp;
---
## SECTION 5 : Assigning PK & FK constraints

&nbsp;

### 5.1 : Set primary keys sessionid and sku

```SQL
ALTER TABLE IF EXISTS public.product_index
    ADD PRIMARY KEY (sku);
```
```SQL
ALTER TABLE IF EXISTS public.session_index
    ADD PRIMARY KEY (sessionid);
```

&nbsp;

### 5.2 : Set foreign keys linking all other tables

```SQL
-- connect analytics to session_index using sessionid
ALTER TABLE IF EXISTS public.session_analytics
    ADD FOREIGN KEY (sessionid)
    REFERENCES public.session_index (sessionid) MATCH SIMPLE
    ON UPDATE NO ACTION
    ON DELETE NO ACTION
    NOT VALID;
```
```SQL
-- connect session_transactions to session_index using sessionid
ALTER TABLE IF EXISTS public.session_transactions
    ADD FOREIGN KEY (sessionid)
    REFERENCES public.session_index (sessionid) MATCH SIMPLE
    ON UPDATE NO ACTION
    ON DELETE NO ACTION
    NOT VALID;
```
```SQL
-- connect session_ecommerceinfo to session_index using sessionid
ALTER TABLE IF EXISTS public.session_ecommerceinfo
    ADD FOREIGN KEY (sessionid)
    REFERENCES public.session_index (sessionid) MATCH SIMPLE
    ON UPDATE NO ACTION
    ON DELETE NO ACTION
    NOT VALID;
```
```SQL
-- connect product_sales to product_index using sku
ALTER TABLE IF EXISTS public.product_sales
    ADD FOREIGN KEY (sku)
    REFERENCES public.product_index (sku) MATCH SIMPLE
    ON UPDATE NO ACTION
    ON DELETE NO ACTION
    NOT VALID;
```
```SQL
-- connect session_transactions to product_index using sku
ALTER TABLE IF EXISTS public.session_transactions
    ADD FOREIGN KEY (sku)
    REFERENCES public.product_index (sku) MATCH SIMPLE
    ON UPDATE NO ACTION
    ON DELETE NO ACTION
    NOT VALID;
```

&nbsp;

### CONCLUSION
- all tables are connected via PK-FK constraints