# AWS Cloud Architecture — Portfolio Project 1

A progressive, hands-on AWS architecture project built to demonstrate real cloud operations skills: provisioning, access control, storage management, cost optimization, monitoring, and fault diagnosis. Each phase intentionally introduces a misconfiguration that is traced and resolved using AWS-native logging tools.

**Status:** Phases 1–5 complete | Phases 6–7 in progress  
**Services used so far:** EC2, IAM, S3, VPC, ALB, RDS, Auto Scaling, Launch Templates, CloudWatch, SNS, AWS Budgets, CloudTrail, VPC Flow Logs  
**Target roles:** Cloud Support Associate · Junior Systems Administrator · Cloud Operations Specialist · Cloud Engineer · DevOps Engineer

---

## Architecture Overview

![Architecture Diagram](architecture.svg)

> Current state: Phases 1–5 complete. Diagram regenerated at the end of each phase.

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
Stopped the application process on the EC2 instance to simulate a service outage. Watched the CloudWatch alarm transition from OK to ALARM state, received the SNS email notification, and confirmed the failure in the CloudWatch alarm history. This validated the full monitoring pipeline: failure → metric breach → alarm → notification.

![CloudWatch alarm in ALARM state](screenshots/phase1-CloudWatchAlarm.png)
![SNS email notification](screenshots/phase1-SNS-Email.png)

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

![Lifecycle rule configuration](screenshots/phase2-LifecycleRule.png)

### What I broke and how I found it
Applied a bucket policy with `"Effect": "Deny", "Principal": "*", "Action": "s3:*"` — a deny-all that locked out every principal in the account, including my own admin user. Confirmed the break by attempting to access the Objects tab and receiving an Access Denied error.

Traced the misconfiguration in CloudTrail Event History by filtering on the bucket name and locating the `PutBucketPolicy` event. The full JSON of the deny policy was visible in the event record, showing exactly when the change was made and under which identity.

![CloudTrail event record showing the deny policy](screenshots/phase2-EventRecord-Deny.png)

Recovery required signing in as the root user — the only principal that retains bucket policy management access regardless of resource-based deny policies. Replaced the deny-all with the original scoped allow policy and confirmed access was restored.

### What I learned
- S3 versioning does not prevent deletion — it prevents permanent deletion. Delete markers make objects invisible without destroying the underlying versions
- S3 has two separate logging systems for two separate audiences: server access logs capture object-level request activity (ops visibility), CloudTrail captures API-level control plane activity (security auditing)
- `"Principal": "*"` with `"Effect": "Deny"` is an account-wide lockout. A safe deny policy always includes a condition that excludes the account root or a specific trusted principal
- IAM policies and bucket policies are evaluated independently — both must allow, and either can deny

---

## Phase 3 — VPC + Networking

**Services:** VPC, subnets, Internet Gateway, NAT Gateway, route tables, VPC Flow Logs

### What I built
- Created a custom VPC (`10.0.0.0/16`) to replace the default VPC — a flat network with no isolation is not appropriate for production workloads
- Provisioned four subnets across two Availability Zones: two public (`10.0.1.0/24`, `10.0.2.0/24`) and two private (`10.0.3.0/24`, `10.0.4.0/24`). Spreading across AZs means the architecture survives an AZ-level failure
- Created and attached an Internet Gateway, then configured the public route table with a `0.0.0.0/0 → IGW` rule — this route is what makes a subnet public, not its name
- Configured a private route table with no IGW route, ensuring private subnets are unreachable from the internet at the routing level
- Launched a NAT Gateway in `project1-public-1a` with an Elastic IP, then added a `0.0.0.0/0 → NAT` route to the private route table — private resources can initiate outbound connections but remain unreachable inbound
- Re-launched the EC2 web server into `project1-public-1a`, replacing the default VPC deployment with a properly isolated network
- Enabled VPC Flow Logs on the VPC, shipping to a CloudWatch Logs log group with 7-day retention

![VPC Resource Map showing full topology](screenshots/phase3-VPC-ResourceMap.png)

![Public route table with 0.0.0.0/0 → IGW](screenshots/phase3-PublicRouteTable.png)
![Private route table with 0.0.0.0/0 → NAT](screenshots/phase3-PrivateRouteTable.png)

![NAT Gateway with attached Elastic IP](screenshots/phase3-NATGateway.png)
![EC2 web server running in project1-public-1a](screenshots/phase3-EC2-InVPC.png)
![VPC Flow Logs configured with 7-day retention](screenshots/phase3-FlowLogs-Config.png)

### What I broke and how I found it
Deleted the `0.0.0.0/0 → IGW` route from the public route table to simulate a misconfigured network. The EC2 instance stayed running and healthy at the status-check level, but the site was unreachable — browser requests timed out, and `curl` confirmed a connection timeout with no response.

