# CI-CD-Deployment-for-Serverless-ETL-Pipeline
Build a CI/CD pipeline that automatically deploys your entire ETL infrastructure (S3, Glue, Crawlers, IAM, Athena config) whenever you push code.


We are building an automated factory. When you change your Python code or your AWS infrastructure settings and "Push" to GitHub, AWS will automatically test it, build it, and deploy it to the cloud.

# Phase 1: The Local Setup
Create a folder on your computer named aws-etl-pipeline. This will be your Git repository.

# 1. The Project Structure
Create these files exactly as shown:

Plaintext
aws-etl-pipeline/
│
├── template.yaml       <-- Infrastructure (S3, Glue, IAM)
├── buildspec.yml      <-- The "Robot" instructions for CodeBuild
├── scripts/
│   └── etl_job.py      <-- Your actual Python ETL logic
└── tests/
    └── test_logic.py   <-- Simple test to prevent deploying broken code

# Phase 2: Writing the Code

Copy these snippets into your files.

📄 scripts/etl_job.py
This is a simple script that Glue will run. It reads data and prints a message (you can add complex Spark logic later).

Python
import sys
from awsglue.utils import getResolvedOptions

print("🚀 ETL Job Starting...")

# Your Spark/Python transformation logic goes here

print("✅ ETL Job Completed Successfully.")

📄 template.yaml (Infrastructure as Code)

This file tells AWS exactly what "Hardware" to turn on. We are using AWS CloudFormation.

YAML
AWSTemplateFormatVersion: '2010-09-09'
Description: Serverless ETL Infrastructure

Resources:
  # 1. Storage for our data and scripts
  DataBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub "my-etl-data-bucket-${AWS::AccountId}"

  # 2. Permissions for Glue to run
  GlueRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: glue.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSGlueServiceRole
        - arn:aws:iam::aws:policy/AmazonS3FullAccess

  # 3. The ETL Job itself
  MyGlueJob:
    Type: AWS::Glue::Job
    Properties:
      Name: sales-etl-job
      Role: !GetAtt GlueRole.Arn
      Command:
        Name: glueetl
        ScriptLocation: !Sub "s3://my-etl-data-bucket-${AWS::AccountId}/scripts/etl_job.py"
      GlueVersion: "4.0"
      WorkerType: G.1X
      NumberOfWorkers: 2
📄 buildspec.yml (The Instructions)
This file tells AWS CodeBuild exactly what to do when it receives your code.

YAML
version: 0.2

phases:
  install:
    runtime-versions:
      python: 3.9
  pre_build:
    commands:
      - echo "Running unit tests..."
      - python tests/test_logic.py
  build:
    commands:
      - echo "Tests passed! Syncing script to S3..."
      - aws s3 cp scripts/etl_job.py s3://my-etl-data-bucket-$AWS_ACCOUNT_ID/scripts/
      - echo "Deploying CloudFormation Stack..."
      - aws cloudformation deploy --template-file template.yaml --stack-name my-etl-stack --capabilities CAPABILITY_NAMED_IAM
Phase 3: Setting up the AWS "Factory"
Now, we go to the AWS Console to connect these pieces.

# Step 1: Push to GitHub
Create a new repository on GitHub.

Push your local files:

Bash
git init
git add .
git commit -m "initial commit"
git remote add origin <your-repo-url>
git push -u origin main

# Step 2: Create the Pipeline (CodePipeline)
Go to AWS CodePipeline -> Create pipeline.

1. (Current Step)
Category: Keep Deployment selected.

Template: Select Deploy to CloudFormation.

Next: Click the orange Next button.

2. Connect Your Source (The Brain)

Source Provider: Select GitHub (via GitHub App).

Connection: If you haven't connected before, click "Connect to GitHub".

A popup will appear. Give the connection a name (like my-github-connection).

Click "Install a new app" and follow the prompts to authorize AWS to access your specific repository (etl-pipeline-repo).

Repository Name: Choose your repo from the list.

Branch Name: Select main.

Output artifact format: Leave this as CodePipeline default.

Click Next.

3. Deploying

Stack Name: Type etl-pipeline-stack.

Template: * Artifact: Select SourceArtifact.

File Path: Type template.yaml.

CloudFormationResourcePermissions (The "IAM Role")
This is where many people get stuck. CloudFormation needs a "Service Role" to act on your behalf to create the S3 bucket and the Glue job.

Since you are seeing a "Required" error, you need to provide a JSON policy that allows CloudFormation to manage these resources. Copy and paste the following into that box:

JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "glue:*",
                "iam:CreateRole",
                "iam:PutRolePolicy",
                "iam:AttachRolePolicy",
                "iam:GetRole",
                "iam:PassRole",
                "cloudformation:*"
            ],
            "Resource": "*"
        }
    ]
}
Note: In a professional production environment, we would restrict "Resource" to specific ARNs, but for this learning project, "*" ensures your pipeline has the power to build everything without hitting permission walls.

3. Retention Policy
Selection: Delete

Confirmation: This is Correct for a demo or development project.

What it means: If you delete the CodePipeline or the "Stack," AWS will attempt to clean up and delete the resources (like the Glue Job) to save you money.

Warning: If you have important data in your S3 bucket, a Delete policy might try to delete the bucket too. For learning, Delete is fine, but for a real company database, you would choose Retain.

-> create pipeline from template 

# While pipline is creating parallel creat codebuild project

Project name: Select the etl-build-project you just created.

Project type: default project

source -> github
    Use override credentials for this project only -> tick
    Connection -> select 
    Repository

Make sure new service role is selected in enviroment section 

Go to the CodeBuild Console.

Select your project: etl-build-project.

Click Edit -> Environment.

Scroll down to Additional configuration and find Environment variables.

Add one:

Name: AWS_ACCOUNT_ID

Value: 528582359305

Click Update environment.

Buildspec -> Use a buildspec file (Select) -> buildspec.yml

Create Build Project

How to Edit the Pipeline to Add the Build Stage
Now that the project exists, let's put it into your pipeline.

Go to CodePipeline and click on your pipeline name (etl-pipeline-stack or whatever you named it).

Click the Edit button at the top right.

Add the Stage:

Find the gap between the Source stage and the Deploy stage.

Click + Add stage.

Stage name: Build.

Add the Action:

In the new Build stage, click + Add action group.

Action name: Run-CodeBuild.

Action provider: AWS CodeBuild.

Input artifacts: SourceArtifact.

Project name: Select the etl-build-project you just created.

Output artifacts: Type BuildArtifact.

Click Done.

Important: Click Save at the top of the pipeline page to apply the changes.

Part 3: What happens next?
Your pipeline should now look like this:
Source (GitHub) ➔ Build (CodeBuild) ➔ Deploy (CloudFormation).

🚀 To Test It:
Go to your pipeline.

Click Release change.

Watch the Build stage. It will start your "Machine," read your buildspec.yml, run your Python tests, and upload your script to S3.

Once Build turns green, the Deploy stage will automatically start and create your Glue job.

