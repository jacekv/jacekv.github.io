# AWS Certified Security Specialty

I am currently doing a Udemy course on AWS Certified Security Specialty. I will
be posting my notes here.

## Thread Detection and Incident Response

### AWS Guard Duty

#### What is it?

* Intelligent Threat Discovery to protect your AWS account using Machine Learning
algorithms, anomaly detection, and 3rd party data.

* Input data to GuardDuty includes:
    * VPC Flow Logs - unusual internet traffic, unusual IP address, etc
    * CloudTrail Event Logs - unusual API calls, unauthorized deployments, etc.
    * DNS Logs - unusual DNS queries like encoded data within DNS queries, etc.
    * Optional features - EKS Audit logs, RDS & Aurora, EBS, Lambda, and more

* It can also protect against CryptoCurrency attacks, since it has a dedicated
"finding" for it.

* You can manage multiple accounts in GuardDuty,
  * through an AWS Organization
  * sending invitation through GuardDuty

* The Administrator account can add and remove member accounts, and manage GuardDuty
withing the associated member accounts. It can also manage findings, suppression
rules, trusted IP lists, threat lists.

* In an AWS Organization, you can specify a member account as the Organization's
delegated administrator for GuardDuty.

#### Findings Automated Response

* Findings are potential security issues for malicious events happening in you AWS
account.

* You are able to automate responses to findings, by using EventBridge. Send alerts
to SNS, SQS, Lambda, etc. which can trigger a response.

* Events are published to both the administrator account and the member account
that it is originating from.

* Each finding has a severity value between 0.1 to 8+.

#### Trusted and Thread IP Lists

* Works only for IP addresses

* Trusted IP lists are IP addresses and CIDR ranges that you trust and you do not
want to be flagged as malicious.

* Thread IP lists are IP addresses and CIDR ranges that you do not trust and you
want to be flagged as malicious.

#### Suppression Rules

* Set of criteria that automatically filter and archive new findings.

* You can suppress entire findings types or specific findings defined on more granular
criteria.

* Suppressed findings are NOT sent to Security Hub, S3 Detective, or Event Bridge.

* Suppressed findings can be still be viewed in the Archive.

#### Potential Question in the Exam

* Problem: GuardDuty is activated but it didn't generate any DNS based findings.

* Reason: GuardDuty only processes DNS logs if you use the default VPC DNS
resolver. All other types of DNS resolver won't generate DNS based findings.

* If GuardDuty is suspended or disabled, then no finding types are generated.

* Best practice to enable GuardDuty in all regions.


### AWS Security Hub

#### What is it?

* A central security tool to manage security across multiple AWS accounts and
and to automate security checks.

* It aggregates findings, insights, and security scores from multiple
regions to a single aggregation region.

* You can also have AWS Organizations Integration, where all accounts are covered
by Security Hub.

* The integrated dashboard shows your current security and compliance status
to quickly take actions.

* It automatically aggregates alerts in predefined or personal findings format
from various AWS services & AWS partner tools.

* In order to use Security Hub, you need to enable AWS Config. If you have
multiple accounts, due to an Organization, you need to enable AWS Config in
all accounts.

* Guard Duty is enabled automatically when you enable Security Hub. Guard Duty
will send findings to Security Hub in AWS Security Finding Format (ASFF).

* You can also create Custom Actions in Security Hub, which are automated
remediation actions that can be executed on specific findings.

* A finding is going to send a message to EventBridge, which will trigger a
Lambda function to execute the remediation action.

### Amazon Detective

* GuardDuty, Macie, and Security Hub are used to identify potential security
issues or findings.

* Amazon Detective analyzes, investigates, and quickly identifies the root cause
of security issues or suspicious activities (using Machine Learning and graphs).

* It automatically collects and processes events from VPC Flow Logs, CloudTrail,
GuardDuty and creates a unified view.

* Produces visualizations with details and context to get to the root cause.

### Penetration Testing on AWS

* AWS customers can carry out security assessments or penetration tests against
their own AWS infrastructure without prior approval for 8 services:

1. EC2, NAT Gateways, and Elastic Load Balancers
2. RDS
3. CloudFront
4. Aurora
5. API Gateway
6. Lambda and Lambda Edge
7. Lightsail
8. Elastic Beanstalk environments

Prohibited activities include:

1. DNS zone walking
2. Denial of Service (DoS) attacks
3. Port flooding
4. Protocol flooding
5. Request flooding

You can contact AWS Support to request permission for prohibited activities.

### DDoS Simulation Testing on AWS

* Controlled DDos attack which enables you to evaluate the resilience of your
applications against DDoS attacks.

* It must be performed by an approved AWS DDoS Test Partner.

* The target can be either Protected Resources or Edge-Optimized API Gateway
that is subscribed to Shield Advanced.

* The Attack must not originate from AWS resources, not exceed 20 Gigabits/second,
not exceed 5 million packets/second for Cloudfront and 50000 packets/second for
any other service.

### Compromised resources

#### Compromised EC2 Instances

