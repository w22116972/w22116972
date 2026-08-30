

# Connect to MGN data and control planes over a private network
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network"></a>

*Dipin Jain and Mike Kuznetsov, Amazon Web Services*

## Abstract

This pattern explains private connectivity to AWS Transform MGN control and data planes through interface VPC endpoints. Use it when migration policy prevents replication traffic from traversing public internet paths.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network-summary"></a>

This pattern explains how you can connect to an AWS Transform MGN data plane and control plane on a private, secured network by using interface VPC endpoints.

MGN is a highly automated lift-and-shift (rehost) solution that simplifies, expedites, and reduces the cost of migrating applications to AWS. It enables companies to rehost a large number of physical, virtual, or cloud servers without compatibility issues, performance disruption, or long cutover windows. MGN is available from the AWS Management Console. This enables seamless integration with other AWS services, such as AWS CloudTrail, Amazon CloudWatch, and AWS Identity and Access Management (IAM).

You can connect from a source data center to a data plane—that is, to a subnet that serves as a staging area for data replication in the destination VPC—over a private connection by using Site-to-Site VPN services, AWS Direct Connect, or VPC peering in MGN. You can also use [interface VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html) powered by AWS PrivateLink to connect to an MGN control plane over a private network. 

## Prerequisites and limitations
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network-prereqs"></a>

