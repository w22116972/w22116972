

# Access container applications privately on Amazon EKS using AWS PrivateLink and a Network Load Balancer
<a name="access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer"></a>

*Kirankumar Chandrashekar, Amazon Web Services*

## Abstract

Use AWS PrivateLink with a Network Load Balancer when consumers need private, cross-VPC or cross-account access to an EKS-hosted service. Compare it with internal load balancers, VPC peering, Transit Gateway, and Gateway API before choosing it.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer-summary"></a>

This pattern describes how to privately host a Docker container application on Amazon Elastic Kubernetes Service (Amazon EKS) behind a Network Load Balancer, and access the application by using AWS PrivateLink. You can then use a private network to securely access services on the Amazon Web Services (AWS) Cloud. 

The Amazon EKS cluster running the Docker applications, with a Network Load Balancer at the front end, can be associated with a virtual private cloud (VPC) endpoint for access through AWS PrivateLink. This VPC endpoint service can then be shared with other VPCs by using their VPC endpoints.

The setup described by this pattern is a secure way to share application access among VPCs and AWS accounts. It requires no special connectivity or routing configurations, because the connection between the consumer and provider accounts is on the global AWS backbone and doesn’t traverse the public internet.

## Prerequisites and limitations
<a name="access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer-prereqs"></a>

