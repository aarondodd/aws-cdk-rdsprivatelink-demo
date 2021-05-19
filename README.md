
# Cross-account RDS access using AWS Privatelink demo

This demo shows how to leverage AWS Privatelink to publish a Relational Database Service (RDS) database from one account to other accounts. This method allows for point-to-point connectivity between accounts without relying on routing subnets. It leverages a Lambda function to keep the Privatelink's associated Network Load Balancer (NLB) updated with the RDS endpoint which is triggered whenever the RDS cluster enters a failover state.

Reasons this demo was created:
1. A use case was presented where routing between VPCs was not allowed and the database team and application team were in separate accounts. This enabled the resources to remain where they were but still be connected.
2. Another use case where the application and database were in separate VPCs that had overlapping CIDR blocks. This meant VPC peering could not be used (not without some fancy NATing). This architecture does not rely on routing so enabled communication between the application and database.

Note: check the AWS documentation for NLB features. As of publishing this demo, an NLB only supports an EC2 instance or IP address as targets, hence the need for the associated Lambda function. Should NLB ever support DNS entries as targets, the Lambda won't be needed.

Running this CDK code generates the left side of the below architecture diagram.

![Architecture diagram](architecture.png)

This demo creates the following:
1. A Virtual Private Cloud (VPC) with two isolated (no Internet) subnets
2. An RDS MySQL multi-availability zone cluster
3. A Simple Notification Service (SNS) topic for receiving RDS failover events
4. An RDS event subscription filtered by failover notices and published to the SNS topic
5. An NLB and associated PrivateLink (aka VPC Endpoint Service) endpoint
6. A Lambda function, triggered by SNS, that resolves the RDS endpoint DNS name to the IP address of the active instance and updates the NLB's target group accordingly

**Terms to be aware of**
- VPC Endpoint Service: This is the AWS Console and CDK name for a PrivateLink connection. In the console, you'll see this under VPC / Endpoint Services. An Endpoint Service is what you create to share out. Endpoint Services need an associated NLB.
- VPC Endpoint: This is where you create an endpoint to use, as opposed to above for sharing. An Endpoint can be to an AWS service (such as to access S3 or other services without Internet access in your VPC) or a custom share you've been granted permission to use, such as in this demo.

**Caveats / limitations**
- This code is as-is and is not endorsed or supported by AWS or anyone. Use at your own risk.
- The AWS NLB also has a start delay on performing a health check on targets. This doesn't appear to be configurable and can added a minute or so until traffic is directed to the new IP. This means, using RDS natively, a failover may take ~30 seconds to be active but in this architecture, may take a few minutes.
- Your RDS, PrivateLink, and the consuming accounts must be in the same region and same underlying AWS Availability Zones. Ensure you have subnets in the consuming accounts with the same AZ-IDs (check the AWS Console / Subnets and column Availability Zone ID).

## Pre-requisites

To use this demo, you need:
1. The [AWS Cloud Development Kit (CDK) ](https://aws.amazon.com/cdk/) installed. See [Getting Started With AWS CDK](https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html) for installation and usage.
2. An AWS account to deploy this demo in.
3. Another AWS account to consume the database from.
4. This other account will need a VPC in the same region as where you deploy the CDK stack, and subnets created. Ideally, subnets should exist in all Availabity Zones, but at a minimum will need them in the same AZ-IDs as the RDS account's. You can see subnet AZ-IDs in the AWS Console under / VPC / Subnets in the column "Availability Zone IDs" and will, for us-east, look like "use-azN".

## Usage

### RDS Account
1. Clone this repo locally.
2. Open app.py and adjust the "props" section with appropriate values. At least "principals_to_share_with" needs to be adjusted.
3. Deploy the CDK stack: `cdk deploy --all`.
4. Watch the output for "function ARN:" and for "Endpoint service ID: " and make note of the values.
5. Once deployed, kick off the Lambda function for initially populating the target group, either by logging into the AWS Console and executing the function or running `run_lambda.py [arn from output]`.

### Consumer Account
1. From the AWS account listed in "principals_to_share_with", log in and go to VPC / Endpoints and create a new Endpoint. 
2. Choose custom and search by the "Endpoint service ID" from the output. 
3. Scroll down and ensure appropriate subnets are selected. Note: you will need a VPC already created with subnets in the same Availability Zones as the RDS account. The security group you assign should allow for the DB port from the instances/applications you'll be querying RDS.

## Testing

**Connectivity**: In the AWS account listed under "principals_to_share_with", you can create an EC2 instance that can route to the Endpoint you created and use the MySql cli to connect.

**Failover**: In the AWS console, reboot the RDS instance and choose "reboot with failover". Watch the target group's members in another tab and in a few minutes it should update to a new IP address.

## Enhancements

Below are ideas for taking this further, especially if production-izing:
1. Instead of using "allowed_principals" in the privatelink_stack.py and specifying an array principals, leverage AWS Resource Access Manager (RAM) and share with an AWS Organization or OU.
2. Move the props dictionary to AWS Systems Manager (SSM) Parameter Store or at least source from a separately managed configuration file so adding/removing shared accounts doesn't requiring editing the core stack.
3. Create a multi-account CDK and a stack to create the endpoint in the consumer account.
