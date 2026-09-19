# Data Engineering — Question Bank

Extracted from `~/chat_classify/questions.csv` (team = `data-eng`).

- **2814** data-eng questions total, **2732** after de-duplicating (82 exact dupes removed)
- **2500** labeled `interview`, **232** `maybe-interview`
- **401** source conversations

Questions are grouped by the conversation they came from. `~` marks a `maybe-interview` question.

---

## Securing Healthcare Data with Node.js  (1)

- ~ What is CockroachDB?

## Highest Performer Query  (1)

- ~ How would you find the highest performer for each designation?

## Combined Document Chain  (3)

- ~ Given an employee table with id, name, and manager_id, write a query to get all people who report to a given manager, directly or indirectly.
- ~ How would you handle a hierarchy of 1000 levels in that query?
- ~ Given two tables, how would you find records in table 2 that are not in table 1?

## GCP Data Services Overview  (1)

- ~ What are the main GCP data services and their use cases?

## Data Engineer Interview GCP  (4)

- What are all GCP services and their use cases for a data engineer?
- What is a data warehouse?
- What are the types of data sources?
- What is Apache Airflow?

## Azure Data Engineer 28/mor  (29)

- What file types are supported in Azure?
- How do you handle wrong data in a CSV file in Azure?
- How do you drop records with wrong data in Azure?
- How can you avoid fetching wrong records in Azure?
- How do you avoid reading wrong data in Databricks?
- How do you update table B based on table A when both have id and repo_name?
- How do you update salary column with specific values for each emp_id in SQL?
- How do you shift salary data one row down in a large table?
- What is the difference between cache and persist in Spark?
- How does the cache work in Spark?
- How would you architect processing 1GB of data in Spark, including number of nodes and partitions?
- What is the difference between partition and bucket in Spark?
- When do we use liquid clustering?
- What is the Catalyst optimizer in Spark?
- What is the result of inner join between table A (1,1,2,null,null,3) and table B (1,1,2,null,3)?
- What is the result of left join between those tables?
- What is the result of right join between those tables?
- How can you trigger a pipeline in Data Factory when source data is updated?
- Can that trigger work for Databricks?
- Can Kafka also be used for triggering?
- How to drop duplicate records in Spark?
- What are Delta Lake tables?
- Where should Delta Lake tables be used?
- What advantages do Delta Lake tables provide?
- What specifically is Delta Lake commonly used for?
- What is Unity Catalog and its advantages?
- How many metastores can we create?
- What is the advantage of materialized views in Delta Lake and when not to use them?
- Which is more costly: materialized views or regular views?

## SQL Pivot Query  (1)

- ~ Write a SQL pivot query without hardcoding values, using dynamic SQL generation.

## PowerCenter vs IDA Comparison  (1)

- What are the differences between PowerCenter and Informatica Data Architecture (IDA)?

## Informatica Interview Prep  (1)

- How do you upload a non-structured dataset in Informatica?

## Python Azure Data Engineer  (27)

- What is Azure Synapse Analytics?
- How is Databricks integrated and what are its use cases in an NFT marketplace project?
- How do you implement a data lake for real-time data needs?
- How do you ensure data is up-to-date from Kafka or Event Hub, what sort of data is used, and what connectors are needed?
- What is the architecture for this in PySpark?
- How many components or objects are in this architecture?
- In a data warehouse in Synapse, how do you handle different schemas?
- What is the difference between all layers (bronze, silver, gold) if they have the same schema?
- In the bronze layer, what data format is used?
- In the data lake, what happens to the file?
- How do you handle upserts in this architecture?
- If another tool wants to see this data, how can we send it?
- If we don't have a connector, how can we send data?
- How would you host and expose an API?
- For data sources, how many different data sources do we ingest data from and where do they come from?
- How do you ensure data is stored quickly and performantly?
- Where was the final data warehouse located?
- What kind of data checks do you perform on data?
- In which layers are these checks performed?
- What happens if checks fail in the bronze layer?
- In Kafka, what messages are used?
- What logging mechanisms are used in Kafka?
- How do you manage Databricks notebooks and version control?
- When to use Databricks versus stored procedures?
- What challenges arise when dealing with terabytes of data?
- What is the silver layer in a data lake?
- What is the gold layer in a data lake?

## SQL Unique Pair Filtering  (3)

- ~ How would you write a SQL query to concatenate values (e.g., names) grouped by an ID?
- ~ How would you implement the same concatenation logic in Informatica?
- ~ Explain the Expression Transformation in Informatica in detail.

## Data Engineering Interview Prep  (5)

- What are ETL pipelines?
- What is the difference between a data warehouse and a data lake?
- What is the difference between ELT and ETL?
- What is multimodel data and how do you store it? Which database is used?
- What is HIPAA compliance in data engineering?

## Project Overview Explanation  (16)

- What is Azure Data Factory (ADF) and how does it work?
- How do you create an automated ETL pipeline in ADF?
- How do you debug and monitor data pipelines in ADF?
- How do you write and optimize SQL queries in Databricks?
- What are Delta Tables in Databricks and why are they useful?
- What is the difference between cache() and persist() in Databricks?
- What is Unity Catalog in Azure Databricks?
- What optimization techniques do you use in Databricks?
- How do you handle monitoring and logging in ADF and Databricks?
- What are your data quality best practices?
- How do you handle deployment from one environment to another in Databricks?
- How is Delta Lake better than Parquet?
- Explain the medallion data architecture (Bronze, Silver, Gold layers).
- How do you handle incremental data loading in ADF?
- What's your experience with Azure Synapse Analytics?
- How do you handle large-scale data ingestion from diverse sources?

## Duplicate Records Query  (1)

- ~ How do you identify duplicate records in a database?

## ETL with DBT Analytics  (10)

- How would you design an ETL pipeline for dashboards and analytics using Postgres and DBT, including normalization best practices?
- What tools and approaches would you recommend for building a data-driven analytics platform on messy Postgres data?
- How do you structure a DBT project for team collaboration and best practices?
- What normalization strategies are important when transforming messy data into actionable insights?
- How do you handle data quality and testing in a DBT pipeline?
- What is the role of DBT in modern data engineering versus traditional ETL?
- How do you optimize DBT models for performance with large volumes of data?
- Explain the concept of incremental models in DBT and when to use them.
- How do you manage dependencies and documentation within a DBT project?
- What are common pitfalls when using DBT for analytics and how do you avoid them?

## Golang Interview Prep  (3)

- What are the differences between PostgreSQL, ScyllaDB, and Bigtable?
- What is the difference between ScyllaDB and PostgreSQL?
- What are the challenges of migrating from ScyllaDB to Bigtable compared to Bigtable?

## Sr Eng Approach to Projects  (3)

- ~ How should one approach cleaning and storing data in a data lake?
- ~ What is predictive analysis?
- ~ How can you optimize costs and implement financial control over a data pipeline when dealing with extensive amounts of data?

## Spring Boot Basics  (2)

- Design an ETL pipeline for dashboards monitoring and analytics using Postgres to create actionable insights from messy data.
- What is dbt and how is it used in data engineering?

## Snowflake Technical Q&A  (10)

- ~ Explain Snowflake architecture and its key components.
- ~ What are the different types of tables in Snowflake?
- ~ How does Snowflake handle concurrency and scaling?
- ~ Explain Snowflake's storage and compute separation.
- ~ What is cloning in Snowflake and when would you use it?
- ~ Describe how Time Travel and Fail-safe work in Snowflake.
- ~ What are Snowflake stages and how do you use them?
- ~ Explain the difference between tasks and streams in Snowflake.
- ~ How do you optimize query performance in Snowflake?
- ~ What is data sharing in Snowflake and how is it implemented?

## Data Engineer Interview Prep  (11)

- What is FHIR (Fast Healthcare Interoperability Resources) in the context of data storage?
- What are the key things to take care of during data migration?
- How do you ensure optimizations are done after a data migration?
- What is antivity and how is it relevant to data engineering?
- How do you ensure data quality and handle a lot of data ingestions?
- How do you optimize costs in Azure for data engineering work?
- What are the transmission layers in a data pipeline and how do you design them?
- How do you design a transformation layer to keep it clean and avoid overlapping?
- When a transformation layer is already implemented, how do you ensure it is clean? (Give bullet points)
- How do you normalize data coming to PostgreSQL when the data warehouse is Redshift?
- How do you design a data sync mechanism between a transactional database and a data warehouse?

## SQL Query for Orders  (2)

- ~ Explain what this SQL query does: SELECT first_name, last_name, COUNT(*) AS total_orders, SUM(amount) AS total_amount_spent, SUM(CASE WHEN status = 'Pending' THEN 1 ELSE 0 END) AS pending_orders, SUM(CASE WHEN status = 'Delivered' THEN 1 ELSE 0 END) AS delivered_orders FROM orders WHERE status IN ('Pending', 'Delivered') GROUP BY first_name, last_name HAVING COUNT(*) > 0 ORDER BY total_orders DESC;
- ~ What is the difference between the two queries you provided?

## Data Engineering Interview Prep  (1)

- What is the difference between ETL and ELT?

## Azure Data Engineer Prep  (9)

- What is the difference between normalization and denormalization?
- How do star schema and snowflake schema work?
- How to perform joins optimally when joining millions of records that are updated frequently?
- What optimization techniques are used when joining multiple or single notebooks in Databricks?
- What is the pivot function in PySpark?
- How to implement dynamic pivoting and optimize it in Azure Synapse when the number of columns changes over time?
- How to determine and optimize highly complex queries in SQL?
- How to decide when to use Parquet?
- What is the difference between Parquet, Tabloo, and ORC? How to decide which one to use?

## GoLang | Nishant | Interview  (1)

- Write a SQL query for the second highest marks on a particular subject given tables for student, marks, and subjects.

## PySpark Word Vowel Processing  (1)

- ~ Write a PySpark program to process a text and count the number of vowels in each word.

## Golang Use Cases  (5)

- How to find the highest pay in each department?
- How to query nested JSON documents in a NoSQL database where department and employee are related?
- How to handle a million employees across 200 departments?
- Explain the MongoDB Aggregation Pipeline.
- How does the MongoDB Aggregation Pipeline work internally?

## Python DevOps Interview Prep  (1)

- How can big data achieve high observability?

## Data Modernisation Overview  (2)

- ~ What is data modernisation?
- ~ What are key things to remember during data migration?

## SQL Query for Employees  (3)

- Why did you use a LEFT JOIN instead of an INNER JOIN?
- There is a department that might not have any employees. How would you handle departments with no employees in the employee table?
- Assume you have a data pipeline in Python or PySpark that extracts records from SQL Server and pushes them to a data lake. It runs daily, incrementally, and is huge. Sometimes accessing records causes a bottleneck and takes a long time for certain dates. How can you improve and eliminate the bottleneck when you cannot see any errors?

## Data Engineering Interview Prep  (8)

- What is a delta table?
- What are the types of delta tables?
- What happens to files when you truncate an external delta table?
- What is stored in logs?
- How does time travel work in delta tables?
- What is Microsoft Fabric and why use Azure?
- Explain the architecture you used for a medical project.
- How would you plan a real-time application on the gold layer in Fabric with event-driven data?

## Terraform lookup function  (1)

- ~ What is Databricks in Azure?

## Azure Data Engineer Prep  (26)

- What are the optimizations in Teradata?
- What is the difference between Azure and AWS?
- How is data ingestion done? What are the techniques?
- Explain the Spark architecture and flow.
- What is data skewness?
- How do you handle data skewness?
- What is the difference between repartitioning and coalesce in PySpark?
- What are wide and narrow transformations in Spark? Which should you prefer?
- What are Spark window functions?
- What are analytical functions in Spark?
- Compare LAG and LEAD functions.
- What is the difference between ORDER BY and SORT BY in Spark?
- What are the types of joins in Spark?
- Explain cross join with an example: 5 rows in one table and 4 in another.
- What is data modeling?
- What is a star schema?
- What is intermediate data load?
- How do you implement data modeling in Databricks?
- Can MERGE be used in Databricks?
- What are the file formats supported in Databricks?
- What are the types of clusters in Databricks?
- What is the RANK function in Spark?
- What are the types of data pipelines in Azure Data Factory?
- How many types of data pipelines are there in ADF?
- How many types of workflows are there in Databricks?
- How do you create a workflow in Databricks?

## QA Automation Interview Guide  (1)

- What is the pandas library and what are DataFrames?

## Azure Data Engineer Prep  (1)

- What is the difference between table types in Azure data engineering contexts?

## Custom KPI Examples  (9)

- ~ What are custom KPIs?
- ~ What are the types of datasets used in KPIs?
- ~ What statistical tools are used for development?
- ~ What is Spark?
- ~ What are R packages?
- ~ How do you detect errors in data?
- ~ How do you solve the average system problem when comparing two systems?
- ~ Given two systems with the same schema, one has an average of 4.2 and the other 3.9, what could be the cause of the discrepancy?
- ~ What are data validation scenarios?

## DevOps Interview Preparation  (1)

- What is the role of a data engineer in the migration of complete frameworks from Terraform to Databricks?

## Go AWS Interview Prep  (3)

- Write migration scripts to bring initial SQL database (steps, no code).
- If migration is in order and I get a dirty error when starting the service, what should be done?
- What are RLS (Row Level Security) policies in PostgreSQL?

## Data Migration Strategies  (7)

- What are the different data migration strategies?
- How do you determine the best data migration strategy and come up with a plan?
- How would you migrate batch data from SQL Server to AWS Glue?
- How would you migrate streaming data?
- Should we freeze the timestamp during streaming migration?
- How do you point Kafka to the new platform?
- How do you ensure continuous validation for batch data?

## Python Snowflake AWS R Interview  (15)

- What is Snowflake and how do you connect to it using Python and AWS?
- How would you clean a CSV with unfiltered data, duplicate records, and null values using Python/Pandas?
- How do you handle data quality issues in Snowflake?
- How is sensitive data secured in Snowflake, and what steps protect data?
- How do you automate an ETL pipeline, specifically using Snowflake?
- Given an employee table (emp_id, emp_name, dept_id) and a department table (dept_id, dept_name), write SQL to return each employee name with their department name.
- What is AWS Glue and how do you create and manage a data workflow/catalog with it?
- How do you remove duplicates from a file using PySpark?
- Write a filter function to filter rows where the column 'score' has value > 100.
- How do you use PySpark for big data that is larger than memory?
- How do you protect sensitive information in an ETL pipeline?
- How do you use AWS Lambda in ETL?
- Given an employee table (id, name, dept_id) and a department table (id, name), write SQL to get employee names with their department names.
- In a DataFrame with product_id and sales columns, write a Pandas function to get total and average sales per product.
- What are the challenges in Teradata to Databricks migrations?

## Python Snowflake Interview Prep  (4)

- How to set up a Snowflake database with Python and AWS?
- How does Snowflake work?
- Design an ETL pipeline to read daily sales CSV files from S3, transform to calculate total quantity and revenue per product, and load into Snowflake.
- How would you handle large data in an ETL pipeline? What improvements would you make?

## Data Analyst Interview Prep  (3)

- What is GA4?
- What is Databricks?
- What are the components of GA4?

## Python Backend Dev Interview  (14)

- What is Apache Spark?
- What is the difference between streaming and batch streaming in Spark?
- What are the parameters in the spark-submit command?
- What are the deploy modes for PySpark?
- How to find an application running in YARN?
- How are stages and tasks determined in Spark?
- What is an example of a narrow transformation in Spark?
- What is an example of an action in Spark?
- How to get which process is running on executors and drivers in a Spark program?
- What are memory issues in PySpark?
- How to handle memory issues in PySpark?
- How to tune a Spark job and use the history server?
- What is the rule to set the number of executors in Spark?
- How to cast all 50 columns to string in Spark? Write the code.

## Semantic Modeling in Fabric  (1)

- ~ What are the types of semantic models in Microsoft Fabric?

## Shipping Data Analysis  (9)

- What are the disadvantages of storing data in databases?
- What are the next considerations for denormalization?
- Why is moving to the cloud from on-premises better, even when it costs more?
- What are your thoughts on CI/CD for batch data?
- What is a factless fact table?
- In Databricks, I have several tables with 5 million records each and I need to union them. What issues can arise and how to solve them?
- How to design the best infrastructure for Databricks?
- How to monitor Databricks infrastructure?
- What are the challenges of moving to Databricks with Unity Catalog, including pre-tag and multiple joins?

## Complex PySpark Transformation  (6)

- ~ How do you design a complex PySpark transformation involving three joins with varying data sizes?
- ~ In what order should you join large, medium, and small datasets in PySpark?
- ~ How do you handle join results that increase data volume instead of reducing it?
- ~ What is the best approach for filtering after joining multiple mappings with multiple conditions?
- ~ How do you design ETL stages when the data size surprises you mid-join?
- ~ What are the challenges when dealing with balances in an ETL pipeline?

## GoLang AWS Interview Prep  (1)

- Write a SQL query to find student details with the second highest total marks, given tables: Student (id, name, total_marks), Subject (id, name), and Enrollment (student_id, subject_id, marks).

## Data Engineering Interview Prep  (17)

- What is Azure Fabric?
- What are some go-to tools for a complete data pipeline?
- We are building a data warehouse using Databricks. How should we design the data lake architecture?
- Explain the bronze-silver-gold layer architecture for a data lake.
- I have a table in the gold layer called dim_property that is updated frequently. Every time there is a change, I want to send that transaction through an event bus as a topic so my subscriber can know what changes occurred (update, delete, insert). Define an end-to-end data pipeline and architecture to do this.
- What is the difference between managed tables and external tables?
- What happens when we truncate an external table?
- What happens when we truncate and load an external table built in Unity Catalog?
- Will the file also be deleted when truncating an external table?
- If I am loading fact and dimension tables, which should I load first?
- What happens when a dimension table gets truncated? Can we reload it or reload the fact table?
- What is data fabric?
- Compare data fabric and Databricks.
- Describe a project built on data fabric, its architecture and flow.
- What is dimensional modeling?
- I want to load 50-60 tables from SQL Server on-premises into a bronze layer, perform incremental loads, and use a data lakehouse as the destination. Describe the data pipeline for this.
- What if we have big data?

## Tech Profile Introduction  (13)

- What are the different types of data sources that a data engineer works on?
- What is an analytical database?
- In GA4, what is the exact process to visualize data in Power BI and Tableau?
- Where should data cleaning be done in analytical tools?
- Why is data cleaning important?
- In Google Analytics (GA4), what are the different types of integration models?
- What are attribution models?
- What is the difference between engagement rate and bounce rate?
- When creating a dataset for visualization using BigQuery data, what are the steps of conversion?
- If there is a discrepancy between Google Analytics and Power BI, what could be possible reasons?
- What is the difference between Universal Analytics (UA) and GA4?
- What things should I consider in GA4 that are not in UA?
- In Power BI, when should I use a KPI card?

## Dictionary Merge in Python  (10)

- ~ How do you delete duplicate rows in a Spark DataFrame where id and name are duplicate but address is different?
- ~ What is the lifecycle of a Spark application?
- ~ What are stages in Spark?
- ~ What are accumulators in Spark?
- ~ How would you design an ETL pipeline reading from S3, transforming, and writing to Redshift?
- ~ What is the difference between coalesce and repartition in Spark?
- ~ What are some optimizations for Spark jobs?
- ~ How would you handle a data file with millions of rows where the id has the same value throughout?
- ~ How do you get the maximum value from an entire table across all columns in SQL without using GREATEST?
- ~ How do you find all elements in table A that are not in table B in SQL without using NOT IN?

## Star Schema Overview  (12)

- What is data staging in ETL?
- What is an Operational Data Store (ODS)?
- Describe an Azure-based architecture for a data engineer for an Ethereum-based marketplace.
- What reporting tools are used in this architecture?
- How do you validate data quality in the final layer using Databricks or manually?
- What is Unity Catalog (UC)?
- What is serverless SQL?
- What is the current Databricks Runtime (DBR) version?
- What types of clusters are available in Databricks?
- What is the difference between star schema and snowflake schema?
- What are the latest features of Databricks?
- What are the latest optimization features in Databricks?

## DevOps Interview Prep  (10)

- What is Microsoft Fabric?
- What are the main tools in Microsoft Fabric?
- What is the difference between OneLake and Lakehouse?
- Where should you use a Lakehouse?
- How would you describe using OneLake in your project, even if you haven't actually used it?
- What are the basic steps for designing a pipeline in Microsoft Fabric?
- What is the difference between a pipeline in Microsoft Fabric and a pipeline in Databricks?
- What is Azure Synapse?
- In which kind of project would you use Azure Synapse?
- What are the different data sources in Azure Synapse?

## DevOps Engineer Interview Prep  (4)

- What is the difference between Snowflake and Databricks?
- What ETL tools do you use?
- What are the stages in Snowflake?
- What is the data retention period in Snowflake?

## Incremental Load and Upsert  (14)

- ~ What is incremental load and upsert?
- ~ If redundancy occurs, which function should be used to remove it?
- ~ What is the difference between delta and catalog?
- ~ Which format does Delta Lake store data in?
- ~ What is a partition?
- ~ What is the difference between a view and a volume in Unity Catalog?
- ~ What is a volume in Unity Catalog?
- ~ What is Azure Key Vault and how is it used?
- ~ What are some challenges with Azure Data Factory (ADF) and their solutions?
- ~ What are ADF triggers?
- ~ What is thread polling?
- ~ What is thread pooling?
- ~ How do you handle log maintenance in Databricks?
- ~ If I have a data pipeline in ADF, how can I send a notification email to a list of users when the pipeline starts?

## GCP Data Architecture  (15)

- ~ How is data pushed to Pub/Sub?
- ~ How does data move from Pub/Sub to BigQuery?
- ~ Why use Kafka for event-based architecture?
- ~ What are cheaper alternatives to Kafka?
- ~ What is the use of Airflow in data pipelines?
- ~ For data ingestion from a new source, should you use Kafka or a read replica? Which one to choose and why?
- ~ How would you design the architecture to consume 100 tables into BigQuery?
- ~ Compare Kafka and database table for data ingestion.
- ~ What serverless services can be used for data processing?
- ~ What is the clustering strategy in BigQuery?
- ~ How does costing work in BigQuery?
- ~ What can be controlled to save costs in BigQuery?
- ~ Explain data cleansing and data quality in GCP.
- ~ How would you handle a very large table with historical data issues and duplicates, considering data frequency of use?
- ~ How to determine when a customer is at their home address using GPS logs and customer data?

## Data Engineering Intro Summary  (1)

- What is your experience with Postgres and SQL?

## Databricks Workspace Setup Guide  (11)

- ~ How do you set up a new Databricks workspace, and what are the key considerations for security and compliance?
- ~ How do you monitor performance and operations in Databricks?
- ~ How do you manage user and group permissions in Databricks? What value does it add, and how would you configure it?
- ~ What are the steps to set up the Databricks CLI (dbx cli)?
- ~ What is the difference between Delta Live Tables and Delta tables?
- ~ How does Delta Live Tables integrate with other features of Databricks?
- ~ How can you use SQL and Python to debug data pipelines in Databricks?
- ~ How do you set up monitoring in Databricks?
- ~ What CloudWatch metrics could you capture for Databricks?
- ~ How do you handle rollback in Databricks?
- ~ What are the considerations for data governance and regulatory compliance in Databricks?

## Resume Based Q&A  (3)

- What are the requirements for an ETL pipeline?
- How do you manage tables in Databricks?
- What are the properties of managed tables in Databricks?

## AWS Data Engineer Intro  (13)

- What AWS services are commonly used by data engineers?
- What is Apache Iceberg?
- What are the advantages of Apache Iceberg over traditional partitioning?
- How do you migrate data from Parquet to Iceberg?
- When would you use a graph database over a relational database?
- How do you implement vector search using AWS services?
- How would you design a serverless data pipeline to load data into S3?
- How would you design a cost-effective data lake architecture that balances performance and cost?
- How do you troubleshoot ETL job performance when processing data from RDS to Redshift?
- How would you design a Redis cache layer for high traffic?
- What are common Redis-related tools used in data engineering?
- How do you optimize a PostgreSQL query that has multiple joins?
- How do you implement change data capture (CDC) from PostgreSQL to Amazon Redshift?

## GCP Scaling for Million Users  (1)

- ~ Explain Snowflake.

## AWS Data Engineer Interview  (10)

- How do you implement CDC (Change Data Capture) from PostgreSQL to a data warehouse?
- How does PostgreSQL handle concurrency control?
- What is BRIN index in PostgreSQL?
- What are the steps to implement Redis caching?
- Explain Redis schemas/data structures.
- What are the advantages of using Apache Spark and data lakes?
- How does Redis handle high availability?
- What is the function of time travel in Apache Iceberg?
- How does Apache Iceberg optimize metadata operations?
- How would you implement a concurrent, fault-tolerant pipeline in Golang?

## Databricks Admin Interview Prep  (13)

- What is a reconciliation framework?
- How do you set up Databricks for scalability, custom roles, autoscaling, and what parameters should be considered?
- What is the difference between serverless and dedicated clusters in Databricks in terms of cost and when would you not choose serverless?
- How do you implement RBAC, row-level and column-level security in Databricks?
- How does data mesh work with business domains owning data products and contracts controlling access; how would you implement this in Databricks?
- How do you set up access control and SSO in Databricks?
- How do you troubleshoot a critical production issue in Databricks?
- What is z-ordering and how does it improve performance?
- What is data vacuuming in Databricks?
- As a Databricks admin, if multiple nodes are underperforming compared to others, how do you debug and fix?
- How do you perform recovery in Databricks?
- What is the difference between interactive and job clusters?
- How do you manage costs in Databricks clusters?

## Aurora PG vs AlloyDB PG  (1)

- ~ What are the differences between Amazon Aurora PostgreSQL and Google Cloud AlloyDB for PostgreSQL?

## Data Engineer Interview Prep  (34)

- What is Neo4j?
- What is dbt?
- What are the best practices for dbt?
- How do you handle sensitive data in dbt?
- How do you handle encryption and decryption in dbt?
- Is base64 encryption?
- Why use base64?
- How do you document models in dbt?
- How do you manage end-to-end lineage in dbt?
- How do you debug a complex pipeline that takes too much time?
- What happens when there is a schema change in upstream data, and how do you handle it in dbt?
- What specific tools does dbt offer for handling schema changes?
- How do you detect and guard against hidden problems in dbt?
- How do you use the `dbt run` command?
- What does the 'impostation' function refer to in dbt?
- How would you break down a complex JSON file for ingestion and transformation?
- How do you add another ingestion source to an existing pipeline, and how should you structure it for dbt?
- What database does dbt use?
- What would change if the database is Snowflake?
- What functions in dbt need to change for Snowflake compatibility?
- How do you offload processing between transformation and environment when using Snowflake?
- How do you set up dbt development environment?
- How do you use dbt macros?
- How complex can dbt macros be?
- What criteria do you use for building a dbt macro?
- What is the approach for choosing when to use Snowflake?
- What types of Snowflake warehouses are there?
- How would you design an ETL pipeline dumping data from Salesforce to Snowflake, handling large volumes?
- What are slowly changing dimensions?
- How do you design a pipeline for change data capture (CDC)?
- How does CDC work for a particular record?
- Compare data vault models and dimensional models.
- What are data marts and how do you create one?
- How do you handle integration from Snowflake, including cloning and mapping?

## Airflow DAG Overview  (1)

- ~ What is an Airflow DAG?

## ML Interview Prep Guide  (1)

- How was data ingestion done in that project?

## Tech Terms Breakdown  (1)

- What are Data Factory, Data Lake, and Data Warehouse?

## AWS Data Engineer Prep  (3)

- How would you design a system to connect to an external system using REST APIs (each costing 10 cents), ingest all data into a database, and keep it synced with high accuracy?
- How do you manage cost and user requirements in a system where you are a middleman for API calls and pay per call?
- How can webhooks be integrated into this system?

## Data Engineer Interview Prep  (10)

- Write a SQL query to find the top 3 performers in each designation based on salary/age ratio.
- Explain Spark optimization techniques and their use cases.
- What are common memory issues in Spark?
- How to handle out-of-memory errors in Spark?
- What is the difference between partition and repartition in Spark?
- What is the output when you do partition in Spark?
- What is the output when using df.write.partitionBy('somefield') in PySpark?
- How to install and run PySpark in VS Code step by step?
- Using PySpark, extract the date from the trade_date field in the given CSV data.
- Why is partitioning directories useful in Spark?

## Python Django Interview Prep  (3)

- What is N+1 query problem and how to optimize it?
- How to optimize SQL graph queries?
- Design a simple database schema with tables and relationships.

## Data Engineering Interview Prep  (4)

- List Azure services used in data engineering.
- How can you containerize an environment to run and test code for an agent?
- The agent generates code for thousands of users; how do you ensure the containerized app is scalable?
- How do you handle multiple concurrent users using agents?

## Interview Prep and Tech Stack  (2)

- Explain the logic behind concurrent features.
- How did you use Airflow in your project?

## Databricks Admin Azure Prep  (14)

- What is the integration between AWS and Databricks?
- What is platform administration in Databricks?
- What is Databricks administration exposure?
- What is Okta AD and how does it integrate with Databricks?
- What is disaster recovery in Databricks?
- What are common issues in Terraform for Databricks?
- What are DAP templates in Databricks?
- How do you handle disaster recovery for Databricks?
- What is the maintenance of volumes in Databricks?
- What are the types of cluster policies in Databricks?
- What instance types are used for cluster policies in Databricks?
- How do you manage costs for dashboards and Databricks?
- When you create AD groups, how do you enable users to start using Databricks?
- How do you create specific policies for a user in Databricks?

## Senior DBA Role Overview  (8)

- How to assign permissions to groups using Unity Catalog?
- How to implement Always On high availability?
- How to normalize a schema in a healthcare context (IQVIA)?
- What is performance tuning for databases?
- How to design database architecture and scaling?
- How to handle slow-running queries in a high-transaction system?
- What tools and strategies are used for disaster recovery?
- How to handle recovery models and disaster recovery logs?

## Snowflake Interview Questions  (1)

- What are some common Snowflake interview questions?

## 3PM-Manoj (GenAI) Deetsdigital  (1)

- Given a directory of large CSV files, find a solution to aggregate and summarize the files handling strings. What packages would be used and what is the flow?

## 7 Year Dev Summary  (1)

- How do you unload data from Snowflake to an external stage like S3 or Azure Blob Storage? What are the best practices?

## AI ML GenAI Data  (1)

- How do you handle late-arriving data in a Snowflake ETL pipeline?

## Real-Time N-th Retrieval Issues  (1)

- Review this code and highlight the issues and the solutions for a system that processes key-value pairs from multiple concurrent input streams and provides the ability to retrieve the N-th smallest value.

## 8PM - Screening  (7)

- In Databricks, when you create a feature branch, how are changes isolated to that user or accessible by others?
- When merging a feature branch to the developer branch in Databricks, does it impact existing jobs?
- In Databricks, jobs point to notebooks with tags. When merging to main and developer branches, how do changes merge to main since job names are different?
- In Databricks, what are the different types of job triggers (scheduled, table update, etc.)? When moving to production, how do you carry the five tables and their trigger configurations?
- If a job's JSON configuration does not include trigger information, what alternative can be used?
- How do you deploy notebooks, jobs, tables, and pipelines via CI/CD in Databricks?
- A view has 100 lines of code with schema and catalog info specific to dev. When deploying to prod, it should not use dev-specific schema. What are the best practices for deploying views in Databricks?

## Data Engineer Interview Prep  (20)

- What are the features in Snowflake to be used?
- How to unload data in Snowflake?
- What is the command to unload data in Snowflake?
- Do we need stages between unloading, like internal or external storage?
- What is Snowpipe?
- What is the use of secondary roles in Snowflake?
- What are stored procedures in Snowflake?
- How to create a stored procedure and capture errors to send email to a distribution list?
- Can we incorporate CDC in this?
- How to send email in Snowflake?
- How to schedule a unique fixture for teams in SQL?
- What tools are used for data modeling?
- What are DBS files?
- What is Erwin and its uses?
- How to handle late arrival data in real-time data loading into Snowflake?
- How to use dbt for late arrival data?
- What is the no-code approach?
- What are hybrid tables and iceberg tables in Snowflake?
- What are ETL rules?
- What are ETL tools?

## AI/ML Interview Introduction  (4)

- What is a use case for Kafka?
- What is a broker in Kafka?
- What is partition and replication factor in Kafka?
- How is monitoring implemented in Kafka?

## Real-Time N-th Value Retrieval  (1)

- ~ Design a system that processes key-value pairs from multiple concurrent input streams and provides the ability to retrieve the N-th smallest value.

## MLOps interview prep  (2)

- What are projects and domains in AWS Data Zones?
- How do you maintain scalability and reliability in a pipeline?

## AI/ML interview prep  (1)

- What is a data validation framework and how is it used?

## Interview intro preparation  (18)

- How will you identify missing indexes?
- How will you analyze the execution plan?
- What do we mean by statistics here?
- How does SQL Server identify that statistics are outdated?
- What are the types of backup?
- What is a differential backup?
- How does SQL Server identify which changes have been made after a full backup?
- How can we see the extents in DCM pages?
- How can we demonstrate the transaction file?
- What is your understanding of point-in-time recovery?
- When have you done point-in-time recovery?
- What is log shipping?
- What are the log shipping modes?
- Explain multiple secondary servers in log shipping.
- What is the difference between standby and no recovery mode?
- What are your suggestions for configuring a 10 TB database?
- How to log backup a 10 TB database?
- What is tempdb and how do you set it up?

## Monkey Patching in Python  (3)

- ~ What are the differences between fillna, dropna, and replace for handling missing values in pandas?
- ~ What is the difference between ROW_NUMBER and LIMIT in SQL?
- ~ What is a Common Table Expression (CTE)?

## Interview Prep and Tech Stack  (32)

- In a SQL query using * with joins, how can I optimize it?
- If you found that some indexes are missing, how would you create an index?
- In an execution plan, what is the key factor for a query?
- What is the difference between execution plan and explain plan?
- What is the less command for excessive page I/O?
- How would you troubleshoot excessive page I/O?
- What is less contention?
- What is ledge contention?
- What is log contention?
- Is it important to manage log contention or can we avoid it?
- What is a spinlock in SQL Server?
- How does a spinlock impact performance?
- How does SQL Server handle lock acceleration?
- What is lock acceleration?
- How does SQL Server handle the connection log file internally?
- Is it important to manage the connection log file or can we avoid it?
- What is virtual log file (VLF) fragmentation?
- How does SQL Server maintain VLFs?
- What is a trace event?
- What is a trace event notification?
- What are extended events?
- What is SQL Server Profiler trace?
- What is a deadlock situation?
- What key events should be tracked for CPU usage?
- How can you track long-running queries using extended events?
- How many types of backups are there and what is the plan for each?
- How would you take a backup of only specific tables?
- What is the maximum backup size you have taken in your projects and how long did it take to back up 80GB?
- How can I reduce backup time?
- What is a differential change map?
- What are performance counters?
- What is auto state?

## Nishant Databricks, Final Round  (12)

- What is the difference between narrow and wide transformations in Spark?
- How does Spark handle data shuffling and how can it be optimized?
- What is a DAG in Spark and why is it important?
- How do you see access control in Unity Catalog in Databricks?
- What is the difference between RANK, DENSE_RANK, and ROW_NUMBER in SQL?
- How do you optimize complex SQL queries?
- What is the difference between EXPLAIN and EXPLAIN ANALYZE in SQL?
- What are window functions in SQL?
- What is a slowly changing dimension (SCD) and how do you handle it?
- What is list comprehension in Python and what are its benefits?
- I have a table ingested from GCS. How would you design for parallelizing the creation of a Delta table?
- If I am making external API calls to fill in columns in real time, how can I parallelize them for speed?

## SQL Query Optimization  (9)

- How would you optimize a SQL query?
- What is VACUUM in the context of Delta Lake or databases?
- You have a Delta table with 1 billion records. Provide all strategies to optimize performance, prioritized.
- How do you implement role-level security (RLS)?
- How do you implement row-level security (RLS)?
- What are the best practices for migrating table data?
- What else can you do to achieve zero-downtime migration?
- Compare Tableau and Power BI.
- What are the uses of Redshift?

## Term Explanations Made Simple  (5)

- What is Apache Iceberg and why is it used? What are the alternatives? Provide a technical example.
- What is Redis and why is it used? What are the alternatives? Provide a technical example.
- What are Teradata, Databricks, and Data Lakes? Explain with examples.
- Explain PostgreSQL with a technical example.
- Explain Graph/Vector Databases with a technical example.

## Data Engineering Interview Prep  (19)

- How do you set up a cluster for data processing, decide compute resources, and manage processing, especially on AWS?
- What kind of transformations are applied in the silver and gold layers of a data lake?
- How are CI/CD pipelines handled in data engineering?
- How do you run data pipelines using Apache Airflow?
- Write PySpark code to transform a table with duplicate IDs and multiple keys into a deduplicated table with aggregated values.
- How can you use maps in PySpark for data transformations?
- Write PySpark code to deduplicate records based on ID and keep the latest update timestamp.
- How does the withColumn method work in PySpark when casting a column to timestamp?
- What alternatives are there to using withColumn in PySpark?
- Why would you use withColumn over select in PySpark?
- Write PySpark code to aggregate values from col2 by distinct col1, concatenating them with pipes.
- What is the difference between partitionBy and groupBy in PySpark?
- When can you replace groupBy with partitionBy in PySpark?
- How do you monitor performance and observability using Spark UI and Spark Sight?
- What key metrics should you look at in the Spark UI?
- How do you determine the optimal number of shuffle partitions in Spark?
- What are some techniques for optimizing PySpark performance besides partitioning?
- How has Delta Lake improved data lake management and what benefits has it brought to the industry?
- What is Unity Catalog in Databricks?

## Databricks Overview and Use Cases  (2)

- ~ What is the ELK stack?
- ~ What is the difference between RabbitMQ and AWS SQS?

## Interview preparation JD tech stack  (8)

- What is a data lake?
- Compare block container and data lake.
- When should you use a block container versus a data lake?
- What is the difference between a data lake and a Delta Lake?
- Compare blob storage, data lake, and Delta Lake.
- What are best practices to optimize the performance of a table reload and query optimization?
- If memory and CPU are at 85% consumption and memory is still reserved, how does this work as a bottleneck?
- How can I optimize performance using a notebook?

## SQL Consecutive Login Query  (4)

- ~ Write a SQL query to find consecutive logins for users.
- ~ How can you make the query concise and optimized?
- ~ Explain the logic of the query.
- ~ How would you handle the case when there is no end date?

## Interview prep guidelines  (13)

- How to configure Kafka for streaming data and give a use case?
- What are the differences between consumer and producer in Kafka?
- What is a DAG in Airflow?
- How is Java used in data engineering?
- What is the difference between RDD and DataFrame in Spark?
- Given table A with 5 records and table B with 4 records with 2 matching records, what are the total records in a full outer join and a cross join?
- Write a query to find names that appear more than once in an employee table with columns name and email.
- Which is better: count(*) or count(name)?
- What are query optimization techniques?
- What is the difference between a CTE and a query?
- What is the difference between a CTE and a subquery?
- How are materialized views analogous to CTEs?
- How do you define the scope of a CTE?

## Structured OCR Output Tips  (1)

- ~ How is a page processed in an OCR pipeline?

## Airflow vs Spark  (1)

- ~ What is Apache Airflow and Apache Spark?

## Interview Prep and Overview  (1)

- After extracting features, do we store those in a database?

## Interview answer example  (4)

- Give a brief introduction to Python, AWS, and ETL.
- What is the best approach for partitioning a data pipeline?
- What do you do when a data pipeline breaks down?
- How do you handle slow query optimization?

## Interview prep overview  (7)

- Do you have any experience in ETL technologies, Informatica, EDF, and pipelines?
- How do you create a DataFrame in PySpark and how do you insert data into a table?
- How do you partition data?
- How do you create functions and libraries in Databricks or pandas?
- How do you parallelize using PySpark in data processing?
- Give me one use case where you did end-to-end PySpark, including requirement and output.
- Detail the PySpark notebook you did for the Cignet project.

## Interview preparation JD queries  (13)

- What are the advantages of Databricks over Teradata?
- Which AWS services are relevant for data engineering?
- What is the difference between PostgreSQL and SQL?
- How do you handle column mismatch and schema invalid errors during migration?
- How do you handle a failed union operation with multiple joins?
- How do you utilize Spark in this context?
- What are the challenges when migrating from Teradata to Databricks while ensuring optimization?
- How do you approach validating the schema of a large dataset?
- How do you handle data skewness and ensure data efficiency?
- How do you handle timeout issues in partitioned large datasets where the job completes but data is missing?
- What is the approach to create a Databricks job that reads data from multiple sources?
- What is Databricks Runtime?
- What is the latest version of Databricks Runtime?

## GenAI Interview Prep  (12)

- What are constraints in SQL?
- What is the difference between WHERE and HAVING?
- What is the difference between UNION and UNION ALL?
- What is the difference between RANK and DENSE_RANK?
- What is the difference between TRUNCATE and DROP?
- How to read a specific sheet from an Excel file into a DataFrame?
- How to drop duplicates from a DataFrame?
- What are the arguments of drop_duplicates?
- What command shows the bottom of a pandas DataFrame?
- How to find missing elements in a DataFrame?
- How to handle duplicates in a DataFrame?
- How to drop NA without removing the row?

## Interview prep intro and Q&A  (3)

- How do you design a pipeline using OCR? Describe the steps.
- What is modularization in Databricks?
- What is the difference between TF-IDF for Spark and embeddings?

## Backend engineering interview prep  (4)

- What is the difference between LEFT JOIN and INNER JOIN in PostgreSQL?
- What are common data types supported by PostgreSQL?
- How do you perform data manipulation in PostgreSQL?
- What is a subquery in PostgreSQL?

## Hyperparameters in migration  (1)

- ~ What are hyperparameters involved in data migration?

## Simplifying DevOps Experience  (6)

- Have you done migrations to Microsoft Fabric ensuring lineage tracking?
- How do you handle CI/CD pipelines, version control, and pipelines in Microsoft Fabric?
- How do you monitor Fabric workload pipeline executions?
- What are your strategies for data governance?
- How do you manage roles and security across domains?
- How do you tune KQL queries?

## Excitement answer generation  (12)

- How would you set up an S3 bucket for data storage?
- Describe your experience using AWS Lambda for data processing.
- How would you design a cost-efficient and scalable data solution using AWS services?
- How would you approach data storage optimization in Google Cloud for a scalable data solution?
- What challenges might you face when migrating an on-premises data solution to AWS?
- Can you explain the difference between conceptual, logical, and physical data models?
- Describe a situation where you had to revise a data model due to changing business needs.
- How do you ensure data quality in real-time data processing?
- How would you design a comprehensive data quality management strategy for an organization?
- Write a simple Python script to transform data from one format to another and explain the steps.
- What are some advanced features of PySpark that you have used in projects?
- How would you architect a PySpark-based solution for real-time data processing?

## Data engineering interview prep  (8)

- What core data science problems could be solved in a blockchain project?
- What frameworks are used in data engineering?
- In a live stream transaction, how do you determine the size of a Kafka cluster?
- What is the difference between Python data processing and PySpark?
- How many workers can be used in ETL and what does it depend on?
- If a transaction is missed through Kafka, how do you recover it?
- What machine learning algorithms are used in data pipelines?
- What are the challenges faced in data pipelines?

