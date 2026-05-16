# AWS Lambda Word Count Lab 📬

A serverless pipeline that automatically counts words in a text file uploaded to S3 and sends the result via email using SNS.

---

## Architecture Overview

```
S3 Bucket  →  triggers  →  Lambda Function  →  publishes to  →  SNS Topic  →  Email
```

---

## Objectives

- Create a Lambda function in Python to count words in a text file
- Configure an S3 bucket to automatically invoke Lambda on file upload
- Create an SNS topic to deliver the word count result via email

---

## AWS Services Used

| Service | Purpose |
|---|---|
| AWS Lambda | Serverless function to process the file and count words |
| Amazon S3 | Storage bucket — triggers Lambda on file upload |
| Amazon SNS | Sends the word count result as an email notification |
| Amazon CloudWatch | Logging and monitoring Lambda executions |
| AWS IAM | `LambdaAccessRole` — grants Lambda access to S3, SNS, CloudWatch |

---

## Step-by-Step Setup

### Step 1: Create an SNS Topic & Email Subscription

1. Go to **SNS → Topics → Create topic**
2. Type: **Standard**, Name: `WordCountTopic`
3. Click **Create topic**
    <img width="1366" height="768" alt="Screenshot (2)" src="https://github.com/user-attachments/assets/fcea18d2-4a57-493c-b05e-2916eae87f59" />
4. Click **Create subscription**
   - Protocol: `Email`
   - Endpoint: your email address
5. **Confirm the subscription** by clicking the link AWS sends to your email
   <img width="1366" height="768" alt="Screenshot (4)" src="https://github.com/user-attachments/assets/326db824-e068-4bf0-b157-4db896324354" />


> ⚠️ Subscription must show **Confirmed** status before emails will be delivered

---

### Step 2: Create an S3 Bucket

1. Go to **S3 → Create bucket**
   <img width="1333" height="492" alt="image" src="https://github.com/user-attachments/assets/5ffb20a8-d1f9-47aa-87b6-063e48af2c6b" />

2. Give it a unique name e.g. `wordcount-bucket-yourname`
3. Select the **same region** as your SNS topic
4. Leave defaults and click **Create bucket**
   <img width="1366" height="768" alt="Screenshot (7)" src="https://github.com/user-attachments/assets/714d60dc-108c-42ef-a864-e04f0ccdea6a" />


---

### Step 3: Create the Lambda Function

1. Go to **Lambda → Create function → Author from scratch**
   <img width="961" height="208" alt="image" src="https://github.com/user-attachments/assets/e2310f22-d527-4b5c-9f5e-10516cc090f0" />

2. Settings:
   - **Name:** `WordCountFunction`
   - **Runtime:** Python 3.12
   - **Execution role:** Use existing role → `LambdaAccessRole`
3. Click **Create function**
   <img width="1366" height="768" alt="Screenshot (10)" src="https://github.com/user-attachments/assets/8f84791c-d197-45cb-8688-a5d993ce23ac" />


---

### Step 4: Lambda Function Code

```python
import boto3
import urllib.parse

def lambda_handler(event, context):

    SNS_TOPIC_ARN = "arn:aws:sns:REGION:ACCOUNT-ID:WordCountTopic"  # Replace with your ARN

    s3_client = boto3.client('s3')
    sns_client = boto3.client('sns')

    # Get file details from the S3 trigger event
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    file_name = urllib.parse.unquote_plus(
        event['Records'][0]['s3']['object']['key']
    )

    # Read the file content
    response = s3_client.get_object(Bucket=bucket_name, Key=file_name)
    file_content = response['Body'].read().decode('utf-8')

    # Count the words
    word_count = len(file_content.split())

    # Build the message
    message = f"The word count in the {file_name} file is {word_count}."

    # Send via SNS
    sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="Word Count Result",
        Message=message
    )

    return {
        'statusCode': 200,
        'body': message
    }
```

> ⚠️ Replace `SNS_TOPIC_ARN` with your actual Topic ARN copied from **SNS → Topics**

After pasting the code, click **Deploy**.
<img width="1366" height="768" alt="Screenshot (11)" src="https://github.com/user-attachments/assets/59c45b00-067b-4791-967e-44a7d90c4532" />

---

### Step 5: Add the S3 Trigger

1. On the Lambda function page, click **+ Add trigger**
3. Select **S3**
4. Choose your bucket
   <img width="1373" height="768" alt="image" src="https://github.com/user-attachments/assets/4362e711-5b03-4564-9f36-f70630568bc5" />

5. Event type: **PUT** (or All object create events)
6. Optional suffix filter: `.txt`
   <img width="1373" height="768" alt="image" src="https://github.com/user-attachments/assets/367d5cb0-8fd8-465b-bffa-e05d4a8b55f6" />

