# Amazon-Macie-and-Lake-Formation-TBAC
Amazon Macie using Machine Learning to discover sensitive data in S3 buckets. Amazon Lake Formation provides Tag-Based Access Control to provide a scalable and flexible way to manage data access in the data lake. In this lab, we will configure a Amazon Macie job to automatically detect sensitive data in an S3 bucket and apply appropriate tags in Lake Formation.

:information_source: You will run this lab in your own AWS account and running this lab will incur some costs. Please follow directions at the end of the lab to remove resources to avoid future costs.

## Overview

This this lab, we will use Amazon Macie to identify sensitive data in a data lake, and automatically AWS Lake Formation Tag Based Access Control (TBAC) technicals to secure the data and ensure there is no unauthorized access.

Amazon Macie is a fully managed data security and data privacy service that uses machine learning and pattern matching to discover and protect your sensitive data in AWS. AWS Lake Formation is a service that makes it easy to set up a secure data lake in days, allowing you to ensure secure access to your sensitive data using granular controls at the column, row, and cell-levels. Lake Formation tag-based access control (LF-TBAC) works with IAM's attribute-based access control (ABAC) to provide fine-grained access to your data lake resources and data.

In this lab, we will load some simple customer data into a bucket, configure AWS Lake Formation access controls and catalog the data with AWS Glue. We will tag the data by default as "Sensitive". We will then trigger a Macie classification job to scan the data and identify any potentially sensitive data in the bucket. These findings will be passed to a AWS Step Functions state machine to orchestrate the process of updating the tags to mark individual columns identified by Macie as personal or financial information.  Finally, we will use Amazon Athena to query the data and ensure we only have access to the right data.

## Architecture

In this lab, we will create the following architecture.
![Architecture Overview](/images/architecture.png)

The end solution will perform the following steps:
1. An AWS Glue crawler will populate the data catalog and create the "Customers" table in the "Customer" database in the catalog. 
2. An Amazon Macie job will sample the data in the bucket and identify any sensitive or private data in the customer data in the S3 bucket.
3. Any findings from the classification job will be published as an event, and an Amazon Event Bridge rule with initiate an AWS Step Functions state machine.
4. Within the state machine, an AWS Lambda function will transform the findings.
5. For each column containing sensitive data, the state machine will update the value of the Classification tag in AWS Lake Formation.
6. An SNS message will be sent once the update is complete, and an email will be sent to the registered email address with the fields that have been updated.

We will also be using Amazon Athena to query the data and validate that access to sensitive data has been removed based on the values set for the Classification LFTag.

## Setup

In order to streamline this lab, a Cloud Formation template is used to create some initial resources. 

Click on the following link to launch the Cloud Formation template in the us-east-1 region. 

[Launch](https://TBD/)

This template will create the following resources in your account:

- IAM Role - MacieFindingsStepFunctionExecutionRole - this role will be used by the AWS Step Functions state machine. 
- IAM Role - MacieFindingsGlueExecutionRole - this role will be used by the AWS Glue crawler.
- IAM Role - MacieTransformLambdaExecutionRole - this role will be used by the AWS Lambda function when processing the classification job findings.
- S3 bucket - <accountID>-macielflab-source - this bucket will hold the customer data.
- S3 bucket - <accountID>-macielflab-athena - this bucket will be used for the Amazon Athena queries.
- Glue Crawler - macielflab-customer-data-crawler - this crawler will parse the data and create the tables in the catalog. 
- Lambda function - MacieFindingsTransform - this function will transform the classification job findings to update the LFTag values.
- Step Function State Machine - an empty state machine to be configured.

## Step 1 - Configure AWS Lake Formation

## Step 2 - Populate the Data Catalog

## Step 3 - Set Lake Formation permissions and tag values

## Step 4 - Validate permissions will Amazon Athena

## Step 5 - Extend the AWS Step Functions state machine

## Step 6 - Create an AWS EventBridge rule

## Step 7 - Create the Amazon Macie classification job

## Step 8 - Validate the outcomes

## Conclusions and Next Steps

## Clean Up

1. Navigate to Amazon Macie and delete the classification job
2. Delete the Catalog resources
3. Delete contents of the S3 buckets
4. Delete the AWS EventBridge rule
5. Delete the CloudFormation stack
