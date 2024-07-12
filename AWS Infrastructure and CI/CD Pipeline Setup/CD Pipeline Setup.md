# AWS Infrastructure and CI/CD Pipeline Setup

## Introduction

This document outlines the design and implementation of a highly available and scalable web application on AWS, including infrastructure provisioning, CI/CD pipeline setup, and monitoring configurations.

## Architecture Design and Provisioning

### Architecture Diagram AmorServ LLC

Architecture Diagram can be found in this link: https://drive.google.com/file/d/17T2-2hMm-F28puh2lxdLDQXMLi0ozqoz/view?usp=sharing

### Infrastructure Components

#### VPC and Subnets

##### Create a new VPC.

###### Settings:

- Name: MyVPC

- IPv4 CIDR block: 10.0.0.0/16

- Enable DNS hostnames: Yes

- Enable DNS support: Yes

##### Create Subnets:

- Action: Create four subnets within the VPC.

###### Settings:

- Public Subnet 1:
  Name: PublicSubnet1
  Availability Zone: us-east-1a
  IPv4 CIDR block: 10.0.1.0/24

- Public Subnet 2:
  Name: PublicSubnet2
  Availability Zone: us-east-1b
  IPv4 CIDR block: 10.0.2.0/24

- Private Subnet 1:
  Name: PrivateSubnet1
  Availability Zone: us-east-1a
  IPv4 CIDR block: 10.0.3.0/24

- Private Subnet 2:
  Name: PrivateSubnet2
  Availability Zone: us-east-1b
  IPv4 CIDR block: 10.0.4.0/24

##### Action: Create an Internet Gateway and attach it to the VPC.

###### Settings:

- Name: MyInternetGateway

##### Action: Create route tables for the public and private subnets.

###### Settings:

- Public Route Table:
  Name: PublicRouteTable
  Routes: 0.0.0.0/0 pointing to the Internet Gateway
  Associate with PublicSubnet1 and PublicSubnet2

- Private Route Table:
  Name: PrivateRouteTable
  Routes: No default route
  Associate with PrivateSubnet1 and PrivateSubnet2

- EC2 Instance (Bastion Host)
  Launch EC2 Instance:
  Action: Create an EC2 instance in one of the public subnets.

###### Settings:

- Name: BastionHost
  AMI: Amazon Linux 2
  Instance Type: t2.micro
  Subnet: PublicSubnet1
  Security Group: Allow SSH (port 22) from your IP address

##### AWS Lambda Functions for the backend services

- **Function**: Node.js backend services deployed in private subnets.
- **API Gateway**: Exposing Lambda functions as HTTP endpoints.

###### Settings:

- Name: MyBackendLambda
- Runtime: Node.js 14.x
- Role: Create a new role with basic Lambda permissions
- VPC: MyVPC
- Subnet: PrivateSubnet1, PrivateSubnet2
- Security Group: Allow necessary inbound/outbound traffic

#### Launch a MySQL RDS instance in the private subnets

- **Instance**: MySQL RDS instance in private subnets.
- **Security**: Configured with Multi-AZ for high availability.

###### Settings:

- Name: MyRDSInstance
- Engine: MySQL
- Instance Class: db.t3.micro
- Multi-AZ Deployment: Yes
- Subnet Group: Include PrivateSubnet1 and PrivateSubnet2
- Security Group: Allow inbound MySQL traffic (port 3306) from the Lambda security group

### Infrastructure as Code (IaC)

#### Deploying the CloudFormation Stacks

There are 6 six stacks to be deployed which involves:

1. VPC and Subnets Script (vpc.yaml)
2. Lambda Functions Script (lambda.yaml)
3. RDS Instance Script (rds.yaml)
4. Amplify Deployment Script (amplify.yaml)
5. API Gateway Script (api-gateway.yaml)
6. IAM Roles and Policies Script (iam.yaml)

###### To deploy these stacks, follow these steps:

- Upload the YAML files to S3 (for cross-referencing).
- Create a CloudFormation stack for each script:
- Navigate to the AWS CloudFormation console.
- Click "Create stack" and select "With new resources (standard)".
- Upload the respective YAML file and follow the prompts to create the stack.
  Repeat for each script.

#### CI/CD Pipeline Configuration Guide

Using AWS services to automate the build and deployment of your frontend and backend applications.

##### Prerequisites

1. **Install AWS CLI and Configure**:
   ```sh
   aws configure
   ```
2. **Install AWS CloudFormation**:

   - AWS CloudFormation is part of the AWS CLI.

