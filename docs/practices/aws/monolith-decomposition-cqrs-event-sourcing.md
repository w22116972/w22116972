

# Decompose monoliths into microservices by using CQRS and event sourcing
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing"></a>

*Rodolfo Jr. Cerrada, Dmitry Gulin, and Tabby Ward, Amazon Web Services*

## Abstract

CQRS and event sourcing can support incremental decomposition when read and write workloads have different scaling, consistency, or audit requirements. Apply them selectively: they add operational and data-consistency complexity and are not a default microservices pattern.

## Table of Contents

- [Summary](#summary)
- [Prerequisites and limitations](#prerequisites-and-limitations)
- [Architecture](#architecture)
- [Implementation](#epics)
- [Related resources](#related-resources)

## Summary
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-summary"></a>

This pattern combines two patterns, using both the command query responsibility separation (CQRS) pattern and the event sourcing pattern. The CQRS pattern separates responsibilities of the command and query models. The event sourcing pattern takes advantage of asynchronous event-driven communication to improve the overall user experience.

You can use CQRS and Amazon Web Services (AWS) services to maintain and scale each data model independently while refactoring your monolith application into microservices architecture. Then you can use the event sourcing pattern to synchronize data from the command database to the query database.

This pattern uses example code that includes a solution (\*.sln) file that you can open using the latest version of Visual Studio. The example contains Reward API code to showcase how CQRS and event sourcing work in AWS serverless and traditional or on-premises applications.

To learn more about CQRS and event sourcing, see the [Additional information](#decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-additional) section.

## Prerequisites and limitations
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-prereqs"></a>

**Prerequisites **
+ An active AWS account
+ Amazon CloudWatch
+ Amazon DynamoDB tables
+ Amazon DynamoDB Streams
+ AWS Identity and Access Management (IAM) access key and secret key; for more information, see the video in the *Related resources* section
+ AWS Lambda
+ Familiarity with Visual Studio
+ Familiarity with AWS Toolkit for Visual Studio; for more information, see the *AWS Toolkit for Visual Studio demo* video in the *Related resources* section

**Product versions**
+ [Visual Studio 2019 Community Edition](https://visualstudio.microsoft.com/downloads/).
+ [AWS Toolkit for Visual Studio 2019](https://aws.amazon.com/visualstudio/).
+ .NET Core 3.1. This component is an option in the Visual Studio installation. To include .NET Core during installation, select **NET Core cross-platform development**.

**Limitations**
+ The example code for a traditional on-premises application (ASP.NET Core Web API and data access objects) does not come with a database. However, it comes with the `CustomerData` in-memory object, which acts as a mock database. The code provided is enough for you to test the pattern.

## Architecture
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-architecture"></a>

**Source technology stack**
+ ASP.NET Core Web API project
+ IIS Web Server
+ Data access object
+ CRUD model

**Source architecture**

In the source architecture, the CRUD model contains both command and query interfaces in one application. For example code, see `CustomerDAO.cs` (attached).

![Connections between application, service interface, customer CRUD model, and database.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/1cd3a84c-12c7-4306-99aa-23f2c53d3cd3.png)


**Target technology stack **
+ Amazon DynamoDB
+ Amazon DynamoDB Streams
+ AWS Lambda
+ (Optional) Amazon API Gateway
+ (Optional) Amazon Simple Notification Service (Amazon SNS)

**Target architecture **

In the target architecture, the command and query interfaces are separated. The architecture shown in the following diagram can be extended with API Gateway and Amazon SNS. For more information, see the [Additional information](#decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-additional) section.

![Application connecting with serverless Customer Command and Customer Query microservices.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/1c665697-e3ac-4ef4-98d0-86c2cbf164c1.png)


1. Command Lambda functions perform write operations, such as create, update, or delete, on the database.

1. Query Lambda functions perform read operations, such as get or select, on the database.

1. This Lambda function processes the DynamoDB streams from the Command database and updates the Query database for the changes.

## Tools
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-tools"></a>

**Tools**
+ [Amazon DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html) – Amazon DynamoDB is a fully managed NoSQL database service that provides fast and predictable performance with seamless scalability.
+ [Amazon DynamoDB Streams](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html) – DynamoDB Streams captures a time-ordered sequence of item-level modifications in any DynamoDB table. It then stores this information in a log for up to 24 hours. Encryption at rest encrypts the data in DynamoDB streams.
+ [AWS Lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) – AWS Lambda is a compute service that supports running code without provisioning or managing servers. Lambda runs your code only when needed and scales automatically, from a few requests per day to thousands per second. You pay only for the compute time that you consume—there is no charge when your code is not running.
+ [AWS Management Console](https://docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/learn-whats-new.html) – The AWS Management Console is a web application that comprises a broad collection of service consoles for managing AWS services.
+ [Visual Studio 2019 Community Edition](https://visualstudio.microsoft.com/downloads/) – Visual Studio 2019 is an integrated development environment (IDE). The Community Edition is free for open-source contributors. In this pattern, you will use Visual Studio 2019 Community Edition to open, compile, and run example code. For viewing only, you can use any text editor or [Visual Studio Code](https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/welcome.html).
+ [AWS Toolkit for Visual Studio](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/welcome.html) – The AWS Toolkit for Visual Studio is a plugin for the Visual Studio IDE. The AWS Toolkit for Visual Studio makes it easier for you to develop, debug, and deploy .NET applications that use AWS services.

**Code **

The example code is attached. For instructions on deploying the example code, see the *Epics* section.

## Epics
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-epics"></a>

### Open and build the solution
<a name="open-and-build-the-solution"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Open the solution. | 1. Download the example source code (`CQRS-ES Code.zip`) from the *Attachments* section, and extract the files.<br />2. In the Visual Studio IDE, choose **File**, **Open**, **Project Solution**, and navigate to the folder where you extracted the source code.<br />3. Choose **AWS.APG.CQRSES.sln**, and then choose **Open**. The entire solution is loaded into Visual Studio. | App developer | 
| Build the solution. | Open the context (right-click) menu for the solution, and then choose **Build Solution**. This will build and compile all the projects in the solution. It should compile successfully.<br />Visual Studio Solution Explorer should show the directory structure.+ `CQRS On-Premises Code Sample` contains an example of using CQRS on premises.<br />+ `CQRS AWS Serverless` contains all the CQRS and event-sourcing example code using AWS serverless services. | App developer | 

### Build the DynamoDB tables
<a name="build-the-dynamodb-tables"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Provide credentials. | If you don't have an access key yet, see the video in the *Related resources* section.1. In Solution Explorer, expand **CQRS AWS Serverless**, and then expand the **Build **solution folder.<br />2. Expand the **AwS.APG.CQRSES.Build** project and view the `Program.cs` file.<br />3. Scroll to the top of `Program.cs` and look for `Program()`.<br />4. Replace `YOUR ACCESS KEY` with your account access key, and replace `YOUR SECRET KEY`with your account secret key. Note that in a production environment, you would not hardcode your keys. Instead, you could use [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/) to store and retrieve the credentials. | App developer, Data engineer, DBA | 
| Build the project. | To build the project, open the context (right-click) menu for the **AwS.APG.CQRSES.Build** project, and then choose **Build**. | App developer, Data engineer, DBA | 
| Build and populate the tables. | To build the tables and populate them with seed data, open the context (right-click) menu for the **AwS.APG.CQRSES.Build** project, and then choose **Debug**,** Start New Instance**. | App developer, Data engineer, DBA | 
| Verify the table construction and the data. | To verify, navigate to **AWS Explorer**, and expand **Amazon DynamoDB**. It should display the tables. Open each table to display the example data. | App developer, Data engineer, DBA | 

### Run local tests
<a name="run-local-tests"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Build the CQRS project. | 1. Open the solution, and navigate to the **CQRS AWS Services/CQRS/Tests** solution folder.<br />2. In the **AWS.APG.CQRSES.CQRSLambda.Tests** project, open **BaseFunctionTest.cs**, and replace *AccessKey *and *SecretKey* with the IAM keys that you created.<br />3. Save the changes.<br />4. To compile and build the test project, open the context (right-click) menu for the project, and then choose **Build**. | App developer, Test engineer | 
| Build the event-sourcing project. | 1. Navigate to the **CQRS AWS Services/Event Source/Tests** solution folder. <br />2. In the **AWS.APG.CQRSES.EventSourceLambda.Tests** project, open **BaseFunctionTest.cs**, and replace *AccessKey *and *SecretKey* with the IAM keys that you created. <br />3. Save the changes.<br />4. To compile and build the test project, open the context (right-click) menu for the project, and then choose **Build**. | App developer, Test engineer | 
| Run the tests. | To run all tests, choose **View**, **Test Explorer**, and then choose **Run All Tests In View**. All tests should pass, which is indicated by a green check mark icon.  | App developer, Test engineer | 

### Publish the CQRS Lambda functions to AWS
<a name="publish-the-cqrs-lambda-functions-to-aws"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Publish the first Lambda function. | 1. In Solution Explorer, open the context (right-click) menu for the AWS.APG.CQRSES.CommandCreateLambda project, and then choose **Publish to AWS Lambda**.<br />2. Select the profile that you want to use and the AWS Region where you want to deploy the Lambda function, and the function name.<br />3. For the remaining fields, keep the default values, and choose **Next**.<br />4. In **Role Name **dropdown list, select **AWSLambdaFullAccess**.<br />5. To provide your account keys, choose **Add**, and enter `AcessKey` as the variable and your access key as the value. Then choose **Add** again, enter `SecretKey` as the variable and your secret key as the value.<br />6. For the remaining fields, keep the default values, and choose **Upload**. After the Lambda test function uploads, it appears in Visual Studio automatically.<br />7. Repeat steps 1-6 for the following projects:AWS.APG.CQRSES.CommandDeleteLambdaAWS.APG.CQRSES.CommandUpdateLambdaAWS.APG.CQRSES.CommandAddRewardLambdaAWS.APG.CQRSES.CommandRedeemRewardLambdaAWS.APG.CQRSES.QueryCustomerListLambdaAWS.APG.CQRSES.QueryRewqardLambda | App developer, DevOps engineer | 
| Verify the function upload. | (Optional) You can verify that the function was successfully loaded by navigating to AWS Explorer and expanding **AWS Lambda**. To open the test window, choose the Lambda function (double-click). | App developer, DevOps engineer | 
| Test the Lambda function. | 1. Enter the request data, or copy an example request data from **Test data** in the [Additional information](#decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-additional) section. Make sure that you select data that is for the function you are testing.<br />2. To run the test, choose **Invoke**. The response and any errors are displayed in the **Response** text box, and logs are shown in the **Logs** text box or in CloudWatch Logs.<br />3. To verify the data, in AWS Explorer, choose the DynamoDB table (double-click).All CQRS Lambda projects are found under the `CQRS AWS Serverless\CQRS\Command Microservice` and` CQRS AWS Serverless\CQRS\Command Microservice` solution folders. For the solution directory and projects, see **Source code directory** in the [Additional information](#decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-additional) section. | App developer, DevOps engineer | 
| Publish the remaining functions. | Repeat the previous steps for the following projects:+ AWS.APG.CQRSES.CommandDeleteLambda<br />+ AWS.APG.CQRSES.CommandUpdateLambda<br />+ AWS.APG.CQRSES.CommandAddRewardLambda<br />+ AWS.APG.CQRSES.CommandRedeemRewardLambda<br />+ AWS.APG.CQRSES.QueryCustomerListLambda<br />+ AWS.APG.CQRSES.QueryRewqardLambda | App developer, DevOps engineer | 

### Set up the Lambda function as an event listener
<a name="set-up-the-lambda-function-as-an-event-listener"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Publish the Customer and Reward Lambda event handlers. | To publish each event handler, follow the steps in the preceding epic.<br />The projects are under the `CQRS AWS Serverless\Event Source\Customer Event` and `CQRS AWS Serverless\Event Source\Reward Event` solution folders. For more information, see *Source code directory* in the [Additional information](#decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-additional) section. | App developer | 
| Attach the event-sourcing Lambda event listener. | 1. Log in to the AWS Management Console using the same account you use when you publish the Lambda projects.<br />2. For the **Region**, select **US East 1** or the Region where you deployed the Lambda functions in the previous epic.<br />3. Navigate to the **Lambda** service.<br />4. Select the `EventSourceCustomer` Lambda function.<br />5. Choose **Add Trigger**.<br />6. In the **Trigger configuration** dropdown list, select **DynamoDB**.<br />7. In the **DynamoDB table** dropdown list, select **cqrses-customer-cmd**.<br />8. In the **Starting position** dropdown list, select *Trim horizon* from . Trim horizon means that the DynamoDB trigger will start reading at the last (untrimmed) stream record, which is the oldest record in the shard.<br />9. Select the **Enable trigger** check box.<br />10. For the remaining fields, keep the default values, and choose **Add**.After the listener is successfully attached to the DynamoDB table, it will be displayed on the Lambda designer page. | App developer | 
| Publish and attach the EventSourceReward Lambda function. | To publish and attach the `EventSourceReward` Lambda function, repeat the steps in the previous two stories, selecting **cqrses-reward-cmd** from the **DynamoDB table** dropdown list. | App developer | 

### Test and validate the DynamoDB streams and Lambda trigger
<a name="test-and-validate-the-dynamodb-streams-and-lambda-trigger"></a>


| Task | Description | Skills required | 
| --- | --- | --- | 
| Test the stream and the Lambda trigger. | 1. In Visual Studio, navigate to AWS Explorer.<br />2. Expand AWS Lambda, and choose the **CommandRedeemReward** function (double-click). In the function window that opens, you can test the function.<br />3. In the **Request** text box, enter the request data in JavaScript Object Notation (JSON) format. For an example request, see **Test data** in the [Additional information](#decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-additional) section.<br />4. Choose **Invoke**. | App developer | 
| Validate, using the DynamodDB reward query table. | 1. Open the **cqrses-reward-query** table.<br />2. Check the points of the customer that redeemed the reward. The redeemed points should be subtracted from the customer's total aggregated points. | App developer | 
| Validate, using CloudWatch Logs. | 1. Navigate to CloudWatch and choose **Log groups**.<br />2. The **/aws/lambda/EventSourceReward** log group contains the logs for the `EventSourceReward` trigger. All Lambda calls are logged, including the messages you placed in `context.Logger.LogLine` and `Console.Writeline` in the Lambda code. | App developer | 
| Validate the EventSourceCustomer trigger. | To validate the `EventSourceCustomer` trigger, repeat the steps in this epic, using the `EventSourceCustomer` trigger's respective customer table and CloudWatch logs. | App developer | 

## Related resources
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-resources"></a>

**References **
+ [Visual Studio 2019 Community Edition downloads](https://visualstudio.microsoft.com/downloads/)
+ [AWS Toolkit for Visual Studio download](https://aws.amazon.com/visualstudio/)
+ [AWS Toolkit for Visual Studio User Guide](https://docs.aws.amazon.com/toolkit-for-visual-studio/latest/user-guide/welcome.html)
+ [Serverless on AWS](https://aws.amazon.com/serverless/)
+ [DynamoDB Use Cases and Design Patterns](https://aws.amazon.com/blogs/database/dynamodb-streams-use-cases-and-design-patterns/)
+ [Martin Fowler CQRS](https://martinfowler.com/bliki/CQRS.html)
+ [Martin Fowler Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html)

**Videos**
+ [AWS Toolkit for Visual Studio demo](https://www.youtube.com/watch?v=B190tcu1ERk)
+ [How do I create an access key ID for a new IAM user?](https://www.youtube.com/watch?v=665RYobRJDY)

## Additional information
<a name="decompose-monoliths-into-microservices-by-using-cqrs-and-event-sourcing-additional"></a>

**CQRS and event sourcing**

*CQRS*

The CQRS pattern separates a single conceptual operations model, such as a data access object single CRUD (create, read, update, delete) model, into command and query operations models. The command model refers to any operation, such as create, update, or delete, that changes the state. The query model refers to any operation that returns a value.

![Architecture with service interface, CRUD model, and database.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/3f64756d-681e-4f0e-8034-746263d857b2.png)


1. The Customer CRUD model includes the following interfaces:
   + `Create Customer()`
   + `UpdateCustomer()`
   + `DeleteCustomer()`
   + `AddPoints()`
   + `RedeemPoints()`
   + `GetVIPCustomers()`
   + `GetCustomerList()`
   + `GetCustomerPoints()`

As your requirements become more complex, you can move from this single-model approach. CQRS uses a command model and a query model to separate the responsibility for writing and reading data. That way, the data can be independently maintained and managed. With a clear separation of responsibilities, enhancements to each model do not impact the other. This separation improves maintenance and performance, and it reduces the complexity of the application as it grows.

![The application separated into command and query models, sharing a single database.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/12db023c-eb81-4c27-bbb9-b085b13176ae.png)


 

1. Interfaces in the Customer Command model:
   + `Create Customer()`
   + `UpdateCustomer()`
   + `DeleteCustomer()`
   + `AddPoints()`
   + `RedeemPoints()`

1. Interfaces in the Customer Query model:
   + `GetVIPCustomers()`
   + `GetCustomerList()`
   + `GetCustomerPoints()`
   + `GetMonthlyStatement()`

For example code, see *Source code directory*.

The CQRS pattern then decouples the database. This decoupling leads to the total independence of each service, which is the main ingredient of microservice architecture.

![Separate databases for command and query models.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/016dbfa8-3bd8-42ee-afa1-38a98986c7d5.png)


 Using CQRS in the AWS Cloud, you can further optimize each service. For example, you can set different compute settings or choose between a serverless or a container-based microservice. You can replace your on-premises caching with Amazon ElastiCache. If you have an on-premises publish/subscribe messaging, you can replace it with Amazon Simple Notification Service (Amazon SNS). Additionally, you can take advantage of pay-as-you-go pricing and the wide array of AWS services that you pay only for what you use.

CQRS includes the following benefits:
+ Independent scaling – Each model can have its scaling strategy adjusted to meet the requirements and demand of the service. Similar to high-performance applications, separating read and write enables the model to scale independently to address each demand. You can also add or reduce compute resources to address the scalability demand of one model without affecting the other.
+ Independent maintenance – Separation of query and command models improves the maintainability of the models. You can make code changes and enhancements to one model without affecting the other.
+ Security – It's easier to apply the permissions and policies to separate models for read and write.
+ Optimized reads – You can define a schema that is optimized for queries. For example, you can define a schema for the aggregated data and a separate schema for the fact tables.
+ Integration –  CQRS fits well with event-based programming models.
+ Managed complexity – The separation into query and command models is suited to complex domains.

When using CQRS, keep in mind the following caveats:
+ The CQRS pattern applies only to a specific portion of an application and not the whole application. If implemented on a domain that does not fit the pattern, it can reduce productivity, increase risk, and introduce complexity.
+ The pattern works best for frequently used models that have an imbalance read and write operations.
+ For read-heavy applications, such as large reports that take time to process, CQRS gives you the option to select the right database and create a schema to store your aggregated data. This improves the response time of reading and viewing the report by processing the report data only one time and dumping it in the aggregated table.
+ For the write-heavy applications, you can configure the database for write operations and allow the command microservice to scale independently when the demand for write increases. For examples, see the `AWS.APG.CQRSES.CommandRedeemRewardLambda` and `AWS.APG.CQRSES.CommandAddRewardLambda` microservices.

*Event sourcing*

The next step is to use event sourcing to synchronize the query database when a command is run. For example, consider the following events:
+ A customer reward point is added that requires the customer total or aggregated reward points in the query database to be updated.
+ A customer's last name is updated in the command database, which requires the surrogate customer information in the query database to be updated.

In the traditional CRUD model, you ensure consistency of data by locking the data until it finishes a transaction. In event sourcing, the data are synchronized through publishing a series of events that will be consumed by a subscriber to update its respective data.

The event-sourcing pattern ensures and records a full series of actions taken on the data and publishes it through a sequence of events. These events represent a set of changes to the data that subscribers of that event must process to keep their record updated. These events are consumed by the subscriber, synchronizing the data on the subscriber's database. In this case, that's the query database.

The following diagram shows event sourcing used with CQRS on AWS.

![Microservice architecture for the CQRS and event sourcing patterns using AWS serverless services.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/cc9bc84a-60b4-4459-9a5c-2334c69dbb4e.png)


1. Command Lambda functions perform write operations, such as create, update, or delete, on the database.

1. Query Lambda functions perform read operations, such as get or select, on the database.

1. This Lambda function processes the DynamoDB streams from the Command database and updates the Query database for the changes. You can also use this function also to publish a message to Amazon SNS so that its subscribers can process the data.

1. (Optional) The Lambda event subscriber processes the message published by Amazon SNS and updates the Query database.

1. (Optional) Amazon SNS sends email notification of the write operation.

On AWS, the query database can be synchronized by DynamoDB Streams. DynamoDB captures a time-ordered sequence of item-level modifications in a DynamobDB table in near-real time and durably stores the information within 24 hours.

Activating DynamoDB Streams enables the database to publish a sequence of events that makes the event sourcing pattern possible. The event sourcing pattern adds the event subscriber. The event subscriber application consumes the event and processes it depending on the subscriber's responsibility. In the previous diagram, the event subscriber pushes the changes to the Query DynamoDB database to keep the data synchronized. The use of Amazon SNS, the message broker, and the event subscriber application keeps the architecture decoupled.

Event sourcing includes the following benefits:
+ Consistency for transactional data
+ A reliable audit trail and history of the actions, which can be used to monitor actions taken in the data
+ Allows distributed applications such as microservices to synchronize their data across the environment
+ Reliable publication of events whenever the state changes
+ Reconstructing or replaying of past states
+ Loosely coupled entities that exchange events for migration from a monolithic application to microservices
+ Reduction of conflicts caused by concurrent updates; event sourcing avoids the requirement to update objects directly in the data store
+ Flexibility and extensibility from decoupling the task and the event
+ External system updates
+ Management of multiple tasks in a single event

When using event sourcing, keep in mind the following caveats:
+ Because there is some delay in updating data between the source subscriber databases, the only way to undo a change is to add a compensating event to the event store.
+ Implementing event sourcing has a learning curve since its different style of programming.

**Test data**

Use the following test data to test the Lambda function after successful deployment.

**CommandCreate Customer**

```
{  "Id":1501,  "Firstname":"John",  "Lastname":"Done",  "CompanyName":"AnyCompany",  "Address": "USA",  "VIP":true }
```

**CommandUpdate Customer**

```
{  "Id":1501,  "Firstname":"John",  "Lastname":"Doe",  "CompanyName":"Example Corp.",  "Address": "Seattle, USA",  "VIP":true }
```

**CommandDelete Customer**

Enter the customer ID as request data. For example, if the customer ID is 151, enter 151 as request data.

```
151
```

**QueryCustomerList**

This is blank. When it is invoked, it will return all customers.

**CommandAddReward**

This will add 40 points to customer with ID 1 (Richard).

```
{
  "Id":10101,
  "CustomerId":1,
  "Points":40
}
```

**CommandRedeemReward**

This will deduct 15 points to customer with ID 1 (Richard).

```
{
  "Id":10110,
  "CustomerId":1,
  "Points":15
}
```

**QueryReward**

Enter the ID of the customer. For example, enter 1 for Richard, 2 for Arnav, and 3 for Shirley.

```
2 
```

**Source code directory**

Use the following table as a guide to the directory structure of the Visual Studio solution. 

*CQRS On-Premises Code Sample solution directory*

![Solution directory with Command and Query services expanded.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/4811c2c0-643b-410f-bb87-0b86ec5e194c.png)


**Customer CRUD model**

CQRS On-Premises Code Sample\\CRUD Model\\AWS.APG.CQRSES.DAL project

**CQRS version of the Customer CRUD model**
+ Customer command: `CQRS On-Premises Code Sample\CQRS Model\Command Microservice\AWS.APG.CQRSES.Command`project
+ Customer query: `CQRS On-Premises Code Sample\CQRS Model\Query Microservice\AWS.APG.CQRSES.Query` project

**Command and Query microservices**

The Command microservice is under the solution folder `CQRS On-Premises Code Sample\CQRS Model\Command Microservice`:
+ `AWS.APG.CQRSES.CommandMicroservice` ASP.NET Core API project acts as the entry point where consumers interact with the service.
+ `AWS.APG.CQRSES.Command` .NET Core project is an object that hosts command-related objects and interfaces.

The query microservice is under the solution folder `CQRS On-Premises Code Sample\CQRS Model\Query Microservice`:
+ `AWS.APG.CQRSES.QueryMicroservice` ASP.NET Core API project acts as the entry point where consumers interact with the service.
+ `AWS.APG.CQRSES.Query` .NET Core project is an object that hosts query-related objects and interfaces.

*CQRS AWS Serverless code solution directory*

![Solution directory showing both microservices and the event source expanded.](http://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/images/pattern-img/9f1bc700-def4-4201-bb2d-f1fa27404f15/images/23f8655c-95ad-422c-b20a-e29dc145e995.png)


 

This code is the AWS version of the on-premises code using AWS serverless services.

In C\# .NET Core, each Lambda function is represented by one .NET Core project. In this pattern's example code, there is a separate project for each interface in the command and query models.

**CQRS using AWS services**

You can find the root solution directory for CQRS using AWS serverless services is in the `CQRS AWS Serverless\CQRS`folder. The example includes two models: Customer and Reward.

The command Lambda functions for Customer and Reward are under `CQRS\Command Microservice\Customer` and `CQRS\Command Microservice\Reward` folders. They contain the following Lambda projects:
+ Customer command: `CommandCreateLambda`, `CommandDeleteLambda`, and `CommandUpdateLambda`
+ Reward command: `CommandAddRewardLambda` and `CommandRedeemRewardLambda`

The query Lambda functions for Customer and Reward are found under the `CQRS\Query Microservice\Customer` and `CQRS\QueryMicroservice\Reward`folders. They contain the `QueryCustomerListLambda` and `QueryRewardLambda` Lambda projects.

**CQRS test project**

The test project is under the `CQRS\Tests` folder. This project contains a test script to automate testing the CQRS Lambda functions.

**Event sourcing using AWS services**

The following Lambda event handlers are initiated by the Customer and Reward DynamoDB streams to process and synchronize the data in query tables.
+ The `EventSourceCustomer` Lambda function is mapped to the Customer table (`cqrses-customer-cmd`) DynamoDB stream.
+ The `EventSourceReward` Lambda function is mapped to the Reward table (`cqrses-reward-cmd`) DynamoDB stream.

## Attachments
<a name="attachments-9f1bc700-def4-4201-bb2d-f1fa27404f15"></a>

To access additional content that is associated with this document, download and unzip the following file: [attachment.zip](samples/p-attach/9f1bc700-def4-4201-bb2d-f1fa27404f15/attachments/attachment.zip)
