#### Final-Project-Transforming-and-Analyzing-Data-with-SQL

&nbsp;

# Project/Goals
### Project Outline
- Get oriented with a PostgreSQL database and **analyze data** from a hypothetical e-commerce database.
- Transform raw **data into meaningful insights**.
### Personal Project Goals
- **Apply SQL fundamentals, GitHub, and Markdown** in a practical setting.
- **Create a portfolio piece** demonstrating data analysis skills for prospective employers.
- Develop an **understanding of database** tables and variables.
- **Normalize and streamline** the database, addressing data redundancies and establishing key constraints.

&nbsp;

# Process
### Step 1: Database Setup and Data Import
- **Created a PostgreSQL database** in pgAdmin4.
- **Set up tables** for each CSV file, determining variable data types through inspection in Excel.
- Created backup tables before beginning the analysis.
### Step 2: Data Cleaning and Normalization
- Completed **initial exploration to identify issues** needing cleaning, such as duplicate product SKUs, anomalies in product names, lack of primary/foreign keys, redundant columns, and aggregate sales data inconsistencies.
- **Began normalization**: finalized tables connected by product SKU and session ID numbers and established FK and PK constraints.
- Addressed inconsistencies and missing values to ensure data integrity.
### Step 3: Data Analysis Based on Questions
- Addressed five specific **questions outlined in the project brief**, ranging from user interactions to transaction revenue analysis.
- Utilized (partially) normalized tables to **extract insights**.
### Step 4: Exploratory Data Analysis
- **Developed an additional set of questions** to uncover insights, including visitor conversion rates, channel grouping trends, product viewing trends, visitor frequency ranges, and correlations between time on site and transactions.
### Step 5: Quality Assurance
- Executed QA processes to **identify and address remaining data quality issues**, highlighting discrepancies in product prices, quantities, and revenue.
### Step 6: ERD Generation
- Generated an Entity-Relationship Diagram (ERD) in pgAdmin to **showcase the database structure** and relationships.

&nbsp;

# Results
Analysis provided insights into visitor behavior and product performance. Key findings included:

- **Transaction data trends by location** (country, city), with the USA and San Francisco being top grossers.
- **Popularity trends in products and categories**, such as electronics and clothing.
- **Customer behavior patterns**, like higher site engagement correlating with purchases.
- These insights, in a real-world context, could guide marketing and product strategy development.

&nbsp;

# Challenges
### Lack of Data Dictionary
- Encountered challenges due to the **absence of database documentation**.
- Made numerous **assumptions about table columns** due to a lack of clarity and the presence of null values.
### Normalization Complexities
- Faced difficulties in **normalizing product SKUs** and names across multiple tables.
- **Ambiguities in session IDs** and inconsistencies in visitor data added to the complexity.
### Code Management and Organization
- Initially **followed several analysis tangents** without sufficient documentation.
- Ended up with an extensive amount of SQL files and code, complicating the **consolidation of findings**.

&nbsp;

# Future Goals
Given more time, I would focus on:

### Database Improvements
- **Testing assumptions** made during normalization.
- Creating a comprehensive **data dictionary**.
- **Filling in null values** with inferred data.
### Project File Enhancements
- Refining the **documentation of the data cleaning** process.
- **Including more visual elements**, like table outputs, to enhance reader understanding.
- Standardizing the **table naming scheme** for ease of understanding and reproducibility.
### Lessons for Next Projects
- Emphasize **ongoing documentation** and organization.
- **Utilize GitHub** for effective version control.
- Maintain a **project journal** for structured goal-setting and reflection.