**Prerequisites **
+ [Docker](https://www.docker.com/), installed and configured on Linux, macOS, or Windows.
+ An application running on Docker.
+ An active AWS account.
+ [AWS Command Line Interface (AWS CLI) version 2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html), installed and configured on Linux, macOS, or Windows.
+ An existing Amazon EKS cluster with tagged private subnets and configured to host applications. For more information, see [Subnet tagging](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html#vpc-subnet-tagging) in the Amazon EKS documentation. 
+ Kubectl, installed and configured to access resources on your Amazon EKS cluster. For more information, see [Installing kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) in the Amazon EKS documentation. 

## Architecture
<a name="access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer-architecture"></a>

![Use PrivateLink and a Network Load Balancer to access an application in an Amazon EKS container.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/ce977924-012c-4fb6-8e51-94d6e5c829a6/images/378456a3-f4d1-4a57-bb36-879c240cabfb.png)


**Technology stack  **
+ Amazon EKS
+ AWS PrivateLink
+ Network Load Balancer

**Automation and scale**
+ Kubernetes manifests can be tracked in a Git-based repository and reconciled through a GitOps-compatible CI/CD workflow.
+ You can use AWS CloudFormation to create this pattern by using infrastructure as code (IaC).

## Tools
<a name="access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer-tools"></a>
+ [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) – AWS Command Line Interface (AWS CLI) is an open-source tool that enables you to interact with AWS services using commands in your command-line shell.
+ [Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html) – Elastic Load Balancing distributes incoming application or network traffic across multiple targets, such as Amazon Elastic Compute Cloud (Amazon EC2) instances, containers, and IP addresses, in one or more Availability Zones.
+ [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html) – Amazon Elastic Kubernetes Service (Amazon EKS) is a managed service that you can use to run Kubernetes on AWS without needing to install, operate, and maintain your own Kubernetes control plane or nodes.
+ [Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) – Amazon Virtual Private Cloud (Amazon VPC) helps you launch AWS resources into a virtual network that you've defined.
+ [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) – Kubectl is a command line utility for running commands against Kubernetes clusters.

## Epics
<a name="access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer-epics"></a>

### Deploy the Kubernetes deployment and service manifest files
<a name="deploy-the-kubernetes-deployment-and-service-manifest-files"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
|  Create the Kubernetes deployment manifest file. | Create a deployment manifest file by modifying the following sample file according to your requirements.<pre>apiVersion: apps/v1<br />kind: Deployment<br />metadata:<br />  name: sample-app<br />spec:<br />  replicas: 3<br />  selector:<br />    matchLabels:<br />      app: nginx<br />  template:<br />    metadata:<br />      labels:<br />        app: nginx<br />    spec:<br />      containers:<br />        - name: nginx<br />          image: public.ecr.aws/z9d2n7e1/nginx:1.19.5<br />          ports:<br />            - name: http<br />              containerPort: 80</pre>This is a NGINX sample configuration file that is deployed by using the NGINX Docker image. For more information, see [How to use the official NGINX Docker image](https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/) in the Docker documentation. | DevOps engineer | 
| Deploy the Kubernetes deployment manifest file. | Run the following command to apply the deployment manifest file to your Amazon EKS cluster:<br />`kubectl apply –f <your_deployment_file_name> ` | DevOps engineer | 
|  Create the Kubernetes service manifest file.  | Create a service manifest file by modifying the following sample file according to your requirements.<pre>apiVersion: v1<br />kind: Service<br />metadata:<br />  name: sample-service<br />  annotations:<br />    service.beta.kubernetes.io/aws-load-balancer-type: nlb<br />    service.beta.kubernetes.io/aws-load-balancer-internal: "true"<br />spec:<br />  ports:<br />    - port: 80<br />      targetPort: 80<br />      protocol: TCP<br />  type: LoadBalancer<br />  selector:<br />    app: nginx</pre>Make sure that you included the following `annotations` to define an internal Network Load Balancer:<pre>service.beta.kubernetes.io/aws-load-balancer-type: nlb<br />service.beta.kubernetes.io/aws-load-balancer-internal: "true"</pre> | DevOps engineer | 
| Deploy the Kubernetes service manifest file. | Run the following command to apply the service manifest file to your Amazon EKS cluster:<br />`kubectl apply -f <your_service_file_name>` | DevOps engineer | 

### Create the endpoints
<a name="create-the-endpoints"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Record the Network Load Balancer’s name.  | Run the following command to retrieve the name of the Network Load Balancer:<br />`kubectl get svc sample-service -o wide`<br />Record the Network Load Balancer’s name, which is required to create an AWS PrivateLink endpoint. | DevOps engineer | 
| Create an AWS PrivateLink endpoint. | Sign in to the AWS Management Console, open the Amazon VPC console, and then create an AWS PrivateLink endpoint. Associate this endpoint with the Network Load Balancer, this makes the application privately available to customers. For more information, see [VPC endpoint services (AWS PrivateLink)](https://docs.aws.amazon.com/vpc/latest/userguide/endpoint-service.html) in the Amazon VPC documentation.If the consumer account requires access to the application, the consumer account’s [AWS account ID](https://docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html) must be added to the allowed principals list for the AWS PrivateLink endpoint configuration. For more information, see [Adding and removing permissions for your endpoint service](https://docs.aws.amazon.com/vpc/latest/userguide/add-endpoint-service-permissions.html) in the Amazon VPC documentation. | Cloud administrator  | 
| Create a VPC endpoint. | On the Amazon VPC console, choose **Endpoint Services**, and then choose **Create Endpoint Service**. Create a VPC endpoint for the AWS PrivateLink endpoint.<br />The VPC endpoint’s fully qualified domain name (FQDN) points to the FQDN for the AWS PrivateLink endpoint. This creates an elastic network interface to the VPC endpoint service that the DNS endpoints can access.  | Cloud administrator | 

## Related resources
<a name="access-container-applications-privately-on-amazon-eks-using-aws-privatelink-and-a-network-load-balancer-resources"></a>
+ [Using the official NGINX Docker image](https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/)
+ [Network load balancing on Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/load-balancing.html) 
+ [Creating VPC endpoint services (AWS PrivateLink)](https://docs.aws.amazon.com/vpc/latest/userguide/endpoint-service.html) 
+ [Adding and removing permissions for your endpoint service ](https://docs.aws.amazon.com/vpc/latest/userguide/add-endpoint-service-permissions.html)