## Introduction and project description  (4)

- What is data streaming?
- How is Python used in ELT?
- Compare Apache Kafka and Spark Structured Streaming.
- How does Snowflake integrate with Azure and AWS?

## Data engineering interview prep  (17)

- What is the difference between SQL and PostgreSQL?
- What are the advantages and disadvantages of SQL and PostgreSQL over each other?
- How do you optimize a slow PostgreSQL query?
- How do you migrate data from SQL to PostgreSQL? What tools and mapping techniques are used?
- What is the difference between star schema and snowflake schema, and when should each be used?
- How do you troubleshoot deadlocks in SQL Server?
- What is a trigger in Azure Data Factory?
- How do you handle incremental load in Azure Data Factory?
- How do you implement CI/CD for Azure Data Factory?
- How do you optimize a Spark job in Databricks?
- How do you enforce data quality in a data pipeline?
- What is schema evolution and how do you handle changes in Delta Lake / Databricks?
- How do you handle sensitive data in a data pipeline?
- If an ETL pipeline is failing in production, how do you debug and fix it?
- How do you support both batch and streaming data pipelines?
- How do you optimize a data pipeline in Azure Data Factory that is taking a long time?
- If a Databricks job that processes transactions keeps failing, how do you ensure it does not happen again?

## Pankaj - 7PM Data Engineer  (11)

- Explain SmartPath.
- Explain PySpark in the context of your project.
- What did you do for optimization of a PySpark query?
- How did partitioning help in joins?
- How do you join large tables without a memory overflow error?
- What databases have you used with Spark?
- If you have a file and a table, how many partitions will Spark use?
- How do you set upper bound and lower bound using Spark?
- How did you use DBT in your projects?
- Why did you use DBT and not Spring Boot or another framework?
- Describe a job orchestration project you worked on.

## Project explanation and intro  (1)

- Can you describe a complex ETL pipeline you've built using PySpark, detailing the challenges you faced and how you optimized its performance?

## Interview introduction and projects  (10)

- Explain dbt and describe a project you did with it.
- Why did you choose dbt over PySpark?
- In Airflow, if a job sequence has failed, what steps should you take?
- How do you rerun a DAG in Airflow?
- How do you optimize Spark jobs?
- When a Spark job fails with out-of-memory (OOM) and overflow errors, how do you debug and resolve it?
- What other common errors have you encountered with Spark and how did you handle them?
- Describe a project you have done with Kafka.
- How do events get produced to Kafka?
- Who or what consumes the events from Kafka?

## Data engineering interview prep  (9)

- What kind of work do you do in Databricks?
- What is the fundamental problem in this query?
- What is the mistake in this query? (after line 3)
- How can you make this query more efficient?
- How many times are we reading the files?
- If the files are really large, how many times is the join being done?
- Why use row_number() instead of rank()?
- Would it not work with rank() or dense_rank()?
- What is the difference between a normal cluster and a job cluster in Databricks?

## Data engineering interview prep  (17)

- Assume you have many different upstream sources to be consumed and you need to transform those data points like master data. The transaction database is PostgreSQL. Build a pipeline for upstream and what considerations should be aware of?
- In ADF copy activity, if it is failing because of data mismatch, will it progress for all the rest of the rows or just one?
- You identify data mismatch between source and destination and copy activity is failing. How to handle?
- How to handle data mismatches in copy activity when destination is Postgres? Should we use copy activity or something else?
- How to perform transformations in ADF? Does it happen in copy activity?
- How to have transformations in ADF?
- In SQL you have 3 schemas and 5 tables as source. You want to bring it to ADF without hardcoded values. What transformations and schema mapping approach should be used?
- What activities need to be done one by one in ADF for this whole scenario?
- How to achieve the 1st and 2nd step in detail?
- In PostgreSQL, everything is in lower case. If that is the destination, how to transform the source schema to match?
- You want to migrate one table to PostgreSQL from SQL database. How to map the schema in the pipeline?
- In ADF, any database, say you have connection pooling issue in Postgres where application is reaching threshold. How to investigate and solve?
- How does pgBouncer work with connection pooling? How would it solve the problem?
- There is a follow up issue: idle connections are causing issues. How to handle idle connections?
- You notice during a certain period in a day the application is slow, database response is slow only at certain times. How to investigate that?
- You have identified there is a deadlock, something is blocking on UI. How to handle this scenario from a data engineering side?
- Given the SQL query: SELECT d.department_name, ROUND(AVG(e.salary), 2) AS average_salary FROM Employees e JOIN Departments d ON e.department_id = d.department_id GROUP BY d.department_name HAVING COUNT(e.employee_id) > 2 ORDER BY average_salary DESC; how would you write the same in PySpark?

## Create FastAPI GET endpoint  (4)

- ~ Explain what a data pipeline is and why orchestration is important for managing these pipelines.
- ~ How is Python used in data pipeline orchestration and what tools or libraries are used for this?
- ~ What are common challenges faced in data pipeline orchestration with Python?
- ~ How do you handle version control and collaboration when multiple teams are working on a data pipeline codebase?

## 2:45PM - Mahesh Data Engineer  (2)

- What are the Azure services used for data engineering?
- What is the role of cloud training in data engineering?

## Interview prep guide  (23)

- What is DataOps?
- What visualization tools are used in data engineering?
- What scenarios could occur in data engineering when fixing a problem in production, and how would you fix them?
- In a live production environment, how do you handle breaking things and user complaints?
- What is DBR?
- What are the different DBR versions?
- How do you optimize a Databricks environment?
- What batching strategies are used in Databricks?
- In real-time streaming, what batching strategy should be used?
- How do you fine-tune streaming jobs?
- What is throttling?
- Why, when, and where should throttling be done?
- How do you control the number of records in a stream?
- If a notebook is receiving messages from a broker, how do you control the number of records?
- Can you control the number of files and records in Databricks?
- What happens if you delete the checkpoint in Databricks?
- What are common problems in Databricks?
- What are common problems in AWS Glue?
- What is a Databricks cluster?
- When you create a notebook, how many types of clusters are there in Databricks?
- Where do Databricks jobs run?
- Can Databricks be used in a non-serverless architecture?
- How do you fix out-of-memory errors in Databricks jobs?

## AWS Python Interview Prep  (5)

- Design a fault-resilient system for processing high-volume data (1000 events per minute) with large JSON payloads to be loaded into an ERP. List all components and explain why each is used. Then design a low-volume system.
- In a high-volume data system, where can Kafka and Lambda be used? If the payload is huge, what cost-effective alternatives exist?
- What are the key differentiators between the high-volume and low-volume system designs you provided?
- How do you make a high-volume system idempotent?
- If a parent sends the same booking 5 times, how do we detect and handle such duplicate requests?

## Simple data analytics answers  (15)

- What is Composer?
- What is Dataflow?
- What is Dataproc?
- What are operators in Composer?
- How does dbt work with Airflow?
- How would you design a pipeline using Airflow to run dbt models when a file arrives in GCS, handling delays?
- What is BigQuery?
- What is the architecture of BigQuery?
- Write a BigQuery query to get students absent yesterday from a student table with data for the last 2 years.
- What are dbt materializations?
- What is the structure of a dbt project?
- What are dbt snapshots?
- Describe a streaming data pipeline using Dataflow.
- What is PySpark?
- What are optimization techniques in PySpark?

## Interview prep assistance  (10)

- What is the difference between batch and streaming data in a medallion architecture?
- What was done in a Teradata to Databricks migration project?
- What are the key tools for a data engineer?
- Describe some projects you worked on as a senior data engineer.
- What did you do at Sonara.ai and Imbio as a data engineer?
- Explain the Chainlake project and its complexity as a data engineer.
- What was the volume of data in Chainlake?
- How would you design a real-time data pipeline with multiple data sources for scalability?
- How do you handle schema evolution in Delta Lake or medallion architecture?
- In medallion architecture, the bronze layer is close to the source. How do you define schema in a data lakehouse, what triggers changes in schema, and how does it propagate to silver and gold?

## Interview question format  (1)

- What is the Data Engineer data stack?

## Interview preparation assistance  (16)

- What are the key tools and libraries for a Data Architect role and their typical use cases?
- Describe a Teradata to Databricks migration project: key steps, tech stack, and considerations.
- How do you choose between multiple databases for a given use case?
- Design a data pipeline for a company like Juniper Square to ingest data from OLTP systems into a data warehouse and serve BI dashboards to customers.
- Where should the silver layer be placed in a data pipeline?
- Where should delta tables for the silver layer be saved?
- Describe the gold layer in a data pipeline: structure, purpose, and implementation details.
- For a data warehouse like Redshift, do we need an additional layer for delta tables, or is the silver layer sufficient?
- What kind of schema should be used for a data pipeline (e.g., star schema, snowflake)?
- Is BigQuery suitable for handling a large volume of BI requests? If not, what are the alternatives?
- How can you ensure data isolation in a multi-tenant data architecture so that customers only see their own data?
- In a data pipeline, is everything orchestrated by the orchestrator? What is the orchestrator's role?
- If you have a Python script to extract data from MySQL to S3, what orchestration tool would you use and why?
- How do you handle schema changes (e.g., column addition) in a data pipeline?
- What monitoring and alerting strategies would you implement for a data pipeline?
- What actions would you take if no alerts are triggered but no data is loaded?

## New chat  (1)

- What are the key differences between data engineering concepts, presented in a table with bullet points?

## Interview question answers  (1)

- What is the difference between IN and EXISTS in SQL?

## AI and Data Science intro  (1)

- What are the key concepts and features of data science?

## Manoj - 2:30PM  (10)

- What is DBT (Data Build Tool)?
- What is Jinja and where is it used? Give an example.
- What data quality checks are important? What problems can arise for data quality and how to tackle them?
- Give me syntax to deduplicate in Python and SQL for a customer table.
- What would be the difference if we use DENSE_RANK instead of ROW_NUMBER in this query?
- What is an incremental model in dbt?
- In the context of Redshift, how does incremental load work? Explain.
- Can incremental load be related to table partitioning? Will it help? What is table partitioning?
- What difference will it have if we don't partition and then do incremental load vs if we do partition and then incremental load?
- Give me a query for a rolling average for the last 3 days. We have a table with product_id, sales_amount, and date.

## Mahesh - 2PM  (23)

- What is the data size for real-time and batch processing?
- In batch ingestion, do we use SCD1 or SCD2?
- What is the use case of SCD1?
- Design an ETL pipeline for batch data of 15 TB.
- Apart from medallion architecture, what other approaches are there?
- How do you handle API data ingestion?
- How to handle schema drift?
- Which tool for ETL?
- What is a dead letter queue?
- What is the maximum number of joins you have used?
- Explain star schema modeling.
- Design a star schema for sales data with region, customer, product, sales, and sales_item, and provide analytical queries.
- What tools are used for schema design?
- Do we need a fact table for customer and product?
- What is a surrogate key in dimension tables?
- What is the difference between data lake and data warehouse, and when to use which?
- Can we use both data lake and data warehouse in a project?
- How to design a real-time analytics dashboard on Azure to process application logs? What architecture and tools?
- Same scenario on AWS: design a real-time analytics dashboard.
- How to handle Lambda timeout in an Airflow orchestrated pipeline?
- What partitioning strategy for Parquet files for patient vitals data to optimize queries?
- What if queries are sliced by minute or hour? How does partitioning strategy change?
- How to estimate cost of a data pipeline for big data?

## Manoj - 3PM  (8)

- How did you use DBT and BigQuery together?
- Are you aware of BigQuery snapshot?
- What is table clone in BigQuery?
- How do you create a table using 'SELECT * FROM table_clone' and how is it different?
- How is insert overwrite different in DBT?
- Performance-wise, which is better?
- How did you use DBT, Data Factory, BigQuery, and Airflow all together in a single project?
- Where did you use Altrix here?

## Handling data issues  (8)

- ~ How do you handle data issues in a pipeline (e.g., data mismatch, corruption) so the pipeline does not fail?
- ~ What built-in features does Databricks provide to handle data issues?
- ~ Given a user status (on/off) from login/logout events, how would you generate session ID, start time, end time, and duration using PySpark?
- ~ What is the difference between data lake, data mart, and data warehouse?
- ~ How do you maintain historical data in a data pipeline?
- ~ What are the different types of slowly changing dimensions (SCD)?
- ~ What are the differences between Redshift and S3 for data storage/querying?
- ~ An AWS Glue job that previously took a few minutes is now taking hours. How would you debug this and what could be the possible issues?

## Interview intro and Q&A  (6)

- How did we manage data ingestion in the Homecure project?
- What is Microsoft Purview?
- What role does Microsoft Purview play in a data role?
- How did you implement lineage classification in Microsoft Purview?
- How did you design the silver/gold architecture to support performance in projects?
- Can you tell me the approach for multi-cloud integration (e.g., Azure, GCP) when designing data solutions?

## Explain PubSub in GCP  (1)

- ~ What is PubSub in GCP and what problem does it solve?

## Interview preparation answers  (1)

- What is your experience with data modeling using Cosmos DB?

## Group anagrams at scale  (4)

- ~ In AWS, how would you design data storage for a microservices architecture needing OLTP, analytics, and ML feature serving? Justify store choices, schemas, and data movement.
- ~ How do you enforce data contracts across services to prevent schema drift from breaking downstream analytics and features?
- ~ How would you model bronze/silver/gold Delta tables and choose partition columns, apply Z-ordering, manage OPTIMIZE and VACUUM?
- ~ Which columns would you partition vs. Z-order on for streaming upserts, and how would you handle concurrent merge conflicts?

## Tabular comparison Hadoop Spark  (13)

- ~ What is the difference between Hadoop and Spark?
- ~ When loading a 10 GB file into Spark and Hadoop, what happens in the backend first in Hadoop?
- ~ How does a large file get chunked in Hadoop?
- ~ What is data skewness in Spark and why does it happen?
- ~ How do you handle data skewness in Spark?
- ~ What are the types of join strategies in Spark?
- ~ What information can you get from the Spark UI?
- ~ If one particular run is taking a lot of time, how do you debug and fix it in Spark?
- ~ What is Adaptive Query Execution (AQE) in Databricks?
- ~ What is the difference between Spark Streaming and Autoloader?
- ~ What is time travel in Databricks and how does it work?
- ~ How does the OPTIMIZE command work in Databricks?
- ~ How does the VACUUM command work in Databricks?

## Spreetail overview  (6)

- ~ How would you optimize a database to query a billion records in milliseconds without optimizing based on the query?
- ~ How do you resolve the problem of most queries hitting a particular set of data, causing performance issues?
- ~ How do you solve a hot shard problem when data is grouped in a way that one shard receives most queries?
- ~ How does Dagster work?
- ~ Compare Dagster and Airflow in a table and explain when to choose which.
- ~ What is Trino?

## Data engineer profile summary  (8)

- Describe your experience with AWS Lambda for data processing.
- Describe a scenario where you optimized a cloud-based data source for better performance.
- How would you ensure compliance and security in a multi-cloud environment for data integration?
- What is the difference between conceptual, logical, and physical data models?
- What are the best practices for ensuring data consistency and reliability in data models?
- Describe a time when you had to resolve a major data quality issue.
- How would you design a comprehensive data quality management strategy in an organization?
- What advanced features of PySpark have you used in your projects?

## Concurrent streaming data meaning  (2)

- ~ What is the meaning of handling concurrent streaming data?
- ~ How do you design optimized indexes for bidding data lookups in MongoDB?

## Profile introduction creation  (2)

- Describe your role in Go-based services that ingest real-time NFT transaction data via WebSocket, including building Kafka consumer modules for high-frequency bidding streams and caching auction metrics in Redis.
- Design a more complex problem involving real-time data ingestion from WebSockets, Kafka processing, and Redis caching for high-frequency NFT auctions, focusing on concurrency, error handling, and schema evolution.

## Interview prep intro tech stack  (1)

- Write a SQL query to get data from two tables.

## Interview introduction and project  (3)

- What is the format and what are the sources of claims information?
- How is tabular data stored?
- How is information stored in RDBMS used and retrieved?

## AWS vs Data Eng Comparison  (14)

- What kind of Spark optimization can we apply for large datasets?
- How are these techniques usable in PySpark?
- Are Glue and EMR usable for these optimizations?
- What is the difference between DataFrames and Dynamic Frames?
- What is data wrangling?
- What Python libraries are used for data wrangling?
- What type of validation should you do when migrating data from one source to another, before transformations, and what quality checks?
- What are some data quality tools, including AWS native ones?
- What is Deequ?
- How do you handle schema drift with heterogeneous data sources?
- You have PySpark jobs in production producing intermittent data corruption issues. How do you find the solution?
- How to ensure reliability and performance for AWS Redshift?
- What strategies can be used for root cause analysis and remediation of issues?
- What is the purpose of setting max_active_runs in a DAG?

## Intro and project explanation  (1)

- What is your experience with schema designing and data modeling in a warehouse?

## Archita - 12:30PM  (2)

- Write a SQL query to fetch the employee with the 5th highest salary.
- Write an SQL update query to rename employee 'abc' to 'xyz'.

## Resume introduction and projects  (13)

- Describe how you implemented an intermediate data pipeline using Databricks in a project.
- How did you optimize PySpark transformations in your Databricks pipelines?
- How did you ensure the reliability and scalability of your Databricks pipelines when handling increasing data volumes and changes in data sources?
- How do you approach designing an ETL workflow in Azure Data Factory to ensure data integrity and optimize performance?
- Can you describe a complex transformation task you implemented and how you handled errors in schema validation?
- What monitoring techniques did you use in Azure Data Factory to track pipeline performance and quickly identify errors?
- How did you use Python libraries like pandas and PySpark to optimize the data ingestion process in your projects?
- How did you ensure your Python code was clean and maintainable while working on data pipelines?
- What strategies did you use to ensure your Python code scales efficiently when processing growing data volumes in your pipelines?
- What intermediate-level data quality checks have you implemented in your pipelines and how did they improve reliability?
- Can you explain a specific strategy used to improve data governance in a project and the impact it had on data quality?
- Could you explain implementing Azure Data Lake in a recent project and any challenges faced while integrating it with Azure Data Factory?
- Can you describe a situation where you optimized a data pipeline using Azure services, the tools you used, and the outcome?

## Interview intro and projects  (2)

- Describe complex data integration in a project, including decisions you made.
- In the context of pipelines, what is better: writing a pipeline or working without it?

## Interview introduction guide  (6)

- What is an ETL pipeline?
- What is your experience with Databricks?
- What is Databricks and what are notebooks in Databricks?
- What was the data pipeline in the Deviare project?
- Explain a strategy to migrate data from one database to another.
- Explain the pipeline in the Deviare project.

## AWS data engineer prep  (4)

- How to migrate from Teradata to AWS? Discuss potential data anomalies, masking and hashing techniques, GDPR compliance, building a medallion architecture, and necessary transformations.
- If the source does not have Change Data Capture (CDC), how would you handle incremental data ingestion?
- After setting up CDC and triggering it on value changes, how would you scale and implement the process when moving data between layers (e.g., from ingestion to transformation)?
- You migrated and masked data, and need to create a data lakehouse copy in a Redshift environment. One client queries the data warehouse and another queries the data lakehouse. How would you create a data lakehouse without focusing solely on the data warehouse?

## Resume introduction and projects  (1)

- How do you handle large-scale data processing using Pandas and similar tools?

## Resume introduction and projects  (2)

- Have you worked with big data pipelines?
- For big data, which service?

## Interview prep with details  (6)

- What Azure services are available for data engineering?
- What is the Azure equivalent of Amazon QuickSight?
- How do you manage very large DataFrames?
- Should you use repartitioning for better chunking?
- Is repartitioning supported in pandas?
- How can metadata be helpful in data engineering?

## Interview intro and tech stack  (13)

- Describe an intermediate data pipeline using Azure and PySpark. What optimizations for performance?
- How do you optimize Spark jobs within Azure Databricks to improve performance and manage resource utilization?
- How have you handled challenges related to data skewing in Azure Databricks pipelines and what techniques did you use to mitigate it?
- How do you approach designing ETL in Azure Data Factory to ensure both data integrity and performance?
- Can you give an example of a complex transformation task in ADF and how you managed error handling in that process?
- How do you monitor error logs and what steps do you take for pipeline reliability and early resolution?
- Tell me how you've used Python libraries like Pandas and PySpark to optimize data ingestion processes.
- Describe PySpark structured streaming and partitioning for large-scale data ingestion. What are the challenges and how to resolve them using PySpark and Pandas?
- Explain watermarking and salting for data skewing. How to do this in PySpark?
- Describe intermediate level data quality checks in your projects.
- Describe a specific case where data quality checks were enforced and uncovered a critical data issue.
- How did you implement Azure Data Lake in a recent project and what challenges did you face integrating it with ADF?
- What were the challenges in optimizing pipeline performance in Azure and how did you address them?

## AWS vs Data Eng Comparison  (10)

- How are these techniques applicable in PySpark?
- What type of validation should you perform when migrating data from one source to another, before transformations, and what quality checks are needed?
- What is Deequ and how is it used?
- We need to do masking, then write the data back to another folder as CSV or Parquet, adding a batch_id and batch_run_time column. Is this what is happening in the code you provided?
- What three things can we improve in this code?
- How can we improve hashing, standard_df creation, and lineage columns?
- What if the birthdate is not in the right format in standardize_df?
- In the lineage column, it's not a good idea to use a combination of datetime. How can we improve?
- How do you ensure reliability and performance for AWS Redshift?
- What strategies are used for root cause analysis and remediation of issues?

## Interview prep and tech stack  (2)

