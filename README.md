# Security-Monitoring-System-on-AWS
Building a Security Monitoring System on AWS with CloudTrail, CloudWatch, and SNS


Even though protecting your resources is important in today’s virtual world; you also need to be aware of what is happening with your resources at all times. This case study outlines my experience creating a real-time alerting system on Amazon Web Services (AWS) to track who accessed your secret and notify you via email instantly. The intent of this project was to learn about how AWS monitors resource activity (using CloudTrail, CloudWatch, and Simple Notification Service {SNS}) and create a Pipeline for delivering alerts via email every time a secret key is accessed.

What’s the value of having such an alerting system set up? Let’s say you store sensitive data (such as API keys and passwords) on AWS. If an unauthorized person or process retrieves that secret, you want to find out right away. This project demonstrates a step-by-step approach to achieve that kind of visibility and alerting.

So I built a monitoring system that detects GetSecretValue calls and emails me within minutes.



Tools I Used

AWS Secrets Manager — stored a sensitive secret I wanted to protect

AWS CloudTrail — captured the access event (audit trail)

Amazon CloudWatch Logs + Metric Filters — turned that access event into a metric

CloudWatch Alarms — triggered alert logic

Amazon SNS — delivered email notifications

IAM Roles — allowed services talk to each other securely

S3 Bucket — long-term CloudTrail log storage




Secrets Manager
     |
GetSecretValue event
     |
CloudTrail (captures API call)
     |
CloudWatch Logs
     |
Metric Filter (counts access)
     |
CloudWatch Alarm
     |
SNS Topic
     |
Email notification






## Step 1: Securing Sensitive Data with AWS Secrets Manager

Before monitoring anything, I needed a sensitive piece of data to protect. AWS Secrets Manager is the go-to service for securely storing things like database passwords, API keys, or any information that should not be exposed in plain text. It ensures that secrets are encrypted and accessible only to authorized services or users. In other words, if something would cause trouble if leaked, Secrets Manager is a safe vault for it.

For this project, I created a new secret in AWS Secrets Manager (let's call it TopSecretInfo). Since this was a demo, I didn’t use a real password or key; instead, I stored a fun "hot take" string as the secret value – essentially a personal note that “I need 3 coffees a day to function.” ☕️ This gave me a harmless test secret to work with, and it’s something I definitely want to keep secure (and maybe a bit private!).



<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/b202d3a9-a076-4ca9-976f-98db0648d1f4" />





<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/fce10a5c-1233-40f5-a094-7992130a8e4b" />




By setting up this secret, I established the resource to monitor. The plan was that whenever someone (or something) retrieves TopSecretInfo from Secrets Manager, I should get an alert about it. Next, I needed to enable tracking of that access event.

## Step 2: Tracking Access with AWS CloudTrail

AWS CloudTrail serves as your AWS account's audit log by recording every activity that occurs. CloudTrail functions as both a security camera and an audit log, as it provides a record of who performed what actions and when. CloudTrail is useful for not only providing security through the detection of suspicious activities, but is also used for evidence of compliance (showing that rules and policies are being followed), and it can be used to resolve issues (figure out what has changed when something is malfunctioning).

To log all access to the Secret in Secrets Manager, I created a CloudTrail trail specifically to track access to Secrets in Secret Manager. Below are the steps to create a CloudTrail Trail for Secrets Manager:



<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/fce10a5c-1233-40f5-a094-7992130a8e4b" />





<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/55b1dee6-99eb-4c75-97cd-61218c2a0da3" />





<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/a9a6f3ae-cd9e-4750-8da6-1215391e3eaa" />





<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/acd7d378-6776-49c5-88c0-1e1e3f1642db" />


