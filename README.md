# AWS Multi-Account Disk Utilization Monitoring

A comprehensive solution for monitoring disk space utilization across EC2 instances in multiple AWS accounts with QuickSight visualization.

## Architecture Overview

This solution uses a hub-and-spoke architecture to monitor disk utilization across multiple AWS accounts:


### Key Components

1. **CloudFormation Infrastructure**
   - IAM roles for cross-account access
   - EC2 instance for running Ansible
   - S3 bucket for data storage
   - QuickSight data source and dataset

2. **Ansible Monitoring Logic**
   - Cross-account role assumption
   - SSM commands to collect disk usage
   - Data processing with alert thresholds
   - S3 upload with partitioning for QuickSight

3. **Data Flow**

   - Hourly cron job triggers Ansible playbook
   - Ansible assumes roles in child accounts
   - SSM commands collect disk usage data
   - Data is processed and uploaded to S3
   - QuickSight visualizes the data

## File Structure

```
ansible-aws-monitoring/
├── Architecture.png              # Architectural Diagram 
├── ansible/                      # Ansible configuration
│   ├── config.yml                # Target accounts and settings
│   ├── monitor.yml               # Main monitoring playbook
│   ├── monitor-account.yml       # Account-specific tasks
│   └── setup-instructions.md     # Manual setup instructions
├── disk-monitoring.yml           # CloudFormation template
└── README.md                     # This documentation
```

### File Descriptions

- **disk-monitoring.yml**: CloudFormation template that deploys:
  - IAM roles in all accounts
  - EC2 instance, S3 bucket, and QuickSight in master account
  - Basic EC2 setup with Python and Ansible

- **ansible/config.yml**: Configuration file with:
  - Target account IDs
  - Role name for cross-account access
  - AWS region and S3 bucket name

- **ansible/monitor.yml**: Main Ansible playbook that:
  - Loops through target accounts
  - Creates CSV reports with disk utilization data
  - Uploads data to S3 with partitioning
  - Creates QuickSight manifest

- **ansible/monitor-account.yml**: Account-specific tasks:
  - Assumes cross-account role
  - Gets EC2 instances
  - Runs SSM commands to get disk usage
  - Processes data with WARNING/CRITICAL thresholds

- **ansible/setup-instructions.md**: Manual setup instructions for EC2 instance:
  - Directory creation steps
  - Run script creation
  - Cron job configuration
  - Verification steps


## Detailed Setup Instructions

### 1. Deploy CloudFormation StackSet

This step creates and deploys the CloudFormation StackSet to all accounts.

```bash
# Create CloudFormation StackSet
aws cloudformation create-stack-set \
    --stack-set-name disk-monitoring \
    --template-body file://disk-monitoring.yml \
    --parameters ParameterKey=CentralAccountId,ParameterValue=111111111111 \
                 ParameterKey=TargetAccounts,ParameterValue="222222222222,333333333333" \
    --capabilities CAPABILITY_NAMED_IAM
```

**What this does:**
- Creates a CloudFormation StackSet named "disk-monitoring"
- Uses the disk-monitoring.yml template
- Sets the central account ID to 111111111111
- Sets the target accounts to 222222222222 and 333333333333
- Enables IAM capabilities for role creation

```bash
# Deploy StackSet to all accounts
aws cloudformation create-stack-instances \
    --stack-set-name disk-monitoring \
    --accounts 111111111111 222222222222 333333333333 \
    --regions us-east-1
```

**What this does:**
- Deploys the StackSet to all three accounts
- Uses the us-east-1 region for deployment
- Creates all resources defined in the template in each account

### 2. Deploy Ansible Configuration to EC2 Instance

This step deploys the Ansible configuration to the EC2 instance created in the master account.

**Step 2.1: Get the EC2 instance ID**

```bash
# Get the stack ID from the StackSet
STACK_ID=$(aws cloudformation list-stack-instances \
    --stack-set-name disk-monitoring \
    --query "Summaries[?Account=='111111111111'].StackId" \
    --output text)

# Get the instance ID from the stack outputs
INSTANCE_ID=$(aws cloudformation describe-stacks \
    --stack-name $(echo $STACK_ID | cut -d'/' -f2) \
    --query "Stacks[0].Outputs[?OutputKey=='InstanceId'].OutputValue" \
    --output text)

echo "EC2 Instance ID: $INSTANCE_ID"
```

**What this does:**
- Retrieves the CloudFormation stack ID for the master account
- Gets the EC2 instance ID from the stack outputs
- Displays the instance ID for use in subsequent steps

**Step 2.2: Get the S3 bucket name**

```bash
# Get the S3 bucket name from the stack outputs
BUCKET_NAME=$(aws cloudformation describe-stacks \
    --stack-name $(echo $STACK_ID | cut -d'/' -f2) \
    --query "Stacks[0].Outputs[?OutputKey=='MonitoringBucket'].OutputValue" \
    --output text)

echo "S3 Bucket Name: $BUCKET_NAME"
```

**What this does:**
- Gets the S3 bucket name from the stack outputs
- Displays the bucket name for use in subsequent steps

**Step 2.3: Package and upload Ansible files**

```bash
# Create a temporary directory for packaging
mkdir -p /tmp/ansible-package

# Copy Ansible files to the temporary directory
cp -r ansible/* /tmp/ansible-package/

# Create a tarball of the Ansible files
tar -czf ansible-config.tar.gz -C /tmp/ansible-package .

# Upload the tarball to S3
aws s3 cp ansible-config.tar.gz s3://$BUCKET_NAME/config/ansible-config.tar.gz

# Clean up
rm -rf /tmp/ansible-package ansible-config.tar.gz
```