**Prerequisites **
+ **Staging area subnet** – Before you set up MGN, create a subnet to be used as a staging area for data replicated from your source servers to AWS (that is, a data plane). You must specify this subnet in the [Replication Settings template](https://docs.aws.amazon.com/mgn/latest/ug/template-vs-server.html) when you first access the MGN console. You can override this subnet for specific source servers in the Replication Settings template. Although you can use an existing subnet in your AWS account, we recommend that you create a new, dedicated subnet for this purpose.
+ **Network requirements** – The replication servers that are launched by MGN in your staging area subnet have to be able to send data to the MGN API endpoint at `https://mgn.<region>.amazonaws.com/`, where `<region>` is the code for the AWS Region you are replicating to (for example, `https://mgn.us-east-1.amazonaws.com`). Amazon Simple Storage Service (Amazon S3) service URLs are required for downloading MGN software.
  + The AWS Replication Agent installer should have access to the Amazon Simple Storage Service (Amazon S3) bucket URL of the AWS Region you are using with MGN.
  + The staging area subnet should have access to Amazon S3.
  + The source servers on which the AWS Replication Agent is installed must be able to send data to the replication servers in the staging area subnet and to the MGN API endpoint at `https://mgn.<region>.amazonaws.com/`.

The following table lists the required ports.


| 
| 
| Source | Destination | Port | For more information, see | 
| --- |--- |--- |--- |
| Source data center | Amazon S3 service URLs | 443 (TCP) | [Communication over TCP port 443](https://docs.aws.amazon.com/mgn/latest/ug/Network-Requirements.html#TCP-443) | 
| Source data center | AWS Region-specific console address for MGN | 443 (TCP) | [Communication between the source servers and MGN over TCP port 443](https://docs.aws.amazon.com/mgn/latest/ug/Network-Requirements.html#Source-Manager-TCP-443) | 
| Source data center | Staging area subnet | 1500 (TCP) | [Communication between the source servers and the staging area subnet over TCP port 1500](https://docs.aws.amazon.com/mgn/latest/ug/Network-Requirements.html#Communication-TCP-1500) | 
| Staging area subnet | AWS Region-specific console address for MGN | 443 (TCP) | [Communication between the staging area subnet and MGN over TCP port 443](https://docs.aws.amazon.com/mgn/latest/ug/Network-Requirements.html#Communication-TCP-443-Staging) | 
| Staging area subnet | Amazon S3 service URLs | 443 (TCP) | [Communication over TCP port 443](https://docs.aws.amazon.com/mgn/latest/ug/Network-Requirements.html#TCP-443) | 
| Staging area subnet | Amazon Elastic Compute Cloud (Amazon EC2) endpoint of the subnet’s AWS Region | 443 (TCP) | [Communication over TCP port 443](https://docs.aws.amazon.com/mgn/latest/ug/Network-Requirements.html#TCP-443) | 

** Limitations**

MGN isn’t currently available in all AWS Regions and operating systems.
+ [Supported AWS Regions](https://docs.aws.amazon.com/mgn/latest/ug/supported-regions.html)
+ [Supported operating systems](https://docs.aws.amazon.com/mgn/latest/ug/Supported-Operating-Systems.html)

## Architecture
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network-architecture"></a>

The following diagram illustrates the network architecture for a typical migration. For more information about this architecture, see the [MGN documentation](https://docs.aws.amazon.com/mgn/latest/ug/Network-Settings-Video.html) and the [MGN service architecture and network architecture video](https://youtu.be/ao8geVzmmRo).

![Network architecture for Application Migration Service for a typical migration](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/21346c0f-0643-4f4f-b21f-fdfe24fc6a8f/images/546598b2-8026-4849-a441-eaa2bc2bf6bb.png)


The following detailed view shows the configuration of interface VPC endpoints in the staging area VPC to connect Amazon S3 and MGN.

![Network architecture for Application Migration Service for a typical migration - detailed view](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/21346c0f-0643-4f4f-b21f-fdfe24fc6a8f/images/bd0dfd42-4ab0-466f-b696-804dedcf4513.png)


## Tools
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network-tools"></a>
+ [AWS Transform MGN](https://docs.aws.amazon.com/mgn/latest/ug/what-is-application-migration-service.html) simplifies, expedites, and reduces the cost of rehosting applications on AWS.
+ [Interface VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html) enable you to connect to services that are powered by AWS PrivateLink without requiring an internet gateway, NAT device, VPN connection, or AWS Direct Connect connection. Instances in your VPC do not require public IP addresses to communicate with resources in the service. Traffic between your VPC and the other service does not leave the Amazon network.

## Epics
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network-epics"></a>

### Create endpoints for MGN, Amazon EC2, and Amazon S3
<a name="create-endpoints-for-mgn-ec2-and-s3"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Configure the interface endpoint for MGN. | The source data center and staging area VPC connect privately to the MGN control plane through the interface endpoint that you create in the target staging area VPC. To create the endpoint:1. Open the [Amazon Virtual Private Cloud (Amazon VPC) console](https://console.aws.amazon.com/vpc/).<br />2. In the navigation pane, choose **Endpoints**, **Create Endpoint**.<br />3. For **Service category**, choose **AWS services**.<br />4. For **Service Name**, enter `com.amazonaws.<region>.mgn`. For **Type**, choose **Interface**.<br />5. For **VPC**, select a target staging area VPC to create the endpoint. <br />6. For **Subnets**, select the subnets in which to create the endpoint network interfaces.<br />7. To turn on private DNS for the interface endpoint, in the **Additional settings** section, select **Enable DNS Name**.<br />8. Select a security group that allows ingress from the staging area VPC subnet over TCP 443.<br />9. Choose **Create endpoint**.<br />For more information, see [Access an AWS service using an interface VPC endpoint in the Amazon VPC](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-interface.html) documentation. | Migration lead | 
| Configure the interface endpoint for Amazon EC2. | The staging area VPC connects privately to the Amazon EC2 API through the interface endpoint that you create in the target staging area VPC. To create the endpoint, follow the instructions provided in the previous story.+ For service name, enter `com.amazonaws.<region>.ec2`. For **Type**, choose **Interface**.<br />+ The security group must allow inbound HTTPS traffic from the staging area VPC subnet over port 443.<br />+ In the** Additional settings** section, select **Enable DNS name**. | Migration lead | 
| Configure the interface endpoint for Amazon S3. | The source data center and staging area VPC connect privately to the Amazon S3 API through the interface endpoint that you create in the target staging area VPC. To create the endpoint, follow the instructions provided in the first story.+ For **Service Name**, enter `com.amazonaws.<region>.s3`. For **Type**, choose **Interface**.<br />+ The VPC security group must allow inbound HTTPS traffic from the staging area VPC subnet over port 443.<br />+ In the **Additional settings** section, clear **Enable DNS name**. Amazon S3 interface endpoints do not support private DNS names. You use an interface endpoint because gateway endpoint connections cannot be extended out of a VPC. (For details, see the [AWS PrivateLink documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway.html).) | Migration lead | 
| Configure the Amazon S3 Gateway endpoint. | During the configuration phase, the replication server has to connect to an S3 bucket to download the AWS Replication Server’s software updates. However, Amazon S3 interface endpoints do not support private DNS names*,* and there is no way to supply an Amazon S3 endpoint DNS name to a replication server. <br />To mitigate this issue, you create an Amazon S3 gateway endpoint in the VPC that the staging area subnet belongs to, and update the staging subnet’s route tables with the relevant routes. For more information, see [Create a gateway endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html#create-gateway-endpoint-s3) in the AWS PrivateLink documentation. | Cloud administrator | 
| Configure on-premises DNS to resolve private DNS names for endpoints. | The interface endpoints for MGN and Amazon EC2 have private DNS names that can be resolved in the VPC. However, you also need to configure on-premises servers to resolve private DNS names for these interface endpoints.<br />There are multiple ways to configure these servers. In this pattern, we tested this functionality by forwarding on-premises DNS queries to the Amazon Route 53 Resolver inbound endpoint in the staging area VPC. For more information, see [Resolving DNS queries between VPCs and your network](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-overview-DSN-queries-to-vpc.html) in the Route 53 documentation. | Migration engineer | 

### Connect to the MGN control plane over a private link
<a name="connect-to-the-mgn-control-plane-over-a-private-link"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Install AWS Replication Agent by using AWS PrivateLink. | 1. Download the AWS Replication Agent to a private S3 bucket in the destination Region.<br />2. Log in to the source servers to be migrated. The AWS Replication Agent installer needs network access to the MGN and Amazon S3 endpoints. Because your on-premises network isn’t open to MGN and Amazon S3 public endpoints, you must install the Agent with the help of the interface endpoints you created in the previous steps by using AWS PrivateLink.Here’s an example for Linux:1. Download the Agent by using the command:<pre>wget -O ./aws-replication-installer-init.py \ <br />https://aws-application-migration-service-<aws_region>.bucket.<s3-endpoint-DNS-name>/latest/linux/aws-replication-installer-init.py</pre><br />For example, if the DNS name of the Amazon S3 interface endpoint is `vpce-009c8b07adb052a11-qgf8q50y.s3.us-west-1.vpce.amazonaws.com` and the AWS Region is `us-west-1`, you would use the command:<pre>wget -O ./aws-replication-installer-init.py \<br />https://aws-application-migration-service-us-west-1.bucket.vpce-009c8b07adb052a11-qgf8q50y.s3.us-west-1.vpce.amazonaws.com/latest/linux/aws-replication-installer-init.py</pre>`bucket` is a static keyword that you must add before the Amazon S3 interface endpoint DNS name. For more information, see the [Amazon S3 documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/privatelink-interface-endpoints.html#accessing-bucket-and-aps-from-interface-endpoints).<br />2. Install the Agent:If you selected **Enable DNS name** when you created an interface endpoint for MGN, run the command:<pre>     sudo python3 aws-replication-installer-init.py \<br />     --region <aws_region> \<br />     --aws-access-key-id <access-key> \<br />     --aws-secret-access-key <secret-key> \<br />     --no-prompt \<br />     --s3-endpoint <s3-endpoint-DNS-name></pre>If you didn’t select **Enable DNS name** when you created the interface endpoint for MGN, run the command:<pre>     sudo python3 aws-replication-installer-init.py \<br />     --region <aws_region> \<br />     --aws-access-key-id <access-key> \<br />     --aws-secret-access-key <secret-key> \<br />     --no-prompt \<br />     --s3-endpoint <s3-endpoint-DNS-name> \<br />     --endpoint <mgn-endpoint-DNS-name></pre><br />For more information, see [AWS Replication Agent installation instructions](https://docs.aws.amazon.com/mgn/latest/ug/agent-installation.html) in the MGN documentation.<br />After you have established your connection with MGN and installed the AWS Replication Agent, follow the instructions in the [MGN documentation](https://docs.aws.amazon.com/mgn/latest/ug/migration-workflow-gs.html) to migrate your source servers to your target VPC and subnet. | Migration engineer | 

## Related resources
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network-resources"></a>

**MGN documentation**
+ [Concepts](https://docs.aws.amazon.com/mgn/latest/ug/CloudEndure-Concepts.html)
+ [Migration workflow ](https://docs.aws.amazon.com/mgn/latest/ug/migration-workflow-gs.html)
+ [Quick start guide](https://docs.aws.amazon.com/mgn/latest/ug/quick-start-guide-gs.html)
+ [FAQ](https://docs.aws.amazon.com/mgn/latest/ug/FAQ.html)
+ [Troubleshooting](https://docs.aws.amazon.com/mgn/latest/ug/troubleshooting.html)

**Additional resources**
+ [Rehosting your applications in a multi-account architecture on AWS by using VPC interface endpoints](https://docs.aws.amazon.com/prescriptive-guidance/latest/rehost-multi-account-architecture-interface-endpoints/) (AWS Prescriptive Guidance guide)
+ [AWS Transform MGN – A Technical Introduction](https://www.aws.training/Details/eLearning?id=71732) (AWS Training and Certification walkthrough)
+ [AWS Transform MGN architecture and network architecture](https://youtu.be/ao8geVzmmRo) (video)

## Additional information
<a name="connect-to-application-migration-service-data-and-control-planes-over-a-private-network-additional"></a>

**Troubleshooting ***AWS ***Replication Agent installations on Linux servers**

If you get a **gcc** error on an Amazon Linux server, configure the package repository and use the following command:

```
## sudo yum groupinstall "Development Tools"
```