* Steps to address compromised EC2 instances:
    * Capture the instance's metadata
    * Enable Termination Protection (can't be terminated)
    * Isolate the instance (replace instance's SG - no outbound traffic)
    * Detach the instance from any ASG (auto scaling group)
    * Deregister the instance from any ELB
    * Snapshot the EBS volume
    * Tag the EC2 instance (e.g. investigation ticket)
        
* Offline investigation: shutdown instance - look at memory
* Online investigation (e.g. snapshot memory or capture network traffic) 
* Automate the isolation process: Lambda
* Automate memory capture: SSM Run command

#### Compromised S3 Bucket

Identify the compromised S3 bucket using GuardDuty
Identify the source of the malicious activity (e.g. IAM user, role) and the API
calls using CloudTrail or Amazon Detective

Identify whether the source was authorized to make those API calls
Secure your S3 bucket, recommended settings:
    * S3 Block Public Access Settings
    * S3 Bucket Policies and User Policies
    * VPC endpoints for S3
    * S3 Presigned URLs
    * S3 Access Points
    * S3 ACLs (deprecated)

#### Compromised ECS Cluster

Identify the affected ECS Cluster using GuardDuty
Identify the source of the malicious activity (e.g. container, tasks)
Isolate the impacted tasks (deny all ingress/egress traffic to the task using
security groups)
Evaluate the presence of malicious activity (e.g. malware)

#### Compromised RDS Database Instance

Identify the affected RDS Database Instance and DB user using GuardDuty
If it is NOT legitimate behavior:
    * restrict access to the RDS instance (security groups, NACLs)
    * restrict the db access for the suspected user
Rotate the suspected DB user's password
Review DB Audit logs to identify leaked data
Secure your RDS DB instance, recommended settings:
    * Use Secrets Manager to rotate the DB password
    * USe IAM DB Authentication to manage DB users access without passwords

### Compromised AWS Credentials

* Identify the compromised IAM user using GuardDuty
* Rotate the exposed AWS Credentials
* Invalidate temporary credentials by attaching an explicit Deny policy to
the affected IAM user with an STS date condition
* Check CloudTrails logs for other unauthorized activities
* Review your AWS resource (e.g. delete unauthorized resources)
* Verify your AWS account information

#### Compromised IAM Role

* Invalidate temporary credentials bu attaching an explicit Deny policy to the
affected IAM user with an STS date condition
* Revoke access for the identity to the linked AD if any
* Check CloudTrail logs for other unauthorized activity
* Review your AWS resources
* Verify your AWS account information

#### Compromised Account

* Rotate and delete exposed AWS Access Keys
* Rotate and delete any unauthorized IAM users credentials
* Rotate and delete all EC2 key pairs
* Review your AWS resources
* Verify your AWS account information

## Security Logging and Monitoring

### Amazon Inspector

* Allows to run automated security assessments
    * For EC2 instances
        * Leverages the AWS System Manager agent
        * Analyze against unintended network accessibility
        * Analyze the running OS against known vulnerabilities 
    * For Container Images pushed to ECR
        * Assessment of Container Images as they are pushed
    * For Lambda Functions
        * Identifies software vulnerabilities in the Lambda function code and dependencies
        * Assessment of functions as they are deployed
* Reporting & integration with AWS Security Hub
* Send findings to Amazon Event Bridge

#### What do we evaluate

* Only for EC2 instances, Container Images & Lambda functions
* Continuous scanning of the infrastructure, only when needed
* Package vulnerabilities - database of CVE
* Network reachability

A risk score is associated with all vulnerabilities for prioritization

### Logging in AWS for security and compliance

* To help compliance requirements, AWS provides many service-specific security
and audit logs.

* Service logs include:
 * CloudTrail logs - trace all API calls
 * Config Rules - for config & compliance over time
 * CloudWatch logs - for full data retention
 * VPC Flow Logs - IP traffic within your VPC
 * ELB Access Logs - metadata of requests made to your load balancers
 * CloudFront Logs - web distribution logs
 * WAF Logs - web application firewall logs

* Logs can be analyzed using AWS Athena if they are stored in S3.
* You should encrypt logs in S3, control access using IAM & Bucket Policies, MFA
* Move logs to Glacier for cost savings

### AWS Systems Manager (SSM)

* Helps you to manage your EC2 and on-premises systems at scale
* Get operational insights about the state of your infrastructure
* Easily detect problems
* Patching automation for enhanced compliance
* Works for Linux and Windows
* Integrated with AWS Config
* Free Service

* We need to install the SSM agent onto the systems we control
* Amazon Linux & and some Ubuntu AMI have it pre installed
* If an instance can't be controlled with SSM, it's probably an issue with the SSM agent
* Make sure the EC2 instance have proper IAM role to allow SSM actions

#### SSM Documents
* Documents are JSON or YAML
* You define parameters
* You define actions
* Many documents already exist in AWS

* Possible document owned by Amazon - ApplyPathBaseline

#### Run Command
* Execute a document (=script) or just run a command
* Run command across multiple instances (using resource groups)
* Rate Control / Error Control
* Integrate with IAM & CloudTrail
* No need for SSH
* Command Output can be shown in the Console, sent to S3 bucket or CloudWatch Logs
* Send notifications to SNS about command status
* Can be invoked using EventBridge

#### SSM Automation
Simplifies common maintenance and deployment tasks of EC2 instances and other AWS resources
Example: restart instances, create an AMU, EBS snapshot
Automation Runbook
  * SSM Documents of type Automation
  * Defines actions preformed on your EC2 instances or AWS resources
  * Pre-defined runbooks (AWS) ir create custom runbooks
Can be triggered
  * Manually using AWS Console, AWS CLI or SDK
  * By Event Bridge
  * On a schedule using Maintenance Windows
  * By AWS Config for rules remediations

#### SSM Parameter Store
* Secure storage for configuration and secrets
* Optional Seamless Encryption using KMS
* Severless, scalable, durable
* Version tracking of configurations/secrets
* Security through IAM
* Notifications with Amazon Event Bridge
* Integration with CloudFormation

#### SSM Inventory
* Collect Metadata from your managed instances
* Metadata includes installed software , OS drivers, configurations, installed updated, running services
* View data in AWS Console or store in S3 and query and analyze using Athena and QuickSight
* Specify metadata collection interval (minutes, hours, days)
* Query data from multiple AWS accounts and regions
* Create custom Inventory for your custom metadata

#### SSM State Manager

Automate the process of keeping your managed instances in a state that you define

Use cases: bootstrap instances with software, patch OS/software updates on a schedule

State Manager Association
    * Define the state you want your instances to be in
    * Define the schedule for applying the state

Use SSM Documents to create an Association

#### SSM Patch Manager
* Automates the process of patching managed instances
* OS updates, application updates, security updates, ...
* Supports both EC2 instances and on-premises serves
* Path on-demand or on schedule using Maintenance Windows
* Scan instances and generate patch compliance reports (missing patches)
* Path compliance report can be sent to S3
* Path Baseline
  * Defines which patches should and shouldn't be installed on your instances
  * Ability to create custom Patch Baselines
  * Patches can be auto-approved within days of their release
  * By default, install only critical patches and patches related to security
* Path Group
  * Associate a set of instances with a specific Path Baseline
  * Example: create Patch Groups for different environment (dev, test, prod)
  * Instances should be defined with the tag key "Path Group"
  * An instance can only be in one Patch Group
  * Patch Group can be registered with only one Patch Baseline
* SSM - Patch Manager Patch Baseline
  * Pre-Defined Patch Baseline
    * Managed by AWS for different Operating Systems (can't be modified)
    * AWS-RunPatchBaseline (SSM Document) - apply both operating system and application patches
* Custom Path Baseline
  * Create your own Patch Baseline and choose which patches to auto-approve
  * Operating system, allowed patches, rejected patches, ...
  * Ability to specify custom and alternative patch repositories
* SSM Maintenance Windows
  * Defines a schedule for then to perform actions on your instances
  * Example: OS patching, updating drivers, installing software, ...
  * Maintenance Window contains
    * schedule
    * Duration
    * Set of registered instances
    * Set of registered tasks

#### SSM Session Manager
* Allows you to start a secure shell on your EC@ and on-premises servers
* Access through AWS Console, AWS CLI, or Session Manager SDK
* Does not need SSH access, bastion hosts, or SSH keys
* Supports Linux, macOs, and Windows
* Log connections to your instances and executed commands
* Session log data can be sent to S3 or CloudWatch Logs
* CloudTrail can intercept StartSession events
* Iam Permissions:
  * Control which users/groups can access Session Manager and which instances
  * Use tags to restrict access to only specific EC2 instances
  * Access SSM + write to S3 + write to CloudWatch
* Optionally, you can restrict commands that can be run in the session by a user

### AWS CloudWatch

#### Unified CloudWatch Agent
* For virtual servers (EC2 instances, on-premises servers, ...)
* Collect additional system-level metrics such as RAM, processes, used disk space, etc.
* Collect logs to send to CloudWatch logs
  * No logs from inside your EC2 instance will be sent to CLoudWatch without using an agent
* Centralized configuration using SSM Parameter Store
* Default namespace for metrics collected by the agent is *CWAgent* (can be changed)

* procstat Plugin for Agent
    * Collect metrics and monitor system utilization pf individual processes
    * Supports both Linux and Windows servers
    * Example: amount of time the process uses CPU, amount of memory the process uses, ...
    * Select which process to monitor ny:
        * pid_file: name of process identification number (PID) files they create
        * exe: process name that match string you specify (RegEx)
        * pattern: command lines used to start the process (RegEx)
    * Metrics collected by procstat plugin begins with *procstat* prefix

#### Troubleshooting Unified CloudWatch Agent

* CloudWatch Agent Fails to Start
    * Might be an issue with the configuration file
    * Check configuration file logs at /opt/aws/amazon-cloudwatch-agent/logs/configuration-validation.log
* Can't find Metrics Collected by the CloudWatch Agent
    * Check you are using the correct namespace (default is *CWAgent*)
    * Check the configuration file `amazon-cloudwatch-agent.json`
* CloudWatch Agent not pushing Log Events
    * Update to the latest CloudWatch Agent version
    * Test connectivity to the CloudWatch Logs endpoint `logs.<region>/amazonaws.com`
    * Review account, region, and log group configurations
    * Check IAM permissions
    * Verify the system time on the instance is correctly configured
* Check CloudWatch Agent logs at `/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`

#### CloudWatch Logs

* Log groups: arbitrary name, usually representing an application
* Log stream: instances within application/log files/ containers
* Can define log expiration policies (never expire, 1 day to 10 years...)
* CloudWatch Logs can send logs to:
    * Amazon S3
    * Kinesis Data Stream
    * Kinesis Data Firehose
    * Aws Lambda
    * OpenSearch
* Logs are encrypted by default
* Can setup KMS-based encryption with your own keys

#### CloudWatch Logs Sources
* SDK, CLoudWatch Logs Agent, CloudWatch Unified Agent
* Elastic Beanstalk: collection of logs from application
* ECS: collection from containers
* AWS Lambda: collection from Lambda functions
* VPC Flow Logs: collection of network traffic
* API Gateway: collection of API logs
* CloudTrail: collection of API calls
* Route 53: collection of DNS queries

#### CloudWatch Logs Insights

* Interactive log analytics service
* ALlows you to write a query to retrieve, aggregate, and search log data
* Example: find a specific IP inside a log, count occurrences of "Error" in logs, ...
* Provides a purpose=built query language
    * Automatically discovers fields from AWS services and JSON log events
    * Fetch desired event fields, filter based on conditions, calculate aggregate statistics, sort events, limit number of events, ...
    * Can save queries and add them to CloudWatch Dashboards
* Can query multiple Log Groups in different AWS accounts
* Its a query engine, not a real0-time engine

#### CloudWatch Logs S3 Export

* Log data can take up to 12 hours to become available in for export
* The API call is *CreateExportTask*
* Not near-real time or real-time... use Logs Subscription instead

#### CloudWatch Logs Subscription

* Get a real-time log event from CloudWatch Logs for processing and analysis
* Send to Kinesis Data Streams, Kinesis Data Firehose, AWS Lambda
* Subscription Filter - filter which logs are events delivered to the destination
* Log Aggregation with Subscription FIlters from different accounts and regions

#### CloudWatch Alarms

* Alarms are used to trigger notifications for any metric
* Various options (sampling, %, min, max, ...)
* Alarms states: OK, INSUFFICIENT_DATA, ALARM
* Period:
  * Length of time associated with a specific CloudWatch metric
  * High resolution custom metrics: 10 sec, 30 sec or multiple of 60 sec

#### CloudWatch Alarms Target

* Stop, Terminate, Reboot, or Recover an EC2 instance
* Trigger Auto SCaling Action
* Send notification to SNS

#### CloudWatch Alarms - Composite Alarms

* CloudWatch ALarms are on a single metric
* Composite Alarms are monitoring the states of multiple other alarms
* AND / OR logic
* Helpful to reduce "alarm noise" by creating complex composite alarms

##### EC2 Instance Recovery

* Status Check:
  * Instance status: check the EC2 VM
  * System status: check the underlying hardware
  * Attached EBS status: check attached EBS volumes

* Alarms can be created based on CloudWatch Logs Metrics Filters
* To test alarms and notification, set the alarm state to Alarm using CLI

#### CloudWatch Contributor Insights

* Analyze log data and create time series that display contributor data
* Helps you find top talkers and understand who/what is impacting system performance
* Example: find bad hosts, identify the heavies network users, find the URLs that generate the most errors, ...
* Works for any AWS-generated logs (VPC, DNS, ELB, ...)
* Built-in rules created by AWS (leverages your CW logs) or build your own rules

### Amazon Event Bridge

* Schedule: Cron jobs (scheduled scripts)
* Event Pattern: Event riles to react to a service doing something
* Trigger Lambda functions, send SQS/SNS messages, etc.
* Default Event Bus: AWS default event bus
* Partner Event Bus: receive events from SaaS partners
* Custom Event Bus: create your own event bus
* Event buses can be accessed by other AWS accounts using Resource=based Policies
* You can archive events sent to an event bus
* Ability to replay archived events
* Schema Registry:
    * EventBridge can analyze the events in your bus and infer the schema
    * The Schema Registry allow you to generate code for your application, that will know in advance how data is structured in the event bus
    * Schema can be versioned
* Resource-based Policy
    * Manage permissions for a specific event bus
    * Example: allow/deny events from another AWS account or AWS region
    * USe case: aggregate all events from you AWS Organization in a single AWS account or AWS region

### Amazon Athena

* Serverless query service to analyze data stored in S3 using SQL
* Supports CSV, JSON, ORC, Avro, and Parquet
* Commonly used with Amazon Quicksight for reporting/dashboards
* Use cases: Business intelligence/analytics/reporting, analyze & query VPC FLow logs, ELB Logs, CloudTrail logs, ...
* Exam Tip: analyze data in S3 using serverless SQL, use Athena
* Performance Improvement
    * Since you pay per TB data scanned, optimize your queries
    * Use columnar data for cost-savings (less scan)
        * Apache Parquet is ORC is recommended
        * Huge performance improvement
        * Use Glue to convert your data to Parquet or ORC
    * Compress data fir smaller retrievals (bzip2, gzip, lz4, snappy, zlip, zstd, ...)
    * Partition datasets in S3 for easy querying on virtual columns
    * Use the right data format for the right data type
        * s3://yourBucket/pathToTable/<PARTITION_COLUMN_NAME>=<VALUE>/<PARTITION_COLUMN_NAME>=<VALUE>/<PARTITION_COLUMN_NAME>=<VALUE>/etc
        * S3://athena-examples/flight/parquet/year=2019/month=1/day=1/...
    * Use large files (> 128MB) to minimize overhead
* Federated Query
    * Allow you to run SQL queries across data stored in relational, non-relational, object, and custom data sources
    * Uses Data Source Connectors that run on AWS Lambda to run Federated Queries
    * Store the results back in S3
* Troubleshooting
    * Insufficient Permissions When using Athena with QuickSight
        * Validate QuickSight can access S3 buckets used by Athena
        * If the data in the S3 buckets is encrypted using AWS KMS key (SSM-KMS), then QuickSight IAM role mist be granted access to decrypt with the key

### CloudTrail

* Provides governance, compliance and audit for you AWS account
* CloudTrail is enabled by default
* Get an history of events/API calls made within AWS AWS Account by:
    * Console
    * SDK
    * CLI
    * AWS Services
* Can put logs from CloudTrail into CloudWatch Logs or S3
* A trail can be applied to All Regions (default) or a single region
* If a resource is deleted in AWS, investigate CloudTrail first
* Management Events:
    * Operations that are performed n resources in your AWS account
    * Examples:
        * Configuring security
        * COnfiguring rules for routing data
        * Setting up logging
    * By default, trails are configured to log management events
    * Can separate Read Events (that don't modify resources) from Write Events (that do modify resources)
* Data Events:
    * By default, trails do not log data events (due to high volume operations)
    * Amazon S3 object-level activity (e.g. GetObject, DeleteObject, ...) can separate Read Events from Write Events
    * AWS Lambda function execution activity
* CloudTrail Insight Events
    * Enable CloudTrail Insights to detect unusual activity in your AWS account
    * CloudTrail Insights analyzes write management events to detect unusual patterns
        * inaccurate resource provisioning
        * hitting service limits
        * Bursts of AWS IAM actions
        * Gaps in periodic maintenance activity
    * CloudTrails Insights analyzes normal management events to create a baseline
    * And then continuously analyzes write events to detect unusual patterns
        * Anomalies appear in the CloudTrail console
        * Event is sent to Amazon S3
        * An EventBridge event is generated (for automation needs)
* CloudTrail Events Retention
    * Events are stored for 90 days in CloudTrail
    * To keep event beyond this period, log them to S3 and use Athena

### CloudTrail for SysOps

* Log File Integrity Validation
    * Digest Files:
        * References the log files for the last hour and contains a hash of each
        * Stored in the S3 bucket with the log files
    * Helps you determine whether a log file was modified/deleted after CloudTrail delivered it
        * Good for compliance and security
    * Hashing using SHA-256, Digital Signing using SHA-256 with RSA
    * Protect the S3 bucket using bucket policy, versioning, MFA delete protection, encryption, object lock
    * Protect CloudTrail using IAM
* Integration with EventBridge
    * Used to react to any API call being made in your account
    * CloudTrail is not real-time
        * Delivers an event within 15 minutes of an API call
        * Delivers log files to an S3 bucket every 5 minutes
* Organization Trails
    * A trail that will log all events for all AWS accounts in an Organization
    * Log events for management and member accounts
    * Trail with the same name will be created in ever AWS account
    * Member accounts can't remove or modify the organization trail
* CloudWatch Metrics Filter
    * possible use case: Start to many EC2 instances in a short period of time
    * Create a metric filter to count the number of StartInstances API calls > 30
    * Create a CloudWatch Alarm on this metric
    * ability to set thresholds for detection
* Integration with Athena
    * You can use Athena to directly query CloudTrail logs stored in S3
    * Example: analyze operational activity for security and compliance
    * Can create Athena table directly from the CloudTrail console, then specify the S3 bucket location where the CloudTrail logs are stored 

### Monitoring Account activity

* AWS Config Configuration History
    * Must have AWS Config Configuration Recorder On
* CloudTrail Even History
    * Search API history for past 90 days
    * Filter by resource name, resource type, event name
    * Filter by IAM user, assumed IAM role session name, or AWS Access Key
* CloudWatch Logs Insights
    * Search API history beyond the past 90 days
    * CloudTrail Trail mist be configured to send logs to CloudWatch Logs
* Athena Queries
    * Search API history beyond the past 90 days

### Macie

* Amazon Macie is a fully managed data security and data privacy service that uses machine learning and pattern matching to discover and protect your sensitive data in AWS
* Macie helps identify and alert you to sensitive data, such as personally identifiable information (PII)
* Data Identifiers
    * Used to analyze and identify sensitive data in your S3 buckets
    * Managed Data Identifier
        * A set if built-in criteria that Macie uses to analyze and identify sensitive data
        * Examples: AWS credentials, credit card numbers, email addresses, ...
    * Custom Data Identifiers
        * A set of criteria that you define to analyze and identify sensitive data
        * Regular expression, keywords, proximity rule
        * Example: employee IDs, customer account numbers
    * You can use AllowLists to define a text pattern to ignore (e.g. public phone numbers)
* Findings
    * A report of a potential issue or sensitive data that Macie found
    * Each finding has a severity rating, affected resource, datetime, ...
    * Sensitive Data Discovery Result
        * A record that logs details about the analysis of an S3 object
        * Configure Macie to store the results in S3, then quey using Athena
    * Suppression Rules - set of attribute-based filter criteria to archive findings automatically
    * Findings are stored for 90 days
    * Review findings using AWS Console, EventBridge, SecurityHub
    * Policy Findings
        * A report of a potential issue or policy violation that Macie found
        * Example: S3 bucket with public access, S3 bucket with encryption disabled, ...
    * Sensitive Data Findings
        * A detailed report of sensitive data that's found in S3 buckets
        * Examples: Credentials (private keys), Financial (credit card numbers), ...
* Multi-account Strategy
    * You can manage multiple accounts in Macie
    * Associate the Member accounts with the Administrator account
        * Through an AWS Organization
        * Sending an invitation through Macie
        * Supports Delegated Administrator in an AWS Organization
    * Administrator account can:
        * Add and remove member accounts
        * Have access to all S3 sensitive data and settings for all accounts
        * Manage Automated Sensitive Data Discovery and run Data Discovery jobs
        * Manage Data Identifiers and Findings

### S3 Event Notifications

* S3:ObjectCreated, S3:ObjectRemoved, S3:ObjectRestore, S3:Replication, ...
* Object na,e filtering possible (*.jpg)
* Use case: generate thumbnails of images uploaded to S3
* Can create as many S3 events as desired
* S3 event notifications typically deliver events in seconds but can sometimes take a minute or longer
* IAM Policy attached to SNS/SQS/Lambda Resource (Access) Policy to allow S3 to send events to the resource
* with Amazon Event Bridge you can pass it to over 19 AWS services as destination
    * Advanced filtering option with JSON riles
    * Multiple Destinations (Step Functions, Kinesis Streams/ Firehose, Lambda, SNS, SQS, ...)
    * EventBridge Capabilities - Archive, Replay, Schema Registry

### VPC Flow Logs

* Capture information about IP traffic going into your interfaces:
    * VPC Flow Logs
    * Subnet Flow Logs
    * Elastic Network Interface Flow Logs
* Helps tp monitor & troubleshoot connectivity issues
* Flow logs data can go to S3m CloudWatch Logs, and Kinesis Data Firehose
* Captures network information from AWS managed interfaces too (ELB, RDS, ElastiCache, Redshift, WorkSpaces, NATGW, Transit Gateway, ...)
* Query VPC flow logs using Athena on S3 or CLoudWatch Logs Insights
* Troubleshoot SG & NACL issues
    * Look at the ACTION field
    * Incoming Request:
        * Inbound REJECT => NACL or SG
        * Inbound ACCEPT, Outbound REJECT => NACL
    * Outgoing Request:
        * Outbound REJECT => NACL or SG
        * Outbound ACCEPT, Inbound REJECT => NACL
* Architectures
    * VPC Flow Logs -> CloudWatch Logs -> CloudWatch Contributor Insights
    * VPC FLow Logs -> S3 -> Metric Filter -> CW Alarm -(Alarm)> Amazon SNS
    * VPC Flow Logs -> S3 -> Athena -> QuickSight
* Traffic not captured
    * Traffic to Amazon DNS server (custom DNS server traffic is logged)
    * Traffic Amazon Windows license activation server
    * Traffic to and from 169.254.169.254 for EC2 instance metadata
    * Traffic to and from 169.254.169.123 for Amazon Time Sync service
    * DHCP traffic
    * Mirrored traffic
    * Traffic to the VPC router reserver IP address (e.g. 10.0.0.1)
    * Traffic between VPC endpoints ENI and Network Load Balancer ENI

### Traffic Monitoring

* Allows you to capture and inspect network traffic in your VPC
* Route traffic to security appliances that you manage
* Capture the traffic
    * From (Source) - ENIs
    * To (Targets) - an ENI or a Network Load Balancer
* Capture all packets of capture the packets of your interest
* Source and Target can be in the same VPC or different VPCs
* Use cases: Content inspection, threat monitoring, troubleshooting
* VPC Network Analyzer
    * Helps you understand potential network paths to/from your resources
    * Define NEtwork Access Requirements
        * Example: identify publicly available resources
    * Evaluate against them and find issues / demonstrate compliance
        * Evaluate network access to resources in your VPCs
        * Match against the configuration of your VPC resources
    * Network Access Scope - json document contains conditions to define your network security policy

### Route53 - Query Logging
* DNS Query Logging
    * Log information about public DNS queries Route53 Resolver receives
    * Only for public Hosted Zones
    * Logs are sent to CLoudWatch Logs only
* Resolver Query Logging
    * Log all DNS queries
        * Made by resources within a VPC (EC2, Lambda, ...)
        * From on-premises resources that are using Resolver Inbound Endpoints
        * Leveraging Resolver Outbound Endpoints
        * Using Resolver DNS Firewall
    * Can send logs to CloudWatch Logs, S3 bucket, or Kinesis Data Firehose
    * Configurations can be shared with other AWS Accounts using AWS Resource Access Manager

### Amazon OpenSearch Service
* Amazon OpenSearch is successor to Amazon ElasticSearch
* In DynamoDB, queries only exist by primary key or indexes
* WIth OpenSearch you can search any field, even partially matches
* It's common to se OpenSearch as a complement to another database
* Two modes: managed cluster or serverless cluster
* Does not natively support SQL (can be enabled via a plugin)
* Ingestion from Kinesis Data Firehose, AWS IoT, and CloudWatch Logs
* Security through Cognitio & IAM, KMS encryption, TLS
* Comes with OpenSearch Dashboards (visualization)
* Public IP access
    * Accessible from the Internet with a public endpoint
    * Restrict access using Access Policies, Identity-based Policies, and IP-based Policies
* VPC Access
    * Specify VPC, Subnets, Security Groups, and IAM Role
    * VPC Endpoints and ENIs will be created (IAM Role)
    * You need to use VPN, Transit Gateway, managed network, or proxy server to connect to the domain
    * Restrict access using Access Policies and Identity-based Policies
        * Domain Access Policy - specify which actions a principal can perform on te domain subresources

## infrastructure Security

### Bastion Host

* We can use a Bastion Host to SSH into our private EC2 instances
* The bastion is in the public subnet which is then connected to all other private subnets
* Bastion Host security group must allow inbound from the internet on port 22 from restricted CIDR, for example the public CIDR of you corporation
* Security Group of the EC2 Instances must allow the Security Group of the Bastion Host, or the private IP of the Bastion Host

### Site to Site VPN

* Connect your on-premises VPN to your AWS VPC
* Use a Virtual Private Gateway (VGW) and a Customer Gateway (CGW)
* Virtual Private Gateway (VGW)
    * VPN connector on the AWS side of the VPN connection
    * VGW is created and attached to the VPC from which you want to create the Site-to-Site VPN connection
    * Possibility to customize the ASN (Autonomous System Number)
* Customer Gateway (CGW)
    * Software application or physical device on customer side of the VPN connection
    * What IP address to use?
        * Public Internet-routable IP address for you Customer Gateway device
        * If it's behind a NAT device that's enabled for NAT traversal (NAT_T), use the public IP address of the NAT device
    * Important step: enable *Route Propagation* for the Virtual Private Gateway in the route table that is associated with your subnets
    * If you need to ping your EC2 instances from in-premises, make sure you add the ICMP protocol on the inbound of you security groups
* AWS VPN CloudHub
    * Provide secure communication between multiple sites, you have multiple VPN connections
    * Low-cost hub-and-spoke model for primary or secondary network connectivity between different locations (VPN only)
    * It's a VPN connection so it goes over the public internet
    * To set it up, connect multiple VPN connections on the same VGW, setup dynamic routing and configure the routing tables

### Client VPN

* Connect from your computer using OpenVPN to your private network in AWS and on-premises
* Allows you to connect to your EC2 instances over a private IP (just as if you were in the private VPC network)
* Goes over public internet
* Authentication Types
    * Active Directory Authentication
        * Authenticate against Microsoft Active Directory (User Based)
        * AWS Manages Microsoft AD or on-premises AD through AD connector
        * SUpports MFA
    * Mutual Authentication
        * Use certificates to perform the authentication (Certificate Based)
        * Must upload the server certificate to AWS Certificate Manager
        * One client certificate for each user (recommended)
    * Single-Sign On
        * Authenticate against SAML 2.0 identity providers
        * Establish trust relationship between AWS and the identity provider
        * Only one identity provider at a time

### VPC Peering

* Privately connect two VPCs using AWS network
* Make them behave as if they were in the same network
* Must not have overlapping CIDRs
* VPC Peering connection is NOT transitive (must be established for each VPC that need to communicate with one anther)
* You must update route tables in each VPCs subnets to ensure EC2 instances can communicate with each other
* Good to know
    * You can create VPC Peering connection between VPCs in different AWS accounts/regions
    * You can reference a security group in a peered VPC (works cross accounts - same region)

### DNS Resolution in VPC

* DNS Resolution (enableDnsSupport)
    * decides if DNS resolution from Route53 Resolver server is supported for the VPC
    * True (default): it queries the Amazon Provider DNS Server at 169.254.169.253 or the reserved IP address at the base of the VPC IPv4 network range plus 2
* DNS Hostnames (enableDnsHostnames)
    * By default:
        * True -> default VPC
        * False -> newly created VPC
    * Won't do anything unless enable DnsSupport=true
    * If True, assigns public hostname to EC2 instance if it has a public IPv4
* Why enable both settings:
    * If you use custom DNS domain names in a Private Hosted Zone in Route53, you must set both these attributes to true

### VPC Endpoints

* Endpoints allow you to connect to AWS services using a private network instead of the public www network
* They scale horizontally and are redundant
* No more IGT, NAT, etc, to access AWS Services
* VPC Endpoint Gateway (S3 & DynamoDB)
* VPC Endpoint Interface (all except DynamoDB)
* In case of issue
    * Check DNS Setting Resolution in your VPC
    * Check Route Tables
* VPC Endpoint Gateway
    * Only works for S3 and DynamoDB, must create one gateway per VPC
    * Must update route tables entries (no security groups)
    * Gateway is defined at the VPC level
    * DNS resolution mist be enabled in the VPC
    * The same public hostname for S3 can be used
    * Gateway endpoint cannot be extended out of a VPC (VPN, DX, TGW, peering)
* VPC Endpoint Interface
    * Provision an ENI that will have a private endpoint interface hostname
    * Leverage Security Groups for security
    * Private DNS (setting when you create the endpoint)
    * Can be extended out of a VPC (VPN, DX, TGW, peering)
        * The public hostname of a service will resolve to the private Endpoint Interface hostname
        * VPC Setting "Enable DNS hostnames" and "Enable DNS resolution" must be enabled
    * Interface can be accessed from Direct Connect and Site-to-Site VPN

### VPC Endpoint Policy

* Control which AWS principals (AWS accounts, IAM users, IAM Roles) can use the VPC Endpoint to access AWS services
* Can be attached to both Interface Endpoint and Gateway Endpoint
* CAn restrict specific API calls on specific resources
* Doesn't override or replace Identity-based Policies or service-specific policies (e.g. S3 bucket policy)
* Note: can use aws:PrincipalOrdId to restrict access only within the Organization
* Exampple: 
    * SQS only accessible through a VPC Endpoint
        * set SQS Resource Policy which allows only VPC Endpoint to access the SQS queue
        * VOC Endpoint Policy to allow only organization
* Authorization logic
    * VPC Endpoint Policy + Identity-based Policy

### PrivateLink

* Exposing services in your VPC to other VPC
    * Option 1: make it public
        * Goes through the public www
        * Tough to manage access
    * Option 2: use VPC Peering
        * must create many peering relations
        * Open the whole network
    * Option 3: use PrivateLink
        * Most secure & scalable way t expose a service to 100s of VPCs (own or other accounts)
        * Does not require VPC peering, internet gateway, NAT, route tables...
        * REquires a network load balancer (SErvice VPC) and ENI (Customer VPC) or GWLB
        * If the NLB is in multiple AZ, and the ENIs in multiple AZ, the solution is fault tolerant

### Security groups

* Security Groups and NACLs
    * Incoming request, before it enters the subnet, it goes through a NACL
    * After it enters the subnet, it goes through the Security Group
    * Security Groups are stateful
        * If you allow an incoming request, the response is automatically allowed
    * NACLs are stateless
        * You must allow the incoming request and the outgoing response
    * NACLS are like a firewall which control traffic from and to subnets
    * One NACL per subnet, new subnets are assigned the Default NACLS
    * YOu define NACLS rules:
        * Rules have a number (1-32766), higher precedence with a lower number
        * First rule match will drive the decision
        * Example: If you define `#100 ALLOW 10.0.0.10/32` and `#200 ALLOW 10.0.0.10.32`, the IP address will be allowed because 100 has a higher precedence over 200 
        * The last rule is an asterisk (*) which denies all traffic
        * AWS recommends adding rules by increment of 100
    * Newly created NACLS will deny everything
    * Default NACL
        * Accepts everything inbound/outbound with the subnets it's associated with
    * Ephemeral Ports
        * For any two endpoints to establish a connection, they must use ports
        * Clients connect to a defined port, and expect a response on an ephemeral port
        * Different Operating Systems use different port ranges
            * IANA & MS Windows: 49152-65535
            * Linux: 32768-60999
        * NACLS with Ephemeral Ports
            * If a webserver makes a request to a database, the NACL on the webserver database must allow outbound TCP on port 3306 (db port)
            * The NACL of the DB subnet, allows inbound TCP on port 3306
            * The NACL of the DB subnet, allows outbound TCP on ephemeral ports (1025-65535)
            * The NACL of the webserver subnet, allows inbound TCP on ephemeral ports (1025-65535)
    * Security Groups
        * Operate at the instance level
        * Support allow rules only
        * Are stateful
        * All rules are evaluated before deciding whether to allow traffic
        * Applies to an EC2 instance when specified by someone
    * NACLs
        * Operate at the subnet level
        * Support allow and deny rules
        * Are stateless (response must be explicitly allowed by rules)
        * Rules are processed in order (lowest to highest)
        * Automatically applies to all instances in the subnets
* Security Groups - Outbound Rules
    * Default is allowed `0.0.0.0` anywhere
    * But we can remove and just allow specific prefixes
    * Managed Prefix List
        * A set of one or more CIDR blocks
        * Makes it easier to configure and maintain Security Groups and Route Tables
        * Customer-Managed Prefix List
            * Set of CIDRs that you define and managed bu you
            * Can be shared with other AWS accounts or AWS Organization
            * Modify to update many Security Groups at once
        * AWS-Managed Prefix List
            * Set of CIDRs for AWS services
            * You can't create, modify, share, or delete them
            * S3 , CloudFront, DynanomDB, GroundStation
* Security Group - Extras
    * Modifying Security Group Rule NEVER disrupts its tracked connections
    * Existing connections are kept until they time out
    * Use NACLs to interrupt/block connections immediately

### Transit Gateway

* For having transitive peering between thousands of VPC and on-premises, hub-and-spoke (star) connection
* Regional resource, can work cross-region
* Share cross-account using Resource Access Manager (RAM)
* You can peer Transit Gateways can talk with other VPC
* Route Tables: limit VPC can talk with other VPC
* Works with Direct Connect Gateway, VPC connections
* Support IP-multicast (not supported by any other AWS server)
* ECMP = Equal-cost multi-path routing
* Routing strategy to allow to forward a packet over multiple best path
* Use case: create multiple Site-ToSite VPN connections to increase the bandwidth of you connection to AWS
* Share direct Connect between multiple accounts

### Cloudfront Overview

* Content Delivery Network (CDN)
* Improves read performance, content is cached at the edge
* Improves users experience
* 216 Points of Presence globally (edge locations)
* DDoS protection, integration with Shield, AWS WAF
* Origins
    * S3 bucket
        * For distributing files and caching them at the edge
        * Enhanced security with CloudFront Origin Access Control
        * OAC is replacing Origin Access Identity
        * CLoudFront can be used as an ingress (to upload files to S3)
    * Custom Origin (HTTP)
        * Application Load Balancer
        * EC2 instance
        * S3 website (must first enable the bucket as a static S3 website)
        * Any HTTP backend you want
* CloudFront vs S3 Cross Region Replication
    * CloudFront:
        * Global Edge Network
        * Files are cached for a TTL (maybe a day)
        * Great for static content that must be available everywhere
    * S3 Cross Region Replication
        * Must be setup for each region you want replication to happen
        * Files are updated in near real-time
        * Read only
        * Great for dynamic content that needs to be available at low-latency in few regions
* Cloudfront Geo Restriction
    * You can restrict who can access your distribution
        * Whitelist: allow your users to access your distribution only if they're in one of the countries you specify
        * Blacklist: prevent your users from accessing your distribution if they're in one of the countries you specify
    * The country is determined using 3rd party Geo-IP database
    * Use case:
        * Copyright Laws to control access to content
* Signed URL & Signed Cookies
    * You want to distribute paid shared content to premium users over the world
    * You can use CloudFront signed URL or signed cookies
        * Include IRL expiration
        * Includes IP ranges to access the data from
        * Trusted signers (which AWS accounts can create signed URLs)
    * How long should the URL be valid for?
        * Shared content (movie, music) make it short (a few minutes)
        * Private content (private to user) can make it last for years
    * Signed URL = access to individual files (one signed URL per file)
    * Signed Cookies = access to multiple files (one signed cookie for many files)
    * CloudFront Signed URL vs S3 Pre-Signed
        * CloudFront:
            * Allow access to a path, no matter the origin
            * Account wide key-pair, only the root can manage it
            * Can filter by IP, path, date, expiration
            * Trusted signers (which AWS accounts can create signed URLs)
            * Can leverage caching features
        * S3 Pre-Signed URL:
            * Issue a request as the person who pre-signed the URL
            * Uses the IAM key of the signing IAM principal
            * Limited lifetime
            * Query parameters (no headers)
            * Can only be created by the owner of the S3 object
* Field Level Encryption
    * Protect user sensitive information through application stack
    * Adds an additional layer of security along with HTTPS
    * Sensitive information encrypted at the edge close to the user
    * Uses asymmetric encryption
    * Usage
        * Specify set of fields in POST requests that you want to be encrypted (up to 10 fields)
        * Specify the public key to encrypt them
* Origin Access Control with SSE-KMS
    * OAC supports SSE-KMS natively (as requests are signed with Sigv4)
    * Add a statement to the KMS Key Policy to authorize the OAC
* Origin Access Identity with SSE-KMS
    * OAI doesn't support SSE-KMS natively
    * Use Lambda@Edge to sign requests from CloudFront to S3
    * Make sue to disable OAI for this to work
* Other
    * CloudFront Authorization Header
        * Configure CloudFront distribution to forward the Authorization header using cache policy
        * Not supported for S3 origins
    * Restrict access to ALB
        * Revet direct access to your ALB or Custom Origins (only access through CloudFront)
        * First, configure CloudFront to add a Custom HTTP header to requests it sends to the ALB
        * Second, configure the ALB to only allow requests that contain the Custom HTTP header
        * Keep the custom header name and value secret
        * You can also restrict it further bt whitelisting the prefixes of the IP addresses of the CloudFront edge locations
    * Integration with Cognito
        * Use Cognito User Pools to secure your CloudFront distribution
        * User logs into Cognito Hosted UI
        * Cognito returns a JWT token
        * User sends the JWT token to CloudFront and Lambda@Edge verifies the token
        * If the token is valid, the user is allowed to access the content

### AWS WAF
* Protect your web applications from common web exploits (Layer 7 - http layer)
* Deploy an ALB (localized rules)
* Deploy on API Gateway (rules running at the regional or edge level)
* Deploy on CloudFront (rules globally on edge locations)
    * Used to front other solutions: CLB, EC2 instances, custom origins, S3 websites
* Deploy on AppSync (GraphQL)
* WAF is not DDoS protection
* Define Web ACL (Web Access Control List):
    * Rules can include IP addresses, HTTP headers, HTTP body, or URI strings
    * Protect from common attack - SQL injection and Cross-Site Scripting (XSS)
    * Size constraints, geo-match
    * Rate-based rules (to count occurrences of events)
* Rule Actions: Count | Block | Allow | Captcha
* Managed Rules
    * Library of over 190 managed rules
    * REady-to-use rules that are managed by AWS and AWS Marketplace Sellers
    * Baseline Rules Groups - general protection from common threats
        * AWSManagedRulesCommonRuleSet, AWSManagedRulesAdminProtectionRuleSet, ...
    * Use-case specific Rule Groups - protection for many AWS WAF use cases
        * AWSManagedRulesSQLiRuleSet, AWSManagedRulesWindowsRuleSet, AWSManagedRulesPHPRuleSet, AWSManagedRulesWordpressRuleSet, ...
    * IP Reputation Rule Groups - protection from known malicious IP addresses
        * AWSManagedRulesIPReputationList, AWSManagedRulesAnonymousIpList, ...
    * BotControl Managed Rule Groups - block and managed requests from bots
        * AWSManagedRulesBotControlRuleSet
* Logging
    * You can send our logs to an:
        * Amazon CloudWatch Logs log group - 5 MB per second
        * S3 bucket - 5 minutes interval
        * Kinesis Data Firehose - limited by Firehose quotas
* Enhance CloudFront Origin Security with AWS WAF & AWS Secrets Manager
    * WAF  -> CloudFront -> Custom Header from CloudFront -> WAF (filter rule for the custom header) -> ALB
    * Using Secrets manager we auto-rotate the HTTP header with a new value and updated it

### AWS Shield

Protect from DDoS attacks

* DDoS: Distributed Denial of Service - many requests at the same time
* AWS Shield Standard
    * Free service that is activated for every AWS customer
    * Provides protection from attacks such as SYN-UDP Floods, Reflection attacks and other layer 3/layer 4 attacks
* AWS Shield Advanced
    * Optional DDoS mitigation service ($3000 per month per organization)
    * Protect against more sophisticated attack on Amazon EC2, ELB, CloudFront, Global Accelerator, Route 53
    * 24/7 access to the DDoS Response Team (DRP)
    * Protect against higher fees during usage spikes due to DDoS
    * Shield Advanced automatic application layer DDoS mitigation automatically creates, evaluates and deploys AWS WAF rules to mitigate layer 7 attacks
    * CloudWatch Metrics
        * Helps you to detect if there is a DDoS attack
        * DDoSDetected - indicates whether a DDoS event is happening for a specific resource
        * DDoSAttackBitsPerSecond - number of bits per second during a DDoS event for a specific resource
        * DDoSAttackPacketsPerSecond - number of packets per second during a DDoS event for a specific resource
        * DDoSAttackRequestsPerSecond - number of requests per second during a DDoS event for a specific resource

### Firewall Manager

* Manage rules in all accounts of an AWS organization
* Security policy: common set of security rules
    * WAF rules (ALB, API Gateways, CloudFront)
    * Shield Advanced (ALB, CLB, NLB, Elastic IP, CloudFront)
    * Security Groups for EC2, ALB and ENI resources in VPC
    * AWS Network Firewall (VPC Level)
    * Amazon Route 53 Resolver DNS Firewall
    * Policies are created the region level
* Rules are applied to new resources as they are created (good for compliance) across all and future accounts in your Organization

* WAF vs Firewall Manager vs Shield
    * WAF, Shield and Firewall Manager are used together for comprehensive protection
    * Define your Web ACL rules in WAF
    * For granular protection of you resources, WAF alone is the correct choice
    * If you want to use AWS WAF across accounts, accelerate WAF configuration, automate the protection of new resources, use Firewall Manager with AWS WAF
    * Shield Advanced add additional features on top of AWS WAF, such as dedicated support from the Shield Response TEam and advanced reporting
    * If you are prone to frequent DDoS attack, consider purchasing Shield Advanced

### DDoS Protection

* DDoS resilience Edge location Mitigation
    * BP1
        * CloudFront
            * Web Application delivery at the edge
            * Protect from DDoS Common Attack (SYN Floods, UDP reflection, ...)
        * Global Accelerator
            * Access your application from the edge
            * Integration with Shield for DDoS Protection
            * Helpful if your backend is not compatible with CloudFront
    * BP3
        * Route 53
            * Domain Name Resolution at the edge
            * DDoS protection mechanism
* Best practices for DDoS mitigation
    * Infrastructure layer defense (BP1, BP3, BP6)
        * Protect Amazon EC2 against high traffic
        * That includes using Global Accelerator, Route 53, CloudFront, ELB
    * EC2 with Auto Scaling (BP7)
        * Helps scale in case of sudden traffic surges including a flash crowd or a DDoS attack
    * ELB (BP6)
        * ELB scales with the traffic increases and will distribute the traffic to many EC2 instances
* Application Layer Defense
    * Detect and filter web requests (BP1 and BP2)
        * CloudFront cache static content and serve it from edge locations, protecting your backend
        * AWS WAF is used on top of CloudFront and Application Load Balancer to filter and block requests based on request signature
        * WAF rate-based rules can automatically block IPs of bad actors
        * CloudFront can block specific geographic locations
    * Shield Advanced (BP1, BP2, BP6)
        * Shield Advanced automatic application layer DDoS mitigation automatically created, evaluates and deploys AWS WAF rules to mitigate layer 7 attacks
* Attack surface reduction
    * Obfuscating AWS resources (BP1, BP4, BP6)
        * Using CloudFront, API Gateway, Elastic Load Balancing to hide your backend resources (Lambda functions, EC2 instances)
    * Security Groups and Network ACLs (BP5)
        * Use Security Groups and Network ACLs to filter traffic based on specific IP a the subnet or ENI level
        * Elastic IP are protected by AWS Shield Advanced
    * Protecting API endpoints (BP4)
        * Hide EC2, Lambda, elsewhere
        * Edge-optimization mode or CloudFront + regional modes (more control for DDoS)
        * WAF + API Gateway burst limits, headers filtering use API keys

### API Gateway

* API Gateway is a fully managed service that makes it easy for developers to create, publish, maintain, monitor, and secure APIs at any scale
* AWS Lambda + API Gateway = Serverless Architecture
* Support for the WebSocket Protocol
* Handle API versioning (v1, v2)
* Handle different environments (dev, staging, prod, ...)
* Handle security (Authentication and Authorization)
* Create API keys, handle request throttling
* Swagger / OpenAPI import to quickly define APIs
* Transform and validate request and responses
* Generate SDK and API specifications
* Cache API responses
* Integrations
    * Lambda Function
        * Invoke Lambda function
        * Easy way to expose REST API backed by Lambda
    * HTTP
        * Expose HTTP endpoints in the backend
        * Example: internal HTTP API on premise, ALB, ...
        * Why? Add rate limiting, caching, user authentication, API keys, etc.
    * AWS Service
        * Expose any AWS API through API Gateway
        * Example: Start an AWS step function workflow, post a message to SQS
        * Why? Add authentication, deploy publicly, rate control, ...
* Endpoints Types
    * Edge-Optimized (default): For global clients
        * Requests are routed through the CloudFront Edge Locations (improves latency)
        * The API Gateway still lives in only one region
    * Regional
        * For clients within the same region
        * Could manually combine with CloudFront (more control over caching strategies and the distribution)
    * Private
        * Can only be accessed from within VPC using an interface VPC endpoints (ENI)
        * Use a resource policy to define access
* Security
    * User Authentication through
        * IAM roles (useful for internal applications)
        * Cognition (identity for external users)
        * Custom Authorizer (use custom logic)
    * Custom Domain Name HTTPS security though integration with AWS Certificate Manager (ACM)
        * Is using Edge-Optimized endpoint, then the certificate must be in us-east-1
        * If using Regional endpoint, the certificate must be n the API Gateway region
        * Must setup CNAME or A-alias record in Route53
* Resource Policy
    * Define who can access your API Gateway
* Cross VPC Same-Region Access
    * Create VPC Interface Endpoint in the private subnet in account A
    * Create Resource Policy in Gateway in account B
    * No VPC peering required!
* Throttling
    * Account Limit
        * API Gateway throttles requests at 10000 RPS across all APIs
        * Soft limit that can be increased upon request
    * In case of throttling -> 429 Too Many Requests (retriable error)
    * Can set Stage limit & Method limits to improve performance
    * Or you can define Usage Plans to throttle per customer
    * Note: One API Gateway that is overloaded and not limited can cause the other APIs to be throttled

### AWS Artifact (not really a service)
* Portal that provides customers with on-demand access to AWS compliance documentation and AWS agreements
* Artifact Reports - Allows you to download AWS security and compliance documents from third-party auditors, like AWS ISO certifications, PCI, SOC reports
* Artifacts Agreements - Allows oyu to review, accept, and track the status of AWS agreements such as the Business Associate Addendum, or the Health Insurance Portability and Accountability Act for an individual account or in your organization
* Can be used to support internal audit or compliance

### DNSSEC
* DNS Poisoning (Spoofing)
    * Web Browser -> DNS Resolver request for example.com -> Route 53 sends response 
    * attacker was able to inject into our DNS server a wrong IP address for example.com
    * we had been spoofed
* DNSSEC
    * a protocol for securing DNS traffic, verfies DNS data integrity and origin
    * Works only with Public Hosted Zones
    * Route 53 supports both DNSSEC for Domain Registration and Signing
    * DNSSEC signing
        * Validate that a DNS response came from Route53 and has not been tampered with
        * Route53 cryptographically signs each record in the Hosted Zone
        * Two Keys:
            * Managed by you: Key Signing Key (KSK): based non asymmetric CMK in AWS KMS
            * Managed by AWS: Zone signing Key (ZSK)
        * When enabled, Route53 enforces a TTL of one week for all records in the Hosted Zone (records that have a TTL less than one week are not affected)
* How to enable DNSSEC on a hosted zone
    * 1: Prepare for DNSSEC signing
        * Monitor zone availability
        * Lower TTL for records (recommended 1 hour)
        * Lower SOA minium for 5 minutes
    * 2: Enable DNSSEC signing and create a KSK
        * Enable DNSSEC in Route53 for your hosted zone (Console or CLI)
        * Make Route53 create a KSK in the console and link it to a Customer manage CMK
    * 3: Establish chain of trust
        * Create a chain of trust between the hosted zone and the parent hosted zone
        * By creating a Delegation Signer (DS) record in the parent zone
        * It contains a hash of the public key used to sign DNS records
        * Your registrar can be Route53 or a 3rd party registrar

### AWS Network Firewall
* Network Protection on AWS
    * We have seen so far:
        * NACL's
        * VPC security groups
        * AWS
        * Shield & Shield Advanced
        * Firewall Manager
    * But what if we want to protect in a sophisticated way our entire VPC?
* AWS Network Firewall
    * Protect your entire Amazon VPC
    * From Layer 3 to 7
    * Any direction, you can inspect
        * VPC to VPC traffic
        * Outbound to internet
        * Inbound from internet
        * To / from Direct Connect & Site-to-Site VPN
    * Internally, the AWS Network Firewall uses the AWS Gateway Load Balancer
    * Rules can be centrally managed cross-accounts by AWS Firewall Manager to apply to many VPCs
    * Supports 1000s rules
        * IP & Ports - example: 10000s of IP filtering
        * Protocol - example: block SMB protocol for outbound communications
        * Stateful domain list rule groups: only allow outbound traffic to *.mycorp.com or third-party software repo
        * General pattern matching using regex
    * Traffic filtering: Allow, drop, or alert for the traffic that matches the rules
    * active flow inspection to protect against network thread with intrusion-prevention capabilities (like Gateway Load Balancer, but all managed by AWS)
    * Sends logs of rule matches to S3, CloudWatch Logs, or Kinesis Data Firehose
    * Encrypted Traffic
        * supports Deep Packet Inspection for encrypted traffic Transport Layer Security
        * It decrypts the TLS traffic, inspects and blocks any malicious content, then re-encrypts the traffic and forwards it to the destination
        * integrated with AWS Certificate Manager

### Amazon SES
* Fully managed service to send emails securely, globally and at scale
* Allows inbound/outbound emails
* Reputation dashboard, performance insights, anti-span feedback
* Provides statistics such as email deliveries, bounces, feedback loop results, email open
* Supports Domain Keys identified Mail (DKIM) and sender policy framework
* Flexible IP deployment: shared, dedicated, and customer-owner IPs
* Sends emails using your application using AWS console, APIs, or SMTP
* Use case: transaction, marketing and bulk email communications
* Configuration Sets
    * Configuration sets help you to customize and analyze our email send events
    * Event destinations:
        * Kinesis Data Firehose: receives metrics for each email
        * SNS: for immediate feedback on bounce and complaint information
    * IP Pool management: use IP pools to send particular types of emails

## Identity and Access Management

### IAM Policies in Depth

* IAM Policies structure
    * Version
    * Policy Identifier (optional)
    * Statement
        * Statement Identifier (optional)
        * Effect: Allow | Deny
        * Principal: account, user, role, service to which this policy applies to
        * Action: "s3:ListBucket", "s3:*", "ec2:StartInstances"
        * Resource: "arn:aws:s3:::my_bucket", "arn:aws:s3:::my_bucket/*"
        * Condition: {"StringEquals": {"s3:prefix": ["home/"]}} (optional when this policy is in effect)
* IAM Policy
    * NotAction with Allow
        * Provide access to all the actions in an AWS service, except for the actions specified in `NotAction`
        * Use with the `Resource` element to provide scope for the policy, limiting
          the allowed actions to the actions that can be performed on the specified resources
    * NotAction with Deny
        * Use the NotAction element in a state with "Effect": "Deny" to deny access to all listed resources
          except  for the actions specified in the NotAction element
        * This combination does not allow the listed items, but instead explicitly denies the actions not listed
        * You must still allow explicitly actions that you want to allow
        * Restrict to One Region (NotAction) - we can only access eu-central-1 region
            * NotAction: ["cloudfront:*", "iam:*", "route53:*", "support:*"],
              Resource: "*",
              Condition: { "StringNotEquals": { "aws:RequestedRegion": "eu-central-1" } }
    * Principal Options in IAM Policies
        * AWS Account and Root user
        * IAM Roles
        * IAM Role sessions
        * IAM users
        * Federated users (via SAML)
        * AWS Services
        * All principals

|           | Allow                                                                                                           | Deny                                                                                                          |
|-----------|-----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Action    | "Statement": [{   "Effect": "Allow",   "Action": ["iam;*"],   "Resource": "*" }]  -> Allow IAM                  | "Statement": [{   "Effect": "Deny",   "Action": ["iam;*"],   "Resource": "*" }]  -> Deny IAM                  |
| NotAction | "Statement": [{   "Effect": "Allow",   "NotAction": ["iam;*"],   "Resource": "*" }]  -> Allow anything, not IAM | "Statement": [{   "Effect": "Deny",   "NotAction": ["iam;*"],   "Resource": "*" }]  -> Deny anything, not IAM | 

### IAM Condition Operators

* StringEquals / StringNotEquals - Case Sensitive, Exact Matching
    * `"StringEquals": {"s3:prefix": "home/"}`
    * `"StringEquals": {"aws:PrincipalTag/job-category": "iamuser-admin"}`
* StringLike / StringNotLike - Case Sensitive, optional Partial Matching using *,?
    * `"StringLike: {"s3:prefix": ["", "home/*/data", "home/${aws:username}/"]}"`
* DateEqual, DateLessThan,DateGreaterThan
    * `"DateGreaterThan": {"aws:TokenIssueTime": "2019-01-01T00:00:00Z"}`
* ArnLike / ArnNotLike - used specifically for ARN matching
* Bool - check for Boolean value
    * `"Bool": {"aws:secureTransport": "false"}`
* IpAddress / NotItAppress (CIDR format)
    * Resolves to the IP address that the request originates from
    * Public IP only - IpAddress does not apply to requests through VPC Endpoints
        * `"IpAddress": {"aws:SourceIp": "203.0.113.0/23"}`

### IAM Global condition keys

* RequestedRegion
    * The AWS region of the request
    * Used to restrict specific actions in specific AWS regions
    * Example: `"Condition": {"StringEquals": {"aws:RequestedRegion": "eu-central-1"}}`
    * When using a global AWS service (e.g. IAM, Cloudfront, Route53,Support), the AWS region is always us-east-1
    * Work around using NotActionAnd Deny
    * `"Statement": {"Effect": "Deny", "NotAction": ["cloudfront:*", "iam:*", "route53:*", "support:*"], "Resource": "*", "Condition": {"StringNotEquals": {"aws:RequestedRegion": "eu-central-1"}}}`
* PrincipalArn
    * Compare the ARN of the principal that made the request with the ARN specified in the policy
    * Note: For AIM roles, the request context returns the ARN of the role, not the ARN of the user that assumed the role
    * The following types of Principals are allowed:
        * IAM role
        * IAM user
        * AWS Root User
        * AWS STS federated user session
* SourceArn
    * Compare the ARN of the resource making a service-to-service request with the ARN that you specify in the policy
    * This key is included in the request context only if accessing a resource triggers an AWS service to call another service on behalf of the resource owner
    * Example: S3 bucket update triggers and SNS topic sns:Publish API, the policy set the value of the condition key to the ARN of the S# bucket
* CalledVia
    * Looks at the AWS service that made the request on behalf of the IAM User or Role
    * Contains an ordered list of each AWS service n the chain that made request on the principal's behalf
    * Supports:
        * athena.amazonaws.com
        * cloudformation.amazonaws.com
        * dynamodb.amazonaws.com
        * kms.amazonaws.com
* IP & VPC Conditions
    * aws:SourceIp
        * Public requesters I (for example EC2 IP if coming from EC2)
        * Not present if requests through VPC endpoints
    * aws:VpCSourceIp - requester IP through VPC Endpoints (private IP)
    * aws:SourceVpce - restrict access to a specific VPC Endpoint
    * aws:SourceVpc
        * Restrict to a specific VPC ID
        * Request must be made through a VPC Endpoint
    * Common to use these conditions with S3 Bucket Policies
* ResourceTag & PrincipalTag
    * Control access to AWS Resources using Tags
    * aws:ResourceTag - tags that exist on AWS Resources
        * sometimes you will see ec2:ResourceTag (service-specific)
    * aws:PrincipalTag - tags that exist on the IAM user or IAM role making the request

### IAM Permission Boundaries

* IAM Permission Boundaries are supported for users and roles (not groups)
* Advanced feature to use a managed policy to set the maximum permissions and IAM entity can get
* Can be used in combinations of AWS Organizations SCP
    * Use cases:
        * Delegate responsibilities to non administrators within their permission boundaries, for example create new IAM users
        * Allow developers to self-assign policies and manage their own permissions, while making sure they can't escalate their privileges
        * Useful to restrict specific user (instead of a whole account using Organizations & SCP)

### IAM Policy Evaluation Logic

* Simplified Evaluation Logic (Allow//Deny)
    1. By default, all requests are implicitly denied except for the AWS account root user, which has full access
    2. An explicit allow in an identity-based or resource-based policy overrides the default in 1.
    3. If a permission boundary, Organizations SCP, or session policy is preset, an explicit allow is used to limit actions. Anything not explicitly allowed is an implicit deny and may override the decision in 2.
    4. An explicit deny in any policy overrides any allows 
* More detailed one:
    ![IAM Policy Evaluation](/assets/images/AWS_IAM_Policy_Evaluation.png)

### Identity-Based Policies vs Resource-Based Policies

* Cross account:
    * attaching a resource-based policy to resource
    * Or using a role as a proxy
* When you assume a role (user, application or service), you give up your original permissions and take the permissions assigned to the role
* When using a resource-based policy, the principal doesn't have to give up his permissions
* Example: User in account A needs to scan a DynamoDB table in Account A and dump it in an S3 bucket Account B
    * resource based policy will make your life easier
* IAM Roles:
    * Helpful to give temporary permissions for a specific task
    * Allow a user/application to perform many actions in a different account
    * Permissions expire over time
* Resource-based policies:
    * Used to control access to a specific resource (resource-centric view)
    * Allow cross-account access
    * Permanent authorization (as long as it exists in the resource-based policy)
* Resource Policies & aws:PrincipalOrgID
    * aws:principalOrgId can be used in any resource policies to restrict access to accounts that are member of an AWS Organization

### Attribute based access control - ABAC

* Defines fine-grained permissions based on user attributes
* Example: department, job role, team name
* Instead of creating IAM roles for every team, use ABAC to group attributes to identify which resources a set of uses can access
* Allow operations when principal's tags matches the resource tag
* Helpful in rapidly-growing environments
* ABAC v RBAC
    * Role-Based Access Control (RBAC)
        * Access control based on roles, like Administrator, DB Admins, DevOps
        * Create different policies for different job functions
        * Disadvantage: must update policies when resources are added
    * Attribute-Based Access Control (ABAC)
        * scale permissions easily (no need to update policies when new resources added)
        * Permissions automatically granted based on attributes
        * Require fewer policies (you don't create different policies for different job functions)
        * Ability to use users attributes from corporate directory

### IAM MFA

* Users have access to your account and can possibly change configurations or delete resources in your AWS account
* You want to protect your Root Accounts and IAM users
* MFA = password you know + security device you own
* Main benefit of MFA: if a password is stolen or hacked, the account is not compromised
* Different options for MFA
    * Virtual MFA device - Google Authenticator, Authy
    * Security Key - YubiKey
    * Hardware MFA device - Gemalto, SurePassID
* S3 - MFA Delete
    * force users to generate a code on a device before doing important operations on S3
    * MFA will be required to:
        * Permanently delete an object version
        * Suspend Versioning on the bucket
    * MFA won't be required to:
        * Enable versioning
        * List deleted versions
    * To use MFA Delete, Versioning must be enabled on the bucket
    * Only the bucket owner (root account) can enable/disable MFA Delete
* IAM Conditions - MultiFactorAuthPresent
    * Condition key: `aws:MultiFactorAuthPresent`
    * Restrict access to AWS services for users not authenticated using MFA
    * Compatible with the AWS Console and the AWS Cli
* IAM Conditions - MultiFactorAuthAge
    * Condition key: `aws:MultiFactorAuthAge`
    * Grant access only withing a specified time after MFA authentication
* Not Authorized to Perform iam:DeleteVirtualMFADevice
    * This error happens even the user has the correct IAM permissions
    * This happens if someone began assigning a virtual MFA device to a user and then cancelled the process
    * The virtual MFA device is in a state where it's not assigned to any user
        * E.g created an MFA device for the user but never activates it
        * Must delete the existing MFA device then associate the new device
        * AWS recommends a policy that allows a user to delete their own virtual MFA device only if they are authenticated using MFA 
    * To fix this issue, the administrator must use the AWS CLI or AWS API to remove the existing but deactivated device

### IAM Credentials Report

* IAM users and the status of their passwords, access keys, and MFA devices
* Download using IAM Console, AWS API, AWS CLI, or AWS SDK
* Helps in auditing and compliance
* Generated as often as once every 4 hours
* Managing Aged Access Keys through AWS Config Remediations
    * AWS Config can be used to monitor and remediate non-compliant resources
    * Example: monitor IAM users with access keys older than 90 days
    * Create a Config Rule to check for non-compliant resources
    * Create a remediation action to delete the access key
    * Remediation action can be automated or require approval
    * Remediation action can be triggered by AWS Config or AWS Systems Manager

### IAM Roles and PassRole to Services

* Some AWS service will need to perform actions on your behalf
* To do so, we will assign permissions to AWS services with IAM Roles
* Common roles:
    * EC2 Instance Roles
    * Lambda Function Roles
    * Roles for CloudFormation
* Delegate Passing Permissions to AWS Service
    * You can grant users permissions to pass an IAM role to an AWS service
    * Ensure that only approved users can configure AWS service with an IAM role that grants permissions
    * Grant iam:PassRole permissions to the user's IAM user, role, or group
    * PassRole is not an API call (no CloudTrail logs generated)
        * Review CloudTrail log that created or modified the resource receiving the IAM role

### AWS STS - Security Token Service

* Allows to grant limited and temporary access to AWS resources
* Token is valid for up to one hour (must be refreshed)
* AssumeRole
    * Within your own account for enhanced security
    * Cross Account Access: assume role in target account to perform actions there
* AssumeRoleWithSAML
    * return credentials for users logged with SAML
* AssumeRoleWithWebIdentity
    * return credentials for users logged with and IdP (Facebook Login, Google Login, ...)
    * AWS recommends against using this, and using Cognito instead
* GetSessionToken
    * for MFA, from a user or AWS account root user
* Using STS to Assume a role
    * Define an IAM Role within your account or cross-account
    * Define which principals can access this IAM Role
    * Use AWS STS to retrieve credentials and impersonate the IAM Role you have access to (AssumeRole API)
    * Temporary credentials can be valid between 15 minutes to 1 hour
* Cross-Account Access With STS
    * Create a role in Account A that grants access to developers in Account B on service in Account a
    * Grant members of developers IAM Group permission to assume UpdateApp IAM Role
    * They AssumeRole
    * Get STS Role Credentials
    * Access the UpdateApp in Account A
* STS Version 1 & Version 2
    * Version 1:
        * By default, STS is available as global single endpoint
        * Only support AWS Regions that are enabled by default
        * Option to enable "All Regions"
    * Version 2:
        * Version 1 tokens do not work for new AWS Regions (e.g. me-south-1)
        * Regional STS endpoints are available in all AWS Regions* Reduce latency, built-in redundancy, increase session toke validity
        * STS Session Tokens from regional endpoints (STS Version 2) are valid in all AWS Regions
    * Error: An error occurred (AuthFailure) when calling the DescribeInstances operation: AWS was not able to validate the provided access credentials
        * To solve the problem:
            * Use the Regional STS endpoint (any region) which will return STS Tokens Version 2. Use the closes regional endpoint for lowest latency
            * By default, the AWS STS calls to the STS global endpoint issues session tokens which are of Version 1 (Default AWS Regions). You can configure STS global endpoint to issue STS Tokens Version 2 (All AWS Regions)
* STS External ID
    * Granting Access to 3rd Party using External ID
        * External ID is a piece of data that ca be passed to AssumeRole API
        * Allowing the IAM role to be assumed only if a certain value is present (External ID)
        * If solves the Confused Deputy Problem
        * Prevent any other customer from tricking 3rd party into unwittingly accessing your resources
* STS - Revoking IAM Role Temporary Credentials
    * Users usually have a long session duration time (e.g. 12 hours)
    * If credentials are exposed, they can be used for the duration of the session
        * Immediately revoke all permissions to the IAM role's credentials issued before a certain time
        * AWS attaches a new inline IAM policy to the IAM role that denies all permissions if the token is too old
        * Doesn't affect users who assumes the IAM role after you revoke session (don't worry about deleting the policy)

### EC2 Instance Metadata Overview

* Information about an EC2 instance (e.g. hostname, instance type, network settings, ...)
* Can be accessed from withing the EC2 instance itself by making a request to the EC2 metadata service endpoint `http://169.254.169.254/latest/meta-data`
* Can be accessed using EC2 API or CLI tools (e.g. curl or wget)
* Metadata is stored in key-value pairs
* Useful for automating tasks such as setting up an instance's hostname, configuring networking, or installing software
* Example:
    * ami-id, block-device-mapping, instance-id, instance-type, network
    * hostname, local-hostname, local-ipv4, public-hostname, public-ipv4
    * iam - InstanceProfileArn, InstanceId
    * iam/security-credentials-role-name - temporary credentials for the role attached to your instance
    * placement/ - launch region, launch AZ, placement group name, ...
    * security-groups - names of security groups 
    * tags/instance - tags attached to the instance
* Instance Role - How it works
    * Calls to metadata endpoint /latest/meta-data/iam/security-credentials/role-name
    * Returns temporary credentials for the IAM role attached to the instance
* Restrict Access to Instance Metadata
    * You can use local firewall rules to disable access for some or all processes
        * iptables for Linux
    * Turn of access using AWS Console or AWS CLI (HttpEndpoint=disabled)

### IMDSv2 vs IMDSv1

* IMDSv1 is accessing http://169.254.169.254/latest/meta-data directly
* IMDSv2 is more secure and is done in two steps:
    * Get Session Token (limited validity) - using headers & PUT
    * Use Session Token in IMDSv2 calls - using headers
* Requiring the usage of IMDSv2
    * Both IMDSv1 and IMDSv2 are available (enabled by default)
    * The CloudWatch Metric MetadataNoToken provide information on how much MDSv1 is used
    * You can force Metadata Version 2 at Instance Launch using either
        * AWS console
        * AWS CLI: HttpTokens:required
    * You can require IMDSv2 when registering an AMI: --imds-support v2.0
* Require EC2 Role Credentials to be retrieved from IMDSv2
    * AWS credentials provided by the IMDS now include an ec2:RoleDeliver IAM context key
        * 1.0 for IMDSv1
        * 2.0 for IMDSv2
    * Attach this policy to the IAM Role of the EC2 instance
    ```
    {"Effect": "Deny", "Action": "*", "Resource":"*", "Condition": {"NumericLessThan": {"ec2:RoleDelivery": "2.0"}}}
    ``` 
    * Or attach it to an S3 bucket to only required IMDSv2 when API calls are made by an IAM role
    * Or attach is as an SCP in your account
    * IAM Policy or SCP to force IMDSv2:
    ```
    {"Effect": "Deny", "Action": "ec2:RunInstance", "Resource":"*", "Condition": {"StringNOtLike": {"ec2:RoleDelivery": "2.0"}}}
    ```

### S3 - Authorization Evaluation process

* User context
    * Is the IAM principal authorized by the parent AWS account (IAM Policy)?
    * If the parent owns the bucket or object, then bucket policy/ACL or object ACL is evaluated
    * If the parent owns the bucket/object, it can grant permissions to its IAM principals using identity-based policies or resource-based policies
* Bucket Context
    * Evaluates the policies of the AWS account that owns the bucket (check for explicit deny)
* Object Context 
    * Requester must have permission from the object owner (using object ACL)
    * If bucket owner = object owner; then access granted using Bucket Policy
    * If bucket owner != object owner, then access granted using object owner ACL
    * If you want to own all objects in our bucket and only use Bucket Policy and IAM-Based Policies to grant access, enable Bucket Owner Enforced Setting for Object Ownership
        * Then Bucket and objects ACLs can't be edited and are no longer considered for access

### S3 - Cross Account Access to Objects in S3 Buckets

* Use one of hte following to grant cross-account access to S3 objects
* IAM Policies and S3 bucket Policy
* IAM Policies and Access Control Lists (ACLs)
    * Bucket Policy and Access Control Lists (ACLs)* ACLs only works if Bucket Owner Enforced setting = Disabled
    * By default, all newly created buckets have Bucket Owner Enforced = Enabled
    * If a user from Account A uploads an object to a bucket in Account B, the object is owned by Account A and Account B has no access to it
    * Account A user can make sure it gives up object ownership by granting the object ownership to the Account B administrator
    * Using ACL: with adding condition that requests include ACL-specific headers, that either:
        * Grant full permissions explicitly
        * Use Canned ACL
    * Using S3 Object Ownership
* Cross-Account IAM Roles
    * You can use cross-account IAM roles to centralize permission management when providing cross-account access to multiple services
    * Example: access to S3 objects that are stored in multiple S3 buckets
    * Bucket Policy is not required as the API calls to S3 come from within the account (through assumed IAM Role)
* Canned ACL Object Permissions in Cross-Account Setting
    * You can require x-amz-acl header with a Canned ACL granting full control permission to the bucket owner
    * To require the x-amz-acl header in the request, specify s3:x-amz-acl in the condition element of the policy
    * Example: "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}

### S3 - Samples S3 Bucket Policies

* Bucket Policies Force HTTPS in Flight
    * Force all requests to use HTTPS
    * Example:
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:*",
                "Resource": "arn:aws:s3:::mybucket/*",
                "Condition": {
                    "Bool": {
                        "aws:SecureTransport": "false"
                    }
                }
            }
        ]
    }
    ```
* Restrict Public IP address
    * Restrict access to a bucket to a specific IP address
    * Example:
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:*",
                "Resource": "arn:aws:s3:::mybucket/*",
                "Condition": {
                    "NotIpAddress": {
                        "aws:SourceIp": [corparete ip cidr]
                    }
                }
            }
        ]
    }
    ```
* Restrict by User Id
    * Restrict access to a bucket to a specific AWS account
    * Example:
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:*",
                "Resource": "arn:aws:s3:::mybucket/*",
                "Condition": {
                    "StringNotEquals": {
                        "aws:userId": "AWS-account-ID"
                    }
                }
            }
        ]
    }
    ```

### S3 - VPC Endpoint Strategy for S3

* Access S3 through VPC Gateway
    * No costs
    * Only accessed by resources in the VPC where it's created
    * Make sure "DNS Support" is Enabled
    * Keep on using the public DNS of Amazon S3
    * Make sure Outbound rules of SG of EC2 instance allows traffic to S3
* VPC Interface Endpoint for S3
    * ENI(s) are deployed in our Subnets (Security Groups can be attached to ENIs)
    * Can access from on-premise using Direct Connect or VPN
    * Costs $0.01 per hour per AZ
    * Both VPC settings "Enable DNS hostnames" and "Enable DNS Support" must be 'true'
    * No "private DNS name" option for VPC Interface Endpoint for S3
* VPC Endpoint Strategies
    * Single VPC Architecture
        * If everything in VPC use Gateway Endpoint
        * If you have Direct Connect of S2S VPN use Interface VPN
    * Multi-VPC Centralized Architecture - traffic goes through a central VPC
        * Create Interface Endpoint in the Central VPC and all traffic from Direct Connect and/or S2S VPC goes through that
        * The other VPCs get Gateway endpoints, since it is free
* VPC Endpoint Restrictions aws:SourceVpc and aws:sourceVpce
    * aws:SourceVpc - restrict access to a specific VPC ID
    * aws:SourceVpce - restrict access to a specific VPC Endpoint
    * Example:
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:*",
                "Resource": "*",
                "Condition": {
                    "StringNotEquals": {
                        "aws:SourceVpc": "vpc-123456"
                    }
                }
            }
        ]
    }
    ```
