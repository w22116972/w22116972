

# Emulate Oracle DR by using a PostgreSQL-compatible Aurora global database
<a name="emulate-oracle-dr-by-using-a-postgresql-compatible-aurora-global-database"></a>

*HariKrishna Boorgadda, Amazon Web Services*

## Abstract

This pattern compares Oracle disaster-recovery design with Aurora Global Database recovery capabilities. Use it to discuss recovery objectives, replication boundaries, and tested failover rather than assuming equivalent behavior.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="emulate-oracle-dr-by-using-a-postgresql-compatible-aurora-global-database-summary"></a>

Best practices for Enterprise disaster recovery (DR) basically consist of designing and implementing fault-tolerant hardware and software systems that can survive a disaster (*business continuance*) and resume normal operations (*business resumption*), with minimal intervention and, ideally, with no data loss. Building fault-tolerant environments to satisfy Enterprise DR objectives can be expensive and time consuming and requires a strong commitment by the business.

Oracle Database provides three different approaches to DR that provide the highest level of data protection and availability compared to any other approach for protecting Oracle data.
+ Oracle Zero Data Loss Recovery Appliance
+ Oracle Active Data Guard
+ Oracle GoldenGate

This pattern provides a way to emulate the Oracle GoldenGate DR by using an Amazon Aurora global database. The reference architecture uses Oracle GoldenGate for DR across three AWS Regions. The pattern walks through the replatforming of the source architecture to the cloud-native Aurora global database based on Amazon Aurora PostgreSQL–Compatible Edition.

Aurora global databases are designed for applications with a global footprint. A single Aurora database spans multiple AWS Regions with as many as five secondary Regions. Aurora global databases provide the following features:
+ Physical storage-level replication
+ Low-latency global reads
+ Fast disaster recovery from Region-wide outages
+ Fast cross-Region migrations
+ Low replication lag across Regions
+ Little-to-no performance impact on your database

For more information about Aurora global database features and advantages, see [Using Amazon Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database-overview). For more information about unplanned and managed failovers, see [Using failover in an Amazon Aurora global database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover).

## Prerequisites and limitations
<a name="emulate-oracle-dr-by-using-a-postgresql-compatible-aurora-global-database-prereqs"></a>

**Prerequisites **
+ An active AWS account 
+ A Java Database Connectivity (JDBC) PostgreSQL driver for application connectivity
+ An Aurora global database based on Amazon Aurora PostgreSQL-Compatible Edition
+ An Oracle Real Application Clusters (RAC) database migrated to the Aurora global database based on Aurora PostgreSQL–Compatible

**Limitations of Aurora global databases **
+ Aurora global databases aren’t available in all AWS Regions. For a list of supported Regions, see [Aurora global databases with Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.html#Concepts.Aurora_Fea_Regions_DB-eng.Feature.GlobalDatabase.apg).
+ For information about features that aren’t supported and other limitations of Aurora global databases, see the [Limitations of Amazon Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database.limitations).

**Product versions**
+ Amazon Aurora PostgreSQL–Compatible Edition version 10.14 or later

## Architecture
<a name="emulate-oracle-dr-by-using-a-postgresql-compatible-aurora-global-database-architecture"></a>

**Source technology stack****  **
+ Oracle RAC four-node database
+ Oracle GoldenGate

**Source architecture**** **

The following diagram shows three clusters with four-node Oracle RAC in different AWS Regions replicated using Oracle GoldenGate. 

![Oracle RAC in a primary Region and two secondary Regions.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/11d4265b-31af-4ebf-a766-24196193ee01/images/9fc740fc-d339-422e-beaf-1f65690c9d14.png)


**Target technology stack  **
+ A three-cluster Amazon Aurora global database based on Aurora PostgreSQL–Compatible, with one cluster in the primary Region, two clusters in different secondary Regions

**Target architecture**

![Amazon Aurora in a primary Region and two secondary Regions.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/11d4265b-31af-4ebf-a766-24196193ee01/images/8e3deca9-03f2-437c-9341-795ac17e2b42.png)


## Tools
<a name="emulate-oracle-dr-by-using-a-postgresql-compatible-aurora-global-database-tools"></a>

**AWS services**
+ [Amazon Aurora PostgreSQL-Compatible Edition](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.AuroraPostgreSQL.html) is a fully managed, ACID-compliant relational database engine that helps you set up, operate, and scale PostgreSQL deployments.
+ [Amazon Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html) span multiple AWS Regions, providing low latency global reads and fast recovery from the rare outage that might affect an entire AWS Region.

## Epics
<a name="emulate-oracle-dr-by-using-a-postgresql-compatible-aurora-global-database-epics"></a>

### Add Regions with reader DB instances
<a name="add-regions-with-reader-db-instances"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Attach one or more secondary Aurora clusters. | On the AWS Management Console, choose Amazon Aurora. Select the primary cluster, choose **Actions**, and choose **Add region** from the dropdown list. | DBA | 
| Select the instance class. | You can change the instance class of the secondary cluster. However, we recommend keeping it the same as the primary cluster instance class. | DBA | 
| Add the third Region. | Repeat the steps in this epic to add a cluster in the third Region. | DBA | 

### Fail over the Aurora global database
<a name="fail-over-the-aurora-global-database"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Remove the primary cluster from the Aurora global database. | 1. On the Databases page, choose the primary cluster.<br />2. Choose **Remove from Global** to fail over to a secondary cluster. | DBA | 
| Reconfigure your application to divert write traffic to the newly promoted cluster. | Modify the endpoint in the application with that of the newly promoted cluster. | DBA | 
| Stop issuing any write operations to the unavailable cluster. | Stop the application and any data manipulation language (DML) activity to the cluster that you removed. | DBA | 
| Create a new Aurora global database. | Now you can create an Aurora global database with the newly promoted cluster as the primary cluster. | DBA | 

### Start the primary cluster
<a name="start-the-primary-cluster"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Select the primary cluster to be started from the global database. | On the Amazon Aurora console, in the Global Database setup, choose the primary cluster. | DBA | 
| Start the cluster. | On the **Actions** dropdown list, choose **Start**. This process might take some time. Refresh the screen to see the status, or check the **Status** column for the current state of the cluster after the operation is completed. | DBA | 

### Clean up the resources
<a name="clean-up-the-resources"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Delete the remaining secondary clusters. | After the failover pilot is completed, remove the secondary clusters from the global database. | DBA | 
| Delete the primary cluster. | Remove the cluster. | DBA | 

## Related resources
<a name="emulate-oracle-dr-by-using-a-postgresql-compatible-aurora-global-database-resources"></a>
+ [Using Amazon Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database-detaching)
+ [Aurora PostgreSQL Disaster Recovery solutions using Amazon Aurora Global Database](https://aws.amazon.com/blogs/database/aurora-postgresql-disaster-recovery-solutions-using-amazon-aurora-global-database/) (blog post)
