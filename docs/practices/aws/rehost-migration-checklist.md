

# Rehost on-premises workloads in the AWS Cloud: migration checklist
<a name="rehost-on-premises-workloads-in-the-aws-cloud-migration-checklist"></a>

*Srikanth Rangavajhala, Amazon Web Services*

## Abstract

Use this checklist to plan and validate a rehost migration wave from source-server discovery through cutover and handoff. It is most useful when paired with application dependency mapping, rollback criteria, and business acceptance testing.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="rehost-on-premises-workloads-in-the-aws-cloud-migration-checklist-summary"></a>

Rehosting on-premises workloads in the Amazon Web Services (AWS) Cloud involves the following migration phases: planning, pre-discovery, discovery, build, test, and cutover. This pattern outlines the phases and their related tasks. The tasks are described at a high level and support about 75% of all application workloads. You can implement these tasks over two to three weeks in an agile sprint cycle.

You should review and vet these tasks with your migration team and consultants. After the review, you can gather the input, eliminate or re-evaluate tasks as necessary to meet your requirements, and modify other tasks to support at least 75% of the application workloads in your portfolio. You can then use an agile project management tool such as Atlassian Jira or Rally Software to import the tasks, assign them to resources, and track your migration activities. 

The pattern assumes that you're using [AWS Cloud Migration Factory](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/solution-overview.html) to rehost your workloads, but you can use your migration tool of choice.