* Ip Address Restrictions aws:SourceIp and aws:VpcSourceIp
    * aws:SourceIp - restrict access to a specific IP address or range
    * aws:VpcSourceIp - restrict access to a specific IP address or range from a VPC Endpoint
    * Example:
    ```
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Deny",
                "Principal": "*",
                "Action": "s3:*",
                "Resource": "*",
                "Condition": {
                    "NotIpAddress": {
                        "aws:SourceIp": [
                            "192.0.2.0/24",
                            "203.0.113.0/24"
                        ]}
                    }
            }
    }
    ```

### Regain Access to Locked S3 bucket

* If you incorrectly configured your S3 bucket policy to deny access to everyone (Deny s3:, Principal: "*")
* You must delete the S3 bucket policy using the AWS account root user
* Note: Deny statements in IAM policies do not affect the root account

### S3 - Block Public Access Settings

* These settings were created to prevent company data leaks
* If you know your bucket should never be public, leave these on
* Can be set at the account level

### S3 - Access Points

* Example: S3 bucket with lots of data, (/finance, /marketing, /hr)
* Create an access point (can be Read/Write) for each folder and department
* Each access point has its own security
* Benefit: Simple Bucket policy, no need to create complex policies
* Access Points simplify security management for S3 Buckets
* Each access point has
    * its own DNA name (Internet Origin or VPC Origin)
    * an access point policy (similar to bucket policy) - manage security at scale
