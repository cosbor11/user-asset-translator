# user-asset-translator

----
## Table of Contents
1. [ Purpose ](#purpose)
2. [ Implementation Summary ](#implementation-summary) 
3. [ Reasoning ](#reasoning)
4. [ Technology Resources ](#technology-resources)
5. [ Network Diagrams ](#network-diagrams)
6. [ Sequence Diagrams ](#sequence-diagrams)
7. [ Entities ](#entities)
8. [ Validations ](#validations)
9. [Report Generation Strategy](#report-generation-strategy)
10. [ Deployment ](#deployment)

-------

## Purpose

The purpose of this application is to process input from a client that provides data in XML (either via SFTP or HTTP) and translate it into report format for another client to consume over SFTP.

> *Note: This solution includes the AWS cloud setup as well as the Google Cloud equivalent*

## Implementation Summary

This implementation features AWS or GCP Serverless technologies. Essentially, when an xml message is received, it there are two entry points:
  
  (1) An [aws-transfer-family](https://aws.amazon.com/aws-transfer-family) mananged SFTP Service that is backed by a file storage bucket called `user-asset-inbox`. (or an application based sftp server for the GPC case)
  (2) An API Gateway cloud POST endpoint
  
The XML messages are sent to a Cloud Function (Lambda) called `user-asset-event-processor` either by a cloud watch event listener that listens for new files being added, or in the http handling case, is invoked as an endpoint handler in the API Gateway/Endpoint.

The `user-asset-event-processor` Function's duty is to immediatly split a file or payload into single user-asset messages and map those messages to an internal serialized object based on our json spec, validate them, then move the original files to either the `processed` or  `errors` sub folder (in case we need to replay them). Finally, it will then put them onto a message queue to be held for batch processsing.

A Cloud Scheduled Cron job is configurued to invoke a second cloud function or Lambda called `user-asset-report-generator` that consumes all messages on queue (max 100k), converts them to report format along with calculating the account total summary information, then the report is sent to the a file store `user-asset-outbox` or a sftp inbox location, which will immediatly make it available on the SFTP managed service for clients to consume.

## Reasoning

Rather than creating application servers, reverse proxies, squid proxies, setitng up logging services, a database for storing users, onboarding, password management apis, load balancers, etc.. I decided to leverage Cloud Managed Services to do many of the same processes. Amazon provides a fully managed support for file transfers directly into and out of Amazon S3; GCP does not, but we can easily mimick this behavior. Both AWS and GCP feature rich and secure user management. For this usecase, I did not see the need to create a traditional servlet based application with API models, controllers, services, persistance ORMs, sessions handling, connection pooling and auto scaling distributed design.

Durable Message Queues in the cloud let us collect messages and hold them until the batching interval is ready to generate new reports. 

I also wanted to make the services as cost effective and auto-scalable as possible. By taking adavantge of lambda/cloud functions concurrency, and boost capibilities. Based on the provided peek throughput requirements (6k requests-per-min), the full cost of this setup should be less that $50/mo.

There are two lambda/cloud functions total in this implementation, one to collect messages, normalize them into single record json payloads and publish them to queue, and another to consume messages from queue and generate a report file to then upload to s3 or a SFTP server.

Cloud Events and Schedulers are helpful because the have built in logging and alerting capibilities (no need to install and setup Loggly or monitoring tools like Sensu, PagerDuty, Promethus, or New Relic). Also, at any point, any of technologies can be replaced and easily integrated with the existing lambdas and queue flow. For example if you wanted to replace the queue with a topic, event stream, or invocation source.

## Technology Resources

- **SFTP**:
    - AWS: [aws-transfer-family](https://aws.amazon.com/aws-transfer-family)
    - GCP: [embedded ftpserver](https://mina.apache.org/ftpserver-project/embedding_ftpserver.html). Just need to add code to immediatly transfer to filesstore
- **API Gateway**:
    - AWS: [api-gateway](https://aws.amazon.com/api-gateway)
    - GCP: [cloud-endpoints](https://cloud.google.com/endpoints)
- **Managed Service for User Management and Authentication**:
    - AWS: [cognito](https://aws.amazon.com/cognito/)
    - GCP: [firebase-authentication](https://firebase.google.com/docs/auth)
- **File Storage**:
    - AWS: [S3 buckets](https://aws.amazon.com/s3/)
    - GCP: [cloud-firestore](https://cloud.google.com/firestore)
- **Message Queues & Topics**:
    - AWS: [sqs queue](https://aws.amazon.com/sqs/)
    - GCP: [pub/sub](https://cloud.google.com/pubsub/docs/overview)
- **Logging & Monitoring**:
    - AWS: [cloud watch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
    - GCP: [cloud logging](https://cloud.google.com/logging)
- **Events & Schedulers**:
    - AWS: [cloud watch events](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html)
    - GCP: [cloud scheduler](https://cloud.google.com/scheduler)
- **Compute Services**:
    - AWS: [lambdas](https://aws.amazon.com/lambda)
    - GCP: [cloud functions](https://cloud.google.com/functions)
- **DNS**: 
    - AWS: [lambda](https://aws.amazon.com/route53)
    - GCP: [cloud-dns](https://cloud.google.com/dns)
- **Cloud Serverless Deployment**
    - AWS: [CodeDeploy](https://aws.amazon.com/codedeploy)
    - GCP: [deployment manager](https://cloud.google.com/deployment-manager)


## Network Diagrams
 
 **AWS**
 <img width="100%" src="./docs/network-data-flow-diagram-aws.png" />

 **GCP**
<img width="100%" src="./docs/user-asset-translator-network-diagram-gpc.svg" /> 

## Actors and Components

 - **XML Data Provider**:
   This is a user or client system that posts xml payloads to our HTTPS endpoint or an SFTP folder. 
 - **Report Consumer**:
   This is a user or client system that downloads files from our SFTP server
 - **SFTP Service:**
    Only one server is needed. Clients can have separate inbox and outbox folders that only they have access to. Amazon has a managed service as part of their [aws-transfer-family](https://aws.amazon.com/aws-transfer-family) services. You can setup an SFTP inbox to point to a specific s3 buckets and sub folders. See the S3 Bucket/Cloud Filestore section for more information. GCP does not have a managed SFTP service, therefore, we can setup a medium-sized high compute instance with a community docker linux image with SFTP service running and a file listener to immediatly move the files from the instance to a file store.
    The files placed in the inbox should be naned `{clientname}-{timestamp}.xml` so that if multiple files are placed at the same time, our system knows which one to process first.
    The files placed in the outbox will be named `user-asset-report-{timestamp}.txt`

    Note: For both solutions, setup a Route 53 or DNS CNAME to give our clients a pretty url to use.

    > **Security and Authentication:** 
    > *The AWS solution* can be done with an RSA PrivateKey (no need for username and password) 
    > *The GCP solution* would require system admins to ssh into the linux SFTP service to add username and password for our clients. We use Firebase user pool if need be.
    > *IP Whitelisting** to be done for both solutions as an additional layer of security
    >*Firewall* configuration to only allow ssh and SFTP ports to be accessible
 - **AWS Gateway/Cloud Endpoint**
    A Cloud Managed API Gateway can have RESTful endpoint configurations that are flexible to be re-directed to a different subsystem or integration enpoint at any time. 
    One of the advantages of using an API gateway is we can easily leverage a managed service for user management
  - **Cognito/Firebase**  
    We can use `Cognito` or `Firebase` to onboard users into user pools, manages permissions, rotate and store credentials, as well as facilitate client self-serving password update.

    > **Authentication:** 
    > The AWS solution: Clients can use an apiKey managed by the Cognito userpool service
    > The GCP solution can be done using Firebase username/password credentials 
  -  **S3 Bucket/Cloud Filestore**  
    Before setting up the SFTP services will need to create a file store (s3 bucket). In our case we need to setup `user-asset-inbox`, `user-asset-inbox/processed`, `user-asset-inbox/errors`, and `user-asset-outbox`.  
    For multi-tenant support, it is a good idea to setup the folders with client name prefixes like this: `{client-name}/user-asset-inbox`. 
    We can also set a time-to-live policy for these files.  The files will be deleted automatically after 6 months (to reduce storage costs)
  -  **Lambdas/Cloud Functions** 
    Lambdas and Cloud Functions are serverless compute process with minimal scope that have high scalablily and limit the cost to the amount of executions invoked. 
    Modern age tooling like AWS SAM makes it easy for us to specify a CloudFormation template to create and manage the necessary AWS Resources, Policies, Permissions and CloudWatc Events that are required to deploy the compute process. 
    By default there is a 14 min timeout, but we can configure this based on our usecase. Logging features come out-of-the-box, and we can test the processes locally. Lambdas support using nodejs (my preference) or python, as well as Java (which should be avoided because of cold start issues).
    These functions can be tested locally, and can be ran locally against the prod/non enviroments, if need be. 
    *We will need 2 lambdas/cloud functions:*
        1) `user-asset-event-processor`: Tasked with ...
        - receiving xml file content or payload and splitting the batches into single entries 
        - normalizing the data json format that represents our internal format
        - validating against the original form and the normalized form
        - assign identifiers (for logging and tracibiliy)
        - If the input was from s3, the original file is moved from `user-asset-inbox` to `user-asset-inbox/processed`, and if an error occured it is logged and sent to `user-asset-inbox/errors`.
        - putting the events onto Queue, for another lambda to process. 
        2) `user-asset-report-generator`: Tasked with ...
        - receiving all json entries on queue 
        - calculating SUM(UserAssetAccount.amount) for all entries
        - if an error occurs, we will log the error messages and store the failures so we can later replay them. We should also put a redrive policy on the queue that will put the messages on an retry queue, so we can setup alerts and write a script to run replay the event payload. (this is helpful if there is a bug anywhere along the chain of handoffs)
        - lastly, the lambda will put the events on S3 (or in the case of GCP directly on the SFTP endpoint to be consumed)
  -  **SQS/Cloud Pub Sub Queues and Topics** 
      Queues are great to use in this usecase because we can hold messages back and control the flow of user asset entries to either be consumed immediatly or on a scheduled cron job. We just need one queue for the `user-asset-events` (json) and another to store retry failures `user-asset-events-failures`.
  -  **CloudWatch Events & Schedulers /GoogleCloud Triggers & Schedulers**
    Both AWS and Google Cloud have scheduling services and event listeners. 
    These are needed to initiate the transfer of User Asset Files and Events from point-to-point. 
        - Here are the transfer activities that need to be done:
            - **File Store ----> user-asset-event-processor:**
                - *AWS*: Cloud Watch Event (packaged with user-asset-event-processor SAM project) set to listen for S3 PutObject events and immediatly invoke the lambda
                - *GCP*: The Cloud Function can be created with a Cloud Storage Create trigger to invoke the function with the file
            - **Queue/Topic -----> user-asset-report-generator:**
                - *AWS*: Cloud Watch Schedule (packaged with user-asset-event-processor SAM project) set to run on a cron schedule (every few min, every hour, daily, etc)
                - *GCP*: Google Cloud Scheduler can notify a topic to invoke the function to process the messages in a Pub/Sub queue or topic

## Sequence Diagrams
 **SFTP**
 <img width="100%" src="./docs/user-asset-translator-sftp-sequence-diagram.svg" />

  **HTTPS**
   <img width="100%" src="./docs/user-asset-translator-http-sequence-diagram.svg" />

> Note: Both the http and ftp flows converge onto the same lambda (as to reduce duplicate functionality)

## Entities

I find it is best to create a class structure that exactly matches the external systems' payload structure as well as a class structure matching our internal schema, then map the two together in code.
Below are the External and Internal Normalized models. I added the databases data type definitions even though there is not a requirement to store any data, it's there just to get an idea of what the data should look like.

**External System Model (UML)**
```                                
                                                +-------------------------------+     
 +---------------------------+             1..n |       AssetAccount            |                
 |   UserAsset               |                  | ------------------------------|     
 | ------------------------- |                  | PK accountNumber      (string)|              
 | FK userId         (string)|-|--------------|<| FK userId             (string)|               
 |                           |           +----|-| FK assetDescriptionId (string)|
 |                           |           |      |     amount (decimal )         |  
 |                           |           |     1|     type              (string)|         
 +---------------------------+           |      +-------------------------------+
                                         |      
                                         |
                                         |      +----------------------------------+ 
                                         | 1..1 |       AssetDescription           | 
                                         |      | -------------------------------- | 
                                         +----|-| PK assetDescriptionId    (string)| 
                                                |    bankAccountType       (string)| * cardinality = CHECKING|SAVINGS
                                                |    retirementAccountType (string)| * cardinality = 401k | IRA
                                                |    bankName              (string)| 
                                                |    percentStocks         (number)| 
                                                |    percentBonds          (number)| 
                                                |    percentOther          (number)| 
                                                +----------------------------------+
```
**External System Class Structure**

```
class User {
    String userId
    AssetProfile[] assets
}

class AssetAccount {
    String type
    String accountNumber
    Double amount
    AssetDescription description
}

class AssetDescription {
    String accountNumber
    String bankAccountType
    String retirementAccountType
    String bankName
    int percentStocks
    int percentBonds
    int percentOther
}
```

**Internal Normalized Model (UML)**
```
                                                                                 +----------------------------------------+   
                                      +--------------------------------+    0..n |      ASSET_ACCOUNT_ALLOCATIONS         |  
+-----------------------+        1..n |       ASSET_ACCOUNTS           |         | -------------------------------------- | 
|       USERS           | 1           | ------------------------------ |    +--o<| PK asset_account_id         VARCHAR(80)|  
| ----------------------|             | PK asset_account_id VARCHAR(80)|-|-/     |    type                     VARCHAR(20)|                                 
| PK user_id VARCHAR(80)|-|---------o<| FK user_id          VARCHAR(80)|         |    percent                  NUMBER(3,0)|                                
|    external_user_id   |             |    bank_name       VARCHAR(255)|         +----------------------------------------+
+-----------------------+             |    account_number   VARCHAR(80)|         
                                      |    amount          NUMBER(10,2)|                         
                                      |    account_type     VARCHAR(20)|         
                                      |    tax_code          VARCHAR(4)|         
                                      +--------------------------------+ 
```

**Internal System Class Structure**

```
class User {
	String userId
	String externalUserId
	AssetAccount[] assetAccounts
}

class AssetAccount {
	String accountAccountId
	String accountNumber
	Double amount
	AccountTypeEnum accountType
	TaxCodeEnum taxCode
	AssetAccountAllocations[] allocations
}

class AssetAllocation {
	String assetAccountId
	AssetAllocationTypeEnum type
	int percent
}
```

**Internal Normalized Model Example Payload (Json Messages on Queue)**
```
{
    "userId": "<uuid>",
    "externalUserId": "3562662382393",
    "assetsAccounts": [
            {
                "assetAcountId": "<uuid>",
                "accountType": "BANK",
                "accountNumber": "123895769767",
                "amount": "27576",
                "taxCode": "CHECKING",
                "bankName": "CHASE"
            },
            {
                "assetAcountId": "<uuid>",
                "accountType": "BANK",
                "accountNumber": "123895769767",
                "amount": "12546",
                "taxCode": "SAVINGS",
                "bankName": "CITI"
            },
            {
                "assetAcountId": "<uuid>",
                "accountType": "BROKERAGE",
                "accountNumber": "5556876547",
                "amount": "335464",
                "taxCode": "BROKERAGE",
                "bankName": "CITI",
                "allocations": [
                    {type: "stocks": "percent" 50},
                    {type: "bonds": "percent" 20},
                    {type: "other": "percent" 30}
                ]
            },
            {
                "assetAcountId": "<uuid>",
                "accountType": "BROKERAGE",
                "accountNumber": "88876758798",
                "amount": "6615",
                "taxCode": "401K",
                "bankName": "CITI",
                "allocations": [
                    {type: "stock": "percent" 100}
                ]
            }
        ]
    }
}
```

## Validations

  - **User**
    - *externalUserId* should be present
        - return status code 400 (Bad Request) with error message: "Missing User ID"
    - *externalUserId* should match format {USER_ID_REGEX}
        - return status code 400 (Bad Request) with error message: "Invalid User ID {user.externalUserId} value."
    - *assets* count must be greater than 0
        - return status code 400 (Bad Request) with error message: "Missing Asset Accounts (must have at least one Asset for User {user.externalUserId})"
 - **AssetAccount**
    - *accountNumber* should be present
        - return status code 400 (Bad Request) with error message:  "Missing Account Type for User {account.userId}"
    - *accountType* should be present
        - return status code 400 (Bad Request) with error message:  "Missing Asset Account type for User {user.externalUserId} Account Number {account.accountNumber}"
    - *accountType* should match cardinality
        - return 400 Bad Request with error message: "Asset Account Bank Account Type is invalid value. '{account.accountType}' is unknown, for User {user.externalUserId} Account Number {account.accountNumber}"
    - *amount* should be present
        - return status code 400 (Bad Request) with error message:  "Missing Asset Account Amount for User {user.externalUserId} Account Number {account.accountNumber}"
    - *taxCode* should be present
        - return status code 400 (Bad Request) with error message:  "Missing Asset Account Tax Code for User {user.externalUserId} Account Number {account.accountNumber}"
    - *amount* format
        - return status code 400 (Bad Request) with error message: " Asset Account Amount is invalid currency format (should be rounded to the nearest dollar) for User {user.externalUserId} Account Number {account.accountNumber}"
    - *taxCode* should match cardinality
        - return 400 Bad Request with error message: "Asset Account Tax Code is invalid value. '{account.taxCode}' is unknown, for User {user.externalUserId} Account Number {account.accountNumber}"
    - *bankName* should match cardinality
        - return 400 Bad Request with error message: "Asset Account Bank Name is invalid value. '{account.bankName}' is unknown, for User {user.externalUserId} Account Number {account.accountNumber}"
    - *allocations* should be present when accountType is BROKERAGE
        - return 400 Bad Request with error message: "Asset Account Account allocations are missing. for User {user.externalUserId} Account Number {account.accountNumber}" 
- **AssetAllocation**
    - *percent* sum when accountType is BROKERAGE should be 100
        - return status code 400 (Bad Request) with error message:  "Missing Asset Allocation Percent for Type {account.allocations[i].type}, User {user.externalUserId} Account Number {account.accountNumber}"
    - *type* should be present
        - return status code 400 (Bad Request) with error message:  "Missing Asset Allocation Type for User {user.externalUserId} Account Number {account.accountNumber}"
    - *type* should match cardinality
        - "Asset Account Allocation Type is invalid value. '{account.allocations[i].type}' is unknown, for User {user.externalUserId} Account Number {account.accountNumber}"
    - *percent* sum when accountType is BROKERAGE should be 100
        - return 400 Bad Request with error message: "Sum of Asset Account Account allocations is {SUM(account.allocations[i].percent)}. The sum should equal 100, for User {user.externalUserId} Account Number {account.accountNumber}"

## Report Generation Strategy
I would create a mustache template and for each messsage on queue, validate it, then apply the deserialized payload instances to the mustache template, all while accumulating the running total(s) for the account field or whatever fields we would like to have aggregations for in future. Finally we would add the totals at the bottom of the script.

## Deployment

AWS Code Deploy is a helpful service to create AWS SAM resources as handling the deployment of lambdas. 
I would recommend using Github webhooks to trigger post merge to master events and kickoff Jenkins jobs that will utilize CodeBuild and CodeDeploy plugins in order create deployments in AWS.

With GCP, we can accomplish the same thing by using jenkins and then Google Cloud SDK 

 **AWS**
 <img width="100%" src="./docs/jenkins-code-deploy-plugin.png" />

 **GCP**
<img width="100%" src="./docs/gcp-jenkins-cloud-function-deploy.png" />