- What are the roles and responsibilities of a data engineer?
- Explain a project for the role of a data engineer.

## Interview intro and tech stack  (2)

- What are the basic steps in data preprocessing?
- What is the importance of data transformation and normalization?

## Purge in Pub/Sub  (1)

- ~ What does purge do in Pub/Sub?

## AI/ML interview prep  (2)

- Write a script to read a CSV file, create a DataFrame, and print the top 5 rows following best practices and optimization.
- If the DataFrame is empty, print a message; can we also include calculating the length of the DataFrame?

## PAWAN - 12:20PM  (1)

- What is data tearing and replication?

## Resume introduction and projects  (1)

- How have you used AWS Redshift in your previous projects?

## Python backend interview prep  (1)

- How would you use Python to process and clean large datasets?

## Interview prep intro Q&A  (14)

- Can you describe your experience with PostgreSQL, specifically converting R tables and providing input to Tableau?
- How do you handle complex AIR Table to PostgreSQL transformations?
- How do you handle data ingestion when the input/ingestion layer is email?
- How do you handle data quality issues like nulls and duplicates in source tables?
- How do you map Excel data to relational tables using joins?
- In a warehousing scenario with inventory and order management data in Excel, how much time is needed for SQL transformation?
- How do you handle a retail calendar table that is not present in the database and must be brought in externally?
- How do you approach a problem in data engineering?
- Given a set of 10 tables or table structures, how soon can you understand the schema?
- How do you connect to a SQL client?
- How do you perform ETL with Tableau on-premises and in the cloud?
- Why should you not use Tableau for ETL?
- How do you drag and drop fields in Tableau after connecting to a data source?
- What Tableau license should you use?

## Screening introduction preparation  (5)

- Explain dashboard in Power BI with an example for data analysis presentation.
- What is the difference between ADF and ADE?
- What are the key features of PySpark and Azure Databricks?
- How do you optimize performance in PySpark?
- How do you send data in Databricks?

## Resume introduction and projects  (12)

- What are some basic SQL queries you performed in BigQuery?
- How do you manage data partitioning in BigQuery to optimize query performance?
- Explain how clustering works in BigQuery and how it complements partitioning.
- What strategies do you use for cost management while running large queries in BigQuery?
- Explain how you optimize query performance in BigQuery.
- How do you handle data security and privacy in BigQuery?
- Discuss a complex analytical project you worked on with BigQuery, including key challenges and solutions.
- How do you ensure data quality and integrity in data pipelines?
- Discuss architectural decisions you made while designing scalable data pipelines.
- How do you handle data transformation in ETL processes?
- Discuss how you handle error logging and recovery in ETL processes.
- How would you approach migrating an on-premises data warehouse to GCP?

## Resume introduction and projects  (1)

- What challenges did you face while working on IQVIA and OCR, and how did you overcome them?

## Interview prep Data Engineer  (10)

- How do you standardize data schemas across teams?
- What data governance tools and frameworks do you recommend for schema versioning?
- How do you handle schema validation when data types mismatch (e.g., ID as string vs int)?
- Why use dbt over Python for data transformation?
- Explain the SQL DELETE with self-join to remove duplicate pairs.
- Explain the SQL query using CASE to canonicalize start/end pairs and deduplicate.
- How to write a SQL query that returns a specific row if it exists, otherwise all rows?
- Write Python code to conditionally filter a DataFrame.
- Explain a recursive function to merge nested dictionaries and sum values.
- What are alternative approaches to recursive dictionary merging?

## Join types comparison  (1)

- ~ What is the output of inner, left, and outer joins for these two tables?

## Pawan - Questions  (6)

- How did you design and implement integration between a POS system and a payment gateway?
- What approaches have you used to handle network failures or offline mode during POS-to-gateway transactions?
- How do you ensure that POS-gateway integration remains secure and compliant with PCI-DSS?
- How do you manage settlement and reconciliation between POS transactions and gateway reports?
- When integrating with third-party payment processors like Stripe or Adyen, what are the common technical challenges you face?
- How would you implement secure storage of payment data?

## Interview Q&A Preparation  (1)

- Tell me about the IQVIA project you worked on, what challenges you faced, and what did you build?

## SQL query output explanation  (8)

- What will be the output of this SQL query?
- Given this data, what is the output of this other SQL query?
- Given these two tables with an inner join, what will be the output?
- If we have these two tables in Spark SQL, what will be the output?
- In a migration project, if SQL gives us two options we cannot have both, how do we make the case insensitive?
- Given data in this particular join, we store it in time and want it in a delta table using Spark SQL, how do we get the same data in a delta table?
- For the above query, I want to store the output in a delta table where it gives me 4 columns. How?
- We have two pipelines using a normal cluster (not job cluster), the first runs from 2pm to 3pm, the second from 4pm to 6pm. The problem is the cluster is cleaned up and the second pipeline is executing now. What can I do if my second pipeline wants the cluster but it is occupied by the first?

## Rishabh@2  (6)

- What is segment routing or transportation?
- Did you work on routing? What did you do?
- You have multiple pickup points and you have to visit them and pick something from there (inventory or picking from somewhere and dropping to inventory). How do you assign the routing and how do you optimize the routes?
- Consider there is one school and students will be there. You are getting a pickup location, you have cabs, and 3 students could be picked up and then dropped to school. School is fixed, you have locations of students. How to create a routing for that so that minimum distance is achieved? Describe the complete flow.
- Suppose you have historical data. How can you leverage that historical data to be used in this case? What algorithm can we use?
- How can we handle data imbalance in continuous data?

## Manoj - Data Scientist 12:30PM  (1)

- Explain an end-to-end pipeline for IoT devices that detects anomalies from sensor data in a given window.

## Interview prep and project details  (14)

- What is the default cluster manager in Dataproc?
- What is the default storage in BigQuery?
- If I have 1 TB of data daily for clustering in Dataproc, should I use a single cluster or more?
- What are some open-source frameworks that Dataproc supports?
- What are cost-saving techniques for running a cluster?
- How to use Dataproc to migrate an on-premises Hadoop job to GCP?
- Which connector reads Cloud Storage in Data Fusion?
- What is the GCP orchestration tool for scheduled data pipeline jobs?
- What connectors are used with Data Fusion?
- How to design a schema when storing both structured ER data and nested JSON?
- In Dataflow, how can you achieve exactly-once processing?
- What is the minimum scheduling unit in Airflow?
- How to handle retries in a DAG that fails due to an API failure?
- Explain the concept of Catalyst optimizer in PySpark.

## Interview prep and tech stack  (12)

- For the REST API, what kind of data was stored since we get JSON? How to handle pagination when the number of pages is unknown?
- How was data overlap or data duplication handled?
- When using tokens in ADF to authenticate, the token expires. Pagination can take over 30 minutes; how to handle this?
- If an activity starts with an already added token and an API call is made, how can you change the token?
- If the API takes 40 minutes to return data and the token expires in 30 minutes, how do you handle this?
- In ADF, we have a linked dataset for SQL Server with a stored procedure. After running the first part of a pipeline to create lists, a flag is created in the stored procedure. Which ADF activity can handle this?
- What is index fragmentation in SQL?
- How to resolve index fragmentation?
- In SQL, we have data types like nvarchar and varchar, and cached data types. When would you use one over the other? Give a practical scenario.
- For storing keyboard numbers, special characters, etc., which data type should we use?
- What if we use nvarchar for those values?
- In Azure, with dimensions and facts, how would you design a date dimension for today's date?

## Interview prep and intro  (6)

- Describe your experience in building data pipelines.
- How do you handle data inconsistency in a data pipeline?
- What strategies do you employ for ensuring data quality?
- Describe a complex data pipeline design and the challenges involved.
- How do you architect a data pipeline for high-velocity data?
- Give an example of how you optimize a data pipeline for performance and scalability.

## Interview introduction example  (4)

- What is noisy data?
- Is it possible to handle noisy data before training?
- How to handle noisy data using pandas?
- Give me human logic to handle missing and null values, not about pandas.

## Interview prep and tech stack  (7)

- What is AWS Glue?
- What is the difference between dynamic frames and data frames in PySpark?
- What are MDM (Master Data Management) principles?
- Describe batch processing.
- Describe real-time processing.
- What are the steps to initialize a new pipeline to an existing one in AWS?
- What are job bookmarks in AWS Glue?

## Technical interview introduction  (6)

- What is transaction velocity?
- How were Monte Carlo simulations used in the Worldpay project?
- How was the simulation done?
- What were the reasons for the failure of these transactions?
- Describe Ikano.
- What is POS interfacing?

## Interview intro and comparison  (8)

- What have you done in Databricks, things you built and your work?
- Were we using dedicated or shared in your previous one?
- Why choose dedicated and not shared? What are the problems in both?
- How to build an ingestion pipeline where data comes from Kafka and goes to Databricks? Design the pipeline and complete flow, handling both batch and real-time data.
- What will you use for CDC?
- What is the difference between PySpark and Spark SQL?
- What is a broadcast join in PySpark?
- What are the different types of slowly changing dimensions?

## SQL for managers with reports  (3)

- ~ Write a SQL query to find the top 3 salaries per department.
- ~ What is the difference between RANK and DENSE_RANK in SQL?
- ~ In Databricks, when should I use a temp table vs a CTE?

## Interview prep intro tech  (8)

- How does Airflow handle parallelism in DAGs?
- If one scheduler crashes, how do we handle that in Airflow?
- How do you add kill functionality to a DAG in Airflow?
- How does a DAG look in Airflow?
- How do you create an AWS Glue job?
- What are triggers in AWS Glue?
- How do you add a new column to a DataFrame in Python?
- How do you add a derived column C as A * B to a DataFrame in Python?

## Critical issues in Databricks  (8)

- ~ What critical issue have you faced in Databricks?
- ~ What is reconciliation in Databricks for data checks?
- ~ How do you manage clusters and catalogs in Databricks?
- ~ How do you read files from ADLS and Parquet files in Databricks?
- ~ How do you identify duplicate records in Parquet files?
- ~ What are the different types of jobs in Databricks?
- ~ What are the different types of clusters in Databricks?
- ~ What is the difference between a table and a view in Databricks?

## Interview intro preparation  (2)

- How do you implement data-level security to restrict access to certain users at the database level for a particular region?
- Which graph databases have you worked with?

## Bhavesh - 3PM  (3)

- If there are slices, can they be combined into one single file?
- How were your queries executed in cloud quote?
- How to handle deletion with dependent tables: soft delete, hard delete, cascade? How to identify which approach? And if using cascade, how to delete parent and child separately?

## Azure services overview  (1)

- How to find duplicates in thousands of data points using data analysis?

## Embedding model and storage  (8)

- Write PySpark code to create a new column 'grade' in a DataFrame with employee data: if salary > 20k INR then 'A', 20-40k then 'B', >40k then 'C'.
- Why use Unity Catalog instead of Hive metastore?
- What are the types of compute in Databricks, how do they compare, and how do you select one?
- What format of tables do you create in Databricks?
- What are the benefits of Delta format?
- What is the difference between Delta and Parquet?
- How do you improve the performance of a Delta table?
- What happens when you run VACUUM on a Delta table?

## Project explanation and intro  (9)

- Given a solution to scrape data from a government website with similar letter structure, what approach would you take to scrape docs and place them into blob storage? Design the infrastructure.
- When scraping data from similar but not identical government websites, how would you make the scraper generic? What measures would you take?
- If using config files for each source, and you have 1000 sources, what would you do? (scalability problem)
- When we have one doc to scrape, one Azure function works; now we have 1000, what then? (scaling Azure functions)
- How to ingest scraped data for a chatbot? This requires data engineering. Answer as if in an interview.
- After scraping a single doc of 5M characters, how to ingest it for a chatbot? What to embed – 5M characters or 1M words? What?
- In a chatbot ecosystem, how would you design a microservice architecture? (FastAPI)
- Is it right to build a chatbot using microservices, or modular, or monolithic?
- What are the drawbacks of microservice architecture in this chatbot context?

## What is AWS Glue  (3)

- ~ What is AWS DMS?
- ~ How to handle schema changes in AWS Glue?
- ~ How to handle data skew in AWS Glue?

## Interview intro and tech stack  (24)

- Give an example of Databricks in Azure architecture.
- Explain how Databricks is used in the Homecure project.
- Why did we use ADLS?
- What mode did we use for DLT (Delta Live Tables)?
- How do you handle data quality in streaming?
- How do you handle data quality with Flink?
- How do you handle data quality for batch processing with Databricks?
- What built-in data quality features does DLT (Delta Live Tables) provide for Databricks?
- What actions are available for validations and how do you configure them to handle invalid data?
- What happens if you don't provide any action for a validation rule?
- Why use Spark over DLT for the bronze layer?
- How do you handle schema relationships in Databricks?
- Explain Databricks Auto Loader briefly.
- What can optimize performance for a Databricks cluster?
- What is the difference between OPTIMIZE and SETORDER in Databricks?
- How are concurrent writes handled in Delta Lake?
- What features of Unity Catalog are used for governance?
- Explain the masking policy for this.
- What masking policies exist for PII and PHI?
- Where is the final transformed data in the gold layer stored? In Databricks or elsewhere?
- What are the steps for data quality in each layer of the medallion architecture?
- If data volume increases and you cannot meet SLA, so data quality checks take too long, in which layer would you have fewer data checks?
- Where do you see most issues for data validations?
- In the silver layer, if we see bad data, do we drop the row?

## Profile and project intro  (10)

- How do you schedule DMS jobs?
- How do you handle incremental data capture in DMS?
- How does the 'Start CDC automatically via EventBridge + Lambda' flow work?
- How do you convert a PostgreSQL stored procedure to an RDS procedure?
- What are the steps of a data migration project?
- What is the configuration setup for DMS?
- What are distkeys and sort keys in Redshift?
- What are the types of distribution keys in Redshift?
- How can Glue handle PySpark optimizations?
- What is schema evolution and how does Glue handle it?

## Intro and project alignment  (8)

- In the IQVIA project, what challenges did you face in extracting data?
- How was QA done in IQVIA?
- How was skewed data handled in IQVIA?
- How was table extraction done in IQVIA?
- How was OCR used for table extraction?
- What accuracy did you get with Tesseract?
- How were segmented tables handled?
- What if the table has a different segment structure?

## Interview answer preparation  (1)

- What are the downstream processes in this system?

## Resume introduction and projects  (8)

- How do you design a data lake architecture using GCS and Iceberg?
- How do you ensure data consistency while ingesting data using webhooks or REST APIs?
- What is database connection pooling, why is it required, and how do you test it?
- For database pooling, if a query is taking 10-15 seconds, how would you perform RCA and solve it?
- A select query joining 4 tables is taking around 10 seconds. How do you identify and fix the problematic join?
- Explain the use case and example of EXPLAIN or ANALYZE statements in your project.
- How do you validate data correctness when the source system or APIs are not well documented?
- What is your debugging approach when a data pipeline fails?

## Generate questions and answers  (3)

- What scenarios would you use to handle large data sets?
- What is Apache Kafka?
- Explain the steps to publish a message in Kafka.

## Data engineer interview help  (15)

- How would you design a data lake architecture?
- What are the best practices for handling Databricks with AWS services?
- What are the best ways to manage secret credentials with AWS?
- How would you design a data pipeline for disaster recovery?
- How would you design AWS services for disaster recovery?
- What specific AWS services would you use to implement that design?
- Can you describe an end-to-end AWS-based architecture using S3 and CloudWatch monitoring?
- How do you balance batch and streaming data pipelines for performance?
- How do you implement secrets management in Databricks in a multi-environment setup?
- How would you optimize read and write performance in Databricks?
- How would you design an automated CI/CD pipeline with jobs integrating Databricks databases?
- How would you design a data ingestion pipeline using PySpark?
- How do you monitor Spark executor and driver memory configuration?
- How would you design a modular, reusable Python ETL framework?
- How would you design a common ETL Python framework that can run across AWS credentials?

## Resume introduction and projects  (6)

- How would you approach extracting data from two Salesforce CRM instances and inserting it into Snowflake, considering data type and structure differences?
- What steps would you take in Python to handle the data flow between CRM and Snowflake?
- Given 60-70 tables with 100-200 columns each and dependencies, what approach would you use to extract the data?
- Once data is extracted, what would your transformation process look like?
- How would you ensure your Python code is stable and does not fail with such massive data?
- If you create batches, would you run them sequentially or in parallel using multi-threading?

## Interview answer generation  (1)

- What is Airflow and how do you use it?

## Explain tech stack  (1)

- How to build end-to-end ETL pipelines?

## Interview introduction and projects  (16)

- What is Apache Airflow, why is it used, and how have you used it in your project?
- What are tasks and task groups in an Airflow DAG?
- What are the components of Apache Airflow?
- What are hooks in Airflow?
- How do you monitor a workflow in Airflow?
- What is a SubDAG in Airflow?
- How do you maintain data lineage?
- What are XComs in Airflow?
- How do you mount a storage container like S3 in PySpark?
- How do you handle large data challenges like data skewness in PySpark? What techniques can be used to handle it quickly and efficiently?
- How do you optimize PySpark code?
- When performing wide transformations on a small table and a large table, what techniques should you follow to avoid issues?
- What is broadcasting in PySpark?
- What other techniques can decrease shuffling in PySpark?
- Write code to load a CSV file from a path into a DataFrame.
- How do you create a database or schema in PostgreSQL?

## Interview prep tech stack  (7)

- In data engineering, when building a data pipeline from multiple sources, how do you ensure data quality, reliability, and traceability without code?
- Give me a real example where you built a data pipeline in your project, including challenges.
- Describe a recent case where you had to apply solutions in a data engineering pipeline.
- In data versioning, how would you design storing and comparing different versions of data?
- If the data is in a Parquet file, how do you consume that data?
- In Azure, how do you consume Parquet files, similar to Athena?
- Suppose you receive two shape files with admin boundaries. How would you identify the area changes between the country boundaries? (We have geospatial data.)

## Technical interview intro  (2)

- What are the different types of normalization in SQL?
- What are the different types of joins in SQL?

## Interview intro preparation  (3)

- What is in this zip file, what does it do, and what about the markdowns?
- Which Python version is used here?
- What is happening in this Lambda function?

## Interview intro preparation  (19)

- Explain a challenging ETL pipeline you have worked on.
- If a failure occurs during a pipeline, how would you handle it?
- If data has been 90% moved and an error occurs, how do you get notified?
- How do you handle duplicate records or junk characters in data validation?
- Explain the process of using Glue to move data from PostgreSQL to Redshift.
- Why do we need many services to move data from Oracle to Redshift?
- In Glue, there are visual ETL jobs — why not use them?
- What are S3 lifecycle policies and when to use what?
- What are event-based S3 buckets?
- Explain DMS and its high-level architecture.
- How does DMS read data from the source?
- After a full load completes, how do you enable CDC in DMS?
- Explain indexes in Redshift.
- With two nodes and large tables, how do you choose a distribution key?
- What are atomic and non-atomic procedures?
- What are workload management engines in Redshift?
- Explain indexes in PostgreSQL.
- In Redshift, with two schemas (source and target), using a psql procedure that fetches from staging consumes a lot of resources. How would you process the data using another AWS service?
- Can we use PySpark for Oracle to Redshift migration? If yes, tell me the steps without Glue.

## Rishabh@5  (7)

- ~ When processing real-time streaming, where does Redshift fit into your project?
- ~ What are the advantages and disadvantages of AWS Glue vs Step Functions, and when would you choose one over the other?
- ~ If I have an 80GB file and try to process it with Lambda and Step Functions, what challenges could I face?
- ~ What is a materialized view?
- ~ What are the advantages and disadvantages of Redshift?
- ~ In DynamoDB, what are the main indexing methods when searching for a record?
- ~ Between GoodData, QuickSight, and Tableau, which would you choose and why?

## Rishabh@2:30  (14)

- What is the difference between data lake and data warehouse architecture?
- What data file formats are used in data lakes and data warehouses? What are other file formats?
- What is the advantage of storing data in delta format over other formats when ingesting from blob and object sources?
- If a transaction file gets deleted, what is the impact on the pipeline?
- What are the dimensional modeling techniques used in Redshift transformations?
- What are the steps to optimize performance in a data warehouse?
- How to design schema in DynamoDB considering cardinality and access patterns to minimize hot partitions?
- How to implement and maintain data lineage in AWS Glue?
- What are the advantages of data lineage for upstream and downstream data?
- Write an algorithm to read events from an S3 folder, check if file was already processed, and write success/failure logs to CloudWatch.
- What is the Redshift connector and which connection type should be used?
- How to ensure idempotent data ingestion?
- Describe the process to design an ingestion pipeline that handles both batch and streaming data.
- Design a pipeline for multiple sources: batch, streaming, real-time from Kafka.

## Interview intro and tech stack  (1)

- How is data stored and made visible in Google Cloud Storage (GCS) and BigQuery?

## Nomiso - 5PM Data Scientist  (1)

- How would you scale your log analysis pipeline for a 10x increase in data volume?

## Profile intro and projects  (1)

- Write a query to find the second highest salary from the emp table.

## Cloud services for data engineering  (2)

- ~ What are the various cloud services and tools used for data engineering?
- ~ Using AWS services, prepare an end-to-end workflow for data engineering.

## Interview intro and tech stack  (7)

- How do you validate data, including rows and columns?
- What happens with timestamps when migrating the database? How do you handle frozen timestamps?
- How do you create a data pipeline from Azure SQL to Azure DB services, or to Copilot Studio, or to Power Apps?
- How do you optimize ETL for unstructured data when many columns are null?
- What strategies do you use for late-arriving data in a pipeline?
- How do you structure a Python-based ETL project for maintainability and reliability?
- What is data lineage and how do you maintain it?

## Interview intro and tech stack  (3)