* VPC Origin
    * We can define the access point to be accessible only from within the VPC
    * You must create a VPC Endpoint to access the Access Point (Gateway of Interface Endpoint)
    * THe VPC ENdpoint Policy must allow access to the target bucket and Access Point

### S3 - Multi-Region Access Points

* Provide a global endpoint that span S3 buckets in multiple AWS regions
* Dynamically route requests to the nearest S3 bucket (lowest latency)
* Bi-directional S3 bucket replication rules are created to keep data in sync across regions
* Failover Controls - allows you to shift requests across S3 buckets in different regions within minutes (Active-Active or Active-Passive)

### S3 - CORS

* Cross-Origin Resource Sharing (CORS)
* Origin = scheme (protocol) + host (domain) + port
  * example: https://www.example.com (implied port is 443)
* Web Browser based mechanism to allow requests to other origins while visiting the main origin
* Same origin: http://example.com/aa1 and http://example.com/aa2 (path is not part of origin)
* Different origins: http://www.example.com and http://other.example.com
* The requests won't be fulfilled unless the other origin allows for the requests, using CORS Headers (example: Access-Control-Allow-Origin)
* If a client makes a cross-origin request on our S3 bucket, we need to enable the correct CORS headers
* It's a popular exam question
* You can allow for a specific origin or for * (all origins)

