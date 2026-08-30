

# Automate cross-Region failover and failback by using DR Orchestrator Framework
<a name="automate-cross-region-failover-and-failback-by-using-dr-orchestrator-framework"></a>

*Jitendra Kumar, Pavithra Balasubramanian, and Oliver Francis, Amazon Web Services*

## Abstract

Use this pattern to understand how a runbook-based framework can coordinate database failover and failback across AWS Regions. Treat it as a design reference; validate service support, recovery objectives, and failover testing for each workload.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="automate-cross-region-failover-and-failback-by-using-dr-orchestrator-framework-summary"></a>

This pattern describes how to use [DR Orchestrator Framework](https://docs.aws.amazon.com/prescriptive-guidance/latest/automate-dr-solution-relational-database/dr-orchestrator-framework-overview.html) to orchestrate and automate the manual, error-prone steps to perform disaster recovery across Amazon Web Services (AWS) Regions. The pattern covers the following databases:
+ Amazon Relational Database Service (Amazon RDS) for MySQL, Amazon RDS for PostgreSQL, or Amazon RDS for MariaDB
+ Amazon Aurora MySQL-Compatible Edition or Amazon Aurora PostgreSQL-Compatible Edition (using a centralized file)
+ Amazon ElastiCache (Redis OSS)

To demonstrate the functionality of DR Orchestrator Framework, you create two DB instances or clusters. The primary is in the AWS Region `us-east-1`, and the secondary is in `us-west-2`. To create these resources, you use the AWS CloudFormation templates in the `App-Stack` folder of the [aws-cross-region-dr-databases](https://github.com/aws-samples/aws-cross-region-dr-databases) GitHub repository.

## Prerequisites and limitations
<a name="automate-cross-region-failover-and-failback-by-using-dr-orchestrator-framework-prereqs"></a>

**General prerequisites**
+ DR Orchestrator Framework deployed in both primary and secondary AWS Regions
+ Two [Amazon Simple Storage Service](https://aws.amazon.com/s3/) buckets
+ A [virtual private cloud (VPC)](https://aws.amazon.com/vpc/) with two subnets and an AWS security group 

**Engine-specific prerequisites**
+ **Amazon Aurora** – At least one Aurora global database must be available in two AWS Regions. You can use `us-east-1` as the primary Region, and use `us-west-2` as the secondary Region.
+ **Amazon ElastiCache (Redis OSS)** – An ElastiCache global datastore must be available in two AWS Regions. You can `use us-east-1` as the primary Region, and use `us-west-2` as the secondary Region.

**Amazon RDS limitations**
+ DR Orchestrator Framework doesn't check the replication lag before doing a failover or failback. Replication lag must be checked manually.
+ This solution has been tested using a primary database instance with one read replica. If you want to use more than one read replica, test the solution thoroughly before implementing it in a production environment.

**Aurora limitations**
+ Feature availability and support vary across specific versions of each database engine and across AWS Regions. For more information on feature and Region availability for cross-Region replication, see [Cross-Region read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RDS_Fea_Regions_DB-eng.Feature.CrossRegionReadReplicas.html).
+ Aurora global databases have specific configuration requirements for supported Aurora DB instance classes and the maximum number of AWS Regions. For more information, see [Configuration requirements of an Amazon Aurora global database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-getting-started.html#aurora-global-database.configuration.requirements).
+ This solution has been tested using a primary database instance with one read replica. If you want to use more than one read replica, test the solution thoroughly before implementing it in a production environment.

**ElastiCache limitations**
+ For information about Region availability for Global Datastore and ElastiCache configuration requirements, see [Prerequisites and limitations](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastores-Getting-Started.html) in the ElastiCache documentation.

**Amazon RDS p****roduct versions**

Amazon RDS supports the following engine versions:
+ **MySQL** – Amazon RDS supports DB instances running the following versions of [MySQL](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MySQL.html): MySQL 8.0 and MySQL 5.7
+ **PostgreSQL** – For information about supported versions of Amazon RDS for PostgreSQL, see [Available PostgreSQL database versions](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html#PostgreSQL.Concepts.General.DBVersions).
+ **MariaDB** – Amazon RDS supports DB instances running the following versions of [MariaDB](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_MariaDB.html):
  + MariaDB 10.11
  + MariaDB 10.6
  + MariaDB 10.5

**Aurora product versions**
+ Amazon Aurora global database switchover requires Aurora MySQL-Compatible with MySQL 5.7 compatibility, version 2.09.1 and higher

  For more information, see [Limitations of Amazon Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html#aurora-global-database.limitations).

**ElastiCache (Redis OSS) product versions**

Amazon ElastiCache (Redis OSS) supports the following Redis versions:
+ Redis 7.1 (enhanced)
+ Redis 7.0 (enhanced)
+ Redis 6.2 (enhanced)
+ Redis 6.0 (enhanced)
+ Redis 5.0.6 (enhanced)

For more information, see [Supported ElastiCache (Redis OSS) versions](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastores-Getting-Started.html).

## Architecture
<a name="automate-cross-region-failover-and-failback-by-using-dr-orchestrator-framework-architecture"></a>

**Amazon RDS**** architecture**

The Amazon RDS architecture includes the following resources:
+ The primary Amazon RDS DB instance created in the primary Region (`us-east-1`) with read/write access for clients
+ An Amazon RDS read replica created in the secondary Region (`us-west-2`) with read-only access for clients
+ DR Orchestrator Framework deployed in both the primary and secondary Regions

![Diagram of two-Region RDS architecture in a single AWS account.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/8d39561f-924e-4b3e-8175-c5c3cab163bd/images/ad217033-600c-40da-929c-b9f9aecb4c2c.png)


The diagram shows the following:

1. Asynchronous replication between the primary instance and the secondary instance

1. Read/write access for clients in the primary Region

1. Read-only access for clients in the secondary Region

**Aurora architecture**

The Amazon Aurora architecture includes the following resources:
+ The primary Aurora DB cluster created in the primary Region (`us-east-1`) with an active-writer endpoint
+ An Aurora DB cluster created in the secondary Region (`us-west-2`) with an inactive-writer endpoint
+ DR Orchestrator Framework deployed in both the primary and secondary Regions

![Diagram of two-Region Aurora deployment in a single AWS account.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/8d39561f-924e-4b3e-8175-c5c3cab163bd/images/524ec002-5aa7-47b2-8c8d-6d1a3b535e9e.png)


The diagram shows the following:

1. Asynchronous replication between the primary cluster and the secondary cluster

1. The primary DB cluster with an active-writer endpoint

1. The secondary DB cluster with an inactive-writer endpoint

**ElastiCache (Redis OSS) architecture**

The Amazon ElastiCache (Redis OSS) architecture includes the following resources:
+ An ElastiCache (Redis OSS) global datastore created with two clusters:

  1. The primary cluster in the primary Region (`us-east-1`)

  1. The secondary cluster in the secondary Region (`us-west-2`)
+ An Amazon cross-Region link with TLS 1.2 encryption between the two clusters
+ DR Orchestrator Framework deployed in both primary and secondary Regions

![Diagram of a two-Region ElastiCache deployment with Amazon cross-Region link.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/8d39561f-924e-4b3e-8175-c5c3cab163bd/images/cf6620a0-dd42-4042-8dc2-012bf514ffc0.png)


**Automation and scale**

DR Orchestrator Framework is scalable and supports the failover or failback of more than one AWS database in parallel.

You can use the following payload code to fail over multiple AWS databases in your account. In this example, three AWS databases (two global databases such as Aurora MySQL-Compatible or Aurora PostgreSQL-Compatible, and one Amazon RDS for MySQL instance) fail over to the DR Region:

```
{
  "StatePayload": [
    {
      "layer": 1,
      "resources": [
        {
          "resourceType": "PlannedFailoverAurora",
          "resourceName": "Switchover (planned failover) of Amazon Aurora global databases (MySQL)",
          "parameters": {
            "GlobalClusterIdentifier": "!Import dr-globaldb-cluster-mysql-global-identifier",
            "DBClusterIdentifier": "!Import dr-globaldb-cluster-mysql-cluster-identifier" 
          }
        },
        {
          "resourceType": "PlannedFailoverAurora",
          "resourceName": "Switchover (planned failover) of Amazon Aurora global databases (PostgreSQL)",
          "parameters": {
            "GlobalClusterIdentifier": "!Import dr-globaldb-cluster-postgres-global-identifier",
            "DBClusterIdentifier": "!Import dr-globaldb-cluster-postgres-cluster-identifier" 
          }
        },
        {
          "resourceType": "PromoteRDSReadReplica",
          "resourceName": "Promote RDS for MySQL Read Replica",
          "parameters": {
            "RDSInstanceIdentifier": "!Import rds-mysql-instance-identifier",
            "TargetClusterIdentifier": "!Import rds-mysql-instance-global-arn"
          }
        }         
      ]
    }
  ]
}
```

## Tools
<a name="automate-cross-region-failover-and-failback-by-using-dr-orchestrator-framework-tools"></a>

**AWS services**
+ [Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html) is a fully managed relational database engine that's built for the cloud and compatible with MySQL and PostgreSQL.
+ [Amazon ElastiCache](https://docs.aws.amazon.com/elasticache/) helps you set up, manage, and scale distributed in-memory cache environments in the AWS Cloud. This pattern uses Amazon ElastiCache (Redis OSS).
+ [AWS Lambda](https://aws.amazon.com/lambda/) is a compute service that helps you run code without needing to provision or manage servers. It runs your code only when needed and scales automatically, so you pay only for the compute time that you use. In this pattern, Lambda functions are used by AWS Step Functions to perform the steps.
+ [Amazon Relational Database Service (Amazon RDS)](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) helps you set up, operate, and scale a relational database in the AWS Cloud. This pattern supports Amazon RDS for MySQL, Amazon RDS for PostgreSQL, and Amazon RDS for MariaDB.
+ [AWS SDK for Python (Boto3)](https://aws.amazon.com/sdk-for-python/) helps you integrate your Python application, library, or script with AWS services. In this pattern, Boto3 APIs are used to communicate with the database instances or global databases.
+ [AWS Step Functions](https://aws.amazon.com/step-functions/) is a serverless orchestration service that helps you combine AWS Lambda functions and other AWS services to build business-critical applications. In this pattern, Step Functions state machines are used to orchestrate and run the cross-Region failover and failback of the database instances or global databases.

**Code repository**

The code for this pattern is available in the [aws-cross-region-dr-databases](https://github.com/aws-samples/aws-cross-region-dr-databases/tree/main/App-Stack) repository on GitHub.

## Epics
<a name="automate-cross-region-failover-and-failback-by-using-dr-orchestrator-framework-epics"></a>

### Install DR Orchestrator Framework
<a name="install-dr-orchestrator-framework"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Clone the GitHub repository. | To clone the repository, run the following command:<pre>git clone https://github.com/aws-samples/aws-cross-region-dr-databases.git</pre> | AWS DevOps, AWS administrator | 
| Package Lambda functions code in a .zip file archive. | Create the archive files for Lambda functions to include the DR Orchestrator Framework dependencies:<pre>cd <YOUR-LOCAL-GIT-FOLDER>/DR-Orchestration-artifacts<br /><br />bash scripts/deploy-orchestrator-sh.sh</pre> | AWS administrator | 
| Create S3 buckets. | S3 buckets are needed to store DR Orchestrator Framework along with your latest configuration. Create two S3 buckets, one in the primary Region (`us-east-1`), and one in the secondary Region (`us-west-2`):+ `dr-orchestrator-xxxxxx-us-east-1`<br />+ `dr-orchestrator-xxxxxx-us-west-2`<br />Replace `xxxxxx` with a random value to make the bucket names unique. | AWS administrator | 
| Create subnets and security groups. | In both the primary Region (`us-east-1`) and the secondary Region (`us-west-2`), create two subnets and one security group for Lambda function deployment in your VPC:+ `subnet-XXXXXXX`<br />+ `subnet-YYYYYYY`<br />+ `sg-XXXXXXXXXXXX` | AWS administrator | 
| Update the DR Orchestrator parameter files. | In the `<YOUR-LOCAL-GIT-FOLDER>/DR-Orchestration-artifacts/cloudformation` folder, update the following DR Orchestrator parameter files:+ `Orchestrator-Deployer-parameters-us-east-1.json`<br />+ `Orchestrator-Deployer-parameters-us-west-2.json`<br />Use the following parameter values, replacing `x` and `y` with the names of your resources:<pre>[<br />    {<br />         "ParameterKey": "TemplateStoreS3BucketName",<br />         "ParameterValue": "dr-orchestrator-xxxxxx-us-east-1"<br />    },<br />    {<br />        "ParameterKey": "TemplateVPCId",<br />        "ParameterValue": "vpc-xxxxxx"<br />    },<br />    {<br />        "ParameterKey": "TemplateLambdaSubnetID1",<br />        "ParameterValue": "subnet-xxxxxx"<br />    },<br />    {<br />        "ParameterKey": "TemplateLambdaSubnetID2",<br />        "ParameterValue": "subnet-yyyyyy"<br />    },<br />    {<br />        "ParameterKey": "TemplateLambdaSecurityGroupID",<br />        "ParameterValue": "sg-xxxxxxxxxx"<br />    }<br /> ]</pre> | AWS administrator | 
| Upload the DR Orchestrator Framework code to the S3 bucket. | The code will be safer in an S3 bucket than in the local directory. Upload the `DR-Orchestration-artifacts` directory, including all files and subfolders, to the S3 buckets.<br />To upload the code, do the following:1. Sign in to the AWS Management Console.<br />2. Navigate to the Amazon S3 console.<br />3. Select the `dr-orchestrator-xxxxxx-us-east-1 bucket`.<br />4. Choose **Upload**, and then choose **Add folder**.<br />5. Select the `DR-Orchestration-artifacts` folder.<br />6. Choose **Upload**.<br />7. Select the `dr-orchestrator-xxxxxx-us-west-2` bucket.<br />8. Repeat steps 4–7. | AWS administrator | 
| Deploy DR Orchestrator Framework in the primary Region. | To deploy DR Orchestrator Framework in the primary Region (`us-east-1`), run the following commands:<pre>cd <YOUR-LOCAL-GIT-FOLDER>/DR-Orchestration-artifacts/cloudformation<br /><br />aws cloudformation deploy \<br />--region us-east-1 \<br />--stack-name dr-orchestrator \<br />--template-file Orchestrator-Deployer.yaml \<br />--parameter-overrides file://Orchestrator-Deployer-parameters-us-east-1.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback</pre> | AWS administrator | 
| Deploy DR Orchestrator Framework in the secondary Region. | In the secondary Region (`us-west-2`), run the following commands: <pre>cd <YOUR-LOCAL-GIT-FOLDER>/DR-Orchestration-artifacts/cloudformation<br /><br />aws cloudformation deploy \<br />--region us-west-2 \<br />--stack-name dr-orchestrator \<br />--template-file Orchestrator-Deployer.yaml \<br />--parameter-overrides file://Orchestrator-Deployer-parameters-us-west-2.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback</pre> | AWS administrator | 
| Verify the deployment. | If the CloudFormation command runs successfully, it returns the following output:<pre>Successfully created/updated stack - dr-orchestrator</pre><br />Alternatively, you can navigate to the CloudFormation console and verify the status of the `dr-orchestrator` stack.  | AWS administrator | 

### Create the database instances or clusters
<a name="create-the-database-instances-or-clusters"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create the database subnets and security groups. | In your VPC, create two subnets and one security group for the DB instance or global database in both the primary (`us-east-1`) and the secondary (`us-west-2`) Regions:+ `subnet-XXXXXX`<br />+ `subnet-XXXXXX`<br />+ `sg-XXXXXXXXXX` | AWS administrator | 
| Update the parameter file for the primary DB instance or cluster. | In the `<YOUR LOCAL GIT FOLDER>/App-Stack` folder, update the parameter file for the primary Region.<br />**Amazon RDS**<br />In the `RDS-MySQL-parameter-us-east-1.json` file, update `SubnetIds` and `DBSecurityGroup` with the names of resources that you created:<pre>{<br />  "Parameters": {<br />    "SubnetIds": "subnet-xxxxxx,subnet-xxxxxx",<br />    "DBSecurityGroup": "sg-xxxxxxxxxx",<br />    "MySqlGlobalIdentifier":"rds-mysql-instance",<br />    "InitialDatabaseName": "mysqldb",<br />    "DBPortNumber": "3789",<br />    "PrimaryRegion": "us-east-1",<br />    "SecondaryRegion": "us-west-2",<br />    "KMSKeyAliasName": "rds/rds-mysql-instance-KmsKeyId"<br />  }<br />}<br /></pre><br />**Amazon Aurora**<br /> In the `Aurora-MySQL-parameter-us-east-1.json` file, update `SubnetIds` and `DBSecurityGroup` with the names of resources that you created:<pre>{<br />  "Parameters": {<br />    "SubnetIds": "subnet1-xxxxxx,subnet2-xxxxxx",<br />    "DBSecurityGroup": "sg-xxxxxxxxxx",<br />    "GlobalClusterIdentifier":"dr-globaldb-cluster-mysql",<br />    "DBClusterName":"dbcluster-01",<br />    "SourceDBClusterName":"dbcluster-02",<br />    "DBPortNumber": "3787",<br />    "DBInstanceClass":"db.r5.large",<br />    "InitialDatabaseName": "sampledb",<br />    "PrimaryRegion": "us-east-1",<br />    "SecondaryRegion": "us-west-2",<br />    "KMSKeyAliasName": "rds/dr-globaldb-cluster-mysql-KmsKeyId"<br />  }<br />}</pre><br />**Amazon ElastiCache (Redis OSS)**<br />In the `ElastiCache-parameter-us-east-1.json` file, update `SubnetIds` and `DBSecurityGroup` with the names of resources that you created.<pre>{<br />  "Parameters": {<br />    "CacheNodeType": "cache.m5.large",<br />    "DBSecurityGroup": "sg-xxxxxxxxxx",<br />    "SubnetIds": "subnet-xxxxxx,subnet-xxxxxx",<br />    "EngineVersion": "5.0.6",<br />    "GlobalReplicationGroupIdSuffix": "demo-redis-global-datastore",<br />    "NumReplicas": "1",<br />    "NumShards": "1",<br />    "ReplicationGroupId": "demo-redis-cluster",<br />    "DBPortNumber": "3788",<br />    "TransitEncryption": "true",<br />    "KMSKeyAliasName": "elasticache/demo-redis-global-datastore-KmsKeyId",<br />    "PrimaryRegion": "us-east-1",<br />    "SecondaryRegion": "us-west-2"<br />  }<br />}</pre> | AWS administrator | 
| Deploy your DB instance or cluster in the primary Region. | To deploy your instance or cluster in the primary Region (`us-east-1`), run the following commands based on your database engine.<br />**Amazon RDS**<pre>cd <YOUR-LOCAL-GIT-FOLDER>/App-Stack<br /><br />aws cloudformation deploy \<br />--region us-east-1 \<br />--stack-name rds-mysql-app-stack \<br />--template-file RDS-MySQL-Primary.yaml \<br />--parameter-overrides file://RDS-MySQL-parameter-us-east-1.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback</pre><br />**Amazon Aurora**<pre>cd <YOUR-LOCAL-GIT-FOLDER>/App-Stack<br /><br />aws cloudformation deploy \<br />--region us-east-1 \<br />--stack-name aurora-mysql-app-stack \<br />--template-file Aurora-MySQL-Primary.yaml \<br />--parameter-overrides file://Aurora-MySQL-parameter-us-east-1.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback</pre><br />**Amazon ElastiCache (Redis OSS)**<pre>cd <YOUR-LOCAL-GIT-FOLDER>/App-Stack<br /><br />aws cloudformation deploy \<br />--region us-east-1 --stack-name elasticache-ds-app-stack \<br />--template-file ElastiCache-Primary.yaml \<br />--parameter-overrides file://ElastiCache-parameter-us-east-1.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback<br /></pre><br />Verify that the CloudFormation resources deployed successfully. | AWS administrator | 
| Update the parameter file for the secondary DB instance or cluster. | In the `<YOUR LOCAL GIT FOLDER>/App-Stack` folder, update the parameter file for the secondary Region.<br />**Amazon RDS**<br />In the `RDS-MySQL-parameter-us-west-2.json` file, update `SubnetIDs` and `DBSecurityGroup` with the names of resources that you created. Update the `PrimaryRegionKMSKeyArn` with the value of `MySQLKmsKeyId` taken from the **Outputs** section of the CloudFormation stack for the primary DB instance:<pre>{<br />  "Parameters": {<br />    "SubnetIds": "subnet-aaaaaaaaa,subnet-bbbbbbbbb",<br />    "DBSecurityGroup": "sg-cccccccccc",<br />    "MySqlGlobalIdentifier":"rds-mysql-instance",<br />    "InitialDatabaseName": "mysqldb",<br />    "DBPortNumber": "3789",<br />    "PrimaryRegion": "us-east-1",<br />    "SecondaryRegion": "us-west-2",<br />    "KMSKeyAliasName": "rds/rds-mysql-instance-KmsKeyId",<br />    "PrimaryRegionKMSKeyArn":"arn:aws:kms:us-east-1:xxxxxxxxx:key/mrk-xxxxxxxxxxxxxxxxxxxxx"<br />  }<br />} </pre><br />**Amazon Aurora**<br />In the `Aurora-MySQL-parameter-us-west-2.json` file, update `SubnetIDs` and `DBSecurityGroup` with the names of resources you created. Update the `PrimaryRegionKMSKeyArn` with the value of `AuroraKmsKeyId` taken from the **Outputs** section of the CloudFormation stack for the primary DB instance:<pre>{<br />  "Parameters": {<br />    "SubnetIds": "subnet1-aaaaaaaaa,subnet2-bbbbbbbbb",<br />    "DBSecurityGroup": "sg-cccccccccc",<br />    "GlobalClusterIdentifier":"dr-globaldb-cluster-mysql",<br />    "DBClusterName":"dbcluster-01",<br />    "SourceDBClusterName":"dbcluster-02",<br />    "DBPortNumber": "3787",<br />    "DBInstanceClass":"db.r5.large",<br />    "InitialDatabaseName": "sampledb",<br />    "PrimaryRegion": "us-east-1",<br />    "SecondaryRegion": "us-west-2",<br />    "KMSKeyAliasName": "rds/dr-globaldb-cluster-mysql-KmsKeyId"<br />  }<br />}</pre><br />**Amazon ElastiCache (Redis OSS)**<br />In the `ElastiCache-parameter-us-west-2.json` file, update `SubnetIDs` and `DBSecurityGroup` with the names of resources that you created. Update the `PrimaryRegionKMSKeyArn` with the value of `ElastiCacheKmsKeyId` taken from the **Outputs** section of the CloudFormation stack for the primary DB instance:<pre>{<br />  "Parameters": {<br />    "CacheNodeType": "cache.m5.large",<br />    "DBSecurityGroup": "sg-cccccccccc",<br />    "SubnetIds": "subnet-aaaaaaaaa,subnet-bbbbbbbbb",<br />    "EngineVersion": "5.0.6",<br />    "GlobalReplicationGroupIdSuffix": "demo-redis-global-datastore",<br />    "NumReplicas": "1",<br />    "NumShards": "1",<br />    "ReplicationGroupId": "demo-redis-cluster",<br />    "DBPortNumber": "3788",<br />    "TransitEncryption": "true",<br />    "KMSKeyAliasName": "elasticache/demo-redis-global-datastore-KmsKeyId",<br />    "PrimaryRegion": "us-east-1",<br />    "SecondaryRegion": "us-west-2"<br />  }<br />}</pre> | AWS administrator | 
| Deploy your DB instance or cluster in the secondary Region. | Run the following commands, based on your database engine.<br />**Amazon RDS**<pre>cd <YOUR-LOCAL-GIT-FOLDER>/App-Stack<br /><br />aws cloudformation deploy \<br />--region us-west-2 \<br />--stack-name rds-mysql-app-stack \<br />--template-file RDS-MySQL-DR.yaml \<br />--parameter-overrides file://RDS-MySQL-parameter-us-west-2.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback</pre><br />**Amazon Aurora**<pre>cd <YOUR-LOCAL-GIT-FOLDER>/App-Stack<br /><br />aws cloudformation deploy \<br />--region us-west-2 \<br />--stack-name aurora-mysql-app-stack \<br />--template-file Aurora-MySQL-DR.yaml \<br />--parameter-overrides file://Aurora-MySQL-parameter-us-west-2.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback</pre><br />**Amazon ElastiCache (Redis OSS)**<pre>cd <YOUR-LOCAL-GIT-FOLDER>/App-Stack<br /><br />aws cloudformation deploy \<br />--region us-west-2 \<br />--stack-name elasticache-ds-app-stack \<br />--template-file ElastiCache-DR.yaml \<br />--parameter-overrides file://ElastiCache-parameter-us-west-2.json \<br />--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM \<br />--disable-rollback</pre><br />Verify that the CloudFormation resources deployed successfully. | AWS administrator | 

## Related resources
<a name="automate-cross-region-failover-and-failback-by-using-dr-orchestrator-framework-resources"></a>
+ [Disaster recovery strategy for databases on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/strategy-database-disaster-recovery/welcome.html) (AWS Prescriptive Guidance strategy)
+ [Automate your DR solution for relational databases on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/automate-dr-solution-relational-database/dr-orchestrator-framework-overview.html) (AWS Prescriptive Guidance guide)
+ [Using Amazon Aurora global databases](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
+ [Replication across AWS Regions using global datastores](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Redis-Global-Datastore.html)
+ [Automate your DR solution for relational databases on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/automate-dr-solution-relational-database/introduction.html) (AWS Prescriptive Guidance guide)
