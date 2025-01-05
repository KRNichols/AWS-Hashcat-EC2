# Ethical Password Cracking Automation on AWS

This repository provides a complete workflow for ethical password cracking using AWS. It includes secure data handling, automated results upload to S3, and EC2 instance shutdown with Lambda-based termination.

---

## Features
- **Automated Workflow:** Runs Hashcat, uploads results to S3, securely deletes sensitive data, and triggers EC2 instance termination.
- **Secure Data Deletion:** Uses `srm` for securely overwriting and removing sensitive files.
- **Lambda Integration:** Automatically terminates EC2 instances after task completion to prevent unnecessary charges.
- **IAM Role Integration:** Follows AWS least-privilege best practices for access control.

---

## Prerequisites
- AWS account with the necessary permissions
- Access to an EC2 instance with a GPU-optimized AMI (e.g., Ubuntu Server 20.04)
- Installed AWS CLI and Hashcat on the EC2 instance

---

## Setup Instructions

### 1. **Create S3 Bucket**
   - Create an S3 bucket (e.g., `password-cracking-results`) for storing cracked passwords.
   - Enable **encryption** for stored objects.

### 2. **Create IAM Role**
   - Attach the following managed policies:
     - `AmazonS3FullAccess` (restricted to your bucket via inline policy)
     - `AmazonSNSFullAccess`
   - Add an inline policy to restrict S3 access:
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": "s3:*",
           "Resource": [
             "arn:aws:s3:::password-cracking-results",
             "arn:aws:s3:::password-cracking-results/*"
           ]
         }
       ]
     }
     ```

### 3. **Launch EC2 Instance**
   - Use an Ubuntu AMI with GPU support (e.g., `p3.2xlarge`).
   - Attach the IAM role created in Step 2.
   - Allocate at least 20 GB of storage and allow SSH access only from your IP.

### 4. **Install Necessary Tools**
   - Update and install tools on the EC2 instance:
     ```bash
     sudo apt update && sudo apt upgrade -y
     sudo apt install awscli hashcat secure-delete -y
     sudo apt install nvidia-cuda-toolkit -y
     ```

### 5. **Setup Automation Script**
   - Create the automation script (`automate.sh`):
     ```bash
     #!/bin/bash

     # Define variables
     BUCKET_NAME="password-cracking-results"
     RESULTS_FILE="hashcat.potfile"
     HASH_FILE="/home/ubuntu/hashes.txt"
     LOCAL_RESULTS_PATH="/home/ubuntu/$RESULTS_FILE"
     INSTANCE_ID=$(curl http://169.254.169.254/latest/meta-data/instance-id)

     # Wait for Hashcat to finish
     while pgrep hashcat > /dev/null; do
         echo "Hashcat is still running..."
         sleep 30
     done

     echo "Hashcat has finished. Preparing results for upload."

     # Upload results to S3
     if [[ -f "$LOCAL_RESULTS_PATH" ]]; then
         aws s3 cp "$LOCAL_RESULTS_PATH" "s3://$BUCKET_NAME/$RESULTS_FILE"
         echo "Results uploaded to S3."

         # Securely delete sensitive files
         echo "Securely deleting sensitive data..."
         srm -v "$HASH_FILE" "$LOCAL_RESULTS_PATH"
         echo "All sensitive data securely deleted."

         # Notify Lambda to terminate the instance
         aws sns publish --topic-arn <Your-SNS-Topic-ARN> --message "{\"instance_id\":\"$INSTANCE_ID\"}"
         echo "Notification sent to Lambda for instance termination."
     else
         echo "No results found to upload."
     fi
     ```
   - Make the script executable:
     ```bash
     chmod +x automate.sh
     ```

### 6. **Run Hashcat with Automation**
   - Run Hashcat and execute the automation script:
     ```bash
     nohup hashcat -m <hash-type> -a 0 hashes.txt wordlist.txt & ./automate.sh &
     ```

### 7. **Configure Lambda Function**
   - Create an SNS topic (e.g., `TerminateEC2`) and subscribe a Lambda function to it.
   - Lambda function example:
     ```python
     import boto3
     import json

     def lambda_handler(event, context):
         ec2 = boto3.client('ec2')
         message = json.loads(event['Records'][0]['Sns']['Message'])
         instance_id = message['instance_id']

         try:
             ec2.terminate_instances(InstanceIds=[instance_id])
             print(f"Instance {instance_id} terminated successfully.")
         except Exception as e:
             print(f"Error terminating instance {instance_id}: {str(e)}")
     ```

   - Assign a policy to allow Lambda to terminate EC2 instances:
     ```json
     {
       "Version": "2012-10-17",
       "Statement": [
         {
           "Effect": "Allow",
           "Action": "ec2:TerminateInstances",
           "Resource": "*"
         }
       ]
     }
     ```

### 8. **Test and Verify**
   - Ensure the cracked results are uploaded to S3.
   - Confirm sensitive data is securely deleted from the EC2 instance.
   - Verify the EC2 instance is terminated automatically after processing.

---

## Security Considerations
- **IAM Roles:** Use least-privilege policies for both EC2 and Lambda.
- **Data Encryption:** Enable encryption for S3 buckets and EBS volumes.
- **Secure Deletion:** Use tools like `srm` to overwrite sensitive files before deletion.
- **Logging:** Enable AWS CloudTrail for auditing API activities.

---

## Disclaimer
This project is for **educational purposes only**. Ensure you have explicit permission to perform password cracking activities. Unauthorized use is illegal and punishable by law.
