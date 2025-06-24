# EC2 Instance Setup Instructions

This document describes the manual steps needed to set up the EC2 instance for disk monitoring.

## Initial Setup

1. Create the monitoring directory:
   ```bash
   mkdir -p /opt/aws-monitoring
   ```

2. Copy Ansible files to the monitoring directory:
   ```bash
   cp -r /tmp/ansible/* /opt/aws-monitoring/
   ```

3. Create reports directory:
   ```bash
   mkdir -p /opt/aws-monitoring/reports
   ```

## Create Run Script

Create a run script that will execute the Ansible playbook:

```bash
cat > /opt/aws-monitoring/run.sh << 'EOF'
#!/bin/bash
cd /opt/aws-monitoring
source .env
export BUCKET_NAME
ansible-playbook monitor.yml
echo "Disk monitoring complete"
EOF

chmod +x /opt/aws-monitoring/run.sh
```

## Setup Cron Job

Set up a cron job to run the monitoring script hourly:

```bash
echo "0 * * * * /opt/aws-monitoring/run.sh" | crontab -
```

## Verify Setup

1. Check that the cron job is set up correctly:
   ```bash
   crontab -l
   ```

2. Run the monitoring script manually to verify it works:
   ```bash
   /opt/aws-monitoring/run.sh
   ```

3. Check that data is being uploaded to S3:
   ```bash
   aws s3 ls s3://$BUCKET_NAME/data/ --recursive
   ```