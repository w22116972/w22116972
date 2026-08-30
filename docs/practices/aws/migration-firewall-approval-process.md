

# Create an approval process for firewall requests during a rehost migration to AWS
<a name="create-an-approval-process-for-firewall-requests-during-a-rehost-migration-to-aws"></a>

*Srikanth Rangavajhala, Amazon Web Services*

## Abstract

Use this governance pattern to collect network requirements, obtain InfoSec approval, and reduce late firewall changes during rehost migration waves. It is a process reference, not a replacement for customer security policy.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Attachments](#attachments)

## Summary
<a name="create-an-approval-process-for-firewall-requests-during-a-rehost-migration-to-aws-summary"></a>

If you want to use [AWS Transform MGN](https://docs.aws.amazon.com/mgn/latest/ug/what-is-application-migration-service.html) or [Cloud Migration Factory on AWS](https://aws.amazon.com/solutions/implementations/cloud-migration-factory-on-aws/) for a rehost migration to the AWS Cloud, one of the prerequisites is that you must keep TCP ports 443 and 1500 open. Typically, opening these firewall ports requires approval from your information security (InfoSec) team.

This pattern outlines the process to obtain a firewall request approval from an InfoSec team during a rehost migration to the AWS Cloud. You can use this process to avoid rejections of your firewall request by the InfoSec team, which can become expensive and time consuming. The firewall request process has two review and approval steps between AWS migration consultants and leads who work with your InfoSec and application teams to open the firewall ports.

This pattern assumes that you are planning a rehost migration with AWS consultants or migration specialists from your organization. You can use this pattern if your organization doesn’t have a firewall approval process or firewall request blanket approval form. For more information about this, see the *Limitations* section of this pattern. For more information on network requirements for MGN, see [Network requirements](https://docs.aws.amazon.com/mgn/latest/ug/Network-Requirements.html) in the MGN documentation.

## Prerequisites and limitations
<a name="create-an-approval-process-for-firewall-requests-during-a-rehost-migration-to-aws-prereqs"></a>

**Prerequisites **
+ A planned rehost migration with AWS consultants or migration specialists from your organization
+ The required port and IP information to migrate the stack
+ Existing and future state architecture diagrams
+ Firewall information about the on-premises and destination infrastructure, ports, and zone-to-zone traffic flow
+ A firewall request review checklist (attached)
+ A firewall request document, configured according to your organization’s requirements
+ A contact list for the firewall reviewers and approvers, including the following roles:
  + **Firewall request submitter** – AWS migration specialist or consultant. The firewall request submitter can also be a migration specialist from your organization.
  + **Firewall request reviewer** – Typically, this is the single point of contact (SPOC) from AWS.
  + **Firewall request approver** – An InfoSec team member.

**Limitations **
+ This pattern describes a generic firewall request approval process. Requirements can vary for individual organizations.
+ Make sure that you track changes to your firewall request document.

The following table shows the use cases for this pattern.


| 
| 
| Does your organization have an existing firewall approval process? | Does your organization have an existing firewall request form?  | Suggested action | 
| --- |--- |--- |
| Yes | Yes | Collaborate with AWS consultants or your migration specialists to implement your organization’s process. | 
| No | Yes | Use this pattern’s firewall approval process. Use either an AWS consultant or a migration specialist from your organization to submit the firewall request blanket approval form. | 
| No | No | Use this pattern’s firewall approval process. Use either an AWS consultant or a migration specialist from your organization to submit the firewall request blanket approval form. | 

## Architecture
<a name="create-an-approval-process-for-firewall-requests-during-a-rehost-migration-to-aws-architecture"></a>

The following diagram shows the steps for the firewall request approval process.

![Process for firewall request approval from an InfoSec team during a rehost migration to AWS Cloud.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/cf9b58ad-ab6f-43d3-92da-968529c8d042/images/c672f7ce-6e9f-4dbc-bf2c-4272a6c4432b.png)


## Tools
<a name="create-an-approval-process-for-firewall-requests-during-a-rehost-migration-to-aws-tools"></a>

You can use scanner tools such as [Palo Alto Networks](https://www.paloaltonetworks.com/) or [SolarWinds](https://www.solarwinds.com/) to analyze and validate firewalls and IP addresses.

## Epics
<a name="create-an-approval-process-for-firewall-requests-during-a-rehost-migration-to-aws-epics"></a>

### Analyze the firewall request
<a name="analyze-the-firewall-request"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Analyze the ports and IP addresses. | The firewall request submitter completes an initial analysis to understand the required firewall ports and IP addresses. After this is complete, they request that your InfoSec team open the required ports and map the IP addresses. | AWS Cloud engineer, migration specialist | 

### Validate the firewall request
<a name="validate-the-firewall-request"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Validate the firewall information. | The AWS Cloud engineer schedules a meeting with your InfoSec team. During this meeting, the engineer examines and validates the firewall request information.<br />Typically, the firewall request submitter is the same person as the firewall requester. This validation phase can become iterative based on the feedback given by the approver if anything is observed or recommended. | AWS Cloud engineer, migration specialist | 
| Update the firewall request document. | After the InfoSec team shares their feedback, the firewall request document is edited, saved, and re-uploaded. This document is updated after each iteration.<br />We recommend that you store this document in a version-controlled storage folder. This means that all changes are tracked and correctly applied. | AWS Cloud engineer, migration specialist | 

### Submit the firewall request
<a name="submit-the-firewall-request"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Submit the firewall request. | After the firewall request approver has approved the firewall blanket approval request, the AWS Cloud engineer submits the firewall request. The request specifies the ports that must be open and IP addresses that are required to map and update the AWS account.<br />You can make suggestions or provide feedback after the firewall request is submitted. We recommend that you automate this feedback process and send any edits through a defined workflow mechanism.  | AWS Cloud engineer, migration specialist | 

## Attachments
<a name="attachments-cf9b58ad-ab6f-43d3-92da-968529c8d042"></a>

To access additional content that is associated with this document, download and unzip the following file: [attachment.zip](samples/p-attach/cf9b58ad-ab6f-43d3-92da-968529c8d042/attachments/attachment.zip)
