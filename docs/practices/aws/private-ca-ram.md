

# Simplify private certificate management by using AWS Private CA and AWS RAM
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram"></a>

*Everett Hinckley and Vivek Goyal, Amazon Web Services*

## Abstract

This pattern centralizes private certificate authority ownership while sharing issuance capability across AWS accounts through AWS RAM. Use it to support governed private PKI with clear account, organization, and revocation boundaries.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-summary"></a>

You can use AWS Private Certificate Authority (AWS Private CA) to issue private certificates for authenticating internal resources and signing computer code. This pattern provides an AWS CloudFormation template for the rapid deployment of a multi-level CA hierarchy and consistent provisioning experience. Optionally, you can use AWS Resource Access Manager (AWS RAM) to securely share the CA within your organizations or organizational units (OUs) in AWS Organizations, and centralize the CA while using AWS RAM to manage permissions. There is no need for a private CA in every account, so this approach saves you money. Additionally, you can use Amazon Simple Storage Service (Amazon S3) to store the certificate revocation list (CRL) and access logs.

This implementation provides the following features and benefits:
+ Centralizes and simplifies the management of the private CA hierarchy by using AWS Private CA.
+ Exports certificates and keys to customer-managed devices on AWS and on premises.
+ Uses an AWS CloudFormation template for a rapid deployment and consistent provisioning experience.
+ Creates a private root CA along with 1, 2, 3, or 4 subordinate CA hierarchy.
+ Optionally, uses AWS RAM to share the end-entity subordinate CA with other accounts at the organization or OU level.
+ Saves money by removing the need for a private CA in every account by using AWS RAM.
+ Creates an optional S3 bucket for the CRL.
+ Creates an optional S3 bucket for CRL access logs.

## Prerequisites and limitations
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-prereqs"></a>

**Prerequisites **

