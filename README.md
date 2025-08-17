# Automated-Jenkins-Job-Triggered-by-Access-Log-Size
# Automated Log Management with Jenkins, Cron, and AWS S3

> This project demonstrates an automated system for monitoring log file sizes. When a log exceeds a defined threshold (1GB), a cron job triggers a Jenkins pipeline to securely upload the log to an S3 bucket for archival and then clears the local file to free up disk space.

The system provides an end-to-end, hands-off solution for managing log rotation and storage, complete with email notifications.

---

### Table of Contents
- [Features](#features)
- [How It Works](#how-it-works)
- [Tools and Technologies](#tools-and-technologies)
- [Configuration Overview](#configuration-overview)
- [Key Scripts](#key-scripts)
- [Demonstration](#demonstration)

---

### Features

* **Automated Monitoring**: A cron job runs a shell script every five minutes to check the log file size.
* **Conditional Triggering**: The Jenkins job is only triggered when the log file size exceeds a 1GB threshold.
* **Secure Jenkins Integration**: Uses a Jenkins API token for secure, password-less remote job triggering.
* **S3 Archival**: The Jenkins job uploads a timestamped copy of the log file to a designated S3 bucket.
* **Automatic Log Clearing**: After a successful upload, the original log file is cleared to prevent disk space issues.
* **Email Notifications**: A post-build action in Jenkins sends an email notification with the job's status.

---

### How It Works

1.  **Monitoring**: A Bash script (`log_size_monitor.sh`) is executed every five minutes by a cron job.
2.  **Size Check**: The script checks the size of the Nginx access log (`/var/log/nginx/access.log`).
3.  **Triggering**: If the file size is over 1GB, the script makes a remote API call to Jenkins using `curl` and a secure API token.
4.  **Jenkins Job Execution**: The `log-upload-job` in Jenkins is triggered. It receives the path to the log file as a parameter.
5.  **S3 Upload**: The job executes a shell script that uses the AWS CLI to upload a timestamped copy of the log file to an S3 bucket.
6.  **Log Clearing**: Upon a successful S3 upload, the script clears the original log file on the server.
7.  **Notification**: Finally, a post-build action sends an email notification summarizing the build's success or failure.

---

### Tools and Technologies

* **CI/CD**: Jenkins
* **Cloud Storage**: Amazon S3
* **Scripting**: Bash (Shell Scripting)
* **Scheduling**: Crontab
* **CLI Tools**: AWS CLI, curl
* **Notifications**: Jenkins Extended Email Notification Plugin (with Gmail SMTP)
* **Log Source**: Nginx

---

### Configuration Overview

1.  **Jenkins Setup**: Jenkins is installed on an Ubuntu server, with the "Extended Email Notification" plugin.
2.  **AWS CLI Configuration**: The AWS CLI is installed and configured specifically for the `jenkins` system user to grant it permissions to access S3.
3.  **Jenkins Job**: A Freestyle project named `log-upload-job` is created. It's parameterized to accept a `log_file_path` and contains the shell script for the upload and clear logic.
4.  **Monitoring Script**: The `log_size_monitor.sh` script is placed on the server, containing the logic to check the file size and trigger Jenkins.
5.  **Scheduling**: A cron job is set up to run the monitoring script at a regular interval.

---

### Key Scripts

#### 1. Log Monitoring Script (`log_size_monitor.sh`)
This script runs on the server to check the log file size and trigger the Jenkins job.
```bash
#!/bin/bash
# === CONFIGURABLE VALUES ===
LOG_FILE="/var/log/nginx/access.log"
MAX_SIZE=$((1024 * 1024 * 1024)) # 1GB in bytes
JENKINS_URL="http://YOUR_JENKINS_IP:8080"
JENKINS_USER="YourJenkinsUser"
JENKINS_API_TOKEN="YourGeneratedApiToken"
JENKINS_JOB="log-upload-job"
LOG_MONITOR_LOG="/home/ubuntu/jenkins_log_monitor.log"

# === CHECK FILE SIZE ===
CURRENT_SIZE=$(stat -c %s "$LOG_FILE")
if [ "$CURRENT_SIZE" -gt "$MAX_SIZE" ]; then
    echo "$(date) - File size > 1GB. Triggering Jenkins job..." >> "$LOG_MONITOR_LOG"
    curl -X POST "$JENKINS_URL/job/$JENKINS_JOB/buildWithParameters" \
    --user "$JENKINS_USER:$JENKINS_API_TOKEN" \
    --data-urlencode "log_file_path=$LOG_FILE"
else
    echo "$(date) - File is below threshold. No action needed." >> "$LOG_MONITOR_LOG"
fi
```

#### 2. Jenkins Job - "Execute Shell" Script
This script runs inside the Jenkins job to perform the upload and clearing operations.
```bash
#!/bin/bash
LOG_FILE="$log_file_path"
FILE_NAME=$(basename "$LOG_FILE")
S3_BUCKET="log-file-backups1"
TIMESTAMP=$(date +"%Y%m%d-%H%M%S")
S3_PATH="s3://$S3_BUCKET/$FILE_NAME-$TIMESTAMP"

echo "Uploading $LOG_FILE to $S3_PATH"
aws s3 cp "$LOG_FILE" "$S3_PATH"

if [ $? -eq 0 ]; then
    echo "✅ Upload successful. Clearing original log file..."
    : > "$LOG_FILE"
else
    echo "❌ Upload failed. Log file NOT cleared." >&2
    exit 1
fi
```

---

### Demonstration

The final result is a fully automated log rotation system.
* **Jenkins Build**: The `log-upload-job` is triggered and completes successfully.
* **S3 Bucket**: The timestamped log file appears in the S3 bucket.
* **Log File Cleared**: The size of the original `access.log` on the server is reset.
* **Email Notification**: A confirmation email is sent with the build status.
