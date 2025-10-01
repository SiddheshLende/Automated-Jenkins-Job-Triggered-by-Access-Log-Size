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