If you want to share the CA within an AWS Organizations structure, identify or set up the following:
+ A security account for creating the CA hierarchy and share.
+ A separate OU or account for testing.
+ Sharing enabled within the AWS Organizations management account. For more information, see [Enable resource sharing within AWS Organizations](https://docs.aws.amazon.com/ram/latest/userguide/getting-started-sharing.html#getting-started-sharing-orgs) in the AWS RAM documentation.

**Limitations **
+ CAs are regional resources. All CAs reside in a single AWS account and in a single AWS Region.
+ User-generated certificates and keys are not supported. For this use case, we recommend that you customize this solution to use an external root CA. 
+ A public CRL bucket is not supported. We recommend that you keep the CRL private. If internet access to the CRL is required, see the section on using Amazon CloudFront to serve CRLs in [Enabling the S3 Block Public Access (BPA) feature](https://docs.aws.amazon.com/privateca/latest/userguide/crl-planning.html#s3-bpa) in the AWS Private CA documentation.
+ This pattern implements a single-Region approach. If you require a multi-Region certificate authority, you can implement subordinates in a second AWS Region or on premises. That complexity is outside the scope of this pattern, because the implementation depends on your specific use case, workload volume, dependencies, and requirements.

## Architecture
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-architecture"></a>

**Target technology stack  **
+ AWS Private CA
+ AWS RAM
+ Amazon S3
+ AWS Organizations
+ AWS CloudFormation

**Target architecture **

This pattern provides two options for sharing to AWS Organizations:

**Option 1** ─ Create the share at the organization level. All accounts in the organization can issue the private certificates by using the shared CA, as shown in the following diagram.

![Share a CA at the organization level](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/34701b79-c670-4c5d-8c8b-00c7fbd12d06/images/3765d327-3097-4134-a701-28753e1abb14.png)


**Option 2 **─  Create the share at the organizational unit (OU) level. Only the accounts in the specified OU can issue the private certificates by using the shared CA. For example, in the following diagram, if the share is created at the Sandbox OU level, both Developer 1 and Developer 2 can issue private certificates by using the shared CA.

 

![Share a CA at the OU level](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/34701b79-c670-4c5d-8c8b-00c7fbd12d06/images/b8385d18-42d1-4924-aa69-cc4a3e96bf56.png)


## Tools
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-tools"></a>

**AWS services**
+ [AWS Private CA](https://docs.aws.amazon.com/privateca/latest/userguide/PcaWelcome.html) – AWS Private Certificate Authority (AWS Private CA) is a hosted private CA service for issuing and revoking private digital certificates. It helps you create private CA hierarchies, including root and subordinate CAs, without the investment and maintenance costs of operating an on-premises CA.
+ [AWS RAM](https://docs.aws.amazon.com/ram/latest/userguide/what-is.html) – AWS Resource Access Manager (AWS RAM) helps you securely share your resources across AWS accounts and within your organization or OUs in AWS Organizations. To reduce operational overhead in a multi-account environment, you can create a resource and use AWS RAM to share that resource across accounts.
+ [AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html) – AWS Organizations is an account management service that enables you to consolidate multiple AWS accounts into an organization that you create and centrally manage.
+ [Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html) – Amazon Simple Storage Service (Amazon S3) is an object storage service. You can use Amazon S3 to store and retrieve any amount of data at any time, from anywhere on the web. This pattern uses Amazon S3 to store the certificate revocation list (CRL) and access logs.
+ [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) – AWS CloudFormation helps you model and set up your AWS resources, provision them quickly and consistently, and manage them throughout their lifecycle. You can use a template to describe your resources and their dependencies, and launch and configure them together as a stack, instead of managing resources individually. This pattern uses AWS CloudFormation to automatically deploy a multi-level CA hierarchy.

** Code**

The source code for this pattern is available on GitHub, in the [AWS Private CA hierarchy](https://github.com/aws-samples/acmpca-hierarchy) repository. The repository includes:
+ The AWS CloudFormation template `ACMPCA-RootCASubCA.yaml`. You can use this template to deploy the CA hierarchy for this implementation. 
+ Test files for use cases such as requesting, exporting, describing, and deleting a certificate.

To use these files, follow the instructions in the *Epics* section.

## Epics
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-epics"></a>

### Architect the CA hierarchy
<a name="architect-the-ca-hierarchy"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Collect certificate subject information. | Gather certificate subject information about the certificate owner: organization name, organization unit, country, state, locality, and common name. | Cloud architect, Security architect, PKI engineer | 
| Collect optional information about AWS Organizations. | If the CA will be part of an AWS Organizations structure and you want to share the CA hierarchy inside that structure, collect the management account number, the organization ID, and optionally the OU ID (if you want to share the CA hierarchy only with a specific OU). Also, determine the AWS Organizations accounts or OUs, if any, that you want to share the CA with. | Cloud architect, Security architect, PKI engineer | 
| Design the CA hierarchy. | Determine which account will house the root and subordinate CAs. Determine how many subordinate levels the hierarchy requires between the root and the end-entity certificates. For more information, see [Designing a CA hierarchy](https://docs.aws.amazon.com/privateca/latest/userguide/ca-hierarchy.html) in the AWS Private CA documentation. | Cloud architect, Security architect, PKI engineer | 
| Determine naming and tagging conventions for the CA hierarchy. | Determine the names for the AWS resources: the root CA and each subordinate CA. Determine which tags should be assigned to each CA. | Cloud architect, Security architect, PKI engineer | 
| Determine required encryption and signing algorithms. | Determine the following:+ Your organization's encryption algorithm requirements for the public keys that your CA uses when it issues a certificate. The default is RSA\_2048. <br />+ The key algorithm that your CA uses for certificate signing. The default is SHA256WITHRSA. | Cloud architect, Security architect, PKI engineer | 
| Determine certificate revocation requirements for the CA hierarchy. | If certificate revocation capabilities are required, establish a naming convention for the S3 bucket that contains the certificate revocation list (CRL). | Cloud architect, Security architect, PKI engineer | 
| Determine the logging requirements for the CA hierarchy. | If access logging capabilities are required, establish a naming convention for the S3 bucket that contains the access logs. | Cloud architect, Security architect, PKI engineer | 
| Determine certificate expiration periods. | Determine the expiration date for the root certificate (the default is 10 years), end-entity certificates (the default is 13 months), and subordinate CA certificates (the default is 3 years). Subordinate CA certificates should expire earlier than the CA certificates at higher levels in the hierarchy. For more information, see [Managing the private CA lifecycle](https://docs.aws.amazon.com/privateca/latest/userguide/ca-lifecycle.html) in the AWS Private CA documentation. | Cloud architect, Security architect, PKI engineer | 

### Deploy the CA hierarchy
<a name="deploy-the-ca-hierarchy"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Complete prerequisites. | Complete the steps in the [Prerequisites](#simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-prereqs) section of this pattern. | Cloud administrator, Security engineers, PKI engineers | 
| Create CA roles for various personas. | 1. Determine the types of AWS Identity and Access Management (IAM) roles or users in AWS IAM Identity Center (successor to AWS Single Sign-On) required to administer the various levels of the CA hierarchy, such as RootCAAdmin, SubordinateCAAdmin, and CertificateConsumer. <br />2. Determine the granularity of policies needed to separate duties. <br />3. Create the required IAM roles or users in IAM Identity Center in the account that the CA hierarchy resides in. | Cloud administrator, Security engineers, PKI engineers | 
| Deploy the CloudFormation stack. | 1. From the [GitHub repository](https://github.com/aws-samples/acmpca-hierarchy) for this pattern, download the AWSPCA-RootCASubCA.yaml template. <br />2. Deploy the template from the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/) or from the AWS Command Line Interface (AWS CLI). For more information, see [Working with stacks](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacks.html) in the CloudFormation documentation.<br />3. Complete the parameters in the template, including the organization name, the OU name, the key algorithm, the signing algorithm, and other options. | Cloud administrator, Security engineers, PKI engineers | 
| Architect a solution for updating certificates used by user-managed resources. | Resources of integrated AWS services, such as Elastic Load Balancing, update certificates automatically before expiration. However, user-managed resources, such as web servers that are running on Amazon Elastic Compute Cloud (Amazon EC2) instances, require another mechanism. 1. Determine which user-managed resources require end-entity certificates from the private CA. <br />2. Plan a process to be notified about the expiration of user-managed resources and certificates. For examples, see the following:[Using an AWS Config managed rule](https://docs.aws.amazon.com/config/latest/developerguide/acm-certificate-expiration-check.html)[Using Amazon CloudWatch and Amazon EventBridge](https://aws.amazon.com/about-aws/whats-new/2021/03/aws-certificate-manager-provides-certificate-expiry-monitoring-through-amazon-cloudwatch/) <br />3. Write custom scripts to update certificates on user-managed resources and integrate them with AWS services to automate the updates. For more information about integrated AWS Services, see [Services integrated with AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-services.html) in the ACM documentation. | Cloud administrator, Security engineers, PKI engineers | 

### Validate and document the CA hierarchy
<a name="validate-and-document-the-ca-hierarchy"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Validate optional AWS RAM sharing. | If the CA hierarchy is shared with other accounts in AWS Organizations, log in to one of those accounts from the AWS Management Console, navigate to the [AWS Private CA console](https://console.aws.amazon.com/acm-pca/home), and confirm that the newly created CA is shared to this account. Only the lowest-level CA in the hierarchy will be visible, because that is the CA that generates the end-entity certificates. Repeat for a sampling of the accounts that the CA is shared with. | Cloud administrator, Security engineers, PKI engineers | 
| Validate the CA hierarchy with certificate lifecycle tests. | In the [GitHub repository](https://github.com/aws-samples/acmpca-hierarchy) for this pattern, locate the lifecycle tests. Run the tests from the AWS CLI to request a certificate, export a certificate, describe a certificate, and delete a certificate. | Cloud administrator, Security engineers, PKI engineers | 
| Import the certificate chain into trust stores. | For browsers and other applications to trust a certificate, the certificate’s issuer must be included in the browser’s trust store, which is a list of trusted CAs. Add the certificate chain for the new CA hierarchy to your browser's and application’s trust store. Confirm that the end-entity certificates are trusted. | Cloud administrator, Security engineers, PKI engineers | 
| Create a runbook to document the CA hierarchy. | Create a runbook document to describe the architecture of the CA hierarchy, the account structure that can request end-entity certificates, the build process, and basic management tasks such as issuing end-entity certificates (unless you want to allow self-service by child accounts), usage, and tracking. | Cloud administrator, Security engineers, PKI engineers | 

## Related resources
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-resources"></a>
+ [Designing a CA hierarchy](https://docs.aws.amazon.com/privateca/latest/userguide/ca-hierarchy.html) (AWS Private CA documentation)
+ [Creating a private CA](https://docs.aws.amazon.com/privateca/latest/userguide/create-CA.html) (AWS Private CA documentation)
+ [How to use AWS RAM to share your AWS Private CA cross-account](https://aws.amazon.com/blogs/security/how-to-use-aws-ram-to-share-your-acm-private-ca-cross-account/) (AWS blog post)
+ [AWS Private CA best practices](https://docs.aws.amazon.com/acm-pca/latest/userguide/ca-best-practices.html) (AWS blog post)
+ [Enable resource sharing within AWS Organizations](https://docs.aws.amazon.com/ram/latest/userguide/getting-started-sharing.html#getting-started-sharing-orgs) (AWS RAM documentation)
+ [Managing the private CA lifecycle](https://docs.aws.amazon.com/privateca/latest/userguide/ca-lifecycle.html) (AWS Private CA documentation)
+ [acm-certificate-expiration-check for AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/acm-certificate-expiration-check.html) (AWS Config documentation)
+ [AWS Certificate Manager now provides certificate expiry monitoring through Amazon CloudWatch](https://aws.amazon.com/about-aws/whats-new/2021/03/aws-certificate-manager-provides-certificate-expiry-monitoring-through-amazon-cloudwatch/) (AWS announcement)
+ [Services integrated with AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-services.html) (ACM documentation)

## Additional information
<a name="simplify-private-certificate-management-by-using-aws-private-ca-and-aws-ram-additional"></a>

When you export certificates, use a passphrase that is cryptographically strong and aligns with your organization’s data loss prevention strategy.
