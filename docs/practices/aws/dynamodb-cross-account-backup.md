

# Copy Amazon DynamoDB tables across accounts using AWS Backup
<a name="copy-amazon-dynamodb-tables-across-accounts-using-aws-backup"></a>

*Ramkumar Ramanujam, Amazon Web Services*

## Abstract

Use AWS Backup vaults and cross-account copies to separate DynamoDB recovery data from the production account. This reference focuses on recovery isolation, retention, and restore verification.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="copy-amazon-dynamodb-tables-across-accounts-using-aws-backup-summary"></a>

When working with Amazon DynamoDB on AWS, a common use case is to copy or sync DynamoDB tables in development, testing, or staging environments with the table data that is in the production environment. As a standard practice, each environment uses a different AWS account. 

AWS Backup supports cross-Region and cross-account backup and restoration of data for DynamoDB, Amazon Simple Storage Service (Amazon S3), and other AWS services. This pattern provides the steps for using AWS Backup cross-account backup and restore to copy DynamoDB tables between AWS accounts.

## Prerequisites and limitations
<a name="copy-amazon-dynamodb-tables-across-accounts-using-aws-backup-prereqs"></a>

**Prerequisites **
+ Two active AWS accounts that belong to the same organization in AWS Organizations
+ Permissions to create DynamoDB tables in both accounts
+ AWS Identity and Access Management (IAM) permissions to create and use AWS Backup vaults

**Limitations **
+ Source and target AWS accounts should be part of the same organization in AWS Organizations.

## Architecture
<a name="copy-amazon-dynamodb-tables-across-accounts-using-aws-backup-architecture"></a>

**Target technology stack  **
+ AWS Backup 
+ Amazon DynamoDB

**Target architecture **

![Description of copying tables between backup vaults follows the diagram.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/ef6e7393-edb6-4744-be26-43f1cbff9de9/images/fa9f3f2f-7a01-4093-9bd5-fc355e57ba67.png)


1. Create the DynamoDB table backup in the AWS Backup backup vault in the source account.

1. Copy the backup to the backup vault in the target account.

1. Restore the DynamoDB table in the target account by using the backup from the backup vault in the target account.

**Automation and scale**

You can use AWS Backup to schedule backups to run at specific intervals.

## Tools
<a name="copy-amazon-dynamodb-tables-across-accounts-using-aws-backup-tools"></a>
+ [AWS Backup](https://docs.aws.amazon.com/aws-backup/latest/devguide/whatisbackup.html) is a fully-managed service for centralizing and automating data protection across AWS services, in the cloud, and on premises. Using this service, you can configure backup policies and monitor activity for your AWS resources in one place. It allows you to automate and consolidate backup tasks that were previously performed service by service, and removes the need to create custom scripts and manual processes.
+ [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) is a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability.

## Epics
<a name="copy-amazon-dynamodb-tables-across-accounts-using-aws-backup-epics"></a>

### Turn on AWS Backup features in the source and target accounts
<a name="turn-on-bkp-features-in-the-source-and-target-accounts"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Turn on advanced features for DynamoDB and cross-account backup. | In both the source and the target AWS accounts, do the following:1. On the AWS Management Console, open the [AWS Backup console](https://console.aws.amazon.com/backup/).<br />2. Choose **Settings**.<br />3. Under **Advanced features for Amazon DynamoDB backups**, confirm that **Advanced features** is enabled, or choose **Enable**.<br />4. Under **Cross-account management**, for **Cross-account backup**, choose **Enable**. | AWS DevOps, Migration engineer | 

### Create backup vaults in the source and target accounts
<a name="create-backup-vaults-in-the-source-and-target-accounts"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create backup vaults. | In both the source and the target AWS accounts, do the following:1. On the [AWS Backup console](https://console.aws.amazon.com/backup/), choose **Backup vaults**.<br />2. Choose **Create Backup vault**.<br />3. Copy the Amazon Resource Name (ARN) of the backup vault and save it.<br />The ARNs of both the source and the target backup vaults will be required when you copy the DynamoDB table backup between the source and target accounts. | AWS DevOps, Migration engineer | 

### Perform backup and restore using backup vaults
<a name="perform-backup-and-restore-using-backup-vaults"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| In the source account, create a DynamoDB table backup. | To create a backup for the DynamoDB table in the source account, do the following:1. On the AWS Backup **Dashboard** page, choose **Create on-demand backup**.<br />2. In the **Settings** section, for **Resource type**, select **DynamoDB**, and then select the table name.<br />3. In the **Backup vault** dropdown list, select the backup vault that you created in the source account.<br />4. Select the **Retention period** that you want.<br />5. Choose **Create on-demand backup**. <br />A new backup job is created. <br />To monitor the status of the backup job, on the AWS Backup **Jobs** page, choose the **Backup Jobs** tab. All active, in-progress, and completed backup jobs are listed on this tab. | AWS DevOps, DBA, Migration engineer | 
| Copy the backup from the source account to the target account. | After the backup job is complete, copy the DynamoDB table backup from the backup vault in the source account to the backup vault in the target account.<br />To copy the backup vault, in the source account, do the following:1. On the [AWS Backup console](https://console.aws.amazon.com/backup/), choose **Backup vaults**.<br />2. Under **Backups**, choose the DynamoDB table backup.<br />3. Choose **Actions**, **Copy**.<br />4. Enter the AWS Region of the target account.<br />5. For **External vault ARN**, enter the ARN of the backup vault that you created in the target account.<br />6. To copy backups from the source account to the target account, in the target account backup vault, enable access from a different account. | AWS DevOps, Migration engineer, DBA | 
| Restore the backup in the target account. | In the target AWS account, do the following:1. On the [AWS Backup console](https://console.aws.amazon.com/backup/), choose **Backup vaults**.<br />2. Under **Backups**, select the backup that you copied from the source account.<br />3. Choose **Actions**, **Restore**.<br />4. Enter the name of the target DynamoDB table that you want to restore. | AWS DevOps, DBA, Migration engineer | 

## Related resources
<a name="copy-amazon-dynamodb-tables-across-accounts-using-aws-backup-resources"></a>
+ [Using AWS Backup with DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/backuprestore_HowItWorksAWS.html)
+ [Creating backup copies across AWS accounts](https://docs.aws.amazon.com/aws-backup/latest/devguide/create-cross-account-backup.html)
+ [AWS Backup pricing](https://aws.amazon.com/backup/pricing/)
