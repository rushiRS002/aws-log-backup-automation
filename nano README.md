# AWS Log Backup Automation

## Project Overview

This project automates log backup from an EC2 instance to Amazon S3 using AWS Lambda, Systems Manager (SSM), EventBridge, and SNS.

The workflow automatically:

- Creates an S3 bucket
- Creates log files inside EC2
- Uploads logs to S3
- Runs automatically using EventBridge
- Sends notifications using SNS

---

# Architecture

EventBridge

↓
Lambda Function

↓
SSM Command

↓
EC2 Instance

↓
Create Log Files

↓
Upload Logs to S3

↓
SNS Notification

---

# AWS Services Used

- AWS Lambda
- Amazon EC2
- Amazon S3
- AWS Systems Manager (SSM)
- Amazon EventBridge
- Amazon SNS
- AWS IAM

---

# Project Structure


aws-log-backup-automation/

│

├── lambda_function.py

├── requirements.txt

├── README.md

├── screenshots/

│   ├── ec2.png

│   ├── lambda.png

│   ├── s3.png

│   ├── sns.png

│   └── eventbridge.png
│
└── docs/

    └── setup-guide.md

---

# Step 1 — Launch EC2 Instance

## Open EC2 Console

Launch Ubuntu EC2 instance.

### Configuration

- AMI → Ubuntu
- Instance Type → t2.micro
- Key Pair → Create or select existing
- Allow SSH (Port 22)

### Important

Enable:
- Auto-assign Public IP

---

# Step 2 — Attach IAM Role to EC2

## Create IAM Role

Open IAM Console.

### Trusted Entity
- AWS Service
- EC2

### Attach Policies

- AmazonS3FullAccess
- AmazonSSMManagedInstanceCore

### Role Name

```text
EC2-SSM-S3-Role
```

---

## Attach Role to EC2

Go to:

EC2 → Actions → Security → Modify IAM Role

Select:

```text
EC2-SSM-S3-Role
```

Save.

---

# Step 3 — Install AWS CLI

SSH into EC2.

Run:

```bash
sudo apt update
sudo apt install awscli -y
```

Verify:

```bash
aws --version
```

---

# Step 4 — Install SSM Agent

Install SSM Agent:

```bash
sudo snap install amazon-ssm-agent --classic
```

Enable Service:

```bash
sudo systemctl enable snap.amazon-ssm-agent.amazon-ssm-agent.service
```

Start Service:

```bash
sudo systemctl start snap.amazon-ssm-agent.amazon-ssm-agent.service
```

Check Status:

```bash
sudo systemctl status snap.amazon-ssm-agent.amazon-ssm-agent.service
```

---

# Step 5 — Create Lambda IAM Role

Open IAM Console.

Create new role.

### Trusted Entity

- AWS Service
- Lambda

### Attach Policies

- AmazonS3FullAccess
- AmazonSSMFullAccess
- AmazonEC2FullAccess

### Role Name

```text
Lambda-SSM-S3-Role
```

Save role.

---

# Step 6 — Create Lambda Function

Open Lambda Console.

### Configuration

- Function Name → log-backup-function
- Runtime → Python 3.x

### Execution Role

Select:

```text
Lambda-SSM-S3-Role
```

Create Function.

---

# Step 7 — Deploy Python Code

Replace default Lambda code with:

```python code

import boto3
import time
from botocore.exceptions import ClientError

# AWS Clients
s3 = boto3.client('s3', region_name='ap-south-1')
ssm = boto3.client('ssm', region_name='ap-south-1')

# Variables
bucket_name = 'rushi-log-bucket-2026-123'
region = 'ap-south-1'
instance_id = 'YOUR_INSTANCE_ID'

def wait_for_ssm(instance_id):

    while True:

        response = ssm.describe_instance_information()

        ids = [
            i['InstanceId']
            for i in response['InstanceInformationList']
        ]

        if instance_id in ids:
            print("SSM Online")
            break

        print("Waiting for SSM...")
        time.sleep(10)

# Create bucket automatically
def create_bucket():

    try:
        s3.head_bucket(Bucket=bucket_name)

        print(f"Bucket {bucket_name} already exists")

    except ClientError:

        if region == 'us-east-1':

            s3.create_bucket(
                Bucket=bucket_name
            )

        else:

            s3.create_bucket(
                Bucket=bucket_name,
                CreateBucketConfiguration={
                    'LocationConstraint': region
                }
            )

        print(f"Bucket {bucket_name} created successfully")

def lambda_handler(event, context):

    # Step 1: Create Bucket
    create_bucket()

    # Step 2: Wait for SSM
    wait_for_ssm(instance_id)

    # Step 3: Create Log Files
    commands = [

        "mkdir -p /home/ubuntu/logs",

        "echo 'System Log File' > /home/ubuntu/logs/system.log",

        "echo 'Application Log File' > /home/ubuntu/logs/app.log",

        "date >> /home/ubuntu/logs/system.log",

        "date >> /home/ubuntu/logs/app.log",

        f"aws s3 sync /home/ubuntu/logs s3://{bucket_name}/ec2-logs/{instance_id}/"
    ]

    response = ssm.send_command(
        InstanceIds=[instance_id],
        DocumentName='AWS-RunShellScript',
        Parameters={
            'commands': commands
        }
    )

    return {
        'statusCode': 200,
        'body': 'Logs uploaded successfully'
    }
```

Deploy function.

---

# Step 8 — Configure EventBridge Trigger

Open EventBridge Console.

Create Rule.

### Rule Type

- Schedule

### Example

```text
rate(5 minutes)
```

OR

```text
cron(0 18 * * ? *)
```

### Target

- Lambda Function

Select:

```text
log-backup-function
```

Create Rule.

---

# Step 9 — Configure SNS Notification

Open SNS Console.

## Create Topic

Type:
- Standard

Topic Name:

```text
log-backup-topic
```

---

## Create Subscription

Protocol:
- Email

Endpoint:
- Your Email Address

Confirm subscription from email.

---

# Step 10 — Test Lambda Function

Open Lambda Function.

Click:

```text
Test
```

Expected Result:

- S3 bucket created
- Log files created
- Logs uploaded to S3
- SNS notification sent

---

# Expected S3 Output

```text
ec2-logs/
   └── instance-id/
         ├── system.log
         └── app.log
```

---

# Screenshots

Add screenshots inside:

```text
screenshots/
```

Example screenshots:
- EC2 instance
- IAM roles
- Lambda function
- S3 bucket
- EventBridge trigger
- SNS topic

---

# Future Improvements

- CloudWatch logging
- Terraform support
- CI/CD pipeline
- Docker deployment
- Multi-instance support
- CloudFormation template

---

# Author

Rushikesh Shinde

---

# GitHub Topics

aws  
lambda  
ec2  
s3  
ssm  
sns  
eventbridge  
cloud  
devops  
automation