- How do you create a DataFrame from a dictionary in Pandas?
- What methods are available to handle null values in a DataFrame?
- How do you create a scatter plot using Pandas?

## Rishabh@1  (8)

- What is your expertise with ClickHouse in analytics, including data modeling?
- Describe your ability to design denormalized OLAP datasets for BI.
- What is your understanding of ETL and ELT pipelines using Airbyte, dbt, and Airflow?
- What are Python-based transformations?
- Where have you used PySpark?
- Describe your hands-on experience with reporting and visualization frameworks.
- What is your experience with real-time data streaming using Kafka?
- What is your experience with relational and analytical databases like PostgreSQL, ClickHouse, and Elasticsearch?

## Interview preparation answers  (1)

- Describe your experience pulling data from data pipelines, data stores, and creating dashboards. What is your background in analytics and ETL pipelines?

## Pawan | 7:00 PM  (3)

- How do you handle missing data in Azure Databricks?
- How can you optimize Spark jobs in Databricks?
- How do you manage PySpark configuration in Databricks?

## Profile intro and projects  (5)

- What are the key components of a modern data architecture on AWS?
- How would you design a scalable data pipeline on AWS?
- Describe your experience with AWS Glue.
- How do you ensure security in data pipelines on AWS?
- Describe a real-time analytics architecture on AWS.

## Interview intro prep  (24)

- What factors will you consider to create a platform when looking into a company's data ecosystem?
- What database did you use to implement partitioning in the carbon emission platform?
- How do you partition data in Redshift?
- How do you build a data lake in S3?
- How can you archive data in S3?
- How do you manage hot data and cold data?
- What is the difference between Snowflake and Redshift?
- What kind of instance is used in Redshift?
- Design a data lake with structured and semi-structured data.
- In which project did you utilize SCT?
- What are the challenges using SCT?
- How exactly did you use DQ?
- Where did you use OpenMetadata and Grafana?
- If Glue is serverless, why do we need to monitor it?
- How will you access or query the data stored in S3?
- What exactly do you do in the catalog?
- What are crawlers?
- What is data federation?
- What is data virtualization?
- What is FDW in Redshift?
- Where did you use EMR?
- When should you denormalize?
- Explain denormalization with a customer sales table example.
- What are star schema and snowflake schema, and when should you use each?

## Prepare intro and projects  (16)

- What is the difference between event time and ingestion time in Flink?
- How do watermarks work in Flink?
- Describe Flink checkpoint and savepoint.
- What is the role of state backends in Flink?
- How do you implement a Flink streaming application using the DataStream API?
- Explain backpressure in Flink.
- Why is Java multithreading important in streaming jobs?
- How do you integrate Flink with AWS?
- What are the best practices for storing streaming data in S3?
- When would you choose Redshift, Athena, or DynamoDB for streaming data storage?
- Describe a real-time architecture using Flink and AWS.
- How do you ensure exactly-once semantics in Flink?
- How do you manage schema evolution in a streaming pipeline?
- Explain key metrics to monitor in a Flink streaming job.
- How do you optimize AWS cost for streaming jobs?
- How does Flink handle late arrival events?

## Microsoft Fabric usage flow  (7)

- ~ What is the usage and flow of Microsoft Fabric?
- ~ How to design a lakehouse on medallion architecture?
- ~ What are the key factors to consider while creating an ingestion pipeline?
- ~ How to implement governance and data masking strategies?
- ~ What are the strategies to optimize performance issues in Microsoft Fabric like Synapse?
- ~ Describe the approach to design data models for Power BI.
- ~ How to monitor data pipelines and ensure reliability?

## Kafka Connect and Debezium  (2)

- ~ What is Kafka Connect and Debezium?
- ~ What is the difference between a table in Kafka Connect and Debezium?

## Rishabh@12:30  (13)

- What is subject naming strategy in data engineering?
- What is topic record name strategy?
- What if offset?
- What are the types of values in offset?
- In a connector with 10 tables, how will the offset be stored for these 10 different tables?
- What configuration specifically tells the connector where to start and where to continue?
- Does MERGE INTO target_table T USING source_table S ON T.id = S.id perform a full scan or partial scan?
- Why does MERGE INTO target_table T USING source_table S ON T.id = S.id perform a partial scan?
- If I have a MySQL data source and a target table, is MERGE the best option for moving data from source to target, or are there other ways?
- How does MERGE know which data needs to be scanned?
- If I run MERGE INTO target_table T USING source_table S ON T.id = S.id every 5 minutes, it's expensive — what other ways can improve performance?
- What is partition pruning?
- Explain BigQuery merge.

## DBT in modern data stacks  (1)

- ~ What is dbt and its role in modern data stacks?

## Resume Summary Creation  (15)

- What are the basic steps for performance tuning of ETL pipelines in terms of optimization?
- How do you handle a scenario where customer data is in a Pandas DataFrame and today's data is in a PySpark DataFrame?
- What are the steps for optimizing SQL queries?
- For an employee table, what could be the indexes?
- Can I put an index on employee name and employee city as part of indexing?
- In the case of an employee table, where could partitioning be applied (employee ID, name, city, joining date)?
- Where should I use the REST API in Azure Data Factory and what are the steps to perform?
- How do you create a REST Linked Service in Azure Data Factory?
- How do you execute a stored procedure using Azure Data Factory?
- How do you select data from more than two tables?
- What other ways besides joins can you combine data from multiple tables?
- Write a query to join 8 tables (t1 to t8) using an ID column.
- How do you select all columns from all 8 tables in a join?
- What is the best way to handle INSERT, UPDATE, SELECT, DELETE, DROP, TRUNCATE, and CREATE on more than two tables in one go?
- Can we use a CROSS JOIN in a multi-table join scenario like joining 8 tables on ID?

## Interview answers in points  (7)

- How would you design a solution to retrieve 10 lakh rows of data, call an API, and store the data efficiently?
- How would you retrieve 10 lakh rows and get the result as fast as possible?
- How do you find the highest salary in SQL?
- How do you find the 3rd highest salary in SQL?
- What is another approach to get the 3rd highest salary in SQL?
- If there are 10 people with 10k salary, how would dense rank rank them?
- If there are 100 people, 10 have the same highest salary, how do you rank to get the 3rd highest using dense rank? What is the 3rd highest for [10,10,10,10,90,80,67] using dense rank?

## What is Kafka  (9)

- ~ What is Kafka?
- ~ What is the end-to-end flow in Kafka?
- ~ What are the Kafka connector configurations?
- ~ What is the difference between partition key, pre-combine key, and record key, and how to use them?
- ~ What is Hudi table?
- ~ What is Apache Flink?
- ~ What architecture does this use?
- ~ What is lambda architecture?
- ~ What architecture does Hotstar use?

## Kafka connectors explained  (5)

- ~ What are Kafka connectors and Debezium?
- ~ What configurations are required to use Kafka Connect and Debezium?
- ~ Where are these configurations done?
- ~ What is a Hudi table?
- ~ What is GCS?

## Interview introduction generation  (2)

- How do you handle schema evolution?
- When was Hive introduced?

## Project descriptions extraction  (20)

- What was the volume of data?
- What are the data pipeline components in this project?
- How was data validation done in Lufthansa?
- What were the challenges in the validation process and how were they fixed?
- Explain the Microsoft one lack and heterogeneous data handling tools.
- How to design a data pipeline when needing to handle unstructured data and put it in one source?
- Where did you use Kafka in your projects?
- Which databases are you familiar with?
- What is the purpose of Neo4j?
- What data transformation tools have you used?
- What batch processing tools have you used?
- How do you transform millions of records in Databricks using ETL tools?
- How would you optimize joins in Spark SQL?
- How would you optimize a table query for millions of records?
- How do you debug Spark jobs?
- How do you debug ETL pipelines, and how to deal with the same problem?
- What have you done in the IDoc processor?
- How did you check with the help of Databricks?
- Have you built a Microsoft-centric pipeline in your previous projects?
- What is the difference between Redshift and Snowflake data formats?

## Interview preparation guide  (2)

- What kinds of data models have you created?
- What are techniques for optimizing Databricks?

## Answering questions in bullets  (14)

- How do you ensure scalability in a data pipeline?
- What are the advantages of Microsoft Fabric or data architecture?
- Describe the data ingestion ETL pipeline architecture in ADF.
- What is Genie?
- How do you handle failure in a pipeline and monitoring?
- What is Microsoft Preview and Fabric?
- What are the hidden costs in Microsoft Fabric?
- When should you choose Synapse versus Fabric?
- What are the advantages of Fabric over Databricks?
- What are the limitations of Fabric?
- How is Python used in data architecture?
- How do you ensure data security in ADF and Fabric?
- What is your approach for metadata enrichment?
- How do you ensure observability and reliability?

## Interview introduction summary  (6)

- What monitoring tools do you use for projects?
- How do you manage cluster configuration in Databricks?
- How do you version control notebooks in Databricks?
- What is the difference between Spark SQL and Hive SQL?
- What is the purpose of Delta Lake Change Data Feed?
- How do you integrate Databricks with AWS or Azure? What are the steps?

## Interview intro and guidance  (3)

- What is DBT? Give an example.
- What are the best practices for data modeling in general?
- What methods, apart from those mentioned, can be used for query performance and optimization?

## Profile review options  (3)

- ~ Explain the process of exporting data from source to target tables.
- ~ How do you manage incremental and full loads in ETL?
- ~ How can you compare large datasets without a full table scan in SQL?

## Interview intro preparation  (1)

- Explain the Lufthansa migration project from Teradata to Databricks, as a data engineer.

## Pawan | 12:00  (2)

- What dbt tools have you used? Explain Snowflake.
- Have you designed a Snowflake solution specifically for retail?

## Tech intro and stack  (22)

- Describe the Sonara.ai project, your role, and responsibilities.
- What did you use for data ingestion?
- How did you pull data from APIs?
- How do you transfer data to BigQuery? What library do you use?
- How do you load data into BigQuery using a pipeline script?
- What transformations did you perform using dbt?
- What is the difference between Redshift compute and BigQuery compute?
- What projects have you done in Azure Data Factory? What tasks did you perform?
- What kind of schema modeling in dbt? What architecture is followed? (e.g., star schema)
- How do you implement medallion architecture?
- Explain a project in a finance company. How did you do data modeling and analytics?
- What did you use for data ingestion in the finance project?
- What is the difference between ETL and ELT, and when would you choose each?
- What are the steps to transfer data from Postgres to Redshift using AWS Glue?
- How do you implement watermarks when pulling data from Postgres to Redshift to avoid pulling all data?
- How does CDC capture changes?
- Are there other ways to capture changes besides CDC?
- How do you optimize queries in BigQuery?
- Can you partition a BigQuery table on a VARCHAR column?
- What is the difference between partitioning and clustering in BigQuery?
- If you have 100 columns, how do you decide which columns to use for partitioning or clustering?
- For a table with timestamp columns, if you partition by date and cluster by order_timestamp, will a query with a filter on order_timestamp use partitioning or clustering?

## Tech stack extraction  (1)

- What services are in the tech stack from this resume: S3, Redshift, Glue, EMR, Athena, Lake Formation, DynamoDB, API Gateway, Step Functions, EventBridge, ECS/Fargate?

## Interview prep and comparison  (7)

- How do you test incremental load, and how do you validate the incremental load?
- What is the difference between SCD Type 1 and Type 2, and how do you validate each?
- What does data quality mean, and when do you ensure that the data quality is good enough to be moved to production?
- Write a SQL query to find duplicate keys in a table.
- Explain the SQL query: SELECT business_key, COUNT(*) AS record_count FROM your_table GROUP BY business_key HAVING COUNT(*) > 1;
- Write a SQL query to fetch the third highest salary from an employee table.
- Explain the SQL query: SELECT salary FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk FROM employee) t WHERE rnk = 3;

## Data QA Engineer interview  (4)

- How do you validate the duplication?
- What is the difference between left join and inner join?
- Explain the difference between RANK and DENSE_RANK in SQL.
- How do you remove duplicates while maintaining order?

## Ajinkya@2  (6)

- How do you handle missing data in a dataset?
- What pandas functions are used to handle missing/null values? List them.
- How does pandas handle large amounts of data?
- Write a SQL query to find the second highest salary from an employee table with columns name, dob, salary.
- Write a SQL query to count employees per gender from an employee table with a gender column.
- Modify the SQL query to filter employees whose date of birth is between 20-10-1987 and 20-11-1990.

## Interview Prep Tech Stack  (3)

- Write an SQL query to get artist statistics including total songs and average duration.
- Add conditions to the email in the query.
- Provide an alternate solution for this SQL problem.

## Jitendra@4  (1)

- What kind of data were we getting and storing in the Airflow orchestrated jobs?

## Rishabh@4PM  (13)

- Tell me about a project where Kafka Connect was used and how it was implemented in IMBIO.
- In a source Kafka connector that brings data to a Kafka topic with transformations, where does the final sync happen?
- How does data go from sync connectors to GCS?
- How is data served from GCS to an analytical dashboard?
- How is a cloud function used in this architecture?
- If you have a MySQL table with billions of records used for analytics, but you only need 3–4 months of data while it has 2 years, how would you reduce the data in MySQL and keep a complete copy in a data lake? Walk through step by step.
- When onboarding a table in Kafka Connect, how do you ensure historical data is captured?
- If you have a connector that has been running for 3 months with incremental pulls and no snapshotting, and now you need historical data for one table, how would you handle that?
- If you update a connector configuration to snapshot only one table out of ten, and that snapshot takes one day, how can you be sure you don't lose CDC changes for the other tables during that day?
- Given a table with records that have updates and deletes over time, how would you reconstruct the latest snapshot from the landing layer in GCS partitioned by ingestion date?
- If you have a 'last_modified_at' column, how would you write a query to get the latest data every hour?
- How do you identify a deleted record when there is no status column and the record is simply missing?
- If tables are in GCS and you are using BigQuery, but performance is poor, what would you do?

## Copilot agent creation steps  (2)

- What are the use cases where pandas was used in the insurance project?
- What is a use case of numpy in your project?

## Interview Prep Senior Data Engineer  (6)

- What is the difference between Kafka source connector and sync connector?
- What is the difference between PySpark and Apache Spark?
- Given a join between two tables, one is 500 GB and the other is 50 MB, how can you improve join performance? Write the code.
- What is the Debezium configuration?
- In Debezium configuration, how do you fine-tune what keys to use?
- What is heartbeat in Debezium?

## Interview Prep Assistance  (1)

- What is Debezium? Tell me more about Kafka.

## Interview Prep and Tech Stack  (3)

- What are 2-3 more steps after handling missing values?
- How do you handle outliers, and which types can be removed versus kept?
- When to use AUC as a metric to evaluate a model?

## Interview prep intro tech  (2)

- What was the role of the data team in this project?
- What did the data team accomplish since I was also on the data side?

## Interview Prep Data Engineering  (21)

