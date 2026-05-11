# AWS Cloud Architecture — Portfolio Project 1

A progressive, hands-on AWS architecture project built to demonstrate real cloud operations skills: provisioning, access control, storage management, cost optimization, monitoring, and fault diagnosis. Each phase intentionally introduces a misconfiguration that is traced and resolved using AWS-native logging tools.

**Status:** Phases 1–2 complete | Phases 3–7 in progress  
**Services used so far:** EC2, IAM, S3, CloudWatch, SNS, AWS Budgets, CloudTrail  
**Target roles:** Cloud Support Associate · Junior Systems Administrator · Cloud Operations Specialist

---

## Architecture Overview

> Diagram will be updated as each phase is completed.

---

## Phase 1 — EC2 + CloudWatch + SNS + Billing Alarm

**Services:** EC2, IAM, CloudWatch, SNS, AWS Budgets

### What I built
- Configured a billing budget alarm before provisioning any resources — cost controls first, infrastructure second
- Created a scoped IAM admin user following least-privilege principles, avoiding root account usage for day-to-day work
- Deployed a live web application on an EC2 t3.micro instance, selected to minimize compute cost while meeting workload requirements
- Created an SNS topic with a confirmed email subscription to route alarm notifications
- Configured CloudWatch alarms tied to the SNS topic — when an alarm fires, an email is delivered automatically

### What I broke and how I found it
Stopped the application process on the EC2 instance to simulate a service outage. Watched the CloudWatch alarm transition from OK to ALARM state, received the SNS email notification, and confirmed the failure in CloudWatch Logs. This validated the full monitoring pipeline: failure → metric breach → alarm → notification.

### What I learned
- Stopped EC2 instances report missing data rather than a failed status check — CloudWatch alarms need to be configured to treat missing data as breaching, otherwise a powered-off instance shows as healthy
- SNS email subscriptions require confirmation before they deliver — an unconfirmed subscription silently drops notifications
- Billing alarms live in us-east-1 regardless of where your resources are deployed; CloudWatch must be set to that region to configure them

---

## Phase 2 — S3 + IAM + Lifecycle + Logging

**Services:** S3, IAM, CloudTrail, S3 Server Access Logs

### What I built
- Created a private S3 bucket with versioning enabled, SSE-S3 encryption, and all public access blocked from the start
- Demonstrated object recovery by simulating the most common Cloud Support ticket: a deleted file. Removed a delete marker to restore the previous version without any backup system
- Configured a lifecycle policy that automates storage cost optimization:
  - Days 0–30: S3 Standard (active access)
  - Days 30–90: S3 Standard-IA (infrequent access)
  - Days 90–365: S3 Glacier Flexible Retrieval (archival)
  - Day 365+: Automatic expiration
- Applied least-privilege access using two independent layers: an IAM policy scoped to the user identity, and a bucket policy scoped to the resource — both must allow the action for access to succeed
- Created a dedicated logging bucket and enabled S3 server access logging (object-level HTTP request activity) and a CloudTrail trail (control-plane API activity) for full audit coverage

### What I broke and how I found it
Applied a bucket policy with `"Effect": "Deny", "Principal": "*", "Action": "s3:*"` — a deny-all that locked out every principal in the account, including my own admin user. Confirmed the break by attempting to access the Objects tab and receiving an Access Denied error.

Traced the misconfiguration in CloudTrail Event History by filtering on the bucket name and locating the `PutBucketPolicy` event. The full JSON of the deny policy was visible in the event record, showing exactly when the change was made and under which identity.

Recovery required signing in as the root user — the only principal that retains bucket policy management access regardless of resource-based deny policies. Replaced the deny-all with the original scoped allow policy and confirmed access was restored.

### What I learned
- S3 versioning does not prevent deletion — it prevents permanent deletion. Delete markers make objects invisible without destroying the underlying versions
- S3 has two separate logging systems for two separate audiences: server access logs capture object-level request activity (ops visibility), CloudTrail captures API-level control plane activity (security auditing)
- `"Principal": "*"` with `"Effect": "Deny"` is an account-wide lockout. A safe deny policy always includes a condition that excludes the account root or a specific trusted principal
- IAM policies and bucket policies are evaluated independently — both must allow, and either can deny

---

## Phases 3–7 — In Progress

| Phase | Focus | Status |
|-------|-------|--------|
| 3 | VPC + Networking (subnets, route tables, IGW, NAT) | Upcoming |
| 4 | ALB + RDS Multi-AZ | Upcoming |
| 5 | Auto Scaling Group | Upcoming |
| 6 | Lambda + SQS (serverless + decoupling) | Upcoming |
| 7 | CloudFront + Route 53 + ACM (optional polish) | Upcoming |

---

## Cost Controls

A billing budget alarm was configured in Phase 1 before any resources were provisioned. All compute uses t3.micro (Free Tier eligible). Resources not in active use are stopped to avoid unnecessary charges.
