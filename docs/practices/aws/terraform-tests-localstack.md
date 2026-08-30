

# Test AWS infrastructure by using LocalStack and Terraform Tests
<a name="test-aws-infra-localstack-terraform"></a>

*Ivan Girardi and Ioannis Kalyvas, Amazon Web Services*

## Abstract

Use Terraform tests with LocalStack for fast, low-cost feedback on supported AWS resource behavior. Keep a separate validation path for real AWS integration, IAM, quotas, and service behavior that emulation cannot prove.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Best practices](#best-practices)
- [Implementation](#epics)

## Summary
<a name="test-aws-infra-localstack-terraform-summary"></a>

This pattern helps you locally test infrastructure as code (IaC) for AWS in Terraform without the need to provision infrastructure in your AWS environment. It integrates the [Terraform Tests framework](https://developer.hashicorp.com/terraform/language/tests) with [LocalStack](https://github.com/localstack/localstack). The LocalStack Docker container provides a local development environment that emulates various AWS services. This helps you test and iterate on infrastructure deployments without incurring costs in the AWS Cloud.

This solution provides the following benefits:
+ **Cost optimization** – Running tests against LocalStack eliminates the need to use AWS services. This prevents you from incurring costs that are associated with creating, operating, and modifying those AWS resources.
+ **Speed and efficiency** – Testing locally is also typically faster than deploying the AWS resources. This rapid feedback loop accelerates development and debugging. Because LocalStack runs locally, you can develop and test your Terraform configuration files without an internet connection. You can debug Terraform configuration files locally and receive immediate feedback, which streamlines the development process.
+ **Consistency and reproducibility** – LocalStack provides a consistent environment for testing. This consistency helps make sure that tests yield the same results, regardless of external AWS changes or network issues.
+ **Isolation **– Testing with LocalStack prevents you from accidentally affecting live AWS resources or production environments. This isolation makes it safe to experiment and test various configurations.
+ **Automation** – Integration with a continuous integration and continuous delivery (CI/CD) pipeline helps you automatically test Terraform [configuration files](https://developer.hashicorp.com/terraform/language/files). The pipeline thoroughly tests the IaC before deployment.
+ **Flexibility** – You can simulate different AWS Regions, AWS accounts, and service configurations to match your production environments more closely.

## Prerequisites and limitations
<a name="test-aws-infra-localstack-terraform-prereqs"></a>

**Prerequisites**
+ [Install](https://docs.docker.com/get-started/get-docker/) Docker
+ [Enable access](https://docs.docker.com/reference/cli/dockerd/#daemon-socket-option) to the default Docker socket (`/var/run/docker.sock`). For more information, see the [LocalStack documentation](https://docs.localstack.cloud/user-guide/aws/lambda/#migrating-to-lambda-v2).
+ [Install](https://docs.docker.com/compose/install/) Docker Compose
+ [Install](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli) Terraform version 1.6.0 or later
+ [Install](https://developer.hashicorp.com/terraform/cli) Terraform CLI
+ [Configure](https://hashicorp.github.io/terraform-provider-aws/) the Terraform AWS Provider
+ (Optional) [Install](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and [configure](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) the AWS Command Line Interface (AWS CLI). For an example of how to use the AWS CLI with LocalStack, see the GitHub [Test AWS infrastructure using LocalStack and Terraform Tests](https://github.com/aws-samples/localstack-terraform-test) repository.

**Limitations**
+ This pattern provides explicit examples for testing Amazon Simple Storage Service (Amazon S3), AWS Lambda, AWS Step Functions, and Amazon DynamoDB resources. However, you can extend this solution to include additional AWS resources.
+ This pattern provides instructions to run Terraform Tests locally, but can you can integrate testing into any CI/CD pipeline.
+ This pattern provides instructions for using the LocalStack Community image. If you're using the LocalStack Pro image, see the [LocalStack Pro documentation](https://hub.docker.com/r/localstack/localstack-pro).
+ LocalStack provides emulation services for different AWS APIs. For a complete list, see [AWS Service Feature Coverage](https://docs.localstack.cloud/user-guide/aws/feature-coverage/). Some advanced features might require a subscription for LocalStack Pro.

## Architecture
<a name="test-aws-infra-localstack-terraform-architecture"></a>

The following diagram shows the architecture for this solution. The primary components are a source code repository, a CI/CD pipeline, and a LocalStack Docker container. The LocalStack Docker Container hosts the following AWS services locally:
+ An Amazon S3 bucket for storing files
+ Amazon CloudWatch for monitoring and logging
+ An AWS Lambda function for running serverless code
+ An AWS Step Functions state machine for orchestrating multi-step workflows
+ An Amazon DynamoDB table for storing NoSQL data

![A CI/CD pipeline builds and tests the LocalStack Docker container and AWS resources.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/34bfbdbf-14e7-42a0-9022-c85a9c30cdcd/images/dc61fac9-b92c-4841-9132-ff8bb865eed9.png)


The diagram shows the following workflow:

1. You add and commit a Terraform configuration file to the source code repository.

1. The CI/CD pipeline detects the changes and initiates a build process for static Terraform code analysis. The pipeline builds and runs the LocalStack Docker container. Then the pipeline starts the test process.

1. The pipeline uploads an object into an Amazon S3 bucket that is hosted in the LocalStack Docker container.

1. Uploading the object invokes an AWS Lambda function.

1. The Lambda function stores the Amazon S3 event notification in a CloudWatch log.

1. The Lambda function starts an AWS Step Functions state machine.

1. The state machine writes the name of the Amazon S3 object into a DynamoDB table.

1. The test process in the CI/CD pipeline verifies that the name of the uploaded object matches the entry in the DynamoDB table. It also verifies that the S3 bucket is deployed with the specified name and that the AWS Lambda function has been successfully deployed.

## Tools
<a name="test-aws-infra-localstack-terraform-tools"></a>

**AWS services**
+ [Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html) helps you monitor the metrics of your AWS resources and the applications you run on AWS in real time.
+ [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) is a fully managed NoSQL database service that provides fast, predictable, and scalable performance.
+ [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) is a compute service that helps you run code without needing to provision or manage servers. It runs your code only when needed and scales automatically, so you pay only for the compute time that you use.
+ [Amazon Simple Storage Service (Amazon S3)](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) is a cloud-based object storage service that helps you store, protect, and retrieve any amount of data.
+ [AWS Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html) is a serverless orchestration service that helps you combine AWS Lambda functions and other AWS services to build business-critical applications.

**Other tools**
+ [Docker](https://www.docker.com/) is a set of platform as a service (PaaS) products that use virtualization at the operating-system level to deliver software in containers.
+ [Docker Compose](https://docs.docker.com/compose/) is a tool for defining and running multi-container applications.
+ [LocalStack](https://localstack.cloud) is a cloud service emulator that runs in a single container. By using LocalStack, you can run workloads on your local machine that use AWS services, without connecting to the AWS Cloud.
+ [Terraform](https://www.terraform.io/) is an IaC tool from HashiCorp that helps you create and manage cloud and on-premises resources.
+ [Terraform Tests](https://developer.hashicorp.com/terraform/language/tests) helps you validate Terraform module configuration updates through tests that are analogous to integration or unit testing.

**Code repository**

The code for this pattern is available in the GitHub [Test AWS infrastructure using LocalStack and Terraform Tests](https://github.com/aws-samples/localstack-terraform-test) repository.

## Best practices
<a name="test-aws-infra-localstack-terraform-best-practices"></a>
+ This solution tests AWS infrastructure that is specified in Terraform configuration files, and it does not deploy those resources in the AWS Cloud. If you want to deploy the resources, follow the [principle of least-privilege](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#grant-least-privilege) (IAM documentation) and properly [configure the Terraform backend](https://developer.hashicorp.com/terraform/language/backend) (Terraform documentation).
+ When integrating LocalStack in a CI/CD pipeline, we recommend that you don't run the LocalStack Docker container in privilege mode. For more information, see [Runtime privilege and Linux capabilities](https://docs.docker.com/engine/containers/run/#runtime-privilege-and-linux-capabilities) (Docker documentation) and [Security for self-managed runners](https://docs.gitlab.com/runner/security/) (GitLab documentation).

## Epics
<a name="test-aws-infra-localstack-terraform-epics"></a>

### Deploy the solution
<a name="deploy-the-solution"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Clone the repository. | In a bash shell, enter the following command. This clones the [Test AWS infrastructure using LocalStack and Terraform Tests](https://github.com/aws-samples/localstack-terraform-test) repository from GitHub:<pre>git clone https://github.com/aws-samples/localstack-terraform-test.git</pre> | DevOps engineer | 
| Run the LocalStack container. | 1. Enter the following command to navigate into the cloned repository:<pre>cd localstack-terraform-test</pre><br />2. Enter the following command to start the LocalStack Docker container in detached mode:<pre>docker-compose up -d</pre><br />3. Wait until the LocalStack Docker container is operational. | DevOps engineer | 
| Initialize Terraform. | Enter the following command to initialize Terraform:<pre>terraform init</pre> | DevOps engineer | 
| Run Terraform Tests. | 1. Enter the following command to run Terraform Tests:<pre>terraform test</pre><br />2. Verify that all tests completed successfully. The output should be similar to the following:<pre>Success! 3 passed, 0 failed.</pre> | DevOps engineer | 
| Clean up resources. | Enter the following command to destroy the LocalStack container:<pre>docker-compose down</pre> | DevOps engineer | 

## Troubleshooting
<a name="test-aws-infra-localstack-terraform-troubleshooting"></a>


| Issue | Solution | 
| --- | --- | 
| `Error: reading DynamoDB Table Item (Files\|README.md): empty` result when running the `terraform test` command. | 1. Reenter the `terraform test` command.<br />2. If that doesn't resolve the error, edit the **main.tf** file to increase the sleep timeout to a value greater than 15 seconds:<pre>resource "time_sleep" "wait" {<br />  create_duration = "15s"<br />  triggers = {<br />    s3_object = local.key_json<br />  }<br />}</pre> | 

## Related resources
<a name="test-aws-infra-localstack-terraform-resources"></a>
+ [Getting started with Terraform: Guidance for AWS CDK and AWS CloudFormation experts](https://docs.aws.amazon.com/prescriptive-guidance/latest/getting-started-terraform/introduction.html) (AWS Prescriptive Guidance)
+ [Best practices for using the Terraform AWS Provider](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/introduction.html) (AWS Prescriptive Guidance)
+ [Terraform CI/CD and testing on AWS with the new Terraform Test Framework](https://aws.amazon.com/blogs/devops/terraform-ci-cd-and-testing-on-aws-with-the-new-terraform-test-framework/) (AWS blog post)
+ [Accelerating software delivery using LocalStack Cloud Emulator from AWS Marketplace](https://aws.amazon.com/blogs/awsmarketplace/accelerating-software-delivery-localstack-cloud-emulator-aws-marketplace/) (AWS blog post)

## Additional information
<a name="test-aws-infra-localstack-terraform-additional"></a>

**Integration with GitHub Actions**

You can integrate LocalStack and Terraform Tests in a CI/CD pipeline by using GitHub Actions. For more information, see the [GitHub Actions documentation](https://docs.github.com/en/actions). The following is a sample GitHub Actions configuration file:

```
name: LocalStack Terraform Test

on:
  push:
    branches:
      - '**'

  workflow_dispatch: {}

jobs:
  localstack-terraform-test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Build and Start LocalStack Container
      run: |
        docker compose up -d

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: latest

    - name: Run Terraform Init and Validation
      run: |
        terraform init
        terraform validate
        terraform fmt --recursive --check
        terraform plan
        terraform show

    - name: Run Terraform Test
      run: |
        terraform test

    - name: Stop and Delete LocalStack Container
      if: always()
      run: docker compose down
```