![Public route table with the IGW route removed](screenshots/phase3-PublicRouteTable-Broken.png)
![curl timing out while the site is down](screenshots/phase3-Curl-Timeout.png)

Opened VPC Flow Logs expecting to see REJECT entries for the dropped traffic. Instead, Flow Logs showed ACCEPT entries for inbound packets from my IP on port 80 — indistinguishable from a healthy site. The inbound packets were passing the security group check on arrival, so Flow Logs recorded them as ACCEPT. The actual failure was on the return path: the EC2 instance had no `0.0.0.0/0 → IGW` route to send the HTTP response back out to the internet, so the response packets never left the subnet. Flow Logs does not surface that kind of routing failure as a REJECT — it simply never records the response side of the conversation.

![Flow Logs showing ACCEPT entries despite the broken route](screenshots/phase3-FlowLogs.png)

This is the lesson the exercise actually teaches: ACCEPT in Flow Logs means a packet was not blocked. It does not mean the connection succeeded. When a site is down but Flow Logs shows ACCEPT, the failure is downstream of the security group — most often a routing or NAT problem on the return path. The diagnostic signal here was the contrast between `curl` (timeout) and Flow Logs (ACCEPT), not the presence of a REJECT entry.

Restored the IGW route and confirmed the site loaded again.

### What I learned
- A subnet is public or private based on its route table, not its name — the `0.0.0.0/0 → IGW` entry is the definition of a public subnet
- Route tables use longest prefix match, not top-to-bottom priority. `10.0.0.0/16 → local` always wins over `0.0.0.0/0 → IGW` for internal traffic because it is more specific
- The Internet Gateway does not filter traffic — it is a dumb pipe. ACCEPT and REJECT decisions happen at the security group, not the IGW
- Flow Logs records security group decisions on each ENI — not connection success. When the IGW route was deleted, Flow Logs kept showing ACCEPT for inbound packets on port 80 because the SG check passed; the failure was on the return path, which never surfaces as a REJECT. ACCEPT means "not blocked," not "delivered." Diagnosing routing failures requires correlating Flow Logs with external evidence (curl, browser timeouts), not Flow Logs alone
- REJECT in Flow Logs means a packet was blocked by a security group or NACL — typically internet scanners hitting closed ports. A REJECT is a useful security signal, but its absence does not mean the network is healthy
- A VPC Flow Log line decoded: `version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status` — the `action` field is ACCEPT or REJECT, `log-status` OK means the log was captured successfully
- NAT Gateways are zonal resources. One NAT per AZ is the production pattern — a single NAT is a single point of failure. This project uses one NAT as a cost tradeoff; production would use two
- Elastic IPs are static public IP addresses owned until explicitly released. As of February 2024, AWS charges $0.005/hour for every public IPv4 address — attached or unattached — so release Elastic IPs as soon as the attached resource is decommissioned
- The private subnets are built and wired correctly but not yet exercised — they will host RDS in Phase 4. The NAT Gateway path (private → internet) remains unused in the current architecture because RDS does not require outbound internet, but it is in place for future workloads that do (e.g., a Lambda inside the VPC, or a private EC2 pulling package updates)

---

## Phase 4 — ALB + RDS Multi-AZ

**Services:** EC2, ALB, RDS MySQL, IAM, S3, Security Groups

### What I built
- Designed a three-layer security group chain: `project1-alb-sg` accepts HTTP from the internet, `project1-sg-web` accepts HTTP only from `project1-alb-sg`, and `project1-rds-sg` accepts MySQL only from `project1-sg-web` — no CIDR ranges, only SG references
- Launched a second EC2 instance in `project1-public-1b` (us-east-2b) to mirror the existing instance in us-east-2a, spreading the application tier across two Availability Zones
- Created an Application Load Balancer in both public subnets with a target group registering both EC2 instances — external traffic now routes through the ALB DNS name, never directly to EC2 public IPs
- Enabled ALB access logs shipping to S3 every 5 minutes; AWS writes an `ELBAccessLogTestFile` on enable to confirm bucket policy permissions are correct
- Created a DB subnet group across `project1-private-1a` and `project1-private-1b`, then launched RDS MySQL Multi-AZ on `db.t3.micro` with public access disabled — the database sits entirely in private subnets, unreachable from the internet
- Attached an IAM role (`project1-ec2-role`) to both EC2 instances with `AmazonS3ReadOnlyAccess` to allow reading ALB logs from the instance without embedded credentials
- Verified network reachability from both EC2 instances to the RDS endpoint on port 3306 using the MySQL CLI. The application running on the web tier is a static page (PIFID — prime index factor ID) and does not query the database; RDS was deployed in this phase to practice Multi-AZ provisioning, security group chaining, and connectivity troubleshooting, not to back the web app

