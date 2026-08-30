

# Validate database objects after migrating from Oracle to Amazon Aurora PostgreSQL
<a name="validate-database-objects-after-migrating-from-oracle-to-amazon-aurora-postgresql"></a>

*Venkatramana Chintha and Eduardo Valentim, Amazon Web Services*

## Abstract

Validate schema objects, data, and application behavior after an Oracle-to-Aurora PostgreSQL migration before declaring cutover complete. Adapt the checks to the application’s critical transactions and acceptance criteria.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="validate-database-objects-after-migrating-from-oracle-to-amazon-aurora-postgresql-summary"></a>

This pattern describes a step-by-step approach to validate objects after migrating an Oracle database to Amazon Aurora PostgreSQL-Compatible Edition.

This pattern outlines usage scenarios and steps for database object validation; for more detailed information, see [Validating database objects after migration using AWS SCT and AWS DMS](https://aws.amazon.com/blogs/database/validating-database-objects-after-migration-using-aws-sct-and-aws-dms/) on the [AWS Database blog](https://aws.amazon.com/blogs/).

## Prerequisites and limitations
<a name="validate-database-objects-after-migrating-from-oracle-to-amazon-aurora-postgresql-prereqs"></a>

**Prerequisites**
+ An active AWS account.
+ An on-premises Oracle database that was migrated to an Aurora PostgreSQL-Compatible database. 
+ Sign-in credentials that have the [AmazonRDSDataFullAccess](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/query-editor.html) policy applied, for the Aurora PostgreSQL-Compatible database. 
+ This pattern uses the [query editor for Aurora Serverless DB clusters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/query-editor.html), which is available in the Amazon Relational Database Service (Amazon RDS) console. However, you can use this pattern with any other query editor. 

**Limitations**
+ Oracle SYNONYM objects are not available in PostgreSQL but can be partially validated through **views** or SET search\_path queries.
+ The Amazon RDS query editor is available only in [certain AWS Regions and for certain MySQL and PostgreSQL versions](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/query-editor.html).

## Architecture
<a name="validate-database-objects-after-migrating-from-oracle-to-amazon-aurora-postgresql-architecture"></a>

 

![](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/7c028960-6dea-46ad-894d-e42cefd50c03/images/be5f8ae3-f5af-4c5e-9440-09ab410beaa1.png)


 

## Tools
<a name="validate-database-objects-after-migrating-from-oracle-to-amazon-aurora-postgresql-tools"></a>

**Tools**
+ [Amazon Aurora PostgreSQL-Compatible Edition](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html) – Aurora PostgreSQL-Compatible is a fully managed, PostgreSQL-compatible, and ACID-compliant relational database engine that combines the speed and reliability of high-end commercial databases with the simplicity and cost-effectiveness of open-source databases.
+ [Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) – Amazon Relational Database Service (Amazon RDS) makes it easier to set up, operate, and scale a relational database in the AWS Cloud. It provides cost-efficient, resizable capacity for an industry-standard relational database and manages common database administration tasks.
+ [Query Editor for Aurora Severless](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/query-editor.html) – Query editor helps you run SQL queries in the Amazon RDS console. You can run any valid SQL statement on the Aurora Serverless DB cluster, including data manipulation and data definition statements.

To validate the objects, use the full scripts in the "Object validation scripts" file in the "Attachments" section. Use the following table for reference.


| 
| 
| Oracle object | Script to use | 
| --- |--- |
| Packages | Query 1 | 
| Tables | Query  3 | 
| Views | Query  5 | 
| Sequences | Query  7 | 
| Triggers |  Query  9 | 
| Primary keys | Query  11 | 
| Indexes | Query  13 | 
| Check constraints | Query  15 | 
| Foreign keys  | Query 17  | 


| 
| 
| PostgreSQL object | Script to use | 
| --- |--- |
| Packages | Query 2 | 
| Tables | Query 4 | 
| Views | Query 6 | 
| Sequences | Query 8 | 
| Triggers | Query 10 | 
| Primary keys | Query  12 | 
| Indexes | Query  14 | 
| Check constraints | Query  16 | 
| Foreign keys | Query  18 | 

## Epics
<a name="validate-database-objects-after-migrating-from-oracle-to-amazon-aurora-postgresql-epics"></a>

### Validate objects in the source Oracle database
<a name="validate-objects-in-the-source-oracle-database"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Run the "packages" validation query in the source Oracle database.  | Download and open the "Object validation scripts" file from the "Attachments" section. Connect to the source Oracle database through your client program. Run the "Query 1" validations script from the "Object validation scripts" file. Important: Enter your Oracle user name instead of "your\_schema" in the queries. Make sure you record your query results. | Developer, DBA | 
| Run the "tables" validation query.  | Run the "Query 3" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "views" validation query.  | Run the "Query 5" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "sequences" count validation.  | Run the "Query 7" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "triggers" validation query.  | Run the "Query 9" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "primary keys" validation query.  | Run the "Query 11" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "indexes" validation query.  | Run the "Query 13" validation script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "check constraints" validation query.  | Run the "Query 15" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "foreign keys" validation query.  | Run the "Query 17" validation script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 

### Validate objects in the target Aurora PostgreSQL-Compatible database
<a name="validate-objects-in-the-target-aurora-postgresql-compatible-database"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Connect to the target Aurora PostgreSQL-Compatible database by using the query editor. | Sign in to the AWS Management Console and open the Amazon RDS console. In the upper-right corner, choose the AWS Region in which you created the Aurora PostgreSQL-Compatible database. In the navigation pane, choose "Databases," and choose the target Aurora PostgreSQL-Compatible database. In "Actions," choose "Query." Important: If you haven't connected to the database before, the "Connect to database" page opens. You then need to enter your database information, such as user name and password. | Developer, DBA | 
| Run the "packages" validation query. | Run the "Query 2" script from the "Object validation scripts" file in the "Attachments" section. Make sure you record your query results. | Developer, DBA | 
| Run the "tables" validation query.  | Return to the query editor for the Aurora PostgreSQL-Compatible database, and run the "Query 4" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "views" validation query.  | Return to the query editor for the Aurora PostgreSQL-Compatible database, and run the "Query 6" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "sequences" count validation.  | Return to the query editor for the Aurora PostgreSQL-Compatible database, and run the "Query 8" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "triggers" validation query.  | Return to the query editor for the Aurora PostgreSQL-Compatible database, and run the "Query 10" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "primary keys" validation query.  | Return to the query editor for the Aurora PostgreSQL-Compatible database, and run the "Query 12" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "indexes" validation query.  | Return to the query editor for the Aurora PostgreSQL-Compatible database, and run the "Query 14" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "check constraints" validation query.  | Run the "Query 16" script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 
| Run the "foreign keys" validation query.  | Run the "Query 18" validation script from the "Object validation scripts" file. Make sure you record your query results. | Developer, DBA | 

### Compare source and target database validation records
<a name="compare-source-and-target-database-validation-records"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Compare and validate both query results.  | Compare the query results of the Oracle and Aurora PostgreSQL-Compatible databases to validate all objects. If they all match, then all objects have been successfully validated. | Developer, DBA | 

## Related resources
<a name="validate-database-objects-after-migrating-from-oracle-to-amazon-aurora-postgresql-resources"></a>
+ [Validating database objects after a migration using AWS SCT and AWS DMS](https://aws.amazon.com/blogs/database/validating-database-objects-after-migration-using-aws-sct-and-aws-dms/)
+ [Amazon Aurora Features: PostgreSQL-Compatible Edition](https://aws.amazon.com/rds/aurora/postgresql-features/)

## Attachments
<a name="attachments-7c028960-6dea-46ad-894d-e42cefd50c03"></a>

To access additional content that is associated with this document, download and unzip the following file: [attachment.zip](samples/p-attach/7c028960-6dea-46ad-894d-e42cefd50c03/attachments/attachment.zip)
