# Amazon Macie and Lake Formation TBAC

This lab is provided as part of [AWS Innovate Data Edition](https://aws.amazon.com/events/aws-innovate/data/). Click [here](https://github.com/phonghuule/aws-innovate-data-edition-2022) to explore the full list of hands-on labs.

:information_source: You will run this lab in your own AWS account and running this lab will incur some costs. Please follow directions at the end of the lab to remove resources to avoid future costs.

Amazon Macie using Machine Learning to discover sensitive data in S3 buckets. AWS Lake Formation provides Tag-Based Access Control to provide a scalable and flexible way to manage data access in the data lake. In this lab, we will configure a Amazon Macie job to automatically detect sensitive data in an S3 bucket and apply appropriate tags in AWS Lake Formation.


## Overview

In this lab, we will use Amazon Macie to identify sensitive data in a data lake, and automatically AWS Lake Formation Tag Based Access Control (TBAC) technicals to secure the data and ensure there is no unauthorized access.

Amazon Macie is a fully managed data security and data privacy service that uses machine learning and pattern matching to discover and protect your sensitive data in AWS. AWS Lake Formation is a service that makes it easy to set up a secure data lake in days, allowing you to ensure secure access to your sensitive data using granular controls at the column, row, and cell-levels. AWS Lake Formation tag-based access control (LF-TBAC) works with IAM's attribute-based access control (ABAC) to provide fine-grained access to your data lake resources and data.

In this lab, we will load some simple customer data into a bucket, configure AWS Lake Formation access controls and catalog the data with AWS Glue. We will tag the data by default as "Sensitive". We will then trigger a Macie classification job to scan the data and identify any potentially sensitive data in the bucket. These findings will be passed to a AWS Step Functions state machine to orchestrate the process of updating the tags to mark individual columns identified by Macie as personal or financial information.  Finally, we will use Amazon Athena to query the data and ensure we only have access to the right data.

## Architecture

In this lab, we will create the following architecture.
![](/images/architecture.png)

The end solution will perform the following steps:
1. An AWS Glue crawler will populate the data catalog and create the "Customers" table in the "Customer" database in the catalog. 
2. An Amazon Macie job will sample the data in the bucket and identify any sensitive or private data in the customer data in the S3 bucket.
3. Any findings from the classification job will be published as an event, and an Amazon Event Bridge rule with initiate an AWS Step Functions state machine.
4. Within the state machine, an AWS Lambda function will transform the findings.
5. For each column containing sensitive data, the state machine will update the value of the Classification tag in AWS Lake Formation.

We will also be using Amazon Athena to query the data and validate that access to sensitive data has been removed based on the values set for the Classification LFTag.

## Before We Begin

Before you start, there are a few things you should know:
- The resources created in this lab may have some costs associated with them. 
- You will need an IAM user or role with appropriate permissions to execute this lab. This includes permissions to create the resources defined in the CloudFormation template, as well as the manual activities performed through the AWS Management Console. 
- It is assumed you're familiar with using and navigating through the AWS Management Console. 
- Changes to AWS Lake Formation will affect any existing resources that are using the data catalog. It is recommended that you complete this lab in a sandbox environment.

## Setup

In order to streamline this lab, a Cloud Formation template is used to create some initial resources. 

This template will create the following resources in your account:

- IAM Role - macielflab-StepFunctionExecutionRole - this role will be used by the AWS Step Functions state machine. 
- IAM Role - macielflab-GlueExecutionRole - this role will be used by the AWS Glue crawler.
- IAM Role - macielflab-EventBridgeExecutionRole - this role will be used by the AWS Glue crawler.
- IAM Role - macielflab-LambdaExecutionRole - this role will be used by the AWS Lambda function when processing the classification job findings.
- S3 bucket - ${AWS::AccountId}-macielflab-data - this bucket will hold the customer data.
- S3 bucket - ${AWS::AccountId}-macielflab-athena - this bucket will be used for the Amazon Athena queries.
- Glue Database - customer - database in the Glue catalog
- Glue Crawler - macielflab-customer-data-crawler - this crawler will parse the data and create the tables in the catalog. 
- Lambda function - macielflab-transformationFunction - this function will transform the classification job findings to update the LFTag values.
- Step Function State Machine - macielflab-macieFindingOrchestration - an empty state machine to be configured.


1. Click on the following link to launch the CloudFormation process. Please double check you are in the correct account and region.
[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=macilflab-foundation&templateURL=https://s3.amazonaws.com/ee-assets-prod-us-east-1/modules/1e02deca779a4be58b9d50513a464cdc/v1/macielflab/macielflab-template.json)
2. On the first screen, the values will be prepopulated by the link. Click next.
3. For "Stack name", leave "macilflab-foundation" and click "Next".
![](/images/cloudformation/stackdetails.PNG)
4. On the "Configure stack options" page, leave all the default values and click "Next".
5. On the final page, scroll to the end of the page, and mark the check box next to "I acknowledge..." and click "Create stack".
![](/images/cloudformation/ackcreate.PNG)
6. It should take roughly 3 minutes to complete.


## Step 1 - Configure AWS Lake Formation

:information_source: If you are already using AWS Lake Formation in this account, you can skip this step, and proceed directly to Step 2. 

:information_source: If you are not already using AWS Lake Formation, but do have resources using the AWS Glue Catalog, please consider impact carefully and permissions will need to be updated. These steps should be completed in a non-production/sandbox account.

By default, AWS Lake Formation uses the IAMAllowedPrincipals group to provide super permissions to users and roles, effectively delegating access control to IAM policies. Before we can use AWS Lake Formation to manage access to our data, we need to revoke access provided by the IAMAllowedPrincipals group. In this way, IAM policies provide coarse grained permssions, while AWS Lake Formation manages fine grained access control to catalog resources and data.
1. Navigate to [AWS Lake Formation](https://us-east-1.console.aws.amazon.com/lakeformation/) in the AWS Management console. Confirm that you are in the correct Region.
2. The first time you use AWS Lake Formation, you need to configure an Adminstrator. On the "Welcome to AWS Lake Formation" dialog, leave "Add myself" selected and click "Get started".
![](/images/lakeformation/welcome.PNG)
3. Click on "Settings, and uncheck the 2 checkboxes under "Data catalog settings", then click "Save".
![](/images/lakeformation/catalogsettings.PNG)
4. Click on "Administrative roles and tasks" in the navigation panel. 
5. Under "Database Creators", select the IAMAllowedPrincipals group, and click "Revoke".
6. Leave "Create database" checked, and click "Revoke".
![](/images/lakeformation/revokedbcreators.PNG)
7. Click on "Databases" in the navigation panel, and select the "customer" database.
8. Under the "Actions" dropdown, select "edit" for the database.
9. In the "Database details" page, uncheck the "Use only IAM access control..." checkbox, then click "Save".
![](/images/lakeformation/editdatabase.PNG)
10. Click on "Data lake permissions" in the navigation panel, and select the "IAMAllowedPrincipals" record for the customer database. Click "Revoke" and confirm that you want to revoke the permission.

## Step 2 - Configure AWS Lake Formation permissions

In this step, we are going to grant permission for the glue crawler to create tables in our database.

1. In AWS Lake Formation, click on "Data lake permissions", then click "Grant".
2. In the "IAM users and roles" dropdown, select the "macielflab-GlueExecutionRole".
![](/images/lakeformation/glueexecution-principal.PNG)
3. Under "LF-Tags" or catalog resources, select "Named data catalog resources".
![](/images/lakeformation/glueexecution-named.PNG)
4. Under "Databases", select the "customer" database. Leave "Tables" and "Data filters" blank.
![](/images/lakeformation/glueexecution-database.PNG)
5. Under "Database permissions", mark the checkbox next to "Create table". 
![](/images/lakeformation/glueexecution-permissions.PNG)
6. Click Grant. 

We have now granted permission for the Glue crawler to create a table in our database.

## Step 3 - Load data into the S3 bucket. 

In this lab, we're going to use a very small dataset that containers some sensitive personal and financial information. 

1. Download the following CSV file with dummy customer data: [customers.csv](data/customers.csv). Note: click on the link, click on "Raw" and save the file to your local disk.
2. Navigate to [Amazon S3](https://us-east-1.console.aws.amazon.com/s3/) in the AWS Management Console, and click on the bucket named "<accountID>-macielflab-data".
3. Click "Create folder", and enter "Customers" as the Folder name - note that this name is case sensitive and ends with an 's'. Click "Create folder".
![](/images/s3/createfolder.PNG)
4. Click into the newly created folder, click "Upload". Select the CSV file you downloaded earlier and click "Upload" to load the data into the bucket.
![](/images/s3/upload.PNG)

## Step 4 - Populate the Data Catalog

1. Navigate to [AWS Glue](https://us-east-1.console.aws.amazon.com/glue) in the AWS Management Console. Confirm that you are in the correct Region.
2. click on "Crawlers" in the navigation pane.
3. Select the checkbox next to the "macielflab-CustomerDataCrawler" crawler, then click "Run crawler".
4. The crawler may take 1-2 minutes to complete. You should see the status change to "Stopping" or "Ready", with 1 table created. 
![](/images/glue/crawler.PNG)

In the next steps, we'll configure AWS Lake Formation tags on the database and check that we can query the data. 

## Step 5 - Create and configure AWS Lake Formation tags

In this step, we're going to configure our AWS Lake Formation LFTag ontology, set LF-Tag permissions and assign values to our customer table and datbase. 

1. Navigate to [AWS Lake Formation](https://us-east-1.console.aws.amazon.com/lakeformation) in the AWS Management Console. Confirm that you are in the correct Region.
2. Click on "Tables" in the navigation panel to confirm that our customer table has been created correctly. 
![](/images/lakeformation/table.PNG)
3. Click on "LF-Tags" in the AWS Management Console.
4. Click on "Add LF-tag".
5. For "Key", enter the "Classification".
6. For "Values", enter "UNCLASSIFIED, PERSONAL_INFORMATION, FINANCIAL_INFORMATION", and click "Add", then "Add LF-tag". The last two values will be set by Amazon Macie. 
![](/images/lakeformation/addLfTag.PNG)
7. Click on "LF-tag permissions" in the navigation pane, then click "Grant".
8. For "IAM users and roles", select the "macielflab-StepFunctionExecutionRole" role. 
![](/images/lakeformation/tag-stepfunction-principal.PNG)
9. Click "Add LF-Tag", select "Classification" as the key and each of "UNCLASSIFIED", "PERSONAL_INFORMATION" and "FINANCIAL_INFORMATION".
![](/images/lakeformation/tag-stepfunction-tags.PNG)
10. Mark the checkboxes next to "Describe" and "Associate" for both LF-tag permissions and Grantable permissions, then click "Grant".
![](/images/lakeformation/tag-stepfunction-permissions.PNG)
11. Next, click on "Data lake permissions" in the navigation panel and click "Grant". 
12. Remaining in the "Data lake permissions" page, click "Grant". 
13. For "Principals", under "IAM users and roles", select the "macielflab-StepFunctionExecutionRole" role.
![](/images/lakeformation/grant-stepfunction-principal-2.PNG)
14. For "LF-Tags or catalog resources", select "Named data catalog resources"
15. In the "Databases" dropdown, select "customer".
16. In the "Tables" dropdown, select "customers".

![](/images/lakeformation/grant-stepfunction-resources.PNG)

17. Under "Table permissions", mark the checkboxes next to "Super" for both "Table" permissions and "Grantable" permissions.

![](/images/lakeformation/grant-stepfunction-tags-2.PNG)

18. Remaining in the "Data lake permissions" page, click "Grant".
19. For "Principals", under "IAM users and roles",select the role or user you are using to complete these labs.
20. For "LF-Tags or catalog resournces", leave "Resources matched by LF-Tags" selected, then click "Add LF-Tag". Select "Classification" as the "Key" and "UNCLASSIFIED" only as the "Values".

![](/images/lakeformation/grant-user-tags.PNG)

21. Under "Database permissions", mark the checkbox next to "Describe".
22. Under "Table permissions", mark the checkbox for "Select" and "Describe". This will grant your current user permission to view and run select statements for any table with tag "Classification=UNCLASSIFIED". 

![](/images/lakeformation/grant-user-permissions.PNG)

The final step in this section is allocate "Classification=UNCLASSIFIED" to the database. Table and column resources in this database will inherit this tag value. 

23. Click on "Database" in the navigation panel, then click on the "customer" database.
24. Click "Edit LF-tags, then click "Assign new LF-Tag".
25. For "Assigned keys", select "Classification". For values, select "UNCLASSIFIED" only. Click Save.

![](/images/athena/managesettings.PNG)

## Step 6 - Validate permissions with Amazon Athena

1. Navigate to [Amazon Athena](https://us-east-1.console.aws.amazon.com/athena) in the AWS Management Console. Confirm that you are in the correct Region.
2. If this is the first time you're using Amazon Athena, click the "Explore the query editor" button; otherwise, click on "Query editor" in the navigation panel.
3. Click on the "Settings" tab, then click on the "Manage" button.
4. In the "Manage settings" page, for "Query result location and encryption" browse to the bucket named "<accountID>-macielflab-athena", then click "Save".
  
![](/images/athena/managesettings.PNG)
  
5. Select the "Editor" tab, and execute the following statement:
```
SELECT * FROM "customer"."customers" limit 10;
```
![](/images/athena/data-before.PNG)
As you can see, the dataset includes some data that is potentially PII or sensitive financial data. In the next steps, we will Amazon Macie and AWS Step Functions to automatically update the tag values. When we re-run this query, we will no longer be able to see the fields. 

## Step 7 - Extend the AWS Step Functions state machine

1. Navigate to [AWS Step Functions](https://us-east-1.console.aws.amazon.com/states/) in the AWS Management Console. Confirm that you are in the correct Region.
2. You will see that a placeholder state machine named "macielflab-macieFindingOrchestration" has been created for you.
3. Click on this state machine, the click the "Edit" button.
4. On the next page, click on the "Workflow Studio" button in the top right hand corner. This will show you the initial state machine.
  
![](/images/stepfunctions/placeholder.PNG)
  
5. In the seach bar on the left hand side, search for "invoke lambda"
  
![](/images/stepfunctions/action-invokelambda.PNG)
  
6. Click and drag the "Invoke Lambda" state to just above the "Success" state in the state machine.
7. With the "Lambda Invoke" state selected, enter "Transform Payload" as the "State name".
8. Under "API Parameters", for "Function name", select "macielflab-transformationFunction$latest".
  
![](/images/stepfunctions/action-transformPayload.PNG)
  
9. In the state search bar, enter "Map"

![](/images/stepfunctions/action-map.PNG)

10. Click and drag a "Map" state to the state machine between the "Transform Payload" and the "Success" states.
11. With the "Map" state selected, enter "For each classification" as the "State name".
12. For "Path to items array", enter "$.body.classifications".
13. For "Maximum concurrency", enter "1".

![](/images/stepfunctions/action-foreach.PNG)

14. In the state search bar, enter "addLFT".

![](/images/stepfunctions/action-addLFT.PNG)

15. Drag the "AddLFTagsToResource" state to the "Drop state here" placeholder.
16. With the "AddLFTagsToResource" state selected, enter "Update Column Classification Tag"
17. For API Parameters, enter the following JSON:
```
{
  "LfTags": [
    {
      "TagKey": "Classification",
      "TagValues.$": "States.Array($.classificationValue)"
    }
  ],
  "Resource": {
    "TableWithColumns": {
      "DatabaseName": "customer",
      "Name": "customers",
      "ColumnNames.$": "$.columns"
    }
  }
}
```
![](/images/stepfunctions/action-updatetags.PNG)

18. Click on the "Apply and exit" button in the top right hand corner of the page, then click "Save". 
19. You will see a popup warning about IAM permissions. Click "Save anyway" to continue.	

## Step 8 - Create an AWS EventBridge rule
Next, let's create an EventBridge rule that will trigger our state machine each time the Amazon Macie classification job completes. 
1. Navigate to [Amazon EventBridge](https://us-east-1.console.aws.amazon.com/events) in the AWS Management Console. Confirm that you are in the correct Region and click "Create rule". 
2. Enter "MacieLFLab-StartFunctionOnJobComplete" as the rule "Name", then click "Next".
![](/images/eventbridge/rule1.PNG)
3. Under "Event pattern" at the bottom of the next page, select "Macie" in the "AWS Service" dropdown.
4. For "Event type", select "Macie Finding", then click "Next".
![](/images/eventbridge/rule2.PNG)
5. On the next page, choose "Step Functions state machine" in the "Select a target" dropdown.
6. For the "State machine", select the "macielflab-macieFindingOrchestration" state machine.
![](/images/eventbridge/rule3.PNG)
7. For the "Execution role", select "Use existing role" then select the "macielflab-EventBridgeExecutionRole" from the dropdown. Click "Next".
![](/images/eventbridge/rule4.PNG)
8. On the "Configure tags" page, click "Next". 
9. On the final "Review and create" page, click "Create rule".

## Step 9 - Create the Amazon Macie classification job
Our final task is to create and run an Amazon Macie classification job, and verify the results. 
1. Navigate to [Amazon Macie](https://us-east-1.console.aws.amazon.com/macie) in the AWS Management Console. Confirm that you are in the correct Region.
2. If this if the first time you've used Amazon Macie in the account, you will need to enable the service. Click on "Get started". Review the Macie information, then click "Enable Macie".
3. Click "Jobs" in the navigation panel, then click "Create job"
4. Mark the checkbox next to the bucket named "<accountID>-macielflab-data". Leave all other buckets unchecked, then click "Next". If you don't see the bucket, try refreshing the page.
![](/images/macie/job1.PNG)
5. Review the S3 buckets, then click "Next".
6. On the "Refine the scope" page, select "One-time job", then click "Next".
![](/images/macie/job2.PNG)
7. For the "Select managed data identifiers" page, leave the defaults and click "Next".
8. On the "Select custom data identifiers" page, click "Next".
9. On the "Enter general settings" page, enter "macielflab-CustomerClassificationJob" as the "Job name". 
![](/images/macie/job3.PNG)
10. Review the job details on the final page, then click "Submit". This job may take 10-12 minutes to complete. 
![](/images/macie/job4.PNG)

## Step 10 - Validate the Changes


1. Navigate to [Amazon Athena](https://us-east-1.console.aws.amazon.com/athena) in the AWS Management Console. Confirm that you are in the correct Region.
2. Select the "Editor" tab, and execute the following statement again:
```
SELECT * FROM "customer"."customers" limit 10;
```
![](/images/athena/data-after.PNG)
As you can now see, the fields identified by the Macie classification job no longer appear in the query, as we don't have permission to view the columns tagged as "FINANCIAL_INFORMATION" or "PERSONAL_INFORMATION"

3. Let's also also check the customer table in AWS Lake Formation.  Navigate to  [AWS Lake Formation](https://us-east-1.console.aws.amazon.com/lakeformation) in the AWS Management Console.
4. Click on "LF-Tags" in the navigation panel, then click on the "Classification" tag to see the values. Note the LF-tag values for each column.
![](/images/lakeformation/final-tags.PNG)

## Conclusions and Next Steps

Congratulations on completing this lab. In this exercise, we have created and configured AWS Lake Formation Tag Based Access Control (TBAC) and used an Amazon Macie job to automatically detect sensitive data and update AWS Lake Formation tags accordingly. 

For the purposes of this lab, we have initiated the Macie job as a One-off Job that we create manually. As a next step, you could look at using AWS EventBridge and AWS Step Functions to create and run the job automatically each time data is loaded to the S3 bucket, or potentially every time the catalog is updated via the crawler. You could also configure the Amazon Macie classification job to run on a scheduled basis. 

## Clean Up

### Empty the S3 buckets
1. To delete the contents of the S3 buckets, navigate to [Amazon S3](https://us-east-1.console.aws.amazon.com/s3) in the AWS Management Console. 
2. For each bucket created in this lab, select the radio button next to the bucket, then click "Empty". 3. Enter the text to confirm and delete the contents.

### Delete the AWS EventBridge rule
1. To delete the Amazon EventBridge rule, navigate to the rule in the AWS Management Console. 
2. Select the rule, and click Delete. 
3. Click "Confirm" to delete.

### Delete the CloudFormation stack
1. To delete the CloudFormation stack, navigate to the stack in the AWS Management Console, and click "Delete". 
2. This will remove all resources defined in the stack. Monitor the progress to ensure that all resources have been removed.

### Revert changes in AWS Lake Formation - optional.
1. To revert the AWS Lake Formation changes, navigate to AWS Lake Formation in the AWS Management Console. 
2. Click on "Settings" in the navigation panel, then mark the two check boxes under "Default permissions for newly created databases and tables". Click Save.
3. Click on "Administrative roles and tasks" in the navigation panel. 
4. Under "Data lake administrators", click "Choose adminstrators". 
5. Remove your users/role by clicking the "X", then click "Save".
6. Under "Database creators, click Grant. Under "IAM users and roles", select the "IAMAllowedPrinciples" group. 
7. Select the checkbox next to "Create database" under "Catalog permissions", then click "Grant".
8. Under "Data lake permissions", select each row we created in this lab, and click "Revoke", then "Revoke" to confirm.
9. Under "LF-Tags" in the navigation panel, select the "Classification" tag we created earlier. Click "Delete", then enter the text to confirm. Click "Delete".

## Survey

Let us know what you thought of this session and how we can improve the presentation experience for you in the future by completing [this](https://amazonmr.au1.qualtrics.com/jfe/form/SV_1U4cxprfqLngWGy?Session=HOL03) event session poll. Participants who complete the surveys from AWS Innovate Online Conference will receive a gift code for USD25 in AWS credits (1, 2 & 3). AWS credits will be sent via email by September 29, 2023.
Note: Only registrants of AWS Innovate Online Conference who complete the surveys will receive a gift code for USD25 in AWS credits via email.
1. AWS Promotional Credits Terms and conditions apply: https://aws.amazon.com/awscredits/
2. Limited to 1 x USD25 AWS credits per participant.
3. Participants will be required to provide their business email addresses to receive the gift code for AWS credits.


