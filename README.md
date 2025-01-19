# Cost Optimization for Jenkins Logs Using Shell Script and AWS S3

<img width="595" alt="image" src="https://github.com/user-attachments/assets/c1452f87-f954-418b-9b7c-362f13606391" />


## Problem Statement

Organization had a **self-hosted ELK stack** (Elasticsearch, Logstash, Kibana) to store logs from applications, Kubernetes, and infrastructure. 
While the ELK stack was critical for log analysis and incident management, the **Jenkins logs** from lower environments (UAT, staging) contributed 
significantly to the overall storage and compute costs.

#### Existing process
```
+-------------------+
|   Jenkins Server  |
+-------------------+
          |
          v
+--------------------------+
|  Generates Logs Daily    |
+--------------------------+
          |
          v
+--------------------------+
| Logs Sent to ELK Stack   |
| (ElasticSearch, Logstash,|
|  Kibana)                 |
+--------------------------+
          |
          v
+--------------------------+
| High Storage Costs in    |
| ELK Stack (Compute +     |
| Storage for Logs)        |
+--------------------------+
          |
          v
+--------------------------+
| Logs Rarely Used for     |
| Analysis (Mostly Backup) |
+--------------------------+
```
### Key Issues:

> [!NOTE]
> Jenkins logs were **rarely analyzed or used for debugging**.
> Logs were stored in the ELK stack as a **backup only**, resulting in **high costs** due to storage and compute.

---

## Solution

To address this, I implemented a **cost optimization strategy**:

### Step 1: Analyze Log Usage
- Confirmed that Jenkins logs from lower environments were not actively used for log analysis.
- Developers relied on Slack and email notifications for debugging build failures.

### Step 2: Migrate Logs to Cost-Efficient Storage
- Suggested migrating Jenkins logs from the ELK stack to **Amazon S3**, a cheaper and scalable storage solution.

### Step 3: Automate the Process
- Developed a **Shell Script** to:
  - Identify new Jenkins logs daily.
  - Upload these logs to an **S3 bucket**.
- Set up **S3 Lifecycle Management** to automatically move older logs to **Glacier** or **Deep Archive**.

#### New process

```
+-------------------+
|   Jenkins Server  |
+-------------------+
          |
          v
+-------------------------+
|  Shell Script (Daily)   |
+-------------------------+
          |
          v
+-------------------+
|  AWS S3 Bucket    |
|  (Standard Tier)  |
+-------------------+
          |
          v
+-----------------------------+
| S3 Lifecycle Management     |
| (Archive Older Logs to      |
| Glacier/Deep Archive)       |
+-----------------------------+
```

---

## Cost Optimization Achieved

By implementing this solution:
- Reduced ELK stack costs by approximately **50%**.
- Logs remained accessible for backup purposes, ensuring operational reliability.
- The process was fully automated, requiring no manual intervention after initial setup.

---

## How to Use This Project

1. **Prerequisites**:
   - AWS account with S3 bucket created.
   - Jenkins server running on an EC2 instance.
   - AWS CLI installed and configured on the EC2 instance.

2. **Shell Script Setup**:
   - Place the shell script (`s3_upload.sh`) in the Jenkins server.
  
```bash

#!/bin/bash

# Author : Mohiadeen
# Description : Script to upload Jenkins logs to S3 bucket for cost optimization

# Variables
JENKINS_HOME="/var/lib/jenkins"  # Path to Jenkins home directory
S3_BUCKET="s3://your-s3-bucket-name"  # S3 bucket name (replace with your bucket)
DATE=$(date +%Y-%m-%d)  # Today's date

# Function to check if AWS CLI is installed
check_aws_cli() {
    if ! command -v aws &> /dev/null; then
        echo "Error: AWS CLI is not installed. Please install it to proceed."
        exit 1
    fi
}

# Function to upload a single log file
upload_log_file() {
    local log_file="$1"
    local s3_path="$2"
    
    aws s3 cp "$log_file" "$s3_path" --only-show-errors
    if [ $? -eq 0 ]; then
        echo "Uploaded: $s3_path"
    else
        echo "Error: Failed to upload $s3_path"
    fi
}

# Function to process logs for a single job
process_job_logs() {
    local job_dir="$1"
    local job_name=$(basename "$job_dir")

    for build_dir in "$job_dir/builds/"*/; do
        local build_number=$(basename "$build_dir")
        local log_file="$build_dir/log"

        # Check if the log file exists and was created today
        if [ -f "$log_file" ] && [ "$(date -r "$log_file" +%Y-%m-%d)" == "$DATE" ]; then
            local s3_path="$S3_BUCKET/$job_name/$build_number.log"
            upload_log_file "$log_file" "$s3_path"
        fi
    done
}

# Function to upload logs to S3
upload_logs() {
    for job_dir in "$JENKINS_HOME/jobs/"*/; do
        process_job_logs "$job_dir"
    done
}

# Main script execution
check_aws_cli
upload_logs

```

   - Grant execution permissions using:
     ```bash
     chmod +x s3_upload.sh
     ```
   - Schedule the script to run daily using a cron job.

3. **S3 Lifecycle Policy**:
   - Configure the S3 bucket lifecycle to move older logs to Glacier or Deep Archive.

---