7. Click **Add**
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/07e40299-ce25-4dbd-ab2e-023c517ea601" />

---

### Step 6: Test

1. Create a plain `.txt` file on your computer with some text
2. Upload it to your S3 bucket
3. Wait ~30 seconds and check your email
   <img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/4b42c636-dd1e-4d46-868b-763deb190886" />


**Expected email:**
```
Subject: Word Count Result
The word count in the sample.txt file is 3.
<img width="1366" height="768" alt="image" src="https://github.com/user-attachments/assets/7ce40084-5794-4e97-97bf-e673c4984530" />

```

---

## Checking Logs (CloudWatch)

### From Lambda Console
1. Lambda → **Monitor tab** → **View CloudWatch Logs**
2. Click the latest **Log Stream**
3. Expand entries to read output

### From CloudWatch Directly
1. CloudWatch → **Logs → Log groups**
2. Find `/aws/lambda/WordCountFunction`
3. Open the latest log stream

### Useful Log Entries

| Entry | Meaning |
|---|---|
| `START RequestId` | Function was triggered |
| `END RequestId` | Function completed |
| `REPORT Duration` | Execution time |
| `errorMessage` | Something failed |

### Add Debug Print Statements

```python
print(f"Bucket: {bucket_name}")
print(f"File: {file_name}")
print(f"Word count: {word_count}")
```

These appear directly in CloudWatch logs.

---

## Problems Faced & How They Were Fixed

---

### ❌ Problem 1: Syntax Error — Missing Comma

**Error:**
```
[ERROR] Runtime.UserCodeSyntaxError: Syntax error in module 'lambda_function':
invalid syntax. Perhaps you forgot a comma? (lambda_function.py, line 23)
```

**Cause:**
Missing comma between parameters in the `sns_client.publish()` call.

**Broken code:**
```python
sns_client.publish(
    TopicArn=SNS_TOPIC_ARN
    Subject="Word Count Result"   # ← missing comma on line above
    Message=message
)
```

**Fix:**
```python
sns_client.publish(
    TopicArn=SNS_TOPIC_ARN,       # ← added comma
    Subject="Word Count Result",  # ← added comma
    Message=message
)
```

---

### ❌ Problem 2: Invalid Parameter — TopicArn (Placeholder Not Replaced)

**Error:**
```
[ERROR] InvalidParameterException: An error occurred (InvalidParameter)
when calling the Publish operation: Invalid parameter: TopicArn
```

**Cause:**
The placeholder ARN `arn:aws:sns:us-east-1:123456789012:WordCountTopic` was never replaced with the real ARN.

**Fix:**
1. Go to **SNS → Topics → WordCountTopic**
2. Copy the real ARN from the top of the page
3. Paste it into the Lambda code replacing the placeholder
4. Redeploy

---

### ❌ Problem 3: Invalid Parameter — Topic Name (Subscription ARN Used Instead of Topic ARN)

**Error:**
```
[ERROR] InvalidParameterException: An error occurred (InvalidParameter)
when calling the Publish operation: Invalid parameter: Topic Name
```

**Cause:**
The ARN copied was a **Subscription ARN** (from SNS → Subscriptions page), not a **Topic ARN**. Subscription ARNs have an extra UUID at the end.

**Wrong ARN (Subscription ARN):**
```
arn:aws:sns:us-west-2:897651944959:WordCountTopic:35820263-27c3-4f0d-acc6-93c5ff826e43
```

**Correct ARN (Topic ARN):**
```
arn:aws:sns:us-west-2:897651944959:WordCountTopic
```

**Fix:**
Remove everything after and including the last colon+UUID segment.
Always copy the ARN from **SNS → Topics**, not from **SNS → Subscriptions**.

---

## Key Lessons Learned

- Always create all resources in the **same AWS region**
- SNS subscription must be **Confirmed** before emails are delivered — check spam folder for the confirmation email
- Always copy the **Topic ARN** from SNS → Topics, never from SNS → Subscriptions
- Python function parameters need **commas** between them — easy to miss
- **CloudWatch Logs** are your best friend for debugging Lambda errors
- Use `print()` statements in Lambda code to log values for easier debugging

---

## IAM Role Used

**Role Name:** `LambdaAccessRole`

| Policy | Purpose |
|---|---|
| `AWSLambdaBasicExecutionRole` | Write logs to CloudWatch |
| `AmazonSNSFullAccess` | Publish messages to SNS topics |
| `AmazonS3FullAccess` | Read files from S3 buckets |
| `CloudWatchFullAccess` | Full CloudWatch monitoring access |

---

## Final Result

✅ Text file uploaded to S3  
✅ Lambda triggered automatically  
✅ Word count calculated  
✅ Email received via SNS  

```
Subject: Word Count Result
The word count in the sample.txt file is 42.
```