![Application Load Balancer configured across both public subnets](screenshots/phase4-ALB.png)
![Target group with both EC2 instances registered](screenshots/phase4-TG.png)
![Static PIFID page served through the ALB DNS hostname](screenshots/phase4-AppViaALB.png)
![RDS MySQL Multi-AZ instance in the private subnets](screenshots/phase4-RDS.png)

### What I broke and how I found it
Deleted the `MySQL/Aurora / 3306 / project1-sg-web` inbound rule from `project1-rds-sg` to simulate a misconfigured security group — the most common Cloud Support ticket pattern for database connectivity failures.

![RDS security group with the MySQL inbound rule removed](screenshots/phase4-RDS_SG.png)

The application continued serving pages normally. The ALB health checks kept passing and no 502s appeared in the access logs, because Apache returns HTTP 200 for every page load regardless of whether the database is reachable. The failure was completely silent at the HTTP layer.

The only signal was the MySQL CLI: a connection attempt from either EC2 instance hung for approximately two minutes before timing out with no response. The root cause was visible in the empty inbound rules on `project1-rds-sg` — the rule that permitted EC2 to reach RDS on port 3306 was gone.

This exercise taught the most important diagnostic lesson of Phase 4: ALB logs show what the load balancer sees, not what the application layer experiences. A broken database connection produces no HTTP errors if the web server handles the response independently. Diagnosing database connectivity failures requires probing the application layer directly, not reading HTTP status codes.

Restored the rule and confirmed the MySQL connection re-established immediately.

### What I learned
- Security group chaining uses SG IDs as sources rather than CIDR ranges — any resource added to the referenced SG automatically inherits the permission without rule changes
- The distinction between public and private subnets is purely a routing decision: a `0.0.0.0/0 → IGW` route makes a subnet public, not its name or the type of IP address assigned. All resources in a VPC receive a private RFC 1918 address regardless of subnet type
- ALB access logs are delivered in 5-minute intervals, not in real time — the `ELBAccessLogTestFile` is written immediately on enable and confirms bucket policy permissions before the first log delivery
- ALB deregistration delay defaults to 300 seconds — targets appear to drain slowly because the ALB waits for in-flight requests to complete before removing them from rotation
- RDS Multi-AZ maintains a synchronous standby in a second AZ. On forced failover (reboot with failover), I observed the connection drop for approximately 2 minutes 13 seconds before recovering automatically through the same DB endpoint. The RDS event log explicitly recorded `Multi-AZ instance failover started → DB instance restarted → Multi-AZ instance failover completed`, and the instance's AZ moved from us-east-2a to us-east-2b — confirming the standby was promoted and AWS repointed the endpoint DNS automatically with no application-side configuration change
- IAM roles attached to EC2 instances are the correct credential pattern for AWS CLI access from within an instance — no access keys stored on disk, credentials rotate automatically via instance metadata

![MySQL CLI loop showing the connection drop and recovery across a Multi-AZ failover — outage from 15:00:59 to 15:03:12, approximately 2 minutes 13 seconds](screenshots/phase4-FailOver.png)
![RDS event log showing the failover sequence and the instance now in us-east-2b](screenshots/phase4-FailOverEvents.png)

---

## Phase 5 — Auto Scaling Group

**Services:** EC2, Auto Scaling, Launch Templates, ALB, CloudWatch

### What I built
- Created a Launch Template from the existing Phase 4 EC2 configuration to serve as a reproducible blueprint for every ASG-launched instance — instance type, AMI, security group, and user data defined once and applied consistently across each launch
- Replaced the two static EC2 instances from Phase 4 with an Auto Scaling Group spanning both public subnets (`project1-public-1a` and `project1-public-1b`), configured with minimum 1, desired 2, and maximum 4. The Phase 4 instances were stopped and deregistered from the target group so the ASG could take over without competing with manually managed compute
- Attached the ASG to the existing target group (`project1-tg`) and enabled ELB health checks at the ASG level with a 300-second instance warmup. The ASG now uses the ALB's health verdict (HTTP 200 on port 80) to decide whether an instance should be replaced, not just EC2 status checks
- Added a target tracking scaling policy targeting 50% average CPU utilization across the group. Target tracking automatically creates two CloudWatch alarms — `TargetTracking-…-AlarmHigh` and `TargetTracking-…-AlarmLow` — which become the diagnostic entry point when scaling appears not to work
- Extended the launch template's user data to inject the instance's hostname into the served page title, making ALB round-robin routing dir