### Cognito User Pools (CUP)

* Create a serverless database of user for your web and mobile apps
    * Simple login: Username (or email) / password combination
    * Password reset
    * Email & Phone Number Verification
    * Multi-Factor Authentication (MFA)
    * Federated Identities: Facebook, Google, SAML, OpenID Connect, ...
* Feature: block users if their credentials are compromised
* Login sends back a JWT token
* Integrations
    * CUP integrated with API Gateway and Application Load Balancer

### Cognito Identity Pools (Federated Identities)

* Get identities for "users" so they obtain temporary AWS credentials
  * We do not trust those users, that's why no IAM roles, and also doesn't scale
* Your identity pool (e.g. identity source) can include:
    * Public providers (Login with Amazon, Facebook, Google, Apple)
    * Users in an Amazon Cognito user pool
    * OpenID Connect Providers & SAML Identity Providers
    * Developer Authenticated Identities (custom login server)
    * Cognito Identity Pools allow for unauthenticated (guest) access
* Users can then access AWS services directly or through API Gateway
    * The IAM policies applied to the credentials are defined in Cognito
    * They can be customized based on the user_id for fine grained control

* IAM roles
    * Default IAM roles for authenticated and guest roles
    * Define rules to choose the role for each user based on the user's ID
    * You can partition your users access using policy variables
    * IAM credentials are obtained by Cognito Identity Pools through STS
    * The roles mus have a "trust" policy of Cognito Identity Pools