3. **Install AWS CodeBuild**:
   - AWS CodeBuild is part of AWS services.

##### Step 1: Prepare Your S3 Bucket

1. **Create an S3 Bucket**:
   ```sh
   aws s3 mb s3://your-pipeline-bucket
   ```

##### Step 2: Define and Deploy the Pipeline

1. **Create `pipeline.yaml`**:

   - Save the provided `pipeline.yaml` script to your local machine.

2. **Deploy `pipeline.yaml` Using AWS CloudFormation**:
   ```sh
   aws cloudformation create-stack --stack-name MyPipelineStack --template-body file://pipeline.yaml --capabilities CAPABILITY_NAMED_IAM
   ```

##### Step 3: Configure AWS CodeBuild Projects

1. **Create `buildspec-frontend.yml`**:

   - Save the provided `buildspec-frontend.yml` script to your frontend project directory.

2. **Create `buildspec-backend.yml`**:
   - Save the provided `buildspec-backend.yml` script to your backend project directory.

##### Step 4: Configure Your GitHub Repository

1. **Generate a GitHub Personal Access Token**:

   - Go to GitHub, navigate to Settings > Developer settings > Personal access tokens, and generate a new token with `repo` and `admin:repo_hook` permissions.

2. **Add the GitHub Token to AWS Systems Manager Parameter Store**:

   ```sh
   aws ssm put-parameter --name /github/token --value your-github-token --type SecureString
   ```

3. **Update `pipeline.yaml`**:
   - Ensure the GitHub token and repository details are correctly referenced in your `pipeline.yaml`.

##### Step 5: Connect AWS CodePipeline to Your GitHub Repository

1. **Set Up Source Stage in CodePipeline**:
   - Configure the Source stage in your pipeline to trigger on code changes pushed to your GitHub repository. This is defined in the `pipeline.yaml` script you already deployed.

##### Step 6: Build and Deploy the Frontend and Backend

1. **Push Changes to Your GitHub Repository**:

   - Make sure your frontend and backend repositories are connected to your GitHub account.
   - Commit and push changes to your GitHub repository. This will trigger the pipeline.

2. **Monitor the Pipeline**:
   - Go to the AWS Management Console, navigate to CodePipeline, and select your pipeline to monitor the stages and progress.
3. **Ensure Successful Builds**:
   - The Source stage will pull the latest code from GitHub.
   - The Build stage will use the `buildspec-frontend.yml` and `buildspec-backend.yml` to build the projects.
   - The Deploy stage will deploy the built artifacts to AWS resources defined in your `pipeline.yaml`.

##### Step 7: Verify Deployments

1. **Check AWS Amplify**:

   - Ensure your frontend is deployed and accessible via AWS Amplify.
   - Navigate to AWS Amplify in the AWS Management Console, and confirm that your React/Next.js application is deployed and running.

2. **Check AWS Lambda**:
   - Ensure your backend Lambda functions are deployed and running.
   - Navigate to AWS Lambda in the AWS Management Console, and confirm that your Node.js functions are deployed and operational.

#### Configuring CloudWatch

This guide will help you set up monitoring and logging for your application using AWS CloudWatch.

##### Prerequisites

1. **Install AWS CloudFormation**:
   - AWS CloudFormation is part of the AWS CLI.

###### Step 1: Define the CloudWatch Configuration

- define configuration by saving in file 'cloudwatch.yaml' file

###### Step 2: Deploy the CloudWatch Configuration

1. **Deploy `cloudwatch.yaml` Using AWS CloudFormation**:
   - Use the AWS CLI to create a CloudFormation stack from your `cloudwatch.yaml` file.
   ```sh
   aws cloudformation create-stack --stack-name CloudWatchStack --template-body file://cloudwatch.yaml --capabilities CAPABILITY_NAMED_IAM
   ```

##### Step 3: Verify the CloudWatch Alarms

1. **Go to the AWS Management Console**:

   - Navigate to the CloudWatch service.

2. **Check Alarms**:

   - In the CloudWatch console, go to the Alarms section.
   - Verify that the `LambdaErrorAlarm` and `RDSCpuAlarm` are listed and configured correctly.

3. **Test Alarms (Optional)**:
   - To test the alarms, you can manually trigger errors in your Lambda functions or increase the load on your RDS instance to see if the alarms are triggered.

#### Set Up SNS for Notifications:

Create an SNS topic and subscribe to it for receiving alarm notifications.