Amazon Macie can help identify sensitive data in your knowledge bases, stored as data sources, model invocation logs, and prompt stores in Amazon Simple Storage Service (Amazon S3) buckets. For more information, see the [Macie documentation](https://docs.aws.amazon.com/macie/latest/user/data-classification.html).

## Prerequisites and limitations
<a name="rehost-on-premises-workloads-in-the-aws-cloud-migration-checklist-prereqs"></a>

**Prerequisites**
+ Project management tool for tracking migration tasks (for example, Atlassian Jira or Rally Software)
+ Migration tool for rehosting your workloads on AWS (for example, [Cloud Migration Factory](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/solution-overview.html))

## Architecture
<a name="rehost-on-premises-workloads-in-the-aws-cloud-migration-checklist-architecture"></a>

**Source platform  **
+ On-premises source stack (including technologies, applications, databases, and infrastructure)  

**Target platform**
+ AWS Cloud target stack (including technologies, applications, databases, and infrastructure) 

**Architecture**

The following diagram illustrates rehosting (discovering and migrating servers from an on-premises source environment to AWS) by using Cloud Migration Factory and AWS Application Migration Service.

![Rehosting servers on AWS by using Cloud Migration Factory and Application Migration Service](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/8e2d2d72-30cc-4e98-8abd-ac2ef95e599b/images/735ad65b-2646-4803-82c9-f7f93369b3a5.png)


## Tools
<a name="rehost-on-premises-workloads-in-the-aws-cloud-migration-checklist-tools"></a>
+ You can use a migration and project management tool of your choice.

## Epics
<a name="rehost-on-premises-workloads-in-the-aws-cloud-migration-checklist-epics"></a>

### Planning phase
<a name="planning-phase"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Groom the pre-discovery backlog. | Conduct the pre-discovery backlog grooming working session with department leads and application owners.  | Project manager, Agile scrum leader | 
|  Conduct the sprint planning working session. | As a scoping exercise, distribute the applications that you want to migrate across sprints and waves. | Project manager, Agile scrum leader | 

### Pre-discovery phase
<a name="pre-discovery-phase"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Confirm application knowledge. | Confirm and document the application owner and their knowledge of the application. Determine whether there's another point person for technical questions. | Migration specialist (interviewer) | 
| Determine application compliance requirements. | Confirm with the application owner that the application doesn't have to comply with requirements for Payment Card Industry Data Security Standard (PCI DSS), Sarbanes-Oxley Act (SOX), personally identifiable information (PII), or other standards. If compliance requirements exist, teams must finish their compliance checks on the servers that will be migrated. | Migration specialist (interviewer) | 
| Confirm production release requirements.  | Confirm the requirements for releasing the migrated application to production (such as release date and downtime duration) with the application owner or technical contact. | Migration specialist (interviewer) | 
| Get server list. | Get the list of servers that are associated with the targeted application. | Migration specialist (interviewer) | 
| Get the logical diagram that shows the current state. | Obtain the current state diagram for the application from the enterprise architect or the application owner. | Migration specialist (interviewer) | 
| Create a logical diagram that shows the target state. | Create a logical diagram of the application that shows the target architecture on AWS. This diagram should illustrate servers, connectivity, and mapping factors. | Enterprise architect, Business owner | 
| Get server information. | Collect information about the servers that are associated with the application, including their configuration details. | Migration specialist (interviewer) | 
| Add server information to the discovery template. | Add detailed server information to the application discovery template (see `mobilize-application-questionnaire.xlsx` in the attachment for this pattern). This template includes all the application-related security, infrastructure, operating system, and networking details. | Migration specialist (interviewer) | 
| Publish the application discovery template. | Share the application discovery template with the application owner and migration team for common access and use. | Migration specialist (interviewer) | 

### Discovery phase
<a name="discovery-phase"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Confirm server list. | Confirm the list of servers and the purpose of each server with the application owner or technical lead. | Migration specialist | 
| Identify and add server groups. | Identify server groups such as web servers or application servers, and add this information to the application discovery template. Select the tier of the application (web, application, database) that each server should belong to. | Migration specialist | 
| Fill in the application discovery template. | Complete the details of the application discovery template with the help of the migration team, application team, and AWS. | Migration specialist | 
| Add missing server details (middleware and OS teams). | Ask middleware and operating system (OS) teams to review the application discovery template and add any missing server details, including database information. | Migration specialist | 
| Get inbound/outbound traffic rules (network team). | Ask the network team to get the inbound/outbound traffic rules for the source and destination servers. The network team should also add existing firewall rules, export these to a security group format, and add existing load balancers to the application discovery template. | Migration specialist | 
| Identify required tagging. | Determine the tagging requirements for the application. | Migration specialist | 
| Create firewall request details. | Capture and filter the firewall rules that are required to communicate with the application.  | Migration specialist, Solutions architect, Network lead  | 
| Update the EC2 instance type. | Update the Amazon Elastic Compute Cloud (Amazon EC2) instance type to be used in the target environment, based on infrastructure and server requirements.  | Migration specialist, Solutions architect, Network lead | 
| Identify the current state diagram. | Identify or create the diagram that shows the current state of the application. This diagram will be used in the information security (InfoSec) request.  | Migration specialist, Solutions architect | 
| Finalize the future state diagram. | Finalize the diagram that shows the future (target) state for the application. This diagram will also be used in the InfoSec request.   | Migration specialist, Solutions architect | 
| Create firewall or security group service requests. | Create firewall or security group service requests (for development/QA, pre-production, and production). If you're using Cloud Migration Factory, include replication-specific ports if they're not already open.  | Migration specialist, Solutions architect, Network lead | 
| Review firewall or security group requests (InfoSec team). | In this step, the InfoSec team reviews and approves the firewall or security group requests that were created in the previous step.  | InfoSec engineer, Migration specialist | 
| Implement firewall security group requests (network team). | After the InfoSec team approves the firewall requests, the network team implements the required inbound/outbound firewall rules.  | Migration specialist, Solutions architect, Network lead | 

### Build phase (repeat for development/QA, pre-production, and production environments)
<a name="build-phase-repeat-for-development-qa-pre-production-and-production-environments"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Import the application and server data. | 1. Verify that you are logged in to your migration execution server as a domain user with local administrator permissions on the in-scope source servers.<br />2. Use the migration intake form to import the attributes for the in-scope source servers. For additional information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/step8.html). If you aren't using Cloud Migration Factory, follow the instructions for setting up your migration tool. | Migration specialist, Cloud administrator | 
| Check prerequisites for source servers. | Connect with the in-scope source servers to verify prerequisites such as TCP port 1500, TCP port 443, root volume free space, .NET Framework version, and other parameters. These are required for replication. For additional information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#prerequisites-2). | Migration specialist, Cloud administrator | 
| Create a service request to install replication agents.  | Create a service request to install replication agents on the in-scope servers for development/QA, pre-production, or production. | Migration specialist, Cloud administrator | 
| Install the replication agents. | Install the replication agents on the in-scope source servers on the development/QA, pre-production, or production machines. For additional information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#install-the-replication-agents). | Migration specialist, Cloud administrator | 
| Push the post-launch scripts. | Application Migration Service supports post-launch scripts to help you automate OS-level activities such as installing or uninstalling software after you launch target instances. This step pushes the post-launch scripts to Windows or Linux machines, depending on the servers identified for migration. For instructions, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#push-the-post-launch-scripts). | Migration specialist, Cloud administrator | 
| Verify the replication status. | Confirm the replication status for the in-scope source servers automatically by using the provided script. The script repeats every five minutes until the status of all source servers in the given wave changes to **Healthy**. For instructions, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#verify-the-replication-status). | Migration specialist, Cloud administrator | 
| Create the admin user. | A local admin or sudo user on source machines might be needed to troubleshoot any issues after migration cutover from the in-scope source servers to AWS. The migration team uses this user to log in to the target server when the authentication server (for example, the DC or LDAP server) is not reachable. For instructions for this step, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/step4.html#add-a-user-to-the-admin-group). | Migration specialist, Cloud administrator | 
| Validate the launch template. | Validate the server metadata to make sure it works successfully and has no invalid data. This step validates both test and cutover metadata. For instructions, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#validate-launch-template-1). | Migration specialist, Cloud administrator | 