### Cognito User Pool User Groups

* Collection of users in a logical group in Cognito User Pool
* Defines the permissions for users in the group by assigning IAM role to the group
* User can be in multiple groups:
    * Assign precedence values to each group (lower will be chosen and its IAM role will be applied)
    * Choose from available IAM roles by specifying the IAM role ARN
You can't create nested groups

### Identity Federation & Cognito

* Give users outside of AWS permissions to access AWS resources in your account
* You don't need to create IAM users (user management outside of AWS)
* Use cases:
    * A corporate has its own identity system (e.g. Active Directory)
    * Web/Mobile application that needs access to AWS resources
* Identity Federation can have many flavours:
    * SAML 2.0
    * Custom Identity Broker
    * Web Identity Federation (Google, Facebook, Amazon)
    * Cognito Identity Pools

* SAML 2.0
    * Security Assertion Markup Language
    * Open standard used by many identity providers
        * Supports integration with Microsoft Active Directory Federations Services (ADFS)
        * Or any SAML 2.0-compatible IdPs with AWS
    * Access to AWS Console, AWS CLI, or AWS API using temporary credentials
        * No need to create IAM Users for each of your employees
        * Need to setup a trust between AWS IAM and SAML 2.0 Identify Provider (both ways)
    * Under the hood: Uses the STS API AssumeRoleWithSAML
    * SAML 2.0 Federation is the "old way", Amazon Single Sign-On (AWS SSO) Federation is the new managed and simpler way