**What this does:**
- Creates a temporary directory for packaging
- Copies all Ansible files to the temporary directory
- Creates a tarball of the Ansible files
- Uploads the tarball to the S3 bucket
- Cleans up temporary files

**Step 2.4: Deploy Ansible configuration via SSM**

```bash
# Deploy Ansible configuration via SSM
aws ssm send-command \
    --instance-ids $INSTANCE_ID \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=[
        "cd /tmp",
        "aws s3 cp s3://'$BUCKET_NAME'/config/ansible-config.tar.gz .",
        "mkdir -p /opt/aws-monitoring",
        "tar -xzf ansible-config.tar.gz -C /opt/aws-monitoring",
        "mkdir -p /opt/aws-monitoring/reports",
        "cat > /opt/aws-monitoring/run.sh << \'EOF\'
#!/bin/bash
cd /opt/aws-monitoring
source .env
export BUCKET_NAME
ansible-playbook monitor.yml
echo \"Disk monitoring complete\"
EOF",
        "chmod +x /opt/aws-monitoring/run.sh",
        "echo \"0 * * * * /opt/aws-monitoring/run.sh\" | crontab -",
        "rm -f /tmp/ansible-config.tar.gz"
    ]'
```

**What this does:**
- Uses AWS Systems Manager (SSM) to run commands on the EC2 instance
- Downloads the Ansible configuration tarball from S3
- Creates the monitoring directory and extracts the Ansible files
- Creates the reports directory for storing CSV files
- Creates a run script that executes the Ansible playbook
- Makes the run script executable
- Sets up a cron job to run the monitoring script hourly
- Cleans up temporary files

**Step 2.5: Clean up S3 temporary files**

```bash
# Remove the tarball from S3
aws s3 rm s3://$BUCKET_NAME/config/ansible-config.tar.gz
```

**What this does:**
- Removes the temporary tarball from the S3 bucket

### 3. Add a New Account to Monitoring

This process adds a new AWS account to the monitoring solution.

**Step 3.1: Update CloudFormation StackSet parameters**

```bash
# Define the new account ID
NEW_ACCOUNT="444444444444"

# Get current target accounts from the StackSet
CURRENT_ACCOUNTS=$(aws cloudformation describe-stack-set \
    --stack-set-name disk-monitoring \
    --query "StackSet.Parameters[?ParameterKey=='TargetAccounts'].ParameterValue" \
    --output text)

# Append the new account to the list
NEW_ACCOUNTS="$CURRENT_ACCOUNTS,$NEW_ACCOUNT"

# Update the StackSet parameters
aws cloudformation update-stack-set \
    --stack-set-name disk-monitoring \
    --use-previous-template \
    --parameters ParameterKey=TargetAccounts,ParameterValue="$NEW_ACCOUNTS" \
    --capabilities CAPABILITY_NAMED_IAM
```

**What this does:**
- Defines the new account ID to add
- Gets the current list of target accounts from the StackSet
- Appends the new account to the list
- Updates the StackSet parameters with the new account list

**Step 3.2: Deploy the StackSet to the new account**

```bash
# Deploy the StackSet to the new account
aws cloudformation create-stack-instances \
    --stack-set-name disk-monitoring \
    --accounts $NEW_ACCOUNT \
    --regions us-east-1
```

**What this does:**
- Deploys the StackSet to the new account
- Creates the necessary IAM roles in the new account

**Step 3.3: Update Ansible configuration**

```bash
# Create a temporary file with the updated config
cat > /tmp/config.yml << EOF
# Disk Monitoring Configuration
target_accounts:
$(echo $NEW_ACCOUNTS | tr ',' '\n' | sed 's/^/  - "/')

role_name: "DiskMonitoringRole"
aws_region: "us-east-1"
bucket_name: "{{ ansible_env.BUCKET_NAME }}"
EOF

# Upload the updated config to S3
aws s3 cp /tmp/config.yml s3://$BUCKET_NAME/config/config.yml

# Update the config on the EC2 instance via SSM
aws ssm send-command \
    --instance-ids $INSTANCE_ID \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=[
        "aws s3 cp s3://'$BUCKET_NAME'/config/config.yml /opt/aws-monitoring/config.yml"
    ]'

# Clean up
rm /tmp/config.yml
aws s3 rm s3://$BUCKET_NAME/config/config.yml
```

**What this does:**
- Creates a temporary file with the updated Ansible configuration
- Uploads the updated configuration to S3
- Uses SSM to update the configuration on the EC2 instance
- Cleans up temporary files

## Verification and Monitoring

### Check Data Collection

```bash
# List S3 data
aws s3 ls s3://$BUCKET_NAME/data/ --recursive

# View critical instances
aws s3 cp s3://$BUCKET_NAME/data/year=$(date +%Y)/month=$(date +%m)/day=$(date +%d)/hour=$(date +%H)/disk_utilization.csv - | grep "CRITICAL"
```

### Access QuickSight Dashboard

1. Go to AWS QuickSight console
2. Use dataset "disk-monitoring-dataset"
3. Create visualizations:
   - Disk usage by account (bar chart)
   - Critical instances over time (line chart)
   - Instance status distribution (pie chart)

