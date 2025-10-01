# Automated-Jenkins-Job-Triggered-by-Access-Log-Size
# Objective
Create a system where a Jenkins job is triggered automatically when an Nginx access log file exceeds 1GB. Once triggered, the Jenkins job must:  
1. Upload the log file to a specific Amazon S3 bucket.  
2. Clear the contents of the original log file after successful transfer.

# Step 1: Attach IAM Role to EC2 (for S3 access)
1. Go to AWS IAM → Roles → Create Role  
2. Choose AWS service → EC2 → Next  
3. Attach AmazonS3FullAccess policy → Next  
4. Name the role EC2_S3_Upload_Role → Create role  
5. Attach the role to your EC2 instance:  
6. Verify role on EC2:
   curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Step 2: Prepare Access Log Permissions  
sudo usermod -aG adm jenkins  
sudo chmod 664 /var/log/nginx/access.log  
sudo chown www-data:adm /var/log/nginx/access.log  
sudo systemctl restart jenkins  

# Step 3: Jenkins Pipeline (UploadLogs Job)  
Jenkinsfile:  
pipeline {
    agent any

    environment {
        LOG_FILE = "/var/log/nginx/access.log"
        S3_BUCKET = "my-log-storage-bucket-321"
        BACKUP_NAME = "access-${env.BUILD_ID}.log"
    }

    stages {
        stage('Upload to S3') {
            steps {
                sh '''
                    echo "Uploading $LOG_FILE to S3..."
                    aws s3 cp $LOG_FILE s3://$S3_BUCKET/$BACKUP_NAME
                '''
            }
        }

        stage('Clear Log File') {
            steps {
                sh '''
                    echo "Clearing log file..."
                    > $LOG_FILE
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Log file successfully uploaded and cleared."
        }
        failure {
            echo "❌ Job failed. Log file not cleared."
        }
    }
}  

# Step 4: Cron Script to Trigger Jenkins  
#!/bin/bash

LOG_FILE="/var/log/nginx/access.log"
MAX_SIZE=$((1024*1024*1024))   # 1GB
JENKINS_URL="http://localhost:8080/job/UploadLogs/build"
JENKINS_USER="admin"
JENKINS_TOKEN="your_jenkins_api_token"  # Replace with your Jenkins token

FILE_SIZE=$(stat -c%s "$LOG_FILE")

if [ "$FILE_SIZE" -ge "$MAX_SIZE" ]; then
    echo "$(date): Log file exceeded 1GB. Triggering Jenkins job..."
    curl -u $JENKINS_USER:$JENKINS_TOKEN -X POST $JENKINS_URL
else
    echo "$(date): Log file size under 1GB. No action needed."
fi  

# Make it executable:  
chmod +x /home/ubuntu/log_monitor.sh  

# Step 5: Schedule Cron Job  
crontab -e  
Add the following line:  
*/5 * * * * /bin/bash /home/ubuntu/log_monitor.sh >> /home/ubuntu/log_monitor.log 2>&1  

# Step 6: Testing  
Generate test logs:  
sudo dd if=/dev/zero bs=1M count=10 | sudo tee -a /var/log/nginx/access.log  
Run Jenkins pipeline manually → check console output:  
Uploading /var/log/nginx/access.log to S3...  
Clearing log file...  
✅ Log file successfully uploaded and cleared.  

Verify in S3:  
aws s3 ls s3://my-log-storage-bucket-321/  