* Custom Identity Broker Application
    * Use only if Identity Provider is NOT compatible with SAML 2.0
    * The identity Broker Authenticates users & requests temporary credentials from AWS
    * THe Identity Broker mist determine the appropriate IAM role
    * Uses the STS API AssumeRole or GetFederationToken
* Web Identity Federation
    * Without Cognito
        * Not recommended by AWS - use Cognito instead
    * With Cognito
        * Preferred over for Web Identity Federation
            * Create IAM Roles using Cognito with the least privilege needed
            * Build trust between the OIDC IdP and AWS
        * Cognito benefits:
            * Supports anonymous users
            * Supports MFA
            * Data Synchronization
        * Cognito replaces a Token Vending Machine (TVM)
    * IAM Policy
        * After being authenticated with Web Identity Federation, you can identify the user with an IAM policy variable
        * Examples:
            * cognito-identity.amazonaws.com:sub
            * www.amazon.com:user_id
            * graph.facebook.com:id
            * accounts.google.com:sub

### SAML 2.0 Metadata File Troubleshooting

* Error: Response signature invalid (service:AWSSecurityTokenService; status code: 400; error code: InvalidIdentityToken)
    * Reason: federation metadata of the identity provider does NOT math the metadata of the IAM identity provider
        * Example: metadata file might have changed to update an expired certificate
    * Resolution:
        * Download the updated SAML 2.0 metadata file from the identity provider
        * Update the IAM identity provider using AWS CLI `aws iam update-saml-provider` 