### Test phase (repeat for development/QA, pre-production, and production environments)
<a name="test-phase-repeat-for-development-qa-pre-production-and-production-environments"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create a service request. | Create a service request for the infrastructure team and other teams to perform application cutover to development/QA, pre-production, or production instances.  | Migration specialist, Cloud administrator | 
| Configure a load balancer (optional). | Configure required load balancers, such as an [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-application-load-balancer.html) or an [F5 load balancer](https://www.f5.com/resources/white-papers/load-balancing-101-nuts-and-bolts) with iRules. | Migration specialist, Cloud administrator | 
| Launch instances for testing. | Launch all target machines for a given wave in Application Migration Service in test mode. For additional information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#launch-instances-for-testing). | Migration specialist, Cloud administrator | 
| Verify the target instance status. | Verify the status of the target instance by checking the bootup process for all in-scope source servers in the same wave. It may take up to 30 minutes for the target instances to boot up. You can check the status manually by logging in to the Amazon EC2 console, searching for the source server name, and reviewing the **Status check** column. The status **2/2 checks passed** indicates that the instance is healthy from an infrastructure perspective. | Migration specialist, Cloud administrator | 
| Modify DNS entries. | Modify Domain Name System (DNS) entries. (Use `resolv.conf` or `host.conf` for a Microsoft Windows environment.) Configure each EC2 instance to point to the new IP address of this host.Make sure that there are no DNS conflicts between on-premises and AWS Cloud servers. This step and the following steps are optional, depending on the environment where the server is hosted. | Migration specialist, Cloud administrator | 
| Test connectivity to backend hosts from EC2 instances. | Check the logins by using the domain credentials for the migrated servers. | Migration specialist, Cloud administrator | 
| Update the DNS A record. | Update the DNS A record for each host to point to the new Amazon EC2 private IP address. | Migration specialist, Cloud administrator | 
| Update the DNS CNAME record. | Update the DNS CNAME record for virtual IPs (load balancer names) to point to the cluster for web and application servers. | Migration specialist, Cloud administrator | 
| Test the application in applicable environments. | Log in to the new EC2 instance and test the application in the development/QA, pre-production, and production environments. | Migration specialist, Cloud administrator | 
| Mark as ready for cutover. | When testing is complete, change the status of the source server to indicate that it's ready for cutover, so users can launch a cutover instance. For instructions, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#mark-as-ready-for-cutover). | Migration specialist, Cloud administrator | 

### Cutover phase
<a name="cutover-phase"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Create production deployment plan. | Create a production deployment plan (including a backout plan). | Migration specialist, Cloud administrator | 
| Notify operations team of downtime. | Notify the operations team of the downtime schedule for the servers. Some teams might require a change request or service request (CR/SR) ticket for this notification. | Migration specialist, Cloud administrator | 
| Replicate production machines. | Replicate production machines by using Application Migration Service or another migration tool. | Migration specialist, Cloud administrator | 
| Shut down in-scope source servers. | After you verify the source servers’ replication status, you can shut down the source servers to stop transactions from client applications to the servers. You can shut down the source servers in the cutover window. For more information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#shut-down-the-in-scope-source-servers). | Cloud administrator | 
| Launch instances for cutover. | Launch all target machines for a given wave in Application Migration Service in cutover mode. For more information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-factory-web-console.html#launch-instances-for-cutover). | Migration specialist, Cloud administrator | 
| Retrieve target instance IPs. | Retrieve the IPs for target instances. If the DNS update is a manual process in your environment, you would need to get the new IP addresses for all target instances. For more information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-command-prompt.html#retrieve-the-target-instance-ip). | Migration specialist, Cloud administrator | 
| Verify target server connections. | After you update the DNS records, connect to the target instances with the host name to verify the connections. For more information, see the [Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/list-of-automated-migration-activities-using-command-prompt.html#verify-the-target-server-connections). | Migration specialist, Cloud administrator | 

## Related resources
<a name="rehost-on-premises-workloads-in-the-aws-cloud-migration-checklist-resources"></a>
+ [How to migrate](https://aws.amazon.com/migrate-modernize-build/cloud-migration/how-to-migrate/)
+ [AWS Cloud Migration Factory Implementation Guide](https://docs.aws.amazon.com/solutions/latest/cloud-migration-factory-on-aws/solution-overview.html)
+ [Automating large-scale server migrations with Cloud Migration Factory](https://docs.aws.amazon.com/prescriptive-guidance/latest/migration-factory-cloudendure/welcome.html)
+ [AWS Application Migration Service User Guide](https://docs.aws.amazon.com/mgn/latest/ug/what-is-application-migration-service.html)
+ [AWS Migration Acceleration Program](https://aws.amazon.com/migration-acceleration-program/)

## Attachments
<a name="attachments-8e2d2d72-30cc-4e98-8abd-ac2ef95e599b"></a>

To access additional content that is associated with this document, download and unzip the following file: [attachment.zip](samples/p-attach/8e2d2d72-30cc-4e98-8abd-ac2ef95e599b/attachments/attachment.zip)
