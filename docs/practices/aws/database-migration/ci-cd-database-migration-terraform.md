

# Set up a CI/CD pipeline for database migration by using Terraform
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform"></a>

*Dr. Rahul Sharad Gaikwad, Ashish Bhatt, Aniket Dekate, Ruchika Modi, Tamilselvan P, Nadeem Rahaman, Aarti Rajput, and Naveen Suthar, Amazon Web Services*

## Abstract

Use IaC and pipeline controls to make database migration changes repeatable, reviewable, and observable. The pattern is a starting point; production adoption still requires backup, rollback, validation, and change-approval controls.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Best practices](#best-practices)
- [Implementation](#epics)

## Summary
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform-summary"></a>

This pattern is about establishing a continuous integration and continuous deployment (CI/CD) pipeline for managing database migrations in a reliable and automated manner. It covers the process of provisioning the necessary infrastructure, migrating data, and customizing schema changes by using Terraform, which is an infrastructure as code (IaC) tool.

Specifically, the pattern sets up a CI/CD pipeline to migrate an on-premises Microsoft SQL Server database to Amazon Relational Database Service (Amazon RDS) on AWS. You can also use this pattern to migrate a SQL Server database that's on a virtual machine (VM) or in another cloud environment to Amazon RDS.

This pattern addresses the following challenges associated with database management and deployment:
+ Manual database deployments are time-consuming, error-prone, and lack consistency across environments.
+ Coordinating infrastructure provisioning, data migrations, and schema changes can be complex and difficult to manage.
+ Ensuring data integrity and minimizing downtime during database updates is crucial for production systems.

This pattern provides the following benefits:
+ Streamlines the process of updating and deploying database changes by implementing a CI/CD pipeline for database migrations. This reduces the risk of errors, ensures consistency across environments, and minimizes downtime.
+ Helps improve reliability, efficiency, and collaboration. Enables faster time to market and reduced downtime during database updates.
+ Helps you adopt modern DevOps practices for database management, which leads to increased agility, reliability, and efficiency in your software delivery processes.

## Prerequisites and limitations
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform-prereqs"></a>

**Prerequisites**
+ An active AWS account
+ Terraform 0.12 or later installed on your local machine (for instructions, see the [Terraform documentation](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli))
+ Terraform AWS Provider version 3.0.0 or later from HashiCorp (see the [GitHub repository](https://github.com/hashicorp/terraform-provider-aws) for this provider)
+ Least privilege AWS Identity and Access Management (IAM) policy (see the blog post [Techniques for writing least privilege IAM policies](https://aws.amazon.com/blogs/security/techniques-for-writing-least-privilege-iam-policies/))

## Architecture
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform-architecture"></a>

This pattern implements the following architecture, which provides the complete infrastructure for the database migration process.

![CI/CD pipeline architecture for migrating an on-premises SQL Server database to Amazon RDS on AWS.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/87845d9f-8e6e-4c51-b9ee-9e7833671d05/images/a1e95458-419a-4de9-85ef-b17d8340700a.png)


In this architecture:
+ The source database is a SQL Server database that is on premises, on a virtual machine (VM), or hosted by another cloud provider. The diagram assumes that the source database is in an on-premises data center.
+ The on-premises data center and AWS are connected through a VPN or AWS Direct Connect connection. This provides secure communications between the source database and the AWS infrastructure.
+ The target database is an Amazon RDS database that is hosted inside the virtual private cloud (VPC) on AWS with the help of a database provisioning pipeline.
+ AWS Database Migration Service (AWS DMS) replicates your on-premises database to AWS. It is used to configure the replication of the source database to the target database.

The following diagram shows the infrastructure set up with different levels of the database migration process, which involves provisioning, AWS DMS setup, and validation.

![CI/CD pipeline details of the migration process from on premises to AWS.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/87845d9f-8e6e-4c51-b9ee-9e7833671d05/images/3aca17e5-6fd7-4317-b578-ab5e485c6efb.png)


In this process:
+ The validation pipeline validates all checks. The integrated pipeline moves to the next step when all necessary validations are complete.
+ The DB provisioning pipeline consists of various AWS CodeBuild stages that perform Terraform actions on the provided Terraform code for the database. When these steps are complete, it deploys resources in the target AWS account.
+ The AWS DMS pipeline consists of various CodeBuild stages that perform tests and then provision the AWS DMS infrastructure for performing the migration by using IaC.

## Tools
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform-tools"></a>

**AWS services and tools**
+ [AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/welcome.html) is a fully managed continuous integration service that compiles source code, runs tests, and produces ready-to-deploy software packages.
+ [AWS CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html) is a fully managed continuous delivery service that helps you automate your release pipelines for fast and reliable application and infrastructure updates.
+ [Amazon Relational Database Service (Amazon RDS)](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) helps you set up, operate, and scale a relational database in the AWS Cloud.
+ [Amazon Simple Storage Service (Amazon S3)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) is an object storage service that offers scalability, data availability, security, and performance.
+ [AWS Database Migration Service (AWS DMS)](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html) helps you migrate data stores into the AWS Cloud or between combinations of cloud and on-premises setups.

**Other services**
+ [Terraform](https://www.terraform.io/) is an IaC tool from HashiCorp that helps you create and manage cloud and on-premises resources.

**Code repository**

The code for this pattern is available in the GitHub [Database Migration DevOps Framework using Terraform samples](https://github.com/aws-samples/aws-terraform-db-migration-framework-samples) repository.

## Best practices
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform-best-practices"></a>
+ Implement automated tests for your database migration to verify the correctness of schema changes and data integrity. This includes unit tests, integration tests, and end-to-end tests.
+ Implement a robust backup and restore strategy for your databases, especially before migration. This ensures data integrity and provides a fallback option in case of failures.
+ Implement a robust rollback strategy to revert database changes in case of failures or issues during migration. This could involve rolling back to a previous database state or reverting individual migration scripts.
+ Set up monitoring and logging mechanisms to track the progress and status of database migrations. This helps you identify and resolve issues quickly.

## Epics
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform-epics"></a>

### Set up your local workstation
<a name="set-up-your-local-workstation"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Set up and configure Git on your local workstation. | Install and configure Git on your local workstation by following the instructions in the [Git documentation](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git). | DevOps engineer | 
| Create a project folder and add the files from the GitHub repository. | 1. Open the [GitHub repository](https://github.com/aws-samples/aws-terraform-db-migration-framework-samples) for this pattern.<br />2. Choose **Code** to see cloning options, and copy the URL provided in the HTTPS tab.<br />3. Create a folder for your project on your workstation.<br />4. Open a terminal and navigate to this folder.<br />5. Clone the GitHub repository:<pre>git clone <github-repository-url></pre><br />where `<github-repository-url>` is the URL you copied in step 2.<br />6. When cloning is complete, go to the cloned repository in your project folder:<pre>cd <folder-name>/aws-terraform-db-migration-framework-samples</pre><br />7. Open this project in an integrated development environment (IDE) of your choice. | DevOps engineer | 

### Provision the target architecture
<a name="provision-the-target-architecture"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Update required parameters. | The `ssm-parameters.sh` file stores all required AWS Systems Manager parameters. You can configure these parameters with the custom values for your project.<br />In the `setup/db-ssm-params` folder on your local workstation, open the `ssm-parameters.sh` file and set these parameters before you run the CI/CD pipeline. | DevOps engineer | 
| Initialize the Terraform configuration. | In the `db-cicd-integration` folder, enter the following command to initialize your working directory that contains the Terraform configuration files:<pre>terraform init</pre> | DevOps engineer | 
| Preview the Terraform plan. | To create a Terraform plan, enter the following command:<pre>terraform plan -var-file="terraform.sample"  </pre><br />Terraform evaluates the configuration files to determine the target state for the declared resources. It then compares the target state against the current state and creates a plan. | DevOps engineer | 
| Verify the plan. | Review the plan and confirm that it configures the required architecture in your target AWS account. | DevOps engineer | 
| Deploy the solution. | 1. Enter the following command to apply the plan:<pre>terraform apply -var-file="terraform.sample"</pre><br />2. Enter `yes` to confirm. Terraform creates, updates, or destroys infrastructure to achieve the target state declared in the configuration files. For more information about the sequence, see the [Architecture](#set-up-ci-cd-pipeline-for-db-migration-with-terraform-architecture) section of this pattern. | DevOps engineer | 

### Verify the deployment
<a name="verify-the-deployment"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Validate the deployment. | Verify the status of the `db-cicd-integration` pipeline to confirm that the database migration is complete.<br />1. Sign in to the AWS Management Console, and then open the [AWS CodePipeline console](https://console.aws.amazon.com/codesuite/codepipeline/home).<br />2. In the navigation pane, choose **Pipelines**.<br />3. Choose the `db-cicd-integration` pipeline.<br />4. Validate that the pipeline execution has completed successfully. | DevOps engineer | 

### Clean up infrastructure after use
<a name="clean-up-infrastructure-after-use"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Clean up the infrastructure. | 1. After your project is complete, clean up the infrastructure you created by using the command:<pre>terraform destroy --var-file=terraform.sample</pre><br />2. Enter `yes` to confirm. | DevOps engineer | 

## Related resources
<a name="set-up-ci-cd-pipeline-for-db-migration-with-terraform-resources"></a>

**AWS documentation**
+ [Getting started with a Terraform product](https://docs.aws.amazon.com/servicecatalog/latest/adminguide/getstarted-Terraform.html)

**Terraform documentation**
+ [Terraform installation](https://learn.hashicorp.com/tutorials/terraform/install-cli)
+ [Terraform backend configuration](https://developer.hashicorp.com/terraform/language/backend)
+ [Terraform AWS Provider documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