### AWS IAM Identity Center (successor to AWS Single Sign-On)

* One login (single sign-on) for all your
    * AWS accounts in AWS Organizations
    * Business cloud applications (e.g. Saleforce, Box, Microsoft 365)
    * SAML2.0-enabled applications
    * EC2 Windows Instances
* Identity providers
    * Built-in identity store in IAM Identity Center
    * 3rd party: Active Directory (AD), OneLogin, Okta, ...

* Fine-grained Permissions And Assignments
    * Multi-Account Permissions
        * Application Access* Manage access across AWS accounts in your AWS Organization
        * Permission Sets - a collection of one or more IAM Policies assigned to users and groups to define AWS access
    * Application Assignments
        * SSO access to many SAML 2.0 business applications (Salesforce, Box, Microsoft 365, ...)
        * Provide required URLs, certificates, and metadata
    * Attribute-Based Access Control (ABAC)
        * Fine-grained permissions based on users' attributes stored in IAM Identity Center Identity Store
        * Example: cost center, title, locale, ...
        * Use case: Define permissions once, then modify AWS access by changing the attributes

### AWS Directory Services

* What is Microsoft Active Directory (AD)?
    * Found on any Windows SErver with AD Domain Services
    * Database of objects: User, Accounts, Computers, Printers, FIle Shares, Security Groups
    * Centralized security management, create account, assign permissions
    * Objects are organized in trees
    * A group of trees is a forest
* What is ADFS (AD Federation Services)?
    * ADFS provides Single Sign-On across applications
    * SAML across 3rd party: AWS Console, Dropbox, ...
* AWS Directory Services 
    * AWS Managed Microsoft AD
        * Create your own AD in AWS, manage users locally, supports MFA
        * Establish "trust" relationships with your on-premise AD
    * Ad Connector
        * Directory Gateway (proxy) to redirect to on-premises AD, supports MFA
        * Users are managed on the on-premises AD
    * Simple AD
        * AD-compatible managed directory on AWS
        * Cannot be joined with on-premises AD
        * No MFA
* AWS Directory Services AWS Managed Microsoft AD
    * Managed Service: Microsoft AD in your AWS VPC
    * EC2 Windows Instances
        * EC2 Windows isntances can join the domain and run traditional AD applications (sharepoint, etc)
        * Seamlessly Domain Join Amazon EC2 Instances from Multiple Accounts & VPCs
    * Integrations
        * RDS for SQL Server, AWS Workspaces, Quicksight, ....
        * AWS SSO to provide access to 3rd party applications
    * Standalone repository in AWS or joined to on-premises AD
    * Multi AZ deployment of AD in 2 AZ, # of DC (Domain Controllers) can be increased for scaling
    * Automated backups
    * Automated Multi-Region replication of your directory
    * Integrations
        * AWS Managed Microsoft AD DC using two-way Forest trust with on-premises AD
    * Connect to on-premise AD
        * Ability to connect your on-premise Active Directory to AWS Managed Microsoft AD
        * Must establish a direct connect (DX) or VPN connection
        * Can setup three kinds of forest trust:
            * One-way trust: AWS -> On-Premise
            * One-way trust: On-Premise -> AWS
            * Two-way trust: AWS <-> On-Premise
        * Forest trust is different than synchronization (replication is not supported)
    * Solution Architecture: Active Directory Replication
        * You may want to create a replica of your AD on EC2 in the cloud to minimize latency of in case DX or VPN goes down
        * Establish trust between the AWS Managed Microsoft AD and EC2
* AWS Directory Services AD Connector
    * AD Connector is a directory gateway to redirect directory requests to your on-premises Microsoft Active Directory
    * No caching capability
    * Manage users solely on-premises, no possibility of setting up a trust
    * VPN or Direct Connect
    * Doesn't work with SQL Server, doesn't do seamless joining, can't share directory
* AWS Directory Services Simple AD
    * Simple AD is an inexpensive Active Directory - compatible service which the common directory features
    * Supports joining EC2 instances, manage users and groups
    * Doesn't support MFA, RDS SQL server, AWS SSO
    * Small: 500 users, large: 5000 users
    * Powered by Samba 4, compatible with Microsoft AD
    * Lower cost, low scale, basic AD compatible, or LDAP compatibility
    * No trust relationship