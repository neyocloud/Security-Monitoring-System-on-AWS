



<img width="556" height="171" alt="image" src="https://github.com/user-attachments/assets/4328db79-a836-42ac-9777-2a3ee8408cd8" />






# Security-Monitoring-System-on-AWS
Building a Security Monitoring System on AWS with CloudTrail, CloudWatch, and SNS


Even though protecting your resources is important in today’s virtual world; you also need to be aware of what is happening with your resources at all times. This case study outlines my experience creating a real-time alerting system on Amazon Web Services (AWS) to track who accessed my secret and notify me via email instantly. The intent of this project was to learn about how AWS monitors resource activity (using CloudTrail, CloudWatch, and Simple Notification Service {SNS}) and create a Pipeline for delivering alerts via email every time a secret key is accessed.

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

For this project, I created a new secret in AWS Secrets Manager (let's call it myfilee/path/secret). Since this was a demo, I didn’t use a real password or key; instead, I stored a fun "hot take" string as the secret value – essentially a personal note that “SonarQube security + code quality gate
”  This gave me a harmless test secret to work with, and it’s something I definitely want to keep secure (and maybe a bit private!).



<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/b202d3a9-a076-4ca9-976f-98db0648d1f4" />





<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/fce10a5c-1233-40f5-a094-7992130a8e4b" />




By setting up this secret, I established the resource to monitor. The plan was that whenever someone (or something) retrieves myfile/path/secret from Secrets Manager, I should get an alert about it. Next, I needed to enable tracking of that access event.

## Step 2: Tracking Access with AWS CloudTrail

AWS CloudTrail serves as my AWS account's audit log by recording every activity that occurs. CloudTrail functions as both a security camera and an audit log, as it provides a record of who performed what actions and when. CloudTrail is useful for not only providing security through the detection of suspicious activities, but is also used for evidence of compliance (showing that rules and policies are being followed), and it can be used to resolve issues (figure out what has changed when something is malfunctioning).

To log all access to the Secret in Secrets Manager, I created a CloudTrail trail specifically to track access to Secrets in Secret Manager. Below are the steps to create a CloudTrail Trail for Secrets Manager:




<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/55b1dee6-99eb-4c75-97cd-61218c2a0da3" />


Trail Configuration: I created a new CloudTrail trail and enabled logging of management events in my AWS account. CloudTrail categorizes events into a few types: Management events (control plane actions like creating or modifying AWS resources), Data events (high-volume data access like S3 object reads/writes or Lambda invokes), Insight events, and Network events. Accessing a secret (via the Secrets Manager API) counts as a Management event – essentially an API call to AWS to retrieve a secret value. I chose to track management events because that covers the security-critical actions (and bonus: AWS lets you log management events at no charge).

Read vs. Write API Activity: Within management events, CloudTrail lets you specify whether to log Read events, Write events, or both.

Read events are actions that retrieve or view data (e.g. reading a secret, listing resources, getting configurations).

Write events are actions that change or create data (e.g. updating a secret, deleting a resource).
For learning purposes, I enabled logging for both read and write management events on my trail. In practice, retrieving a secret’s value is considered a Write management event in CloudTrail (likely because it’s a sensitive action, even though it’s a read operation in a literal sense). By logging both, I ensured I wouldn’t miss the secret access event. (Side note: It was interesting to discover that GetSecretValue is classified as a write API call due to its importance!).

Destination for Logs: I configured CloudTrail to deliver logs to an S3 bucket (for long-term storage) and also to a CloudWatch Logs group (for real-time monitoring). The S3 bucket is useful for retention and forensic analysis, but the CloudWatch Logs integration is what will allow me to set up alerts in the next steps.

At this point, CloudTrail was all set. It would record an entry whenever my secret gets accessed. But I wanted to double-check that it works before moving on to building the alerting mechanism.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/76051e10-3590-4970-ab79-e72298a195ee" />





## Step 3: Verifying CloudTrail Events for Secret Access



After configuring CloudTrail, I performed a quick test to verify that secret access events are indeed being logged. I accessed the secret (myfile/path/secret) in two different ways:

Via the AWS Management Console: I went into the Secrets Manager console, found my secret, and clicked the “Retrieve secret value” button to view its contents. This simulates an administrator or user visually checking the secret in the web interface.

Via the AWS CLI: I used my mac terminal to run the command aws secretsmanager get-secret-value --secret-id myfile/path/secret. This simulates an application or script programmatically retrieving the secret via AWS APIs.




<img width="1512" height="982" alt="image" src="https://github.com/user-attachments/assets/79ff24c8-5dfb-402b-a1bc-5eb0eab83412" />





In both cases, I was effectively reading the secret value. Now, the moment of truth: I headed to the CloudTrail Event History in the AWS console to see if these actions were recorded. Sure enough, I found entries for GetSecretValue events corresponding to my secret retrievals. CloudTrail logged the details of each access, including who made the call, the time it occurred, the API used, etc. It didn’t matter whether I used the console or the CLI – CloudTrail captured both.



This validation demonstrated that CloudTrail was working properly, capturing every interaction with the secret and recording it. In practice, this tells me that if somebody ever retrieves the secret, there will be an entry created in the trail. However, having the trail itself was not sufficient; I needed to be notified of the event immediately. Thus, I began looking into CloudWatch for any potential solutions.





Step 4: Turning CloudTrail Logs into Alerts with CloudWatch Logs & Metrics

Raw logs are like needles in a haystack – they’re only useful if we can analyze them or trigger actions based on them. This is where Amazon CloudWatch comes in. CloudWatch can ingest logs from various AWS services (including CloudTrail) and allows you to filter, search, and build metrics out of those logs. By creating a metric from a log, we can then set alarms on that metric.

Why not just rely on CloudTrail’s event history? CloudTrail’s console view is great for manual inspection and recent events (it retains 90 days of management events for quick access), but it doesn’t actively alert you. CloudWatch Logs lets us watch the stream of events continuously and do something when a specific event appears. It’s also better for long-term analysis and combining logs from multiple sources if needed.

Here’s what I did in CloudWatch:

Log Group Setup: Since I configured CloudTrail to send logs to CloudWatch, a Log Group was receiving a continuous feed of CloudTrail logs (JSON entries of each event). I navigated to the CloudWatch Log Groups section and found the log group for my CloudTrail trail.

Creating a Metric Filter: Within that log group, I created a Metric Filter to look for the specific event of interest. A metric filter in CloudWatch is essentially a rule that scans log entries for a pattern and updates a numeric metric when the pattern is found. In my case, I set the filter pattern to trigger on any log entries where the event name is GetSecretValue (and optionally where the resource name equals myfile/path/secret, to be very specific). Whenever this pattern is matched, the filter will increment a custom CloudWatch metric I defined – let's call it “SecretAccessCount”.




<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/08d80a40-29b3-441a-8046-96a799ba4ac1" />





I configured the metric filter so that each occurrence of a secret access increments the metric by 1. This means if the secret is accessed twice, the metric value would go up by 2, etc.

I also set a default value of 0 for periods with no matching log events (so if no one accesses the secret during a given time window, the metric reports 0 for that period). This prevents any false alarms when nothing is happening.

Why use a metric? CloudWatch Alarms (next step) work on metrics, not on raw logs. By translating “secret accessed” events into a metric, I can easily define threshold-based alerts. The metric essentially counts how many times the secret was accessed over time, which is exactly what I care about for triggering an alarm.

At this stage, I had CloudTrail feeding logs to CloudWatch, and a metric counting the secret access events. Now it was time to set up an alarm to email me when that metric shows unwanted activity.


<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/5b6e1d53-8794-4fba-8f17-3783f86443dd" />







## Step 5 — Create an Alarm + SNS Topic

The metric alone isn’t useful without alerting.

I built:

A CloudWatch alarm watching the metric

Trigger condition:
Sum ≥ 1 in a 5-minute window
(so even one access fires the alarm)

Then I created an SNS topic called something like SecretAccessAlerts, subscribed my email, and confirmed the subscription.

<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/dbfc767f-db76-4ac4-816e-e4e6c176ca3b" />




## Step 6: Testing the Alert and Troubleshooting Issues

I proceeded to simulate a security incident (or just a normal access) to test whether the whole pipeline worked. I went ahead and retrieved the secret one more time (just as I did when verifying CloudTrail earlier). Then I waited for the magic to happen... and waited... and waited. Five minutes passed, and no email arrived in my inbox. 😟

This was obviously not the outcome I hoped for. It meant something in the chain wasn’t working as expected. There were no immediate error messages or obvious failures in AWS, which made troubleshooting a bit challenging. I had to systematically investigate each part of the system to find the misconfiguration. Here’s how I approached the debugging:

CloudTrail Logs: First, I checked CloudTrail’s Event History again to confirm that the new secret access event was indeed recorded. It was. CloudTrail showed the latest GetSecretValue event. So CloudTrail did its job.

CloudWatch Log Ingestion: Next, I went to the CloudWatch Logs for the trail. I navigated into the log group and looked at the latest log stream. I found the entry for the GetSecretValue event there as well. This told me CloudTrail successfully delivered the log to CloudWatch.

Metric Filter: I then inspected the metric filter I set up. I looked at the CloudWatch Metrics section to see if my custom metric SecretAccessCount had registered any data points for the time when I accessed the secret. If configured right, it should have a value of 1 around that timeframe. Here, I suspected the issue might be – and indeed, I realized that the metric showed an update, but the way the alarm was evaluating it might be the culprit.

Alarm State: I checked the CloudWatch Alarm’s status and history. It was still in the OK state and did not show any alarm-triggered events. The alarm’s rule was looking for an average > 1, and apparently it never found that condition true. If my metric only ever hit 1 access in the period, the average was 1, which is not greater than 1 (it would need to be 1.x or more to breach a strict “greater than 1” condition). In hindsight, this was a mistake in how I set the threshold logic.

SNS and Email: Since the alarm never triggered, SNS never got a message, hence no email. I double-checked that my email subscription was confirmed and that there were no emails caught in spam. All good there – the issue clearly lay with the alarm logic.

After reviewing each component, I discovered the root cause: I had configured the CloudWatch alarm incorrectly. I used the “Average” statistic with a threshold of 1, instead of using “Sum”. This meant the alarm expected more than one event (average > 1 in 5 minutes) to trigger. In reality, I wanted it to trigger on even a single event. The fix was straightforward: adjust the alarm to use the Sum of events in the 5-minute period and trigger if the sum ≥ 1.

This was a classic case of a small configuration detail causing the whole system to silently fail. No error was thrown anywhere – the services were all working, just my alarm logic was off. Once I realized this, I updated the alarm settings accordingly.






## Success ✅

I accessed the secret one more time.

This time:

CloudWatch alarm went In ALARM

SNS fired

I received an email in ~1–2 minutes 🎉



<img width="3024" height="1964" alt="image" src="https://github.com/user-attachments/assets/0a51b3f1-faec-4ab7-8d55-e28eb284a0f8" />




That confirmed the full monitoring flow worked end-to-end.

CloudTrail Notifications vs CloudWatch Alarms (What I Learned)

I tried enabling direct SNS notifications from CloudTrail too.





## Result:
My inbox got flooded.

CloudTrail sends notifications for new log deliveries — not for specific actions — so it becomes noisy and hard to interpret fast.

CloudWatch alarms were way better because:

I filtered for exact events

I only got emails that mattered

It felt like real monitoring, not log spam

## Key Takeaways

Logging ≠ Monitoring.
CloudTrail gives full visibility; CloudWatch turns it into targeted alerting.

Small configuration details matter.
One wrong statistic (Average vs Sum) killed the whole workflow silently.

Alert fatigue is real.
CloudTrail notifications can overwhelm you. Filtering first is essential.

This mini setup mirrors real security systems.
It’s the same pattern used for detecting risky actions like:

Root logins

IAM policy changes

Security group exposure

Secret access