- What optimization techniques can be used for a Databricks job that is running longer and costing a lot?
- How to manage governance and security in Databricks?
- How is PII data handled in Databricks?
- How to optimize tables in Databricks?
- How to set up a Delta table?
- What are the benefits of Parquet over CSV?
- What is dbt and where did you use dbt in your project end-to-end?
- What are all the materializations in dbt?
- How to handle data skew?
- What is salting in data processing?
- What is Adaptive Query Execution (AQE) in Spark?
- What are operators in Airflow?
- How to design a DAG with retry and fallback?
- How to do parallel reading from multiple data buckets in Python?
- How to manage rate limiting when getting data from an API?
- Write a SQL query to get the running sum.
- Explain this query: SELECT name, score, SUM(score) OVER (ORDER BY name ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_score FROM scores;
- Explain this query: SELECT name, score, SUM(score) OVER (ORDER BY match_no) AS running_score FROM scores;
- Can we use ROW and RANGE functions?
- Explain this PySpark code that uses row_number to get top record per group.
- What are the starting points for a migration from Teradata to Databricks?

## Rishabh@2  (13)

- How does real-time data get into the bronze layer if it uses batch processing?
- What comes next after the bronze layer in a medallion architecture?
- How do you handle dimension and fact tables and keep them updated?
- How do you manage pipelines and the gold layer?
- How do you handle multiple fact and dimension tables with needs based on historical and live data?
- How do you handle data governance in fact and dimension tables?
- In which layer and stage do you apply these processes?
- How can I combine multiple silver tables using PySpark, reading from a lakehouse and writing to a warehouse using stored procedures?
- Can I use Spark to connect with warehouse tables?
- How to integrate Row-Level Security (RLS) in Power BI?
- What is the most difficult part of your data experience so far?
- Is it possible to enable CDC on SQL Server while using Fabric, given ODBC access and other sources via API and SharePoint?
- How frequently do we need to trigger these CDC processes?

## Interview Self-Introduction Help  (1)

- What is the relationship between CDC and Kafka?

## Pawan | 3:00 PM  (2)

- Do you have NoSQL experience?
- Write an SQL query.

## Answering in Bullet Points  (1)

- Write a SQL query using SELECT and WHERE clause.

## Go Backend Engineer Profile  (1)

- What is ClickHouse database and how is it used?

## Data Engineer Interview Prep  (11)

- What is incremental load and how have you handled it in your projects?
- What optimization techniques can you use for slow Spark jobs?
- Given tables Employee(EMPID, EMPNAME, DEPTID, SALARY) and Department(DEPTID, NAME), write a SQL query to display employee name with respective department.
- Write a SQL query to identify duplicate records for employee name.
- What is another optimized way to identify duplicate records?
- Write a Python code to remove duplicates from a list while preserving order: List_1 = [1,2,3,4,1,2].
- What is data skewness and how do you fix it?
- What is a Delta table?
- What is the difference between list, tuple, and dict?
- What are the use cases of list, tuple, and dict in a pipeline?
- Is a DataFrame mutable or immutable?

## Data Engineer Interview Prep  (2)

- ~ Explain a project where you used Python and SQL in data pipelines with streaming and non-streaming, including NumPy for faster transformations in batch processing.
- ~ How did you combine SQL with Python to generate insights?

## Profile Intro and Project Summary  (1)

- Describe the ingestion pipeline in GCP where data is coming from Drive.

## Ajinkya@4  (17)

- Describe an end-to-end ETL process using coding starting from ingestion.
- How do you handle errors and exceptions in an ETL process, for example if a database connection fails?
- If a file has 10 columns and tomorrow more columns are added, how does ingestion work?
- If a new column is added in between existing columns, how does pandas know? Provide code.
- What are the different incremental load strategies?
- What is the architecture of PySpark and its components?
- Which GCP components are used in the project?
- What are the components of GCS you used?
- How do you store data from PySpark into GCP buckets? What is the command?
- Why use both PySpark and Databricks?
- What is the Airflow command to orchestrate modules like ingest.py, transform.py, and query.py?
- What is a data mart? Be brief.
- What is ELT?
- What are ACID transactions in a database?
- Write a query using the RANK function to sort entries.
- What is lazy transformation or evaluation? Give an example.
- How to optimize SQL queries? What methods?

## Rishabh@1  (2)

- What volume of data did you process?
- Did you use ADF for orchestration and transformation?

## Interview Prep and Intro  (2)

- Find duplicate email addresses in a users table (id and email).
- How to eliminate duplicate emails while keeping the latest record?

## Tech Screening Intro Prep  (9)

- What is the difference between Kafka and Kafka Connect?
- In a corporate stage, how does your pipeline look like end to end? Mention Kafka and Debezium.
- How do you fine-tune Debezium configuration?
- What is a BigQuery merge query?
- Will the merge query 'MERGE INTO target_table T USING source_table S ON T.id = S.id' be partial or full?
- What is the medallion architecture?
- What is the cleanup policy in a Kafka topic?
- If I use cleanup.policy=compact, should I use it or not?
- What happens if a topic is compacted but a message has no keys?

## Interview Prep Data Governance  (13)

- What data quality tools have you used?
- What is your experience with IQD?
- Which version of IQD have you used?
- What is your experience with Talend?
- Do you have experience with MDM? Describe your experience.
- Why would you migrate from Teradata to Databricks? What are the benefits?
- What could be the purpose of this migration?
- Did you create any data quality metrics? Describe your experience.
- Did you automate data quality? How?
- Did you build a scorecard in Tableau?
- How do you perform root cause analysis when there is a data issue?
- Describe a scenario where a data quality rule took a lot of time and how you optimized it.
- What good practices have you implemented in data governance for data standardization?

## AJINKYA IOT  (2)

- What was the business requirement (problem statement) for the IOT project?
- What solution did you implement in the IOT project?

## New chat  (1)

- What is the step-by-step process for implementing audit in Databricks?

## Profile Intro and Tech Stack  (14)

- Explain Medallion architecture and why to use it.
- How do you handle errors in ETL jobs, how to prevent those errors, and what steps to take to solve the error?
- How do you handle late-arriving data for a project that has 100 million records per day? What techniques are used?
- In this scenario, should you use full load or incremental load?
- What is a narrow transformation in Spark?
- What is a wide transformation in Spark?
- Explain delta tables and how you use them in your projects.
- There is a SQL job that has been working fine for a month but the run time has increased. How would you approach this?
- Explain slowly changing data in databases and dimensions.
- Suppose you have a column with date values but the type is varchar. How do you change the data type using PySpark?
- Explain this PySpark code: from pyspark.sql.functions import to_date, col; df = df.withColumn('order_date', to_date(col('order_date_str'), 'yyyy-MM-dd'))
- Given a table with columns id, status, and timestamp, write a query to get the latest record for each id.
- Explain this query: SELECT id, status, timestamp FROM (SELECT *, ROW_NUMBER() OVER (PARTITION BY id ORDER BY timestamp DESC) AS rn FROM table_name) t WHERE rn = 1;
- Why use ROW_NUMBER instead of DENSE_RANK in this scenario?

## Interview Preparation for Pragati  (43)

- Explain the order of execution of a SQL query (SELECT with WHERE and GROUP BY) and what happens when we run it.
- What are window functions in SQL and how do you use them?
- What is normalization and denormalization?
- Write a query to get first name, last name, and id of customers whose shipping is pending.
- Can you group by country (e.g., UK, India) and count how many are pending?
- Add a column to show the total amount spent by that country.
- What is MapReduce in Python/Spark?
- Explain Spark architecture.
- If I have a job running on 5 worker nodes and one node gets terminated in the middle of execution, how does Spark handle this?
- How do you handle data changes during a job? (Delta loads vs streaming data)
- For streaming data, what happens if a worker fails in the middle?
- Why is Spark called lazy?
- What is the execution plan in Spark when I hit run?
- What is the difference between RDD and DataFrame?
- Are RDDs mutable?
- What is Catalyst Optimizer?
- How do you handle large datasets (e.g., 1 TB) in Spark?
- What is partitioning and coalesce?
- How do you handle parallel processing for calling an API at 100 calls per minute in Python?
- Optimize the following Python code: while True: data = fetch_all(); print('Fetched 100 API responses'); time.sleep(60)
- How to handle delta loads when you have files in S3 and you get more files daily?
- What is the difference between Unity Catalog and Glue Catalog?
- If a job runs for 24 hours and fails before writing data back, is there any way to retrieve processed data?
- What is the difference between interactive cluster and job cluster in Databricks?
- How many types of schemas are used when planning a chat application? Which one to consider?
- What schema architecture will we use for a chat application? (star schema vs snowflake)
- What is Airflow used for?
- What is a disaster recovery plan for failed jobs?
- How is SNS different from SQS?
- What is a serverless cluster? In what scenarios can you use it?
- What is a SQL warehouse server?
- What happens if we use a normal server instead of serverless in SQL warehouse?
- When configuring a server, what is Photon in Databricks runtime?
- What is Talend ETL? Difference between Databricks and Talend?
- What are routines in Talend?
- How to migrate a Talend job to Databricks?
- My project has financial data processing – how to validate this data?
- What are all normal forms?
- What is a pivot table? Difference between denormalization and pivot table?
- Difference between LEAD and MAX functions in SQL?
- Difference between RANK and DENSE_RANK?
- How to generate a sequence / auto-generated sequence in SQL?
- I have 4 cricket teams, I need combinations between those teams. How to do this in Python?

## Rishabh@1  (22)

- What all Python libraries using pydicom?
- What is my first landing zone once I have extracted data?
- First we stored data to blob, now what next, what kind of transformation and what is the final landing or presentation layer?
- Where was data stored the final one?
- In data warehousing, what exactly is it and why do we go for it?
- What is medallion architecture and how is it different from regular data warehousing format?
- What are Inmon and Kimball methodologies?
- What is the difference between data mart and data warehouse?
- What is your take on ETL vs ELT?
- What are SCD types?
- Describe a complex data scenario you encountered in ADF.
- How did you solve it?
- How to validate the data? Approach?
- How to go about optimizing slow pipelines?
- In performance improvement of a pipeline using stored procedure, how to make use of CTEs and subqueries, and which one to choose?
- What is your approach when there is an issue with performance or stored procedures?
- What about looking into indexes when dealing with relational tables?
- What is your experience with big data?
- How do you manage data security? What is your experience?
- How does Unity Catalog restrict data?
- If you are given data to analyze, how would you approach understanding the data and coming up with something insightful?
- What is data profiling and what tools are used?

## Interview Introduction Preparation  (16)

- What is the medallion architecture and how is it used in Databricks?
- In Spark, when a job is executed from code to execution, what happens in the backend and what are the steps involved?
- What are narrow and wide transformations in Spark?
- What are the two types of partitioning methods in Spark?
- What is the difference between repartition and coalesce?
- What optimization techniques did you use in your project?
- What is Z-order optimization?
- What are views in Databricks?
- What are the types of views?
- Explain an ETL process you did for a project.
- How do you convert a string column to datetime in PySpark?
- Explain the code: df = df.withColumn('event_date', to_date(col('event_date'), 'yyyy-MM-dd')).
- How do you identify incremental data in a source and bring it to Databricks?
- What is SCD in merge into statements?
- What is a Delta Lake table?
- What is ACID and what does it stand for?

## SQL Query for Mumbai Employees  (1)

- ~ Write a SQL query to find employees who live in Mumbai.

## ADF Data Engineer Interview  (16)

- How do you segregate data and implement layers in ETL pipelines?
- How do you handle schema drift or schema changes in a pipeline?
- If a source has 10 columns and you do a full load into Fabric, but the next day the source has new columns, how do you handle that?
- How do you handle the bronze to silver layer transition in a Fabric environment using PySpark, especially when schema evolution is not enabled?
- Have you worked with semantic models or Power BI?
- How do you optimize data ingestion pipelines when you only need to pull thousands of records from a huge dataset?
- How would you handle visualizing billions of rows of sales data (20 years) in Power BI when performance is slow?
- If aggregation still results in slow performance, what additional steps would you take?
- What is lazy evaluation in PySpark?
- What are the main differences between Azure Data Factory and Azure Synapse Analytics?
- What are the feature differences and how do you design a pipeline in each?
- How would you design an incremental pipeline in Fabric that ingests 100 tables, and what considerations would you take into account for a metadata-driven approach?
- What is Direct Lake storage in Fabric?
- What are shortcuts in Fabric?
- How do you create a shortcut in Fabric?
- What is a lakehouse and a warehouse?

## Rahul - DE - 4:20PM  (3)

- How to optimize performance in Spark?
- What are broadcast variables in Spark?
- How does Spark achieve fault tolerance?

## GCP Data Engineer Interview  (10)

- How do you handle architecture for data coming from Kafka to Bronze and Silver layers?
- How do you handle deleted records (CDC) in a Kafka pipeline (e.g., Shubham Gupta IDs 101 and 102, with 102 deleted)?
- How do Databricks connected sources use Unity Catalog for queries?
- What is a federated query?
- Why do you need external storage when connecting from one Databricks project to another?
- Explain the Teradata to Databricks migration process.
- How did you use PySpark scripts in the Teradata to Databricks migration?
- How did you check the DDL and ensure success of PySpark scripts?
- How do you optimize after running ETL for 10k tables (assuming each table had 2 tables and 2 queries)?
- Tell me about Databricks.

## Pawan | 4:30PM  (2)

- What is a Prefect template and its use?
- Explain subscriber flow, message broker, and event apart — how are they used?

## AWS PySpark SQL Interview Prep  (5)

- How does Kinesis connect with AWS Glue?
- What is the difference between Redshift and Databricks?
- As a data engineer, what questions would you ask a client to understand their use case?
- If an ETL job is taking more than 3 hours, what questions would you ask to identify the root cause?
- For an ML project, should I use AWS Glue or Databricks?

## Profile Intro and Projects  (4)

- ~ Write a query to find the third highest salary from an employees table.
- ~ Make the following query parameterized: SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 2;
- ~ Fetch employees whose salary is greater than their manager's salary.
- ~ Provide a Month-over-Month (MOM) analysis for the above chat.

## Interview Prep Cloud Spanner  (16)

- How is Cloud Spanner different from traditional databases in terms of architecture and consistency?
- What factors should be considered when designing primary keys in Cloud Spanner?
- How does interleaving tables work in Cloud Spanner and when should it be used?
- How do you map a Cosmos DB JSON document into Cloud Spanner?
- How do you query indexed JSON data efficiently in Cloud Spanner?
- How do you design a web-based migration?
- What bulk load options are available for Cloud Spanner?
- How do you migrate data from Teradata to Cloud Spanner and what are the things to consider?
- What challenges could be faced during migration?
- How do you validate data correctness after migration?
- How do you ensure idempotency during incremental load?
- How do you diagnose slow queries in Cloud Spanner?
- What causes hot spotting in Cloud Spanner?
- How do you design rollback and recovery during migration?
- How do you handle billions of rows and mixed access patterns in Cloud Spanner?
- How do you monitor Cloud Spanner system post-migration?

## Ajinkya @4  (17)

- What is the schema registry in Kafka?
- Which strategy have you used for Kafka connectors?
- What configuration did you do for backward compatibility in Kafka?
- What configuration is needed for the S3 connector?
- How much data is flowing through the connector?
- How do you ensure performance of the connector?
- What configuration did you use for Debezium?
- CDC data flows through a source connector with performance issues and deletes not happening – what could be the reason?
- Write a simple merge query in BigQuery.
- How to avoid a full table scan in a merge query when using ON T.id = S.id?
- What is the cleanup policy in Kafka?
- What values can the Kafka cleanup policy take?
- What is the maximum workload for a Kafka connector?
- For 20k records, how many tasks should be configured on the connector?
- How many connectors should be used for a given workload?
- Is the processing sequential or parallel?
- How does a parallel connector work with 3 connectors and 6-7 tasks?

## SQL Filtering and Ordering  (2)

- ~ How do you use a subquery to find the third lowest value?
- ~ How do you use a join for this query?

## Interview Preparation Introduction  (4)

- When should I use partitioning and indexing in BigQuery in this scenario?
- What are views? When would you use a materialized view?
- Why is a materialized view useful for AppSheet?
- What is BigQuery in simple terms?

## Rishabh@3  (4)

- What things should you take care of and what steps should you follow to build a dashboard when given a task?
- What would you do if you see two similar reports for different stakeholders that have the same data but show different numbers?
- If different reports should have the same data for a particular tile, what can we do to ensure consistency?
- When you have a website or portal for a company, if traffic increases but sales decrease, how would you identify the reason and what steps would you take to understand the gaps?

## Interview Intro Prep  (26)

- Is it a full load or incremental?
- How did we make the incremental extraction of data in the pipeline?
- What was the role of DBT in this project?
- Did DBT handle data quality checks and what transformations were used? How did you create transformations?
- What is Jinja in DBT? Give a basic example.
- How did we call DBT models through Airflow?
- What did I build in GCP?
- Explain the Pfizer and GCP use case.
- When extracting data from an API, what controls would you implement to avoid pipeline failure?
- What is the approach for handling failed jobs and rerunning from the failed position?
- In a 3-layer architecture (raw, refine, curated), if raw and refine succeed but curated fails, how do you ensure rerun starts from curated instead of raw?
- You receive a JSON file every day with predefined columns. One day the JSON changes from 100 columns to 12 columns. How do you handle this?
- Explain the Xebia lambda use case.
- Why use both Athena and Redshift?
- Compare tools of AWS and GCP.
- Where did you utilize Azure and its services?
- What is data warehousing? What is the difference between data marts and data warehouses, and what are the approaches and standards?
- Explain the medallion architecture.
- What part of ADF did you use in your project?
- What are the use cases for ETL vs ELT and when should you choose one over the other?
- When using blob storage and S3, in which format do you store data and what is the reason for each format?
- Have you done any parsing of data from PDF and CSV files?
- What challenge did you face in the project and what was the solution?
- Why was Kafka implemented in your project and how was it used?
- Compare Pandas and Spark for data processing.
- What is the approach for optimizing SQL-based processing?

## Tech Intro and Projects  (7)

- How would you design a Cloud Spanner schema to efficiently store and retrieve data from Azure Cosmos DB?
- How would you design a bulk load pipeline for incremental sync to migrate data from DB2 and Cosmos DB into Cloud Spanner?
- How do you tune Cloud Spanner queries and indexes for bulk-loaded datasets that cause read data latency?
- Describe the approach for incremental data sync from DB2 using CDC or timestamps from Cosmos DB change feed into Cloud Spanner.
- Given a DB2 database with parent-child tables, how would you redesign them in Cloud Spanner?
- How would you design a Cloud Spanner schema to support real-time data validation and reconciliation?
- What is a significant data queue in Cloud Spanner, and what steps would you take to resolve the queue and optimize query performance?

## Rishabh@12:30  (4)

- How would you standardize KPI extraction from client data with different terminologies?
- How do you handle unstructured data from PDFs?
- How do you standardize data when one client calls a KPI 'x' and another calls it 'y'?
- How would you set up a real-time analytics dashboard processing 4GB/hour from 5-10 APIs?

## Rahul - DE - 5:00PM  (2)

- What is your experience with Snowflake and Airflow, and what challenges have you faced while working on projects using these technologies?
- How do you handle data when web scraping is involved and the data is loaded into Snowflake?

## Rahul - Golang  - 3PM  (3)

- What kind of data and library did you use for telemetry in your Eicher project?
- If a vehicle sends data constantly and the server is down in production, how can we retrieve the data?
- What is your experience with Kafka?

## Interview Introduction Prep  (6)

- What is the difference between zero-copy cloning and time travel?
- How do you use zero-copy cloning and time travel in a production environment?
- What are the benefits of CTEs versus subqueries?
- What are the best practices for an Airflow DAG to be production-ready?
- When should you use a sensor versus an operator in Airflow?
- Describe how to implement data quality checks in a data pipeline and what to check.

## RAHUL L1  (3)

- What NoSQL database did you use?
- Write an SQL query to get total revenue in descending order.
- How to identify non-active users?

## Coding and Tech Q&A  (3)

- What is the process for migrating from Teradata to Databricks, specifically for AI-based workloads?
- How do you normalize and validate extracted data before running business rules?
- What methods are used to benchmark accuracy of extraction?

## Resume Intro and Projects  (4)

- What are the key differences between Power BI and Tableau?
- How do you handle data connectivity in Power BI vs Tableau?
- Explain how you would optimize a Power BI dashboard for performance.
- Describe a challenging problem you solved using Tableau.

## Interview Prep Tech Intro  (1)

- What are data validation techniques?

## 7+ Years Backend Experience  (1)

- Write a SQL query to find the sum of transaction amounts from the last 7 days.

## Interview Prep and Tech Stack  (5)

- What SQL would be generated in the backend for this query?
- If we have 1 million books, what would you change in that case?
- What does SELECT department, COUNT(*) FROM employees do? Will it work?
- How would you delete duplicate rows (by email) from a users table and keep the latest record (max id)?
- Is this DELETE query optimized? DELETE FROM users WHERE id NOT IN (SELECT MAX(id) FROM users GROUP BY email); Explain and fix if wrong.

## Go Code Refactoring Standards  (1)

- Explain the approach of this SQL query.

## Ajinkya @4  (22)

- What does Boomi do?
- What are EDI documents?
- Describe your experience with Kafka Connect and the sources you have worked on.
- What is the difference between doing CDC with MySQL and PostgreSQL?
- What is a replication slot in PostgreSQL?
- When the source is S3, what file format should be used when capturing CDC and the source is S3?
- What is the average file size in S3?
- What should be the flush interval for a job if the file is 1 GB long?
- In CDC, how many records should be retrieved?
- What is the average event size in Kafka?
- Can there be 20,000 events in 10 minutes in Kafka?
- How big is the data when the event size is 100 KB?
- If you are getting files of 1 GB with 20,000 records per GB, what challenges did you face with the S3 connector?
- How did you handle the Java heap issue when holding ~1 GB of buffered records per partition increased JVM heap usage?
- Where was your Kafka connector deployed in your project?
- As a data engineer, if a backend table is huge (over 2 billion records) and used only for analytics, and the backend team wants to move it to a data lake and then to PostgreSQL (with the data lake having everything and PostgreSQL having 3 months of data), how would you use Kafka Connect to achieve this?
- The backend will have only 90 days of data. How would you handle that?
- How can you ensure historical data is stored into S3 if not using PySpark, using Kafka Connect?
- Suppose a single record with id=1 and name='abc' gets updated 3 times in a day (insert, update, update). How will you reconstruct the table so only the last record is shown, and how will you prevent deletes from appearing in the final table?
- Given the query: SELECT id, name FROM (SELECT id, name, ROW_NUMBER() OVER (PARTITION BY id ORDER BY event_time DESC) AS rn FROM bronze_cdc_table WHERE op != 'D') t WHERE rn = 1; how will you pick up the latest record?
- Categorize events into valid and invalid: user ID cannot be null, amount > 0, currency should be INR or USD.
- How can you convert the same code to PySpark?

## Rahul - SAP - 11AM  (6)

- Explain the deduplication process. How do we merge records when a person has multiple email IDs and other attributes like first name, last name, phone number?
- What is checkpointing in streaming data processing?
- Explain how Delta Lake works in Databricks.
- What are the features of Delta Lake?
- How can I build a report after deleting records in Delta Lake?
- Write a SQL script to calculate balance from transaction data.

## Profile Intro and Projects  (2)

- Analyze this SQL query: SELECT e.id, e.name, e.dept, e.desig, e.sal FROM employee e JOIN address a ON e.id = a.id WHERE a.city = 'Mumbai';
- What is the difference between a primary key (PK) and a foreign key (FK)?

## Data Engineer Role Intro  (2)

- What was the project, what challenges did you face, and how did you overcome them?
- Tell me more about the ETL pipeline in one of your projects.

## Cloud Spanner Interview Prep  (8)

- What is your experience with migrating some RDBMS to Google ecosystem?
- If you have a full database structure (tables, stored procedures, triggers, views, CTE), how would you migrate this to Cloud Spanner? What approach and tools would you use?
- Consider an e-commerce scenario with heavy read traffic: how does the load affect Cloud Spanner during a festive burst (e.g., some products very popular, some not)? How would you distribute this load so it doesn't break?
- How do you avoid hotspots in Cloud Spanner?
- Are there any limits on fetching data or how much can be fetched at a time?
- From a migration perspective, what tooling can you use when migrating from RDBMS to Spanner?
- Suppose a business team decides to change prices suddenly (batch write operation) across all products while the application is live. What kind of issues can arise and how can you do this without issue?
- How do you monitor query performance and find slow queries in Spanner?

## Interview Prep for AIOps  (1)

- What is big data?

## SQL Joins Explained  (1)

- ~ What are SQL joins and how do they work?

## Ajinkya @3  (14)

- Why use PostgreSQL as source and S3 as destination in the CorpStage context?
- What was the volume of data ingested?
- How did you handle that volume of data?
- What was the source of millions of records?
- How did you handle PII and PHI data masking in healthcare?
- How did you hash identifiers for confidentiality?
- Write a query to create a view that repeats a sequence: given column A in table numbers, create column D (rep_seq) that repeats seq*2 as many times as seq.
- How does the generate_series function work in PostgreSQL?
- Explain the WITH RECURSIVE query that generates a repeating sequence based on the numbers table.
- Write a simple query using the numbers table to generate a sequence without using generate_series.
- Write a query for clean transactions that removes deleted ones and keeps only the latest state.
- Explain the clean transactions query line by line, including the ROW_NUMBER and filtering logic.
- If a row is deleted, should its ID be included in the results?
- Design a schema where one student can only take one major class (e.g., Science or Maths as tougher subjects). What problems exist in such a design?

## EDA with Kafka  (2)

- ~ What is event-driven architecture with Kafka?
- ~ How do you optimize database transactions with high concurrency?

## SQL Performance Tuning  (1)

- ~ How do you tune SQL queries for performance?

## Interview Preparation Script  (2)

- Where should you use BigQuery in a Google Apps Script project?
- How do you effectively use BigQuery in Google Apps Script to process data?

## Node.js NestJS Microservices Interview  (1)

- What is Kafka and how is it used in microservices?

## Interview Prep Ext JS  (2)

- Write a complex SQL query involving joins and optimizations.
- How do you manage document-based data and aggregations in MongoDB?

## Atesh Nodejs AWS Technical  (5)

- How was Kafka used in your case, and what was the reason for using it?
- Explain the architecture of Kafka.
- How does Kafka ensure message ordering? How would you handle priority processing (e.g., payments need to be processed first) in Kafka?
- How do you handle message retries in Kafka?
- Once an event has failed, what happens to that event during retries?

## Sencha Ext JS Interview Prep  (8)

- How do you optimize large queries in MySQL and what commands check query optimization?
- Given a students table with marks, student ID, and student name, how do you get the 3rd highest marks?
- What is partitioning in databases?
- On what basis is partitioning done?
- How do you perform joins in MongoDB?
- What does EXPLAIN SELECT COUNT(1) FROM employee; do in MySQL?
- What is sharding?
- What are read capacity units?

## Interview Prep Tech Stack  (8)

- What was the data volume in your pipeline?
- Apart from Airflow, what tools did you use?
- Have you ever exposed APIs for reporting?
- How did you expose the APIs?
- How did you deploy the pipeline?
- Design a pipeline that fetches historical data and daily incremental updates, and expose an API to query the data.
- How would you ensure the pipeline fetches data daily?
- How would you modify the code to initially load five years of historical data and then only run incremental updates?

## Interview Preparation for Senior Data Engineer  (6)

- How do you handle APIs and JSON in Matillion?
- If an API returns millions of records, how can you work with Matillion and that API in terms of performance?
- How do you performance tune SQL when, during the ETL process, you sometimes fetch data one by one or get data from silver or gold layers directly? For example, if you have a stored procedure that is not working properly in terms of performance, how would you solve this?
- Assume you have a 200-300 GB CSV file in S3, a very large file landing in S3 or an SFTP location. How would you load that data?
- What databases have you worked on?
- What is your experience with Databricks ETL?

## Interview Prep and Answers  (9)

- How to create triggers in AWS Glue? What are the different types?
- When a file lands in S3, how does it trigger a Glue job? Which trigger type is used?
- Given a table 'sample' with one column of colors, write a query to show 'yellow' at the top and the rest in any order.
- How can you achieve the same without using a CASE statement?
- Provide an approach without using ORDER BY or CASE statement.
- Explain the following query: SELECT DISTINCT num FROM (SELECT num, LAG(num,1) OVER(ORDER BY id) AS prev1, LAG(num,2) OVER(ORDER BY id) AS prev2 FROM sample) t WHERE num = prev1 AND num = prev2;
- What will be the output of the above query?
- Solve the same problem including the id column.
- Write the solution using a CTE instead of a subquery.

## Interview Introduction and Project Overview  (1)

- What is the difference between BigQuery and a traditional DBMS?

## Rishabh as Pankaj @ 2:30  (7)

- What were the data sources in the Homecure project?
- Was the data streaming or batch?
- Explain the flow of streaming and what tools were used.
- What was the source and why was streaming needed? Which API was used?
- Where were these APIs deployed?
- At the reporting level, what did you do and what was the goal?
- What was the use case we wanted to show at the end at the reporting level?

## HR Screening Interview Answers  (1)

- What other tools exist for data visualization and handling?

## Interview Introduction Guide  (7)

- What is medallion architecture?
- How do you implement medallion architecture in Databricks?
- What is the difference between RDD, DataFrame, and Dataset in Apache Spark?
- How do you optimize a large Spark job?
- Explain the difference between wide and narrow transformations in Spark.
- How do you handle failures in Databricks?
- What is Spark Structured Streaming and how does it work?

## CTR Prediction System Design  (4)

- How would you design the feature pipeline to support frequent updates and new features for an online learning system for ad click-through rate (CTR) prediction using classical ML?
- How would you choose and train models to support incremental or frequent retraining for CTR prediction?
- How would you detect and respond to performance degradation in production for a CTR prediction system?
- How would you balance exploration vs. exploitation (e.g., via A/B tests or bandit-style approaches) while minimizing business risk in a CTR prediction system?

## Interview Prep and Intro  (6)

- How would you handle data ingestion from multiple sources of different types (API, NoSQL, relational DB)?
- How do you implement event triggers in Airflow, and how do you manage task dependencies and parallelism?
- How do you set up alerting when a job is completed and integrate it with Slack?
- How do you perform multiple updates in one table in BigQuery?
- How would you handle large-scale parallel updates in BigQuery to avoid locking issues?
- How would you build a dashboard combining JSON event data and SQL user data, and what KPIs would you include?

## Interview Preparation Data Engineer  (7)

- What is Matillion?
- In Matillion, what are the ETL architectures for migration-based ETL?
- What are the differences between orchestration jobs and transformation jobs in Matillion, and when would you use each?
- How would you implement incremental loading in Matillion with billions of records?
- A Matillion pipeline loading data into AWS Redshift takes almost 4 hours; how can you optimize it?
- How do you implement error handling and logging in Matillion pipelines?
- How would you process 500 million records using Matillion and a data warehouse?

## Interview Preparation BI Developer  (5)

- How did you handle real-time data using Kafka in your project?
- What SQL activities did you perform in your BI project?
- How many data sources did you use in Power BI Desktop and why?
- How do you publish a report to Power BI Service and how does it appear in the service?
- What is SSAS and how do you connect and develop reports with it?

## Data Engineer Interview Questions  (1)

- What are the common questions asked during a data engineer interview?

## Data Engineer Interview Prep  (5)

- What is the role of a data engineer and what are common interview questions for that position?
- Give me scenario-based data engineering questions for an interview.
- What are some interview questions related to Databricks?
- What are some interview questions related to Delta Lake?
- What are some interview questions related to ETL processes?

## Interview Introduction Prep  (13)

- How to implement SCD Type 2 in SQL on BigQuery or Snowflake, including handling overlaps and late updates?
- A business metric defined in SQL is giving inconsistent numbers across dashboards. List a structured debugging approach using only SQL and metadata tools.
- In BigQuery, a table has queries with full scans and high cost. How to reduce cost and improve performance?
- A client complains that the same Snowflake query is fast in dev but slow and costly in prod. What to investigate?
- What types of tests should be put in a new ETL pipeline (unit, integration, data quality) before prod ready?
- How to structure unit tests in Python transformations and SQL models so that they can run in CI and catch regressions?
- A client needs clear SLAs and SLOs for a data pipeline. How to define, measure, and report on reliability and data quality targets?
- How to structure a small Python ETL script so it is testable, configurable, and can be promoted to prod with minimal changes?
- A Python pipeline intermittently fails due to transient network issues. How to design robust retry, backoff, and idempotent mechanisms?
- How to structure a dbt project for a new source, including staging, intermediate, and mart layers? Describe the folder and model layout.
- Describe a typical dbt folder structure (models/staging, models/intermediate, models/marts) and what kind of models go into each layer, including where you define sources and tests.
- How to design dbt tests (generic and custom) to enforce primary keys, uniqueness, referential integrity, and basic metric sanity checks?
- Describe how to set up end-to-end CI for dbt (env, seed data, tests, slim CI) to catch breaking model changes before merge.

## Interview Introduction Guide  (4)

- What were the data sources in the cloud quote project?
- What are the other data sources?
- What is the use case of this architecture in the cloud quote project?
- What is the business logic in the gold layer?

## DataOps Engineer Interview Prep  (7)

- Explain a project where you used Databricks.
- Which components have you used in Databricks?
- How do you optimize performance in Databricks jobs?
- Describe an end-to-end data pipeline you built.
- What tools do you use for ETL and ELT?
- How do you handle failures in ETL?
- How do you schedule and monitor pipelines in Apache Airflow?

## Atesh @ 7:30 p.m  (4)

- How would you approach a media data migration from a source system to a target system using AWS services?
- Given a source system with APIs, what is your high-level approach for the migration?
- If you need to figure out a Cultura API, what is the first thing you would do?
- How would you conduct step 1 investigation for the migration?

## Interview Project Explanation  (31)

- What were the challenges you faced in your Power BI projects and how did you overcome them?
- Explain your experience with Power BI in a tabular format including projects you worked on.
- What steps do you take to fix performance issues in Power BI as a senior BI developer?
- When would you use Direct Query vs Import mode in Power BI?
- What are the benefits of Direct Query over Import, and what are the flaws of Import?
- If my dataset size is 3 GB, but I don't want to import all data, what can I do?
- What is required to set up incremental refresh in Power BI?
- Explain the difference between Drill Down and Drill Through in Power BI.
- How can I compare current month data with previous month data in a Power BI report?
- Can I perform month-over-month comparison within a single table?
- Write a DAX measure to fetch specific department names based on a slicer selection.
- What is the difference between SELECTEDVALUE and other similar DAX functions?
- If no value is selected in a slicer, what will SELECTEDVALUE return?
- How can I bring product information from a master table to a sales table in Power BI?
- What is the design approach for handling many-to-many relationships in a data model?
- Can you use both Import and Direct Query in SSAS tabular models?
- How do you add calculated columns in a tabular model?
- Is Power Query case-sensitive?
- How do you set up a Power BI report to refresh every 30 minutes?
- What is the difference between Power BI Pro and Power BI Builder?
- Why would you need a paginated report in Power BI?
- How do you implement security in Power BI service to control who sees what?
- How do you choose which visualization to use in different scenarios?
- How have you integrated Power BI with other applications?
- What methods do you use for data profiling as a BI professional?
- What are window functions in SQL and give an example query using RANK?
- What is a Cartesian product? Provide a query example.
- Is a FULL JOIN a Cartesian product? If so, write a query. If I have 4 records in each table, how many rows will I get?
- What is the difference between EXISTS and NOT IN in SQL?
- What documentation do you produce for an end-to-end BI project?
- What are possible KPIs for a crypto reporting dashboard?

## Interview Introduction Preparation  (11)

- What challenges have you faced as a data engineer and how did you overcome them, especially in the current era of AI?
- What tools and techniques are you aware of and experienced in?
- Explain the architecture of your current project end to end in Databricks.
- How do you schedule jobs in Databricks?
- Tell me about a complex PySpark job you wrote and what problems you solved.
- How do you handle performance issues in PySpark jobs?
- Have you ever worked on streaming pipelines using Apache Kafka? Describe a real use case.
- What are event sources?
- What is event sourcing and how does it relate to Kafka pipelines?
- If a pipeline fails in production and downstream steps are impacted, how would you handle it?
- What types of transformations happen in each layer of the medallion architecture?

## Interview Prep for Data Engineer  (6)

- How did you use Databricks in your project?
- How does Delta Lake ensure ACID compliance?
- What is time travel in Delta Lake and when have you used it?
- What database components are there and which have you worked on?
- Describe a project where you used Databricks, including challenges and orchestration from scratch end to end, and how you overcame them.
- What is z-ordering?

## AI Lead Interview Prep  (1)

- How do you manage sensitive data, PII, and masking, and at which stage of the pipeline?

## Profile Intro and Project Details  (12)

- What SQL optimizations did you implement in your project?
- Given 1000 lines of SQL with 20 inner-joined tables, how would you optimize it? Provide bullet points.
- In which cases is data partitioning beneficial?
- What was the nature of the data in the Homecure project?
- How did you use Google Tag Manager (GTM) in the Homecure project?
- For an e-commerce platform like Myntra with 20 categories and high traffic, what top 10 KPIs would you show on a dashboard?
- How would you design a data warehouse for unstructured data (reviews, text, images) for Myntra, especially for storing reviews?
- What are the benefits of PySpark over traditional SQL?
- What is the difference between a DataFrame in PySpark and a DataFrame in Pandas?
- When would you use broadcast variables in Spark?
- Describe your experience with Meridian MMM.
- What is MMM validation and what is it?

## Interview Intro and Projects  (6)

- What is your experience with Snowflake?
- When processing initial data, what ETL tools do you use?
- What is your exposure to Hadoop in ETL and development?
- What challenges have you faced with Snowflake ETL and how did you overcome them?
- What about data ingestion challenges?
- I have a source with millions of records; I want to add some changes every night but not all records. How would you handle this?

## Interview Introduction Prep  (5)

- What GCP services have you used?
- What is the difference between Cloud Storage, BigQuery, and Cloud Spanner?
- How do you secure data in GCP?
- Explain the difference between streaming inserts and batch loads.
- How does autoscaling work in Dataflow?

## Data Engineering Interview Prep  (7)

- What are the top 3 challenges you faced with optimization and scalable systems?
- How did you optimize data skewness?
- What architecture was used in Enfrontier?
- What is the architecture of the Enfrontier project?
- What architecture was used for this particular project implementation?
- What is the difference between silver and gold layers?
- What is curated data?

## Shreya AI engineer  (1)

- I have two DataFrames, one with company info of employees, one with personal info, only employee ID common. How to join using employee ID using pandas?

## Rahul - Data Engineer 5PM  (4)

- How do you optimize a query in BigQuery?
- When would you choose to use Data Proc in a given situation?
- How do you manage encryption and transit in GCP?
- What are the key components of a data platform in GCP?

## SQL Consecutive Dates Streak  (3)

- ~ How does changing the approach from using DATE minus ROW_NUMBER to using self joins affect the performance and complexity of finding consecutive login streaks?
- ~ What is the risk of O(n^2) complexity with self joins? Clarify how the ROW_NUMBER method achieves better performance, especially with millions of rows.
- ~ How does the ROW_NUMBER method handle the scenario where a user logs in on non-consecutive days but with multiple gaps of one day? Does the grouping logic still correctly identify separate streaks?

## Data Engineer Role Intro  (9)

- Explain the Databricks pipeline architecture and components.
- How does Delta Lake ensure atomicity, consistency, isolation, and durability (ACID) in transactions?
- What is time travel in Delta Lake and how have you used it in a project?
- How do schema enforcement and schema evolution work in Delta tables?
- How do you optimize a database schema design and related job performance?
- How do you handle the small file problem in Databricks?
- Explain the Databricks architecture.
- What is the difference between all-purpose clusters and job clusters in Databricks?
- How do you manage and optimize cluster costs in Databricks?

## Interview Prep Data Engineer  (4)

- Explain the GCP services used in your recent projects.
- What is the difference between Cloud Storage, BigQuery, and Cloud SQL?
- Explain what BigQuery is and why we use it.
- How do you handle late arrival data in Dataflow?

## HL7 FHIR APIs Explained  (1)

- ~ What is HL7/FHIR APIs?

## Ajay Data engineer @1  (14)

- What are common issues and fixes for Kafka upstream related errors?
- How did you manage PII, masking, and data sensitivity while working on Ikano?
- Does masking happen during processing or when data is stored? Does GTM collect complete information and does BigQuery save it fully?
- Did you work with the default BigQuery schema that gets created when tagging a website or an app?
- How do you unnest nested data in BigQuery? For example, Flipkart data comes in nested form in the default schema.
- What orchestration tool did you use at Kantar?
- Which window functions have you used at Kantar?
- What visualization tools have you worked with?
- If you have n categories and you want to show revenues in a chart, which kind of chart should you use?
- If you want to compare with last year's data, how do you proceed?
- Explain the Lufthansa project, the tech stack, and the end-to-end data flow.
- What was the challenge you faced during migration in the Lufthansa project?
- How do you handle deduplication in parallel pipeline execution?
- If Flipkart approached you to design a delta lake, what key steps would you take and how would you prepare?

## Interview Preparation Guide  (15)

- How did you optimize a PySpark job while transforming legacy T-SQL processing into a scalable Databricks pipeline using FireSpark and T-SQL?
- Describe a challenging scenario where the join strategy in PySpark impacted job efficiency. How did you decide which join to use?
- How do you monitor and maintain these optimized PySpark job performances in Databricks?
- How would you refactor a complex legacy ETL pipeline using Databricks and PySpark to improve scalability and maintainability?
- How do you analyze complex T-SQL scripts to identify essential business logic before migrating those processes to Databricks?
- How do you ensure data integrity when translating T-SQL transformations into PySpark code?
- How do you differentiate between T-SQL features and Databricks capabilities when implementing data transformations, especially when some SQL features do not have direct equivalents in PySpark?
- Can you describe how you debug and optimize PySpark code derived from T-SQL to ensure both performance and accuracy?
- How do you differentiate error handling approaches in Azure Data Factory pipelines when dealing with various source systems?
- Can you explain how you analyze and optimize data ingestion pipelines in Azure Data Factory to improve their performance and reliability?
- How do you monitor data flows and ensure data quality consistently within Azure Data Factory pipelines?
- How do you analyze and identify bottlenecks in Databricks data pipelines to optimize their performance?
- Can you break down your approach to tuning read and write operations in Databricks for better efficiency?
- What monitoring tools and metrics do you use to diagnose and improve the scalability and speed of Databricks pipelines?
- How do you approach establishing best practices within the data engineering team, and what specific strategies have you implemented to ensure team members adhere to those practices?

## Interview Prep Tech Stack  (12)

- Explain the utilities in Databricks.
- What is the difference between batch and real-time data processing?
- Walk through how to debug issues in Databricks and approaches for removing performance bottlenecks.
- What are narrow and wide transformations in Spark? When to use each?
- What is metadata management and how does it relate to data governance?
- How to handle sensitive data in data pipelines?
- How to optimize a data pipeline? What approaches?
- How to provide low latency in a data pipeline?
- Suppose a pipeline is failing with OOM issues and GC time is red. How to troubleshoot and fix?
- Explain repartitioning and coalesce in Spark. When to use which?
- Explain the SQL query that finds 3 or more consecutive days with at least 100 people, line by line.
- Write the same logic using PySpark instead of SQL.

## Interview Intro and Projects  (6)

- Which Python libraries have you worked with?
- Can you briefly explain how you have used pandas?
- How do you handle null values?
- What EDA did you perform in your projects?
- How do you ensure data reliability?
- Can you explain the 'mecure' project?

## Data Engineer Interview Prep  (13)

- What data cleaning and validation techniques did you use in data ingestion?
- How was error logging handled in your pipelines?
- Have you designed end-to-end data pipelines?
- What was your approach for full load and incremental load in pipelines?
- How do you handle late arriving data?
- Where have you used pandas and what other libraries have you used?
- Where have you used partitioning and clustering?
- Can you explain snowflake schema and star schema in data modeling?
- When did you split a dimension table in your project?
- What performance challenges did you face and how did you solve them?
- What migrations have you performed?
- What comparisons do you consider to deem a migration successful?
- What monitoring tools have you used and what common issues did you encounter?

## Resume and Tech Stack  (4)

- Explain the medallion architecture in your project.
- How do you join two tables in PySpark when one has 1 million rows and the other has 5 million rows with skewness on user_id?
- How do you manage data from JDBC for Teradata?
- How do you migrate from SaaS to PySpark?

## Rahul - Data Engineer 11:30AM  (19)

- What data cleaning and validation did you use in data ingestion?
- How was error logging done in pipelines?
- Have you designed end-to-end pipelines?
- What was your approach for full load and incremental load?
- Where have you used Pandas and what libraries have you used?
- In data modeling, what is the difference between snowflake and star schema?
- In your project, when did you split dimension tables?
- What performance challenges did you face and how did you address them?
- How do you design an end-to-end pipeline with different sources and what validations do you perform?
- How do you handle error data logging?
- What is the difference between ELT and ETL, when to choose which, and what are the advantages and disadvantages?
- What challenges did you face while designing data modeling, and how did you implement star and snowflake schemas?
- How was the deployment done in GCP? Explain.
- What is the frequency of the deployment?
- What common issues arise after deployment?
- What is least privilege access and why is it important?
- What is your experience with security in GCP and what strategies did you use related to each service?
- How do you optimize performance in SQL and BigQuery?
- Why is orchestration needed and what are its benefits? How do you handle multiple processes and orchestration steps, and what tools do you use?

## Ajay as rishabh 2:00  (14)

- How do I upload a CSV to a Databricks notebook?
- How do I upload files to a volume in Unity Catalog?
- How do I upload data to DBFS?
- How do I print data in Databricks and perform operations on it?
- How to handle dynamic age parameter when age groups are dynamic?
- How to resolve 'public DBFS Root is disabled' error when writing partitioned parquet?
- What are the steps to use Unity Catalog vs Volumes for writing data?
- Provide all SQL queries for business queries that will be tested.
- How to handle a team that has 90% of data when partitioning?
- Why does my window function rank filter for top 5 per team return 1700+ records instead of 5?
- Can we optimize the window function for average potential per position if players play multiple positions?
- Can we do the optimization only for the 11 standard football positions?
- How to extract valid positions from a column instead of using a hardcoded list?
- How to proceed with question C after collecting top 11 positions?

## AI Platform Engineer Intro  (1)

- How do I optimize queries in PostgreSQL?

## Interview Prep Tech Stack  (10)

- What is your experience with Dashibble?
- What is the difference between streaming APIs and normal REST APIs?
- What was your experience with data ingestion? What was handled in the backend and what was handled in Snowflake?
- In data ingestion, why use S3 instead of Snowflake stage?
- How many tables and explain the schema and relationships for Dashibble?
- Given a survey system with tables: survey, questions, respondents, and responses (many-to-many), how would you maintain relationships and data integrity, including when loading tables sequentially?
- How would you handle dynamic input conditions in a Snowflake stored procedure?
- What languages can be used in Snowflake stored procedures?
- Why can JavaScript be used for Snowflake stored procedures?
- What is your experience with Snowflake Cortex and Cortex Agents/Analysts?

## Excel to Google Apps Script Migration  (8)

- What is a view in BigQuery, and what is the difference between slices in AppSheet and views in BigQuery?
- What is a view in BigQuery?
- What is partitioning in BigQuery, and why do we need it?
- I have a stock data table with 300k rows of data, and I need to create a table in BigQuery. The data is in CSV format. How can I import it into BigQuery, and what is the process?
- When uploading a large file that takes time, how can you skip rows in the file?
- Can we skip rows using skipLeadingRows in BigQuery?
- Is there a way to avoid defining the data schema table explicitly?
- If I have GCP, what roles do I need to access BigQuery?

## Somya DE  (8)

- Explain Medallion Architecture used in the project.
- How to handle skew join in PySpark (1M vs 5M rows on user_id)?
- Explain BigQuery architecture.
- How to manage data ingestion from Teradata using JDBC?
- Explain the SaaS to PySpark migration approach.
- PySpark: filter employees with salary > 60000 and select name & salary.
- What is skew data?
- What is the difference between ROW_NUMBER and RANK in SQL window functions?

## Data Scientist Interview Prep  (1)

- What is p-value?

## Interview Preparation Guide  (12)

- What different data sources have you worked on?
- How do you ingest data from a relational database?
- What common issues have you encountered with JDBC and how did you fix them?
- What parameters do you use in JDBC for chunks and did you tune them?
- What happens if no parameters are passed in JDBC? How many executors will be used if we have 4?
- Explain your experience with ingesting files from SFTP, including the end-to-end architecture.
- If you have 500 GB of data to process in a batch, how many executors are needed so the job does not fail and runs smoothly?
- Explain your experience with Cloud Composer.
- Write PySpark code to identify employees who logged in consecutively for N days and explain it line by line.
- Why choose a rolling window of (-2, 0)?
- Do we require another window function or can we use the same DataFrame?
- Which is more effective: one window function or two window functions?

## Data Engineer Introduction  (4)

- How can you optimize a slow PySpark job?
- Write a PySpark streaming application to read data from Kafka and print it to the console.
- What is Delta Lake and why would you use it?
- Explain Databricks architecture.

## Ajay as Rishabh 6:00PM  (12)

- Describe your experience with real-time streaming in a corporate environment, including end-to-end flow and architecture.
- What kind of throughput were you dealing with and how did you handle latency?
- What kind of queue size did you have? How do you process events while maintaining queue size, given that a smaller queue may lead to loss?
- How do you handle late-arriving data and inconsistent timing in a live streaming pipeline?
- Suppose we have a watermark of half past 10, and 3 events arrive at 6:05 but the window starts at 6:00, and one event comes at 6:15. How would you handle that? How do you reprocess past events?
- How do you handle data duplication? What if you support one kind of semantics – what to do?
- Describe your experience with a slower pipeline: how did you find it, optimize it, and what did you implement? What made the pipeline slower over time?
- If there is a change in the existing pipeline (e.g., a new KPI or any change), how do you test without breaking the testing process?
- Describe your experience with DBT end-to-end.
- What problem does DBT solve and why do we use it?
- How does DBT stand out compared to other tools?
- How does testing work in DBT? What tests have you written?

## Python vs Java for DE  (1)

- ~ Why is Python preferred for data engineering over Java, even though Java and Apache are so closely bound?

## Interview Answer Style  (1)

- What will be your debugging process if you have bugs in production related to data?

## Saniya at 9:30pm  (3)

- What is event-based processing using Kafka? Provide an example of where and how you implemented it.
- What alternatives to Kafka have you worked with?
- Give an example of using SNS and RabbitMQ.

## Rishabh@5  (15)

- What is your experience with dbt? Explain the use of dbt and why it is popular.
- What quality checks do you perform while working with dbt?
- How do you schedule dbt jobs? Explain your approach to building bronze, silver, and gold layers.
- How do you handle incremental data in dbt when building bronze and silver layers?
- What kind of macros have you created in dbt? How do you handle duplication? Explain casting approaches in layers.
- How would you transform a complex 500-line MySQL stored procedure into dbt? How can macros help? What are the migration steps?
- What is ephemeral materialization in dbt?
- How do you optimize a large file processing job that is slow due to joins or operations?
- What is adaptive query execution in Databricks?
- What is Z-ordering and how do you use it to improve scan performance?
- What is your experience with streaming jobs?
- What are the different window types in streaming and which have you worked with?
- How do you handle schema changes when source data adds new columns?
- What is Amazon Kinesis?
- What is Autoloader in Databricks?

## Interview Preparation Guide  (11)

- How does schema evolution work and what are its advantages?
- Explain Delta Lake phases, time travel, CDC, and merge.
- How do you debug production pipelines?
- What optimizations have you done in your overall data engineering experience?
- In a medallion architecture with metadata project, how do you handle adding new columns mid-release? How do you identify and validate the changes?
- How do you handle deployment activities? What artifacts are involved and how do they relate to ETL tools and Data Factory?
- How do you deploy artifacts like ARM templates across environments, and how do you handle resource group details?
- How do you implement Unity Catalog in Databricks and convert a Hive metastore to Unity Catalog?
- What is the difference between clusters and serverless in Databricks, and what are the advantages?
- Explain cost optimization strategies in Databricks.
- When would you use incremental vs. overwrite write modes?

## Profile Intro and Project  (4)

- How were we maintaining the data pipelines in your recent project, including DevOps pipelines and everything in the dashboard? Describe your experience.
- What size of data did you handle in this project?
- Was it real time?
- Where was the data stored?

## AI Interview Prep Data Engineer  (16)

- Can you give a specific example of a SQL query and changes in join or indexing that improved performance?
- How do you decide which column to index in a large data warehouse? Explain the trade-offs.
- How have you used SQL window functions for transformation? Provide an example.
- Can you describe a challenging ETL pipeline you built using Fivetran and ADF? How did you ensure reliability and scalability?
- How do you monitor and handle failure in an ETL pipeline?
- How did you troubleshoot complex ETL steps to ensure failures didn't reoccur?
- What architecture choices support scalability when data volume grows?
- Describe your experience with REST API in a data workflow, including authentication, rate limiting, and error management.
- How do you handle error responses to maintain data consistency in a pipeline?
- How do you parse and transform nested JSON responses from a REST API in a data workflow?
- Describe your experience working with cloud services like Azure for data storage, processing, and ML pipelines. Also compare with AWS.
- How would you optimize a data processing workflow to handle large data?
- How do you automate deployment and manage version control for cloud data pipelines, including rollback?
- How would you implement a filter function that filters files based on content length instead of file extension?
- How does parse_filesystem handle empty folders after filtering if all folders are excluded by the filter?
- How would your function behave if it encounters symbolic links or cycles in recursion?

## Interview Prep Intro Tech Stack  (2)

- Walk through the typical process for extracting and analyzing data in an environment using Python and GCP.
- Describe your approach to clean and validate a large clinical dataset that has inconsistencies.

## Spark Optimization in Databricks  (6)

- How does Spark optimization work in Databricks?
- How do you ensure idempotency in a Spark pipeline?
- How do you ensure durability in Spark?
- How do you optimize SQL queries in Spark?
- How do you achieve data quality in Spark pipelines?
- How do you handle schema changes in Spark?

## Interview Intro and Tech Stack  (14)

- What have you implemented in terms of compliance and how?
- What is the difference between performance cookies and analytics cookies?
- What is the right approach to have correct tags when a client has many data layers?
- How do you correct wrong data in multiple data layers?
- How do you correct incorrect source data and send to downstream without changing the source?
- How to correct wrong SKU and size information in the data layer?
- Describe your experience with A/B testing.
- How many days would you run analysis to understand adoption of a new wallet payment method?
- Would you expose a new payment method to all users or some?
- What tool to use and how to decide which users to include?
- How to decide the number of days for analysis?
- How to do payload validation?
- How to handle a large payload when firing impression tags for 100 products per page?
- How to imitate all client-side tags on server side?

## Playwright vs Selenium  (3)

- ~ Given an employee table with columns id, name, salary, and department, write a SQL query to get the first highest salary.
- ~ Write a SQL query to get the 5th highest salary from the employee table.
- ~ Write a SQL query to find salaries that are duplicated in the employee table.

## Shuru Tech  (3)

- How can you make the condition for the age_group column dynamic?
- How can you get the average potential for each position and age group, then rank and filter for the top rank per position?
- In this case, what will be the count of output?

## Interview Preparation Data Engineer  (1)

- How does BigQuery pricing work and how does it manage cost?

## Interview Preparation Response  (1)

- What is your experience with Fivetran and Airbyte?

## Interview Prep Tech Stack  (30)

- How do you build a scalable data pipeline for both stream and batch data that will come to Snowflake?
- Why streaming only app events and logs CDC, why not others?
- How do you connect to the sources we need data from?
- What volume of data have you worked with and what could be the sources?
- How much amount of data were you handling with each source?
- What are the different sources you have worked with and how did you connect, given different formats?
- How do you convert structured tables to CSV/Parquet and push to S3 then to Snowflake?
- How do you load from S3 to Snowflake?
- How do you establish the connection to Snowflake using Python?
- How do you connect S3 to Snowflake, what tools?
- How do you load streaming data, what stages and how did you do that?
- Why do you need S3?
- What is the difference between Snowpipe and Snowpipe Streaming in Snowflake?
- If we are already processing the data in the processing layer, why in Snowflake layer are we loading raw, silver, gold?
- What will we do with Airflow in this pipeline?
- Once data is in raw table, how to move data to silver and gold using SQL transformations in Snowflake?
- Why clean two times? What do I answer?
- Once data is in raw table, how to move data to silver and gold automatically using SQL transformations in Snowflake?
- What is micropartition in Snowflake and how do you optimize scanning within Snowflake?
- In terms of scalability, explain in Snowflake, how to scale?
- How do you design a warehouse size, how to scale?
- What is multi-cluster?
- What is the scaling policy?
- What are streams, tasks, and dynamic tables in Snowflake?
- What is a task in Snowflake?
- What is a merge in Snowflake?
- What is the difference between a view and a materialized view?
- What are functions and stored procedures in Snowflake?
- What is your experience with handling data modeling in Snowflake?
- Assume you have a sales table and you want to find top two customers by total sales in each region, give me the query (Snowflake compatible).

## Interview Prep Assistance  (5)

- What challenges have you faced while building ETL pipelines?
- What is one significant solution you have used in a project?
- How can you design a decoupled event-driven data ingestion pipeline in AWS?
- Why do hot partitions occur in DynamoDB?
- How do you design the partition key strategy to minimize the risk of hot partitions?

## Interview Prep Snowflake AWS  (1)

- What is your experience with Snowflake and AWS as a data engineer with 5+ years of experience?

## Interview Prep Tech Stack  (9)

- How to implement a data validation framework?
- What are the challenges in SQL to Spark migration?
- Write PySpark code for incremental load.
- Suppose one day data volume increased 1x and the pipeline broke, what will you do?
- Explain how Spark DAG is built internally.
- If you need to migrate 500 SQL jobs to Databricks, what strategy would you use?
- What is the difference between data flow in copy and mapping data?
- Design a complete Azure-based data platform for ingesting 2TB/day from 50 sources with an SLA of 30 minutes and 5-year retention. What is the architecture?
- How does time travel work in Delta Lake?

## Interview Intro Preparation  (5)

- How do you select the 500k records from a large set of transactions?
- What types of sampling can be used in a credit card fraud detection scenario?
- How can you derive regional features from merchant names or pincodes?
- How do you handle scenarios where the fraud rate distribution is highly imbalanced (e.g., 0.1% vs 20% fraud)?
- How would you group or aggregate the data in such a case?

## Interview Intro and Tech Stack  (3)

- What is your experience with Azure Data Lake?
- What is your experience with ETL and ELT pipelines?
- What are the best practices for schema design in BigQuery?

## Python Non-Repeating Char  (2)

- ~ Explain how Debezium handles data schema drift.
- ~ What is Debezium?

## Interview Preparation Guide  (3)

- How do Airtable automations work under the hood?
- What technology should be used to integrate Airtable with Snowflake?
- If Snowflake data is correct but Airtable has inconsistencies, what could be wrong?

## Interview Introduction and Project  (18)

- Tell me about the live stream data or pipeline.
- What is the architecture of live streaming data?
- What happens if a Kafka event arrives late?
- How can we flag late-arriving events in Kafka?
- How do you handle the case where an event doesn't arrive and another event arrives late, requiring completion of the CT scan?
- If a chunk cannot be identified from the chunk data, how do you handle it?
- If we have a 10-minute threshold, do we wait or do something else?
- How do you handle data duplication in CT scan?
- What happens if a consumer picks up an event from Kafka but fails mid-processing?
- How do you decide the latency and queue size in live streaming?
- Give me an example of dimensional modeling from my resume.
- In which project did I use dimensional modeling?
- What does a dbt project structure look like and how can we organize it?
- How do we decide which models should be marts vs. staging layers in dbt?
- What are naming conventions in dbt?
- How do you test dbt models?
- How do you use dbt with snapshots and gain?
- What are the limitations of dbt and alternatives like SQLMesh?

## Saniya ML Eng L2  (2)

- How do you audit file tracking or processing in this workflow?
- We're using AWS Glue. How do we track the audit?

## Interview Preparation Assistance  (2)

- Write a query for employee table (name, salary, department) to get the 8th highest salary.
- Write a query to get the 8th highest salary without using inbuilt functions.

## Interview Intro Prep  (15)

- How did you approach identifying trends and patterns in large data sets?
- What steps do you take to ensure and validate reports?
- How do you handle missing and inconsistent data?
- What is ADF (Azure Data Factory) and how did you utilize it in your projects?
- Explain pipeline and linked services in ADF.
- What types of Azure storage services are there?
- How do you handle retries and failures in ETL pipelines?
- What are the steps to design a data warehouse?
- What is ETL and how did you design it in your projects?
- How do you handle data validation and error handling?
- What challenges did you face while building ETL pipelines and what were your solutions?
- What are the steps to design a database schema?
- What indexing strategy did you use to improve performance?
- Write a SQL query to get the two highest paid employees.
- Write a query to find departments where the average salary is greater than the company average salary.

## Tech Interview Prep  (5)

- Describe a complex data pipeline you designed and the challenges you faced.
- What is your experience with cloud data platforms like Snowflake and BigQuery in your project?
- How do you approach data quality and consistency in data engineering projects?
- How do you collaborate with analytics and product teams to ensure your solutions meet their needs?
- How do you stay current with emerging technologies in data engineering?

## Profile Intro and Projects  (2)

- ~ What is Unity Catalog?
- ~ How is data stored in a catalog?

## Profile Intro and Tech Stack  (1)

- ~ Describe the backend system architecture and data-related tools.

## Senior Python Dev Interview  (2)

- What did you do in data architecture for a data analytics platform?
- Describe a technical project you faced in data analytics.

## Interview Prep for Data Engineer  (7)

- Which project are you referring to?
- Why is legacy TSQL hard to maintain?
- What was your experience with the WIM project? What complexity with TSQL led to moving to PySpark, and how did that solve it?
- When you scan a website, what kind of KPIs would you build for them?
- What is your experience with Step Functions?
- What types of Step Functions are there?
- What is your experience with API Gateways (HTTP vs REST API)? What challenges did you face with REST API Gateway, and why use REST API over HTTP?

## Interview Prep Assistance  (1)

- Describe your experience with PySpark end-to-end pipelines.

## Interview Prep Introduction  (4)

- How to replace Pub/Sub with Kafka?
- How to handle failure in the middle of a pipeline?
- Compare Kubeflow, Airflow, and Argo Workflows. When to use what?
- What are the data sources on GCP? What about cross-cloud?

## Data Engineer Interview Prep  (8)

- What use cases have you implemented with Snowflake and its components in your projects?
- How have you used Iceberg tables in your projects and why did you use them?
- When would you use external tables versus Iceberg tables?
- If you have Iceberg tables or materialized views, are there any performance consequences?
- How did you approach data modeling in your projects and why did you choose that approach?
- How did you use Airflow in your project, and what challenges and solutions did you encounter?
- How do you operate and manage the pipeline?
- What is the Medallion architecture?

## Interview Prep Tech Stack  (5)

- What legacy system were they using, what database and tech stack, and what was the source and target and tools?
- What is medallion architecture, what is its use, and what problem does it solve?
- How would you approach data migration without data loss and maintaining integrity for a custom CRM application for a wide industrial use case tracking sales performance, with 30 years of historical data of unknown types and formats?
- How will we manage the pipelines, cleaning, and transformation — describe the backbone architecture.
- What about the undigitized data they have?

## Interview Prep Tech Stack  (7)

- Were the documents batched or streamed? How were they ingested?
- Presigned URL is synchronous, right? Was it streamed or batched? How did it come to the system?
- Once a file gets uploaded, what was happening in our system?
- If I am from an external system and I don't want to manually upload files one by one, but I accumulate 50-60 files daily and you have many labs, how do you process this bulk upload?
- Now files got uploaded, how do you process them in parallel?
- If we use a queue, why do we need to send 100 messages for 100 files?
- If everything is running, how would you propagate the message to the end user if all 100 files are running?

## Kafka Partition Factors  (4)

- ~ What factors should I consider when deciding the number of partitions in Kafka?
- ~ What is consumer lag and how can I monitor it?
- ~ How do you handle duplicates in a CDC pipeline?
- ~ How can I optimize Snowflake performance?

## Interview Preparation Guide  (1)

- How is document uploading included in the architecture?

## Interview Preparation Assistance  (7)

- What is DynamoDB Accelerator (DAX)?
- Why not use DAX all the time?
- How to improve query performance in DynamoDB?
- What is the difference between GSI and LSI in DynamoDB?
- How do you debug a data pipeline when you have a large volume of data?
- How to optimize SQL queries?
- How do you find out how to optimize a query?

## Interview Prep Golang Dev  (4)

- What is the problem in the query and how would you approach fixing it?
- Explain the SQL query that calculates total delivered orders and total spend per customer.
- Why use the delivered status condition in the JOIN? What would you change to not filter only delivered?
- Why use COUNT(DISTINCT o.order_id)? Why not use the product table? What would happen if you used the product table?

## Profile Intro and Projects  (1)

- Describe a project where you built a pipeline in AWS and the challenges you faced.

## Interview Preparation Guide  (15)

- How do we handle deduplication when a single person has multiple email IDs?
- What is the business requirement behind customer deduplication and identity resolution?
- How do we merge multiple identifiers (email, phone) into a single customer record?
- What fields are used beyond first name, last name, email, and phone number for identity matching?
- How do we identify and merge records when multiple people share the same name (e.g., Rahul)?
- What is the size of streaming data, and how is it measured in data engineering?
- What is checkpointing in streaming data processing, and why is it required?
- What are the different data sources we typically work with in data engineering?
- What Kafka-related and streaming data work have I done in my projects?
- How does Databricks and Delta Lake work at a high level?
- What are the key features of Delta Lake?
- How does Delta Lake handle deleted records?
- If records are deleted, how can we rebuild reports using Delta Lake?
- How do we write an SQL query to calculate running balance from transaction data?
- Can you explain the SQL logic used for calculating running balance?

## Interview Answer Guidance  (12)

- How is BigQuery different from Hive?
- Which one is faster, BigQuery or Hive?
- Convert a Hive query into BigQuery.
- How do you convert 'select id, explode(items) as items from orders' into BigQuery?
- Explain the query 'SELECT id, item FROM project.dataset.orders, UNNEST(items) AS item'; what is UNNEST and how does it relate to cross join?
- What is the difference between UNNEST and cross join?
- Write a query to find duplicate records in BigQuery.
- What is the difference between ELT and ETL, and why use ELT instead of ETL?
- When migrating, data duplication occurs; why does this happen and how do you avoid it?
- How do you read a Parquet file using Python? Write the code.
- What is OTLP and how can we handle it with BigQuery?
- What is the full form of OTLP?

## Pawan | 4:00 PM | L-2  (10)

- What is the size of streaming data, and how is it measured in real systems?
- How does Delta Lake handle deleted records? If records are deleted, how can we rebuild reports using Delta Lake?
- How do you migrate a huge amount of untransitioned live transaction streaming data to the cloud?
- How do you migrate live data without impacting the live transactional dataset when performing batch or streaming operations?
- How do you handle on-premise data migration when there are huge schema changes during the process?
- In a medallion architecture, if gold data was working fine (5-10 sec queries) but after 6 months the dataset becomes huge and queries take 5 minutes, how do you handle that?
- If 100k records per day are streaming, how do you process and publish them to Delta Lake?
- How do you handle late-arriving records in Kafka?
- Have you worked with Unity Catalog in Databricks in your project?
- How do you handle a dataset with transaction and financial highly sensitive records?

## Interview Preparation QA Automation  (18)

- What is the difference between batch and streaming processing? What are the advantages and disadvantages of each?
- You have a Medallion architecture and 100 unknown tables. How would you categorize them?
- In the silver layer, how do you ensure data integrity and decide which technical tables to use?
- How would you validate data in existing tables?
- How would you choose a partition key?
- Explain the logic of the SQL MERGE statement.
- What is the logic for the matching condition in a MERGE statement?
- What are the key components of Kafka?
- What is a Kafka broker?
- If a producer is producing 100 GB of data, does the broker need that much memory?
- What is an offset in Kafka?
- Give an example of a Kafka failure in a project.
- What metrics would you check to identify a lag in Kafka?
- How do you calculate lag per partition?
- What happens if there is consumer rebalancing in Kafka?
- You have a high-speed streaming system and need to backfill the last 6 months of data. What considerations would you take?
- In the same system, how would you handle late-arriving data?
- What metrics would you check daily on Snowflake?

## Pawan | 5:30 PM | Data  (13)

- How do we handle deduplication when a single person has multiple email IDs? What is the business requirement behind customer deduplication and identity resolution?
- How do we merge multiple identifiers (email, phone) into a single customer record? What fields are used beyond first name, last name, email, and phone number for identity matching?
- What is the size of streaming data, and how is it measured in real systems? What is checkpointing in streaming data processing, and why is it required?
- How does Databricks and Delta Lake work at a high level? What are the key features of Delta Lake?
- How do we write an SQL query to calculate running balance from transaction data? Can you explain the SQL logic used for calculating running balance?
- How to design an end-to-end pipeline from Oracle to Databricks? How do you design consumers and the pipeline?
- What is Z-ordering? When should you use it?
- What is the difference between streaming and real-time? What is the difference between Kafka and Azure in-built event house?
- How to implement real-time feature pipelines? What is a feature store and why do we need it?
- How to implement data quality checks? What is Unity Catalog?
- What is the disadvantage of over-indexing in a table?
- How to deal with the N+1 query problem?
- How to implement CI/CD for data pipelines?

## Rishabh@9:30  (11)

- If you have a query running for 20 minutes in Snowflake, how would you investigate it?
- You are tasked to invoke an API and put data into Snowflake, but Python inserted duplicate records. How would you handle it?
- If you don't want to use MERGE, what other options do you have?
- You have a dataset and try to load a CSV that is too large. How would you handle it in Python?
- In Snowflake, should you use streams or pipes? What is better?
- Which is better: pipe or streaming?
- Does Snowflake have a trigger-based mechanism?
- In Snowflake, if you write DELETE FROM table WHERE id, can you generate an event?
- How do you handle null and NaN in Python?
- You have a task scheduled in Snowflake that calls a procedure in the backend. It inserts data into a table hourly but skips some rows. How would you investigate?
- Explain time travel and fail-safe in Snowflake.

## Interview Introduction Guidance  (18)

- Explain the Medallion architecture in the Lufthansa project.
- What are the different sources where data was ingested into the Bronze/Silver layers?
- What is the use case of the materialized view approach?
- What is the difference between orchestration and transformation tools you have used?
- What are the components in Matillion?
- In Databricks, what clusters did you use to run pipelines in jobs?
- How do you differentiate between dev, UAT, and prod environments with different folders and workspaces?
- How do you set up CI/CD in Databricks and how do you handle downloading files?
- Explain repartition and coalesce in Spark.
- Explain the difference between narrow and wide transformations in Spark.
- Explain Spark architecture: job, stage, task, and execution.
- What is the role of the Catalyst optimizer in Spark?
- What is the use of broadcast join in Spark?
- Given a DataFrame, how do you create a session id column with a condition (e.g., >30) and increment the session id? What would be the output?
- Given a DataFrame with file paths and folder names, how do you extract the root folder and folder names? What would be the output?
- What are the different optimization techniques in Spark and their use cases?
- In SQL, if you have a sequence column with values 1-100, then you truncate the table and insert new records, what will be the output?
- How do you calculate Month over Month (MOM) in SQL or data engineering?

