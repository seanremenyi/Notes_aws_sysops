# Notes_aws_sysops

## EC2
changing instance type
- this only works for ebs backed instance
- stop the instance
- Instance setting -> change instance type
- Start instance

Enhanced Networking
enhanced Networking (SR-IOV)
- higher bandwidth, higher pps (packet per second), lower latency
- Option 1: Elastic Networok Adaptor (ENA) up to 100 GBps
- Option 2: Intel 82599 VF up to 10 GBps - Legacy
- Works for newer generation EC2 instances
Elastic Fabric Adapter( EFA)
- Improved ENA for HPC (clusters) only works for Linux
- Great for inter-node communications, tightly coupled workloads
- Leverages Message Passing Interface (MPI) standard
- Bypasses the underlying Linux OS to provide low-latency, reliable transport

Placement Groups
- Sometimes you want control over the EC2 instances placement srategy
- That strategy can be defined using placement groups
- When you create a placement group, you specify one of the following strategies for the group:
Cluster 
- clusters instance into a lot latency group in a single Availabiliy zone
- Same rack (same hardware)
- Same AZ
- low latency
- 10 GBps network
- Pro: Great network (10 GBps bandwidth between instances)
- Cons: If the rack fails, all instances fails at the same time
- Use Cases: Big data job that needs to complete fast
- Application that needs extremely low latency and high network throughput
Spread - spreads instances across underlying hardware (max 7 instance per group per AZ) - critical applications
- instances located on different hardward
- Pros: Can span across Availability zones
- Reduced risk in simultaneous failure
- EC2 instances are on a different physical hardware
- Cons: Limited to 7 instances per AZ per placement group
- Use CAses: Application that needs to maximize high availability
- Critical Applications where each instance must be isolated from failure from each other
Partition - spreads instances across many different partitions (which rely on different sets of racks) within an AZ. Scales to 100s of EC instances per group (Hadoop, Cassandra, Kafka)
- partitions represents a rack 
- up to 7 partitions per az
- Can span across multiple AZs in the same region
- Up to 100s of EC2 instances
- The instances in a partition do not share racks with the instances in the other partitions
- A partition failure can affect many ec@s but wont affect other partitions
- EC2 instances get access to the partitions information as metadata
- Use Cases: Big Data: HDFS, HBase, Cassandra, Kafka

Shudown Behaviour (option on launch)
How should the instance react when shutdown is done using the OS, can you terminate?
- Stop (defualt)
- Terminate
This is not applicable when shutting down from the AWS console
CLI attribute: InstanceInitiatedShutdownBehaviour

Termination Protection
Enable termination protection: To protect against termination in AWS console or cli
ex: We have an instance where shutdown behaviour = terminate and enable terminate protection is ticked
We shutdown the instance from the OS, what will happen
The instance will still be terminated (we are doing it from the OS and not from console)

Launch Troubleshooting:
InstanceLimitExceeded: if you get this error, it means that you have reached your limit of max number of vCPUs per region
- On demand instance limits are set on a per-region basi
- Ex: if you run on-demand (A,C,D,H,I,M,R,T,Z) instance types you'll have 64 vCPUS (default)
- Resolution: Either launch the instance in a different region or request AWS to increase your limit of the region
- Note: vCPU-based limits only apply to running On-demand instances and spot instances
InsufficientInstanceCapacity: If you get this error, it means AWS does not have enough On demand capacity in the particular AZ where the instance i launched
-Resolution: Wait for few mins before requesting again
- If requesting more than 1 request, break down the requests. If you need 5 instances, rather than a single request of 5, request one by one.
- If urgent, submit a request for a different instance type now, which can be resized later
Instance Terminated Immediately (goes from pending to terminated)
- You've reached your EBS vloume limit
- An EBS snapshot is corrupt
- The root EBS volume is encrypted and you do not have permissions to access the KMS key for decryption
- The instance store-backed AMI that you used to launch the instance is missing the required part (an image.part.xx file)
- To find the exact reason, check out the EC2 console of AWS - instances - Description tab, note the reason next to the state transition reason label

SSH Troubleshooting
-Make sre the private key(pem file) on your linux machine has 400 permissions, else you will get "unprotected prifate key file" error
- Make sure the username for the OS is given correctly when logging via SSH, else you will get "Host key not found", "permission denied", or "Connection closed by [instance] port22" error
Possible resons for "Connection timed out" to EC2 instance vis ssh:
- SG is not configured correctly
- Check the route table for the subnet (routes traffice destined outside VPC to IGW) 
- NACL is not configured correctly
- Instance doesn't have a public IPv4
- CPU load of the instance is high
SSH v EC2 instance connect
Connect using SSH: rule (can have one ip)

instance connect: Allows aws ip range /29
user uses es2 instance connet api which pushes one-time ssh public key (valid for 60seconds) to connect to the instance
why you dont need your ssh key
interface with the ec2 instance connect interface

Puchasing options:
on-demand :short workload, predictable pricing
pay for what you use
-Linux: billing per second, after the first minute
All other OS (windows), billing per hour
has highest cost but no upfront payment
no long term commitment
Recommended for short term and un-interrupted workloads, where you can't predict how the application will behave

Reserved: (minimum 1 year)
- Reserved instance: long workloads
up to 75% discount compared to on-demand
reservation period 1 or 3 year (3 year more discount)
purchasing options, no upfront, partial upfront, all upfront (all upfront most discount)
reserve a specific instance type
recommended for stead-state usage applications (tink databas)
- Convertible reserved instances: long workloads with flexible instances
can change the ec2 type
up to 54% discount
- Scheduled reserved Instnaces: ex. every thursday between 3 and 6pm
launch within time window you reserve
when you require a fraction of day/week/month
still commitment over 1 to 3 years

Spot instances: short workloads, cheap, can lose instances (less reliable)
can get up to 90% discounts compared to on-deman
instances that you can "lose" at any point of time if your max price is less than the current spot price
the most cost-efficient instances in aws
useful for workloads that are resillient to failure (batch jobs, data analysis, image processing, any distributed workloads, workloads with a flexible start and end time)
not suitable for critical jobs or database

Dedicated hosts: book an entire physical server, control instance placement
An amazon ec2 dedicated host is a physical server with ec2 instance capcaity fully dedicated to your use. Dedicated hosts can help you address compliance requirements and reduce costs by allowing you to use your existing server bound software licences.
Allocated for your account for a 3-year period reservation
more expensive
Useful for software that have complicated licencing models (BYOL- bring your own licence)
Or for companies that have strong regulatory or compliance needs

EC2 dedicated instances:
Instances running on hardware that's dedicated to you
May share hardware with other instances in same account
No control over inststance placement (can move hardware after stop/start)
per instance billing
less access to underlying hardware than dedicated hosts

Spot instances:
define max spot price and get the instance while current spot price < max
- the hourly spot price varies based on offer ad capacity
- if the current price > your max price you can choose to stop or terminate your instancewith 2 minutes grace period
Spot Block
- block spot instance during a specified time frame (1 to 6 hours) without interruptions
- in rare instances, the instance may be reclaimed
- used for batch jobs, data analysis or workloads that are resilient to failures

requests type: one time or persistent
persistent - valid from to valid until parameters
so if a spot instance is stopped in persistent mode a new spot request will be made
you can only cancel spot requests that are open, active or disabled. Canceling a spot request ddoes not terminate instances 
you must first cancel a spot request, and then terminate the associated spot instances

spot fleets:
-spot fleets = set of spot instances + (optional) on-demand instances
The spot fleet will try to meet the target capacity with price constraints
- Define possible launch pools: instance type(m5.large), os, Availability zone
- Can have multiple launch pools that the fleet can choose
- spot fleet stops launching instances when reaching capacity or max cost
Strategies to allocate spot instances:
- lowestprice: from the pool with the lowest price (cost optimization, short workload)
- diversified: distributed across all pools (great for availability, long workloads)
- capacityOptimizes: pool with the optimal capacity for the number of instances

Spot fleets allow us to automatically request Spot instances with the lowest price

Instance types:
R: applications that needs a lot of RAM - inmemory caches
C: applications that needs good CPU - compute/databse
M: applications that are balanced (think medium)- general - web/app
I: applications that need good local i/o (instance storae) databases
G: applications that need a GPU - video rendering/machine learning
T2/T3: burstable instances (up to a capacity)
T2/T3 - unlimited: unlimited burst

burstable instances:
- AWS has a concept of burstable instances (t2/t3)
- Burst overall, the instance has ok cpu performance
- hen the machine needs to process something unexpected (a spike in load for example), it can burst, and CPU can be very goo
- If the machine bursts, it utilizes "burst credits"
- If all the credits are gone, the CPU becomes BAD
- If the machine stops bursting, credits accumulate over time
- Burstable instances can be amazing to handle unexpected traffic and getting the insurance that it will be handled correctly
- If your instance consistently runs low on credit, you need to move to a different kind of non-burstable instance
- Credit usage can be seen in cloudwatch
- bigger the instance the faster you earn credits
T unlimited:
- unlimited burst credit balance
- you pay extra money if you go over your credit balance, but you don't lose performance
- overall, it is a new offering, so be careful, costs could go high if you're not monitoring the health of your instances
- if average CPU usage over aa 24-hour period exceeds the baseline, the instance is billed for additional usage per vcpu/hour
- Be careful, costs could go high if you're not monitoring the cpu health of your instances
what happens when credit are exhausted:
after the credits are exhausted, the measured cpu utilization drops

Elastic IPS
-when you stop and start an EC2 instance, it changes its public ip
If you need to have a fixed IP, you need an Elastic IP
An Elastic IP is a public IPv4 you own as long as you don't delete it
You can attach it to one instance at a time
You can remap it across instances
You don't pay for the Elastic IP if it's attached to a server
you pay for the elastic ip if it's not attached to the server
- With an elastic ip address, you can mask the filure of an insatance or software by rapidly remapping the address to another instance in your account
- You can only have 5 Elastic IP in your account (you can ask AWS to increase that)
How you can avoid using Elasitc IP:
- Always think if other alternatives are available to you
- You could use a random public IP and register a DNS name to it
- Or use a Load Balancer with a static hostname

EC2 cloudwatch metrics:
AWS provided metrics (AWS pushes them):
- Basic monitoring (defualt): metrics are collected at a 5 min interval
- Detailed Monitoring (paid): metrics are collected at a 1 minute interval
- Includes CPU, Network, Disk, Status Check Metrics
Custom metrics (yours to push):
- Basic resolution: 1 minute resolution
- High resolution: all the way doesn to 1 second resolution
- Include RAM, application level metrics
- Make Sure the IAM permissions on the EC2 instance rol are correct

RC2 included metrics:
- CPU: CPU utilization + credit usage/balance
- Network: Network in/out
- Status check: 
Instance status = check the EC2 VM
System status = check the underlying hardware
- Disk Red/Write for ops/bytes (only for instance store)
- RAM is not included

Unified CloudWatch Agent
- for virtual servers (EC2 instances, onpremises servers, ...)
- collect additional system-level metrics such as RAM, process, used disk space, etc.
- Collect logs to send to Cloudwatchlogs: No logs from inside your ec2 instance will be sent to Cloudwatch logs without using an agent
- Centralized configuration using SSM Parameter store
- make sure iam permissions are correct
- Deafualt namespace for metrics collected by the unified Cloudwatch agent is CWAgent (can be configured/changed)
procstat Plugin
-Colect metrics and monitor system utilization of individual processes
Supports both Linux and Windows servers
Example: amount of time the preocess uses CPU, amount of memory the process uses, ...
Select which processes to monitor by
- pid_file: name of process identification number (PID) files they create
- exe: process nam that match string you specify (RegEX)
- pattern: command lines used to start the process (Regex)
Metrics colected by procstat plugin begins with "procstat" prefix (e.g., procstat_cpu_time, procstat_cpu_usage,...)

Status checks:
- automatic checks to identify hardware and software issues
System Status Checks
- Monitos problems with AWS systems (software/hardware issues on the physical host, loss of system power,...)
- Check personal health dashboard for any scheduled critical maintenance by AWS to your instance's host
- Resolution: stop and start the instance (instance migrated to a new host)
Instance Status Checks
- monitors software/network configuration of your instance (invalid network configuration, exhausted memory,...)
- resolution: reboot the instance or change instance configuration
CW metrics & Recovery:
CloudWatch metrics (1 minute interval)
- StatusCheckFailed_System
- StatusCheckFailed_Instance
- StatusCheckFailed (for both)
Option 1: CloudWatch Alarm
- Recover EC2 instance with the same private/public IP, EIP, metadata and placement group
- send notifications using SNS
Option 2: Auto Scaling Group
- set min/max/desired 1 to recover an instance but won't keep the same private and elastic IP

EC2 hibernate:
we know we can stop, terminate instances
- Stop: the data on disk (EBS) is kept intact in the next start
- Terminate: any EBS volumes (root also set-up to be destroyed is lost
on the start, the following happens:
- First start: the OS boots & the EC2 User Data script is run
- Folowing starts: the OS boots up
- Then your application starts, caches get warmed up and that can take time
introducing EC2 hibernate:
- The in-memory (RAM) state is perserved
- The instance boot is much faster! (the OS is not stopped/restarted)
- Under the hood: the RAM state is written to a file in the root EBS volume
- The root EBS volume must be encrypted
Use Cases:
- long running processing
- save the RAM statess
- services that take time to initialize
Good to know:
- Supported instance familied - C3,4,5 M3,4,5, R3,4,5
- instance ram size - must be less than 150 GB
- intance size - not supported for bare metal instances
- AMI - Amazon Linux 2, Linux AMI, Ubunut....Windows
- Root Volume: must be EBS, encrypted, not instance store and large
- Available on-demand and reserved instances
- an instance cannot be hibernated more than 60 days

## AMI
AMI= Amazon machine image
AMIs are a customization of an ec2 instance
- you add your own software, configuration, operating system, monitoring....
- faster boot/configuration time because all your software is pre-packaged
AMI are built for a specific region (and can be copied across regions)
You can launch EC2 instances from:
- A public AMI: AWS provided
- Your own AMI: you make and maintain them yourself
- An AWS marketplace an AMI someone else made (and potentially sells)

AMI process
- Start an EC2 instance and customize it
- stop the instance (for data integrity)
- build an AMI - this will also create EBS snapshots
- launch instances from other AMIs

AMI no-reboot option
- enables you to create an AMI without shutting down your instance
- by default, it's not selected (AWS will shut down the instance before creating an AMI to maintain the file system integrity)
With no-reboot enabled: Note: OS buffers are not flushed to disk before the snapshot is created

AWS Backup plans to create AMI
AWS backup doesn't reboot the instance while taking EBS snapshots (no-reboot behaviour)
- This won't help you to create an AMI that guarantees file system integrity since you need to reboot the instance
- To maintain integrity you need to provide the reboot parameter while taking images (EventBridge + Lambda + CreateImage API with reboot)

EC2 instance migration between AZ
take ami from instance in one az
launch it in new az

Cross account AMI sharing:
- you can share an AMI with another AWS account
- Sharing an AMI does not affect the ownership of the AMI
- You can only share AMIs that have unecrypted volumes that are encrypted with a customer managed key
- If you share an AMI with encrypted volumes, you must also share any customer managed keys used to encrypt them
AMI sharing with KMS encryption:
- must share the key with the target account
Cross account AMI copy:
- If you copy an AMI that has been shared with your account, you are the owner of the target AMI in your account
- The owner of the source AMI must grant you read permissions for the storage that backs the AMI (EBS snapshot)
- If the shared AMI has encrypted snapshots, the owner must share the key or keys with you as well
- Can encrypt the AMI with your own CMK while copying

EC2 image builder
- used to automate the creation of VMs or container images
- Automate the creation, maintain, validate and test ec2 AMIs
- Can run on a schedule (weekly, whenever packages are updated)
- free service (only pay for underlying resources)
- builder - creates an ec2 (builds componenets) - build ami- test ec2 instance - ami is distributed

AMI in production:
- you can force users to only launch EC2 instances from pre-approved AMIs (AMIs tagged with specific tags) using IAM policies
- Combine with AWS Config to find non-compliant EC2 instance (instnces launched with non-approved AMIs)

Manage EC2 at scale
## Systems Manager
- Helps you manage your EC2 and On-Premises systems at scale
- Get operational insights about the state of your infrastructure
- Patching automation for enhanced compliance
- Works for both Windows and linux OS
- Integrated with CloudWatch metrics/dashboards
- Integrated with AWs Config
- Free service
How it works:
- We need to install the SSm agent onto the system we control
- Installed by default on Amazon linux 2 AMI & some ubuntu ami
- if n instance can't be controlled with SSM, its probably an issue with the SSM agent
- Make sure the ec2 instances have the proper IAM role to allow SSM actions

AWS Tags:
- you can add text key-vaue pairs called Tags to many AWS resources
- Commonly used in EC2
- Free naming, common tags are Name, Environment, Team...
Theyre used for:
- Resource grouping
- Automation
- Cost Allocation
- Better to have too many tags than to have too few
Resource groups:
- create, view or manage logical group of resources thanks to tags
- Allows creation of logical groups of resources such as
Applications, different loayers of an application stack, production versus development environments
- Regional service
- Works with EC2, S3, DynamoDB, Lambda, etc.

SSM- Documents
- Documents can be in JSON or YAML
- you define parameters
- you define actions
- Many documents already exist in AWS
Run command:
- execture a document (=script) or just run a command
- Run command across multiple instances (using resource groups)
- Rate control/Error control
- Integrated with IAM & CloudTrail
- No need for SSH
- Command Output can be shown in the console, sent to s3 bucket or clooudwatch Logs
- Send notifications to SNS about command status ( In progress, success, failed)
- Can be invoked using EventBridge


SSM automation
- simplifies ommon maintenance and deployment tasks of EC2 instances and other AWS resources
- Example: restart insatances, create and AMI, EBS, snapshots
Automation Runbooks
- SSM documents of type automation
- Defines actions performed on your EC2 instances or AWS resources
- Pre-defined runbooks (AWS) or create cutom runbooks
Can be triggered
- Manually using AWS console, AWS CLI or SDK
- By amazon eventbridge
- On a schedule using Maintenance Windows
- By AWS Config for rules remediations

SSM Parameter Store:
- secure storage for configuration and secrets
- Optional Seamless Encryption using KMS
- Serverless, scalable, durable, easy SDK
- Version tracking of configurations/secrets
- Configuration management using papth & iam
- Notifications with Cloudwatch Events
- Integration with CloudFormation

SSM Parameter store hierarchy
./mydepartment/myapp/dev/
-db.url
-db.password
like a file system
can get passwords from amazon linux 2 ec2s
can get secret ids from parameter store
Advanced tier allows 100,000, 8kb max size of parameter value, parameter policies
Parameter Policies:
- allow to assign a ttl to a parameter (expiration dat) to force updating or deleting sensitive data such as passwords
- Can assign multiple policies at a time

SSM - Inventory
- Collect metadata from your managed intances (EC2/On-prmises)
- Metadata includes installed software, OS drivers, configurations, installed update, running services....
- View data in AWS console or store in S# and query and analyze using Athena and QuickSight
- specify metadat collections (minutes, hours, days)
- Query data from multiple AWS accounts and regions
- Create custom Inventory for your custom metadata (e.g. rack location of each managed instance)

SSM - State Manager
- automate the proces of keeping your managed instances (EC2/On-premises) in a state that you define
- Use Cases: bootstrap instances with software, patch OS/Software updates on a schedule
State Manager Association:
- defines the state that you want to maintain to your managed instances
- Ex. port 22 must be closed, antivirus must be installed
- Specify a schedule when this configuration is pplied
Uses SSM Documents to create an associations (e.g. SSM Document to configure CW Agent

SSM- Patch Manager
- Automates the process of patching managed instances
- OS updates, applications updates, security updates,...
- Supports both EC2 instances and on-premise servers
- Supports Linux,  MacOS and Windows
- Patch on demand or on a schedule using Maintenance Windows
- Scan instances and generate patch compliance report (missing patches)
- Patch compliance report can be sent to s3
Patch Baseline
- Defines which patches should and shouldnt be installed on your instances
- Ability to create custom Patch Baseline (specify approver/rejected patches)
- Patches can be auto-approved within days of their release
- By default, install only critical patches and patches related to security
Patch group
- Associate a set of instances with a specific Patch Baseline
- Ex: create Patch Groups for different environments (dev, test, prod)
- Instances should be defined with the tag key Patch Group
- An instance can only be in one Patch Group
- Patch Group can be registered with only one Patch Baseline
Maintenance Windows:
- define a schedule for when to perform actions on your instnces
- Ex. OS patching, updating drivers, installing software
- Maintenance window contains: scedhule, duration, set of registered instances, set of registered tasks

SSM Session Manager:
- allows you to sart a secure shell on your EC2 and on-premises servers
- Access through AWS console, AWS cli, or session Manager SDK
- Does not need SSH access, bastion hosts or SSH keys
- Supports Linux, MacOS and windows
- Log connections to your instances and executed commands
- Session log data can be sent to s3 or CloudWatch logs
- Cloudtrail can intercept Start Session events
IAM permissions
- contorl which users/groups can access session manager and which instances
- use tags to restrict accessss to only specific ec2 instances
- access ssm + write to s3 + write to cloudwatch
- Optionally, you can restrict commands a user can run in a session
- - more control and compliance (everything is tracked)
SSh needs inbound role with port 22 open
SSM session manager needs iam permissions

OpsWorks
- Chef and puppet help you perform server configuration automatically, or repetitive actions
- The work great with EC2 and On premises VM
- AWS OpsWorks = Manager Chef and Puppet
- It's an alternative to AWS SSM
- chef and puppet needed -> OpsWorks
-they help manage configuration as code
helps in having consistent deployments
Works with Linux/Windows
Can automate user accounts, cron, ntp, packages, services ...
They leverage "Recipes" or "Manifests"
Chef/puppet have similarities with SSM/Beanstalk/CloudFormation but they are open source tools that work cros-cloud

#Scalability and High Availabilit
Scalability:
- means that an application/system can handle greater loads by adapting
- There are 2 kinds of scalability, vertical and horizontal (horizontal is also elasticity)
- Scalability is linked but different to high availability
Vertical Scalability:
- means increasing the size of the instance
- Vertical scalability is very common for non distributed systems such as databases
- RDS, ElastiCache are services that can scale vertically
- There's usually a limit to how much you can vertically scale (hardware limit)
Horizontal Scalability:
- means increasing the number of instances/systems for your application
- implies distributed systems
- common for web applications/modern applications
- its easy to horizontally scales thanks to cloud offereings such as ec2
High Availability
- usually goes hand in hand with horizontal scaling
- means running your application/system in at least 2 data centers (== Availability Zones)
- The goal of high availability is to survive a data center loss
- The high availability can be passive (for RDS Multi AZ for example)
- The high availability can be active (for horizontal scaling)

## Elastic Load Balancing
- Load balancers are servers that forward internet traffic to multiple servers (EC2 instances) downstream
Why use one?
- spread oad across multiple downstream instances
- Expose a single point of access (DNS) to your application
- Seamlessly handle failures of downstream instances
- Do regular health cheks to your instances
- PRovide SSL termination (HTTPS) for your websites
- Enforce stickiness with cookies
- High availability across zones
- Seperate public traffic from private traffic
Why use an EC2 load balances
an ELB (EC2 load balancer) is a managed load balancer
- AWS guarantees that it will be working
- AWS takes care of upgrades, maintenance and high availability
- AWS provides only a few configuration knobs
It costs less to setup your own load balancer but it will be a lot more effort on your end
It is integrated with many AWS overings/services

Health Checks
- re crucial for load balancers
- They enable the load balancers to know if instances it forwards traffic to are available to reply to requests
- The health check is done on a port and a route (/health is common)
- It the response is not 200 (OK), then the instance is unhealthy

Types of Load balancers:
AWS has 3 kinds of managed Load balancers
- Classic Load balancer (v1 - old generation) -2009
Http, https, tcp
- Application Load Balancer (v2 - new generation) - 2016
http, https, web socket
- Network Load Balancer (v2 - new generation)- 2017
tcp, TLS (secure tcp) and udp
- Overall it is recommended to use the newer, v2, generation load balancers as they provide more features
- You can setup internal (private) or external (public) ELBs
- - LBs can scale but not instantaneously - contact AWS for a "warm up"
Trouble shooting
- 4xx errors are client induced errors
- 5xx errors are application induced errors
- Load balancer error 503 means at capacity or no registered target
- if the LB can't connet to your application, check your security group.
Monitoring
- ELB access logs will log all access requests (so you can debug per request)
- CloudWatch metrics will give you aggregate statistics (ex. connections count)

Classic Load Balancers
- Supports TCP (layer 4), http and https (layer 7)
- Health checks are tcp or http based
- fixed hostname xxx.region.elb.amazonaws.com

Application Load Balancer (v2)
- Application load balancer is layer 7 (http)
- Load balancinf to multiple HTTP applications across machines (target groups)
- Load Balancing to multiple applications on the same machine (ex. containers)
- Supporot for HTTP/2 and Web Socket
- Supports redirects (from HTTP to HTTPS for example)
Routing tables to different target groups:
- Routing based on path in URL (.com/a and .com/b)
- Routing based on hostname in url (a.example.com and b.example.com
ALB are a great fit for micro services & container-based applications (example: Docker and Amazon ECS)
Has a port mapping feature to redirect to a dynamic port in ECS
In comparison, we'd need multiple classic load ballancers per application
Target Groups
- EC2 instances (can be managed by an Auto Scaling group) - HTTP
- ECS tasks (managed by ECS itself) - HTTP
- Lambda functions - HTTP request is translated into a JSON event
- IP addresses - must be private IPs
- ALBs can route to multiple target groups
- Health checks are done at the target group level
Fixed hostname
The application servers don't see the IP of the client directly
- The true IP of the client is inserted in the header X-Forwarded-For
- We can also get Port (X-Forwaded-Port) and proto (X-ForwardedProto)

Network Load Balancer (v2)
Network load balancers (Layer 4) allow you to:
- Forward TCP and UDP trffice to your instance
- Handles millions of requests per second
- Less latency ~100ms (vs 400 ms for ALB)
NLB has one static IP per AZ (no static dns name) and supports assigning Elastic IP (helpful for whitelisting specific IP)
NLB are used for extreme performance TCP or UDP traffic
Not included in the AWS free tier


Sticky Sessions (Session Affinity)
- It is possible to implement stickiness so that the same client is always redirected to the same instance behind a load balancer.
- This works for Classic Load Balancer nd Application Load Balancer
- The "cookie"used for stickiness has an expiration date you control 1s to 7 days
- Use case: Make sure the user doesn't lose his session data
- Enabling stickiness may bring imbalance to the load over the backend ec2 instances
Cookie Names
- Application-based Cookied
--Custom cookie
---Generated by the target
---Can include any custom attributes required by the application
---Cookie name must be specified individually for each target group
---Dont use AWSALB, AWSALBAPP or AWSALBTG (reserved for use by the ELB)
--Aplication cookie
---Generated by the load balancer
---Cookie name is AWSALBAPP
--Duration-based Cookies
---Cookie generated by the load balancer
---Cookie name is AWSALB for ALB, AWSELB for CLB

Cross Zone-Load Balancing
With cross zone load balancin: each load balancer instance distributes evenly across all registered instances in all AZ (traffice is split between instances)
Without Cross zone-load balancing
Requestrs are distributed in the instances of the node of the ELB (traffice is split between AZ)
- Application Load Balancer
- --Always on (can't be disabled)
- --No charged for inter AZ data
- Network Load Balancer
- --Disabled by default
- --You pay for charges( $) for inter AZ data if enabled
- Classic Load Balancer
- --Through Console => Enabled by default
- --Through CLI/API => Disabled by default
- --No charged for inter AZ data if enabled

SSL/TLS
- An SSL certificate allows traffic between your cients and your load balancer to be encrypted in transit (in-flight encryption)
- SSL refers to Secure Sockets Layer, used to encrypt connections
- TLS refers to Transport Layer Security, which is a newer version
- Noawadays, TLS certificates are mainly used, but people still refer as SSL
- Public SSL certificates are issued by Certificate Authorities (CA)
- Comodo, Symantec, GoDaddy, GlobalSign, Digicert, Letsencrypt, etc,...
- SSL certificated have an expiration date (you set) and must be renewed
Load Balancers - SSL Certificates
- The load balancer uses an X.509 certificate (SSL/TLS server certificate)
- You can manage certificates using ACM (AWS certificate Manager)
- You can create, upload your own certificates alternatively
- HTTPS listener:
- -You must specify a default certificate
- -You Can add optional list of certs to support multiple domains
- -Clients can use SNI (server name indication)to specify the hostname they reach
- -Ability to specify a security policy to support older versions of SSL/TLS (legacy clients)
Server Name Indication
- SNI solves the problem of loading multiple SSL certificated onto one web server (to serve multiple websites)
- It's a "newer" protocol, and requires the client to indicate the hostname of the target server in the initial SSL handshake
- The server will then find the correct certificate, or return the default one
Note:
- Only works for ALB & NLB (newer generation, coudfront 
- Does not work for CLB (older gen)
- Classic load balancer:
- --Suports only one SSL certificate
- --Must use multiple CLB for multiple hostname with multiple SSL certificates
- Application Load Balancer
- --Supports multiple listeners ith multiple SSL certificates
- --Uses Server Name Indication (SNI) to make it work
- Network Load Balancer 
- --Support multiple listeners with multiple SSL certiificates

Connection Draining
Feature naming:
- CLB: Connection draining
- Target group: Deregistration delay (for ALB nd NLB)
- Time to complete "in-flight requests" while the instance is de registering or unhealthy
- Stops sending new requests to the instance which is de-registering
- between 1 to 3600 seconds, default is 300 seconds
- can be disables (set value to 0)
- set to a low value if your requests are short

Health Checks
Target Health Status:
- Initial - registering the target
- Healthy
- Unhealthy
- Unused: target is not registered
- Draining: de-registering the target
- Unavailable: health checks disabled
If a target group contains only unhealthy targets, ELB routes requests accross its unhealthy targets

Error codes
- Successful request: Code 200
- Unsuccessful at client side: 4xx code
400: Bad request
401: Unauthorized
403: Forbidden
460: Client closed connection
463: X-forwarded for header with > 30 IP (malformed request)
- Unsuccessful at server side: 5xx code
500: internal server error would mean some error on the ELB itself
502: Bad Gateway
503: Service unavailable
504: Gateway timeout: probably an issue within the server
561: Unauthorized

Monitoring:
- All load balancer metrics are directly pushed to CloudWatch meterics
- BackendConnectionErrors
- HealthyHostCoulnt/UnhealthyHostCount
- HTTPCode_Backend_2XX: Successful request
- HTTPCode_Backend_3XX:redirected requests
- HTTPCode_Backend_4XX: error codes
- HTTPCode_Backend_5XX: server error codes generated by the load balancer
- Latency
- RequestCount
- RequestCountPerTarget
- SurgeQueueLength: The total number of requests (HTTP listener) or connections (TCP listener) that are pending routing to a healthy instance. Help to scale out ASG. Max value is 1024
- SpilloverCount: The toal number of request that were rejected because the sure queue is full

Troubleshooting:
HTTP 400: Bad reques -> The client sent a malformed request that does not meet HTTP specifications
HTTP 503: Service unavailbel -> Ensure that you have healthy instances in every AZ that your load balancer is configured to respond in. Look for healthyHostCount in Cloudwatch
HTTP 504 Gateway Timeout -> Check if keep-alive settings on your ec@ instances are enabled and make sure that the keep-alive timeout is greater than the idle timeout settings of load balancer

Access Logs:
Access logs from Load balancer can be stored in S3 and contain:
- time
- Client IP adress
- Latencies
- Request paths
- Server response
- Trace ID
Onlny pay for the s3 storage
Helpful for compliance reason
Helpful for keeping access data even after ELB r EC2 instances are terminated
Access Logs are already encrypted

Request Tracing:
- Request tracing - Each http request has an added custom header "X-Amzn-Trace-ID"
- This is useful in logs/distributed tracing platform to track a single request
- Application Load Blancer is not yet integrated with X-Ray

Target groups settings
- deregistration_delay.timeout_seconds: time the load balancer waits before deregistering a target
- slow_start.duration_seconds
- load_balancing.algorithm.type: how the load baalncer selects targets when routing requests (Round robin, Least outstanding requests)
- stickiness.enabled
- stickinees.type: application-based or duration-based cookie
- stickinedd.app_cookie.cookie_name: name of the application cookie
- stickiness.app_cookie.duration_seconds: application-based cookie expiration period
- stickiness.lb_cookie.duration_Seconds: duration-based cookie expiration period
Slow Start Mode:
- By Default, a target receives it's full share of requests once it's registered with the target group
- Slow Start Mode gives healthy targets time to warm-up before the load balancer sends them a full share of requests
- The load balancer linearly incrceasses the number of requests that it sends to the target
- A target exits Slow Start Mode when:
- - The duration period elapses
- - The target becomes unhealthy
- To disable, set Slow start duration value to 0
Request Routing Algorithms - Least Outstnding requests:
- The next instance to receive the request is the instnce that has the lower number of pending/unfinished requests
- Works ith Application Load balancer and Classic load balanceer (HTTP/HTTPS)
Request Routing Algorithms - Round Robin
- Equally choose the targets from the targets from the target group
- Works with Application Load Balancer and Classic Load Balancer (TCP)
Request Routing Algorithms - Flow Hash
- Selects a target based on the protoco, source/destination IP address, source/destination port, and TCP sequence number
- Each TCP/UCP connection is routed to a single target for the life of the connection
- Works with Network Load Balancer

ALB - listener Rules:
Processed in order (default rule, last)
Supported actions (forward, redirect, fixed-response)
Role conditions:
- host-header
- http-request-method
- path-patern
- source-ip
- http-header
- query-string

Target group weighting
- specify weight for each target group on a single rule
- example: multiple versions of your app, blue/green deployment
- Allows you to control the distribution of the traffic to your applications

## Auto Scaling Groups (ASG)
in real life, he load on your websites and applications can change
in the cloud, you can create and get rid of servers very quickly
The goal of an ASG is to:
- Scale out (add EC2 instances) to match an increased load
- Scale in (remove EC2 instances) to match a decreased load
- Ensure we have a minimum and a maximum number of machines running
- Automatically register new instnces to a load balancer
Following attributes:
Launch configuration:
- AMI + Instance type
- EC2 user data
- EBS vlumes
- Security groups
- SSH ey pair
Min size, max size, Initial Capacity
Network + Subnets Information
Load Balancer Information
Scaling Policies
AUTO SCALING ALARMS
- it is possible to scale an ASG based on CloudWatch alarms
- an alarm monitors a metric (such as average CPU)
- Metrics are computed for the overall ASG instances
- Based on the alarm (we careate scale in or scale out events)
New Rules:
It is now possible to define "better" auto scaling rues that are directly managd by ec2
- Target average CPU Usage
- Number of requests on the ELB per instance
- Average network In
- Average network out
These rules are eassier to set up and can make more sense
Auto Scaling Custom Metric
- We can auto scale based on a custom metric (ex: number of connected users)
1. Send custom metric from application on ec2 to clouwatch (PutMetric API)
2. Create CloudWatch alarm to react to low/high values
3. Use the CloudWatch alarm as the scaling policy for ASG
ASG Brain Dump:
- Scaling policies can be on CPU, network, can even be on cutom metrics based on a schedule (if you know your useer patterns)
- ASGs use launch configurations or launch templates (newer)
- To update an ASG, you must provide a new launch configuration/launch template
- IAM roles attached to an ASG will get assigned to EC2 instances
- ASG are free. You pay for the underlying resources being launched
- Having instances under an ASG means that if they get terminated for whatever reason, the ASG will automatically create new ones as a replacement. Extra safety
- ASG can terminate instances marked as unhealthy by an LB (and hence replce them)

ASG - Dynamic scaling policie
target Tracking Scaling:
- Most simple and easy to set-up
- Ex. I want the average ASG CPU to stay around 40%
Simple/Step Scaling:
- When a Cluoudwtch alarm is triggered (ex cpu > 70%) then add 2 units
- When a cloudwatch aarm is triggered (examplae cpu<30%) then remove 1
Scheduled Actions:
- Anticipate a scaling based on known usage patterns
- Example: increase the min capacity to 10 at 5pm on fridays
Predictive scaling:
-predictive scaling: continuosly forecast load and schedule scaling ahead

Good metrics to scale on:
- SPUUtilization: Average CPU utilization across your instances
- RequestCountPerTarget: to make sure the number of requests perec2 instances is stable
- Average netowkr in/out (if you're application is network bound)
- Any custom metric (that you push using CloudWatch)

Scaling cooldown
- After a scaling activity happens, you are in the coldown period (default 300 seconds)
- During the cooldown period, the ASG will not launch or terminate additional instances (to allow for metrics to stabilize)
- Advice: Use a ready-to-use AMI to reduce configuration time in order to be serving requests faster and reduce the cooldown period

Lifecycle hooks
- By default, as soon as an instance is launched in an ASG it's in service
- You can perform extra steps before the instance goes in service (Pending state)
- - Define a script to run on the instances they start
- You can perform some actions before the instance is terminated (teminating state)
- - Pause the instances before they're terminatinated for troubleshooting
- Use Cases: cleanup, log extraction, special heath checks
- Integration with EventBridge, SNS and SQS

Launch Configuration vs Launch Template
both:
- ID of the AMI, the instance type, a key pair, security groups and the other parameters that you use to launch EC2 instances (tags, EC2 user-data)
- You can't edit both launch configuations and launch templates.
Launch Configuration (legacy):
- Must be re-created every time
Launch template (newer):
- Can have multiple versions
- Create parameters subsets (partial configuration for re-use and inheritance)
- Provision using both On-Demand and Spot instances (or a mix)
- Supports Placement Groups, Capacity reservations, dedicated hosts, multiple instance types
- Can use T2 unlimited burst feature
- Recommended by AWS going forward

Health Checks:
- To make sure you have high availability, means you have at least 2 instances running across 2 AZ in your ASG (must configure multi-AZ ASG)
- Health checks available:
- - EC2 Status checks
- - ELB health cheacks
- - Custom health checks: sends instance's health to ASG using AWS CLI or AWS SDK
- ASG will launch a new instance after terminating an unhealthy one
- ASG will not reboot unhealthy hosts for you
- Good to know cli:
- - set-instance-health (use with Custom Health Checks)
- - terminate-instance-in-auto-scaling-group

Troubleshooting ASG issues:
-<number of instances> instance(s) are already running. Launching EC2 instance failed.
- - The auto scaling group has reached the limit set by the MaximumCapacity parameter. Update your Auto Scaling group by providing a new value for the maximum capacity.
- Launching Ec2 instances is failing:
- - The security group does not exist. SG might have been deleted
- - The key pair does not exist. The key pair might have been deleted
- If the ASG fails to launch an instance for over 24 hours, it will automatically suspend the processes (administration suspension)

Cloudwatch metrics for ASG:
- Metrics are collected every 1 minute
- ASG- level metrics (opt-in)
- - GroupMinSize, GroupMaxSize, GroupDesireCapacity
- - GroupInServiceInstances, GroupPendingInstances, GroupStandbyInstances
- - GroupTerminatingInstances, GroupTotalInstances
- - You should enable metric collection to see the metrics
- EC2-level metrics (enabled): CPU Utilization, etc...
- - Basic monitoring: 5 minutes granularity
- - Detailed monitoring: 1 minute granularity

AWS Auto scaling
- Backbone service of auto scaling for scalable resources in AWS:
- Amazon EC2 auto scaling groups: Launch or terminate EC2 instances
- Amazon EC2 Spot Fleet requests: Launch or terminate instances from a spot fleet request, or automatically replace instances that get interrupted for price or capacity reasons.
- Amazon ECS: Adjust the ECS servicce desired count up or down
- Amazon DynamoDB (table or global secondary index): WCU & RCU
- Amazon Aurora: Dynamic Read Replicas Auto Scaling
Scaling Plans:
Dynamic scaling: creates a target tracking scaling policy
- Optimize for availability => 40% of resource utiliation
- Balance availability and cost => 50% of resource utilization
- Optimize for cost => 70% of resource utilization
- Custom => choose your own metric and target value
- Options: Disable scale-in, cooldown period, warmup time (for ASG)
Predictive scaling: continuously forecast load and schedule scaling ahead

## Beanstalk
Developer problems on AWS:
- Managing infrastructure
- Deploying Code
- Configuring all the databases, ,load balancers, etc
- Scaling concerns
- Most web apps hve the same architecture (ALB + ASG)
- All the developers want is for their code to run!
- Possibly, consistently across different applications and environments
Overview:
- Elastic Beanstalk is a developer centric view of deploying an application on AWS
- It uses all the components we've seen before, EC2, ASG, ELB, etc.
- But it's all in one view that's easy to make sense of
- We still have full control over the configuration
- Beanstalk is free but you py for the underlying instances
Elstic Beanstalk:
Managed service
- Instance configuration/os is handled by beanstalk
- Deployment strategy is configurable but performed by Elasstic Beanstalk
Just the application code is the responsibility of the developer
Three architecture models:
- Single instance deployment: good for dev
- LB +ASG: great for production or pre-production web applications
- ASG only: great for non-web apps in production (workers, etc.)

3 components:
- Application
- Application version: each deployment gets assigned a version
- Environment name (dev, test, prod): free naming
You deploy application versions to environments and can promote application versions to the next environment
Rollback feature to previous versions
Full control over lifecycle of environments

Support for many platforms:
- Go, Java SE, Java with Tomcat, .NET on windows Server with IIS, Node.js, PHP, Python, Ruby, Packer Builder, Single container Docker, Multicontainer Docker, Preconfigured Docker
- If not supported, you can write your custom platform (advanced)

## CloudFormation
Infrastructure as code:
- Currently we have been doing a lot of manual work
- All this manual work will be very tough to reproduce:
- - In another region
- - In another AWS account
- - ithin the same region if everything was deleted
- Wouldn't it be great if akk our infrastructure was in code?
- That would be deployed and createe/update/delete our infrastructure

Cloudformation is a declarative way of outlining your AWS infrastructure for any resource (most of them are supported)
Then cloudformation creates those for you, in the right order with the exact configuration that you specify

Benefits:
IaC:
- no resources are manually created, which is excellent for control
- The code can be version controlled for example with git
- Changes to the infrastructure are reviewd through code
Cost:
- Each resource within the stack is stagged with an identifier so you can easily see how much a stack costs you
- You can estimate the cost of your resources using the CloudFormation template
-Savings strategy: In Dev, you could automation deleteion of templates at 5PM and recreate at 8AM, safely
Productivity:
- destroy and re-create an infrastructure cloud on the fly
- automated generation of diagram for your templates
- declarative programming (no need to figure out ordering and orchestration)
Sepearation of concern: create many stacks for many apps and many layers, Ex:
- VPC stacks
- Network stacks
- App stacks
Don't re-invent the wheel:
- Leverage existing templates on the web
- Leverage the documentation

How does it work:
- Templates have to be uploaded in S3 and then referenced in CloudFormation
- To update a template, we can't edit previous ones. We have to re-upload a new version of the template to AWS
- Stacks are identified by a name
- Deleting a stack deletes every single artifact that was created by cloudformation

Deploying CloudFormation templates:
Manual way:
- Editing templates in the CloudFormation Designer
- Using the console to input parameters, etc.
Automated way:
- Editing templates in a YAML file
- Using the AWS CLI (Command Line Interface) to deploy the templates
- Recommended way when you fully want to automate your flow

CloudFormation Building Blocks:
Template components:
1. Resources: Your AWS resources declared in the template ( Mandatroy)
2. Parameters: The dynamic inputs for your template
3. Mappings: The static variables for your template
4. Outputs: References to what has been created
5. Conditionals: List of conditions to perform resource creation
6. Metadata
Template Helpers:
1. References
2. Functions

YAML
- YAML andJSON are the languages you can use for cloudformation
- JSON is horrible for CF
- Yaml is great
- Key - value pairs
- Nested objects
- Support Arrays
- Multi line strings
- Can include comments

Resources:
- Resources are the core of your CloudFormation template (Mandatory)
- They represent the different AWS COmponeents that will be created and configured
- Resources are declared and can reference each other
- AWS figured out creationi, updates and deletes of resources for us
- There are over 224 types of resources (!)
- Resource types identifiers are of the form:
    AWS::aws-product-name::date-type-name
How do we know:
- read the documentation
- over 224 resources
Can i create a dynamic amount of resources?
- No, Everything in the CloudFormation template has to be declared. You can't perform code generation there
Is every AWS supported?
- almost. Only a select few niches are not there yet
- You can work around that using AWS Lambda Custom Resources

Parameters:
- They are a way to provide inputs to your AWS CloudFormation templae
- They're important to know if:
When to use?
- Is this cloudformation resource configuration likely to change in the future
- If so, use  parameter
- You won't have to re-upload a template to change its content

Parameters can be controlled by all these settings:
Type:
- String
- Number
- CommaDelimitedList
- List<Type>
- AWS Parameter (to help catch invalid values - match against existing values in the AWS Account)
Description
Constraints
ConstraintDescription(string)
Min/MaxLength
Min/Max Value
Defaults
AllowedValues (array)
AllowedPattern (regexp)
NoEcho

How to reference a Parameter
- The FN::Ref function can be leveraged to reference parameters
- Parameters can be used anywhere in a template
- The shorthand for this in Yaml is !Ref
- The function can also refereence other elements within the template

Concept: Pseudo Parameters
- AWS offers us pseudo parameters in any Cloudformation template
- These can be used at any time and are enabled by default

Mappings:
- Mappings are fixed variables within your CloudFormation Template
- They're very handy to differentiate between different environments (dev vs. prod), regions (AWS regions), AMI types, et.c
- All the values are hardcoded within the template

When would you use mappings vs. parameters:
Mappings are great when you know in advance all the values that can be taken and that they can be deduced from variables such as
- Region 
- Availability Zone
- AWS Account
- Environment (dev vs. Prod)
- Etc.
They allow safer control over the template
Use parameters when the values are really user specific

FN::FindInMap
- We use FN::FindInMap to return value from a specific key
- !FindInMap [ MapName, TopLevelKey, SecondLevelKey]

Outputs:
- The outputs section declares optional outputs values that we can import into other stacks (if you export them first)
- You can also view the outputs in the AWS console or in using the AWS CLI
- They're very useful for example if you define a network CloudFormation, and output the variables such as VPC ID and your Subnet IDs
- It's the best way to perform some collaboration cross stack, as you let expert handle their own part of the stack
- You can't delete a CloudFormation Stack if it's outputs are being referenced by another CloudFormation stack

Cross Stack Reference
- We can create a second template that leverages that security group 
- For this we use the FN::ImportValue function
- You can't delete the underlying stack until all the references are deleted too

Conditions:
Conditions are used to control the creation of resources or outputs based on a condition
Condtions can be whatever you want them to be, but common ones are
- Environment (dev/test/prod)
- AWS Region
- ny parameter value
Each condition can reference another condition, parameter value or mapping
The logical ID is for you to choose, It's how you name condition
The intrinsic function (logical) can be any of the following:
- Fn::And
- Fn::Equals
- Fn::If
- Fn::Not
- Fn::Or

Intrinsic Functions:
- Ref
- Fn::GetATT
- Fn::FindInMap
- Fn::ImportValue
- Fn::Join
- Fn::Sub
- Condition Functions (Fn::If, Fn::Not, Fn::Equals, etc...)

Fn::Ref
The Fn::Ref function can be leveraged to reference
- Parameters => returns the value of the parameter
- Resources => returns the physical ID of the underlying resource (ex. EC2 ID)
The shorthand for this YAML is !Ref

Fn::GetAtt
Attributes are attached to any resource you create
to know the attributes of your resources, the best place to look is the documentation
For example: the AZ of the EC2 machine

Fn::Join
Join values with a delimiter

Fn::Sub
- Fn::Sub or !Sub as a shorthand, is used to substitue variables from a test. It's a very handy function that will allow you to fully customize your templates
- For example, you can combine Fn::Sub with references or AWS Pseudo variables!
- String must contain ${VariableName} and will substitute them

User Data in EC2 in CloudFormation
- The important thing to pass is the entire script through the function Fn::Base64
- Good to know: user data script log is in /var/log/cloud-init-output.log

cfn-init
- AWS::CloudFormation::Init must be in the MEtadate of a resource
- With the cfn-init script, it helps make complex EC2 configurations readable
- The Ec2 instance will query the CloudFormation srvice to get init data
- Logs go to /var/log/cfn-init.log

cfn-signal & wait conditions
- We still don't know how to tell CloudFormation tht the Ec2 instance got properly configured after cfn-init
For this, we can use the cfn-signal script!
- We can run cfn-signal right after cfn-init
- Tell CloudFormation service to keep on going or fail
We need to define WaitCondition:
- Block the template until it receives a signal from cfn-signal
- We attach a CreationPolicy (also works on ec2, ASG)
Failures and Troubleshooting:
- Ensure that the AMI you're using has the AWS CloudFormation helper scripts installed. If the AMI doesn't include the helper scripts, you can also download them to your instance.
- Verify that the cfn-init &cfn-signal command was successfully run on the instance. You can view logs, such as var/log/cloud-init.log or /var/log/cfn-init.log, to help you debug the instance launch.
- You can retrieve the logs by logging in to your instance, but you must disable rollback on failure or else AWS CloudFormation deletes the instance after your stack fails to create.
- Verify that the instance has a connection to the internet. If the instance is in a VPC, the instance should be able to connect to the Internet through a NAT device if it's in a private subnet or through an Internet Gateway if it's in a public subnet
- For example run a curl command.

Rollbacks:
Stack Creation Fails:
- Default: everything rolls back (gets deleted). We can look at the log
- Option to disable rollback and trouble shoot what has happened
Stack Update Fails:
- The stack automatically rolls back to the previous known working state
- Ability to see in the log what happened and error messages
Update rolling back: if the rollback fails

Nested Stacks:
- Nested stacks are stacks as part of other stacks
- They allow you to isolate repeated pattersn/ common componenets in seperate stacks and call them from other stacks
- Ex. Load Balancer configuration that is re-used
- Nested stacks are considered best practice
- To update a nested stack, always update the parent (root stack)

ChangeSets:
- When you update a stack, you need to know what changes before it happens for greater confidence
- ChangeSets won't say if the update will be successful

Drift:
- Cloudformation allows you to create infrastructure
- But it doesn't protect you against manual configuration changes
- How do we know if our resources have drifted
- We can use CloudFormation Drift
- stack actions -> detect drict
- stack actions -> drift

Retain Data on Deletes:
- You can put a DeletionPolicy on any resource to control what happens when the CloudFormation template is deleted
DeletionPolicy=Retain:
- Specify on resources to preserve / backup in case of Cloudformation deletes
- To keep a resource, specify Retain (works for any resource / nested stack)
DeletionPolicy=Snapshot
- EBS volume, Elasticache cluster, Elasticache ReplicationGroup
- RDS DBInstance, RDS DBCluster, Redshift Cluster
DeletionPolicy=Delete (default policy)
- Note: For AWS::RDS::DBCluster resources, the default policy is Snapshot
- Note: to delete an S3 bucket, you need to first empty the bucket of its content

Termination protection on stacks:
To prevent accidendat deletes of CloudFormation templates, use TerminationProtection

CloudFormation StackSets:
- Create,update or delete stacks across multiple accounts and regions with a single operation
- Administrator account to create StackSets
- Trusted accounts to create, update, delete stack instances from StackSets
- When you update a stackset, all associated stack instances are updated throughout all acoounts and regions
- Ability to set a maximum concurrent actions on targets (# or %)
- Ability to set failure tolerance (# or %)

## EBS
- An EBS (elastic block store) Volume is a network drive you can attach to your instance while they run 
- It allows your instances to persist data, even after their termination
- They can only be mounted to one instance at a time (Or multi attach feature for some EBS)
- They are bound to a specific availability zone
- Analogy: Think of them as a "network USB stick"
- Free tier : 30 GB of free EBS storage of type General purpose (SSD) or magnetic per month
It's a network drive (not a physical drive)
- It uses the network to communicat the instance, which means there might be a bit of latency
- It can be detached from an EC2 instance and attached to another one quickly
Its locked to an AZ:
- An EBS volume in us-east-1a cannot mbe attched to us-east-1b
- To move a volume across, you first need to snapshot it
Have provisioned capacity (size in GBs and IOPS)
- You get billed for all the provisioned capacity
- You can increase the capacity of the drive over time
Delete on termination:
- Controls the EBS behaviour when an ec2 instance terminated
- By default, the root EBS volume is deleted (Attribute enabled)
- Be default, any other attached EBS volume is not deleted (attribute disabled)
- This can be controlled by the AWS console/ AWS CLI
- Use case: preserve root volume when instance is terminated

Instance Store:
- EBs volumes are network drives with good but "limited" performance
- If you need a high-performancee hardware disk, use EC2 Instance Store
- Better I/O performance
- EC2 Instance Store lose their storage if they're stopped (ephemeral)
- Good for buffer/cache/scratch data/temporary content
- Risk of data loss if hardware fails
- Backups and Replication are your responsibility

Volume types:
- gp2/gp2 (SSD): General Purpose SSD volume that balances price and performance for a wide variety of workloads
- io 1/ io2 (SSD): HIghest-performance SSD volume for mission-critical low latency or high or high-throughput workloads
- st 1 (HDD): low cost HDD volume designed for frequently accessed, throughput-intensive workloads
- sc 1 (HDD): Lowest cost HDD volume designed for less frequently access workloads
EBS volumes are characterized in Size/Throughput/IOPS (I/O ops per second)
When in doubt always consult the AWS documentation -it's good
Only gp2/gp3 and io1/io2 can be used as boot vlumes

General puporse SSD:
- cost effective storage, low latency
- system boot volumes, Virtual desktops, Development and test environments
- 1GB - 16TiB
gp3:
- Baseline of 3,000 IOPS and throughput 125 MB/s
- Can increase IOPS up to 16,000 and throughput up to 1000 Mb/s independently
gp2:
- Small gp2 volumes can burst IOPS to 3,000
- Size of the volume and IOPS are linked, max IOPS is 16,000
- 3 IOPS per GB, means at 5,334 GB we are at the max IOPS

Provisioned IOPS (PIOPS) SSD:
- Critical business applications with sustained IOPS performance
- Or applications that need more than 16,000 IOPS
- Great for database workloads (sensitive to storage perf and consistency)
io1/io2 (4Gb - 16 TB):
- Max PIOPS: 64,000 for Nitro EC2 instances and 32,000 for other
- Can increase PIOPS independently from storage size
- io2 have more durability and more IOPS per GB (at the same price as io1)
io2 Block Express (4GB- 64TB)
- Sub-ms latency
- Max PIOPS: 256,000 with an IOPS:GB ratio of 1000:1
Supports EBS Multi-Attach

Hard Disk Drives (HDD):
- Cannot be a boot volume
- 125 MB to 16 TB
Throughput Optimized HDD(st1):
- Big Data, Data Warehouses, Log Processing
- Max throughput 500MB - max IOPS 500
Cold HDD (sc1):
- For data that is infrequently accessed
- Scenarios where lowest cost is important
- Max throughput 250 Mb/s - max IOPS 250

EBS Multi-Attach - io1/io2 family
- Attach the same EBS volume to multiple EC2 instances in the same AZ
- Each instance has full read and write permissions to the volume
Use Cases:
- Acheive higher application availability in clustered linux applications (ex. Teradata)
- Applications must manage concurrent write operations
Must use a file system that's cluster-aware (Not XFS, EX4, etc.)


EBS Volume Resizing:
You can only increase the EBS volumes:
- size (any volume type)
- IOPS (only in IO1)
After resizing an EBS volume, you need to repartition your drive
After increasing the size, it's possible for the volume to be in a long time in the "optimization" phase. The volume is still usable
You can't decrease the size of your EBS volume (create another smaller volume then migrate data)

EBS Snapshots
- Make a backups (snapshot) of your EBS volume at a point in time
- Not necessary to detach volume to do snapshot, but recommended
- Can copy snapshots across AZ or Region

Amazon Data Lifecycle Manager
- Automate the creation, retention and deletion of EBS snapshots and EBS-backed AMIs
- Schedule backups, cross-account snapshot copies, delete outdated backups, ...
- Uses resource tags to identify the resources (EC2 instances, EBS volumes)
- Can't be used to manage snapshots/AMIs created outside DLM
- Can't be used to manage instance store backed AMIS

Fast Snapshot Restore (FSR)
- EBS Snapshots stored in S3
- By default, there's a latency of I/O operations the first time each block is accessed (block must be pulled from S3)
- Solution: force the initialization of the entire volume (using the dd or fio command) or you can enable FSR
- FSR helps you to create a volume from a snapshot that is fully initialized at creation (no I/O latency)
- Enabled for a snapshot in a particular AZ (billed per minute - very expensive $$$)
- Can be enabled on snapshots created by Data Lifecycle Manager

Volume Migration:
- EBS Volumes are only locked to a specific AZ
To migrate it to a different AZ (or region):
- Snapshot the volume
- (optional) Copy the volume to a different region
- Create a volume from the snapshot in the AZ of your choice

Encryption:
When you create an encrypted EBS volume, you get the following:
- Data at rest is encrypted inside the volume
- All the data in flight moving between the instance and the volume is encrypted
- All snapshots are encrypted
- All volumes created from the snapshot
Encryption and decryption are handled transparently (you have nothing to do)
Encryption has a minimal impact on latency
EBS Encryption leverages keys from KMS (AES-256)
Copying an unencrypted snapshot allows encryption
Snapshots of encrypted volumes are encrypted

encrypt an unencrypted EBS volume:
- Create an EBS snapshot of the volume
- Encrypt the EBS snapshot (using copy)
- Create new EBS volume from the snapshot (the volume will also be encrypted)
- Now you can attach the encrypted volume to the original instance

## EFS
- Managed NFS (network file system) that can be mounted on many EC2
- EFS works with EC2 instances in multi-AZ
- Highly available, scalable, axpensive (3x gp2), pay per use
- Use cases: content management, web serving, data sharing, wordpress
- Uses NFSv4.1 protocol
- Uses security group to control access to EFS
- Compatible with Linux based AMI (not Windows)
- Encryption at rest using KMS
- POSIX file system (~ Linux) that has a standard file API
- File system scales automatically, pay-per-use, no capacity planning

EFS scale
- 1000s of concurrent NFS clients, 10 GB +/s throughput
- Grow to Petabyte-scale network file system automatically
Performance mode (set at EFS creation time):
- General Purpose (default): latency- sensitive use cases (web servers, CMS, etc...)
- Max I/O - higher latency, throughput, highly parallel (big data, media processing)
Throughput mode
- Bursting (1 TB = 50 MB/s + burst of up to 100 MB/s)
- Provisioned: set your throughput regardless of storage size, ex: 1 GB/s for 1TB storage
Storage Tiers (lifecycle management feature - move file after N days)
- Standard: For frequently accessed files
- Infrequent access (EFS-IS): cost to retrieve files, lower price to store

EFS vs. EBS
EBS:
- can be attached to only one instance at a time
- are locked at the Availability zone (AZ) level
- gp2: IO increases if the disk size increases
- io1: can increase IO independently
To migrate an EBS volume across AZ:
- Take a snapshot
- Restore the snapshot to another AZ
- EBS backups use IO and you shouldn't run them while your application is handling a lot of traffic
Root EBS Volumes of instances get terminated by default if the EC2 instance gets terminated (you can disable that)
EFS:
- Mounting 100s of instances across AZ
- EFS share website files (wordpress)
- Only for linux Instances (POSIX)
- EFS has a higher price point than EBS
- Can levergae EFS-IA for cost savings

EFS - Access Points:
- Easily manage applications access to NFS environments
- Enforce a POSIX user and group to use when accessing the file system
- Restrict access to a directory within the file system and optionally specify a different root directory
- Can restrict access from NFS clients using IAM policies

EFS - Operations:
Operations can be done in place:
- Lifecycle Policy (enable IA or change IA settings)
- Throughput Mode and Provisioned Throughput Numbers
- EFS Access Points
Operations that require a migration using DataSync (replicated all file attributes and metadata)
- Migration to encrypted EFS
- Performance Mode (e.g. Max IO)

CloudWatch Metrics:
PercentIOLimit:
- How close the file system reaching the I/O limit (General Purpose)
- If at 100% move to Max I/O (migration)
BurstCreditBalance
- The number of burst credits the file system can use to achieve higher throughput lovels
StorageBytes:
- File system's size in bytes (15 minute interval)
- Dimensions: Standard, IA, Total (Standard + IA)

## S3 
- Amazon s3 allows people to store objects (files) in "buckets" (directories)
- Buckets must have a globally unique name
- Buckets are defined at the regional level
Naming Convention
- No uppercase
- No underscore
- 3-63 characters long
- Not an IP
- Must start with lowercase letter or number

Ojects
Objects (files) have a key
The key is the full path
The key is compose of prefix + object name
There's no concept of "directories" within buckets (although the UI tricks you into thinking otherwise)
Just keys with very long names that contain slashes
object values are the content of the body:
- Max object size if 5TB 
- If uploading more than 5GB, must use "multi-part upload"
Metadata (list of text key/ value pairs - system or user metadata)
Tags (Unicode key / value pair - up to 10) - useful for security/lifecycle
Version ID (if versioning is enabled)

Versioning:
- You can version your files in Amazon S3
- It is enabled at the bucket level
- Same key overwrite will increment the version: 1, 2, 3
It is best practice to version your buckets:
- Protect against unintended deletes (ability to restore a version)
- Easy roll back to previous version
Notes:
- Any file that is not versioned prior to enabling versioning will have version "null"

Encryption:
There are 4 methods to encrypt obects in s3:
- SSE-S3: encrypts S3 objects using keys handled & managed by AWS
- SSE-KMS: leverage AWS Key Management Service to manage encryption keys
- SSE-C: when you want to manage your own encryption keys
- Client Side Encryption
It's important to understand which ones are adapted to which situation for the exam 

SSE-S3
- encryption using keys handled & managed by Amazon s3
- Object is encrypted server side
- AES-256 encryption type
- Must set header: "x-amz-server-side-encryption":"AES256"

SSE-KMS
- encryption using keys handled & managed by KMS
- KMS advantages: user control + audit trail
- Object is encrypted server side
- Must set header: "x-amz-server-side-encryption":"aws:kms"

SSE-C
- server side encryption using data keys fully managed by the customer outside of AWS
- Amazon S3 does not store the encryption key you provide
- HTTPS must be used
- Encryption key must provided in HTTP headers, for every HTTP request made

Client Side Encryption
- Client library such as the Amazon S3 Encryption Client
- Clients must encrypt data themselves before sending to S3
- Clients must decrypt data themselves when retrieving from s3
- Customer fully manages the keys and encryption cycle

Encryption in transit (SSL/TLS)
Amazon S3 exposes:
- HTTP endpoint: non encrypted
- HTTPS endpoint: encryption in flight
You're free to use the endpoint you want but HTTPS is recommended
Most clients would use the HTTPS endpoint by default
HTTPS is mandatory for SSE-C
Encryption in flight is also called SSL/TLS

Security:
User-Based
- IAM policies - which API calls should be allowed for a specific user from IAM
Resource Based:
- Bucket Policies - bucket wide rules from the S3 console - allows cross account
- Object Access Control List (ACL) - finer grain
- Bucket Access Control List (ACL) - less common
Note: an IAM principal can access an S3 object if
- the user IAM permissions allow it OR the resource policy ALLOWS it

S3 Bucket Policies
JSON based policies
- Resources: buckets and objects
- Actions: Set of API to allow or Deny
- Effect: Allow/Deny
- Principal: The account or user to apply the policy to
Use S3 bucket policy to:
- Grant public access to the bucket
- Force objects to be encrypted at upload
- Grant access to another account (Cross account)

Bucket Settings for Block Public Access
Block public access to buckets and objects granted through
- new access control lists (ACLs)
- any access control lists (ACLs)
- new public bucket or access point policies
Block public and cross-account access to buckets and objects through any public bucket or access point policies
These settings were created to prevent company data leaks
If you know your bucket should never be public, leave these on
Can be set at the account level

Other Security:
Networking:
- Supports VPC Endpoints (for instance in VPC without www internet)
Logging and Audit:
- S3 access Logs can be stored in other S3 bucket
- API calls can be logged in AWS CloudTrail
User Security:
- MFA Delete: MFA (multi factor authentication) can be required in versioned buckets to delete objects
- Presigned URLs: URLs that are valid only for a limited time (ex: premium video service for logged in users)

Websites:
- S3 can host static websites and have them accessible to the www
The website URL will be 
bucket-name.s3-website-AWS-region.amazonaws.com
If you get a 403 (forbidden) error, make sure the bucket policy allows public reads

CORS:
an origin is a scheme (protocol), host(domain) and port
CORS means cross-origin resource sharing
Web browser based mechanism to allow requests to other origins while visiting the main origin
The requests won't be fulfilled unless the other origin allows for the requests, using CORS headers (ex: Access-Control-Allow-Origin)

S3 CORS:
If a client does a cross-origin request on our S3 bucket, we need to enable the correct CORS headers
It's a popular exam question
You can allow for a specific origin or for * (all origins)

Consistency Model
- Strong consistency as of December 2020:
After a:
- successful write of a new object (new PUT)
- or an overwrite or delete of an existing object (overwrite PUT or DELETE)
any:
- subsequent read request immediately receives the latest version of the object (read after write consistency)
- subsequent list request immediately reflects changes (list consistency)
Available at no additional cost, without any performance impact

S3- MFA Delete
- MFA (multi-factor authentication) forces user to generate a code on a device (usually a mobile phone or hardware) before important operations on S3
- To use MFA-Delete, enable Versioning on the bucket
You will need MFA to 
- permanently delete an object version
- suspend verioning on the bucket
You won't need MFA for
- enabling versioning
- listed deleted versions
Only bucket owner (root account) can enable/disable MFA-Delete
MFA-Delete curently can only be enabled using the CLI

S3 Default Encryption
- One way to "forcee encryption" is to use a bucket policy and refuse any API call to PUT an S3 object without encryption headers
- Another way is to use the "default encryption" option in S3
- Note: Bucket policies are evaluated before "default encryption"

S3 Access Logs
- For audit purposes, you may want to log all access to S3 buckets
- Any request made to S3, from any account, authorized or denied, will be logged into another s3 bucket
- That data can be analyzed using data analysis tools
- Or athena as we'll see later in this section
Warning:
- Do not set your logging bucket to be the monitored bucket
- It will create a logging loop and your bucket will grown in size exponentially

S3 Replication:
- Must enable versioning in source and destination
- CRR cross region replication
- SRR Same region replication
- Buckets can be in different accounts
- Copying is asynchronous
- Must give proper IAM permissions to s3
CRR use cases: compliance, lower latency access, replication across accounts
SRR use cases: log aggregation, live replication between production and test accounts
Notes:
- After activating, only new objects are replicated (not retroactive)
For delete operations:
- Can replicate delete markers from source to target (optional setting)
- Deletions with a version ID are not replicated (to avoid malicious deletes)
There is no "chaining" of replication
- If bucket ! has replication into bucket 2, which has replication into bucket 3
- Then objects created in bucket 1 are not replicated to bucket 3

Pre-signed URLS:
can generate pre-signed URLs using SDK or cli
- For downloads (easy, can use the cli)
- for uploads (harder, must use the SDK)
Valid for a default of 3600 seconds, can change timeout with --expires-in [TIME_BY_SECONDS] argument
USers given a pre-signed URL inherit the permissions of the person who generated the URL for Get/put
Examples:
- Allow only logged-in users to download a premium video on your S3 bucket
- Allow an ever changing list of users to download files by generating URLs dynamically
- Allo temporarily a user to upload a file to a precise location in our bucket

S3 Inventory
- List objects and their corresponding metadata (alternative to S3 List API operation)
Usage examples:
- Audit and report on the replication and sncryption status of your objects
- Get the number of objects in an S3 bucket
- Identify the toal storage of previous object versions
Generate daily or weekly reports
Output files: CSV, ORC or Apache Parquet
You can query all the data using Amazon Athena, Redshift, Presto, Hive, Spark...
You can filter generated report using S3 Select
USe cases: Business, Compliance, Regulatory needs,...

S3 Storage Classes
- Amazon S3 Standard - General Purposes
- Amazon S3 Standard- Infrequent Access (IA)
- Amazon S3 One Zone- Infrequent Access
- Amazon S3 Intelligent Tiering
- Amazon Glacier 
- Amazon Glacier Deep Archive
- Amazon S3 Reduced Redundancy Storage (deprecated - omitted)

Standard - General Purpose:
- High Durability (99.999999999%) of objects acreoss multiple AZ
- If you store 10,000,0000 objects with Amazon S3, you can on average expect to incur a loss of a single object once every 10,000 years
- 99.99% availability over a given year
- Sustain 2 concurrent facility failures
- Use cases: Big Data analytics, mobile & gaming applications, content distribution

Amazon S3 Standard- Infrequent Access (IA):
- Suitable for data that is less frequently accessed but requires rapid access when needed
- Same durability
- 99.9% Availability
- Low cost compared to S3 standard
- Sustain 2 concurrent facility failures
- Use Cases: As a data store for disaster recovery backups
- 30 day minimum

Amazon S3 One Zone- Infrequent Access
- Same as IA but data is stored in a single AZ
- Same durability but if AZ is destroyed, data will be lost
- 99.5% Availability
- Low latency and high troughput performance
- Supports SSL for data at transit and encryption at rest
- Low cost compared to IA (by 20%)- Use Cases: Storing secondary backup copies of on-premise data, or storing data you can create
- 30 day minimum

Amazon S3 Intelligent Tiering:
- Same latency and high throughput performance of s3 standard
- small montly monitoring and auto-tiering fee
- Automatically moves objects between 2 access tiers based on changing access patterns
- Designed for durability 99.999999999% of objects across multiple Availability Zones
- Resilient against events that impact an entire Availability Zone
- Designed for 99.9% availability over a given year
- 30 day minimum

Amazon Glacier:
- Low cost object storage meant for archiving/backup
- Data is retained for longer term (10s of years)
- Alternative to on-premises magnetic tape storage
- Average annual durability is 99.999999999%
- Cost per storage per month ($.0.004/GB) + retrieval cost
- Each item in Glacier is called "Archive" (up to 40TB)
- Archives are stored in "Vaults"
3 retirveal options:
- Expedited (1 to 5 minutes)
- Standard (3 to 5 hours)
- Bulk (5 to 12 hours)
- Minimum storage duration of 90 days

AMazon Glacier Deep Archive:
- for longer storage
- cheaper
Retrieval:
- Standar (12 hours)
- Bulk (48 hours)
Minimum storage duration of 180 days

Lifecycle rules:
- you can trnasition objects between storage classes
- for infrequently accessed object, move them to Standard_IA
- For archive objects you don't need in real time, Glacier or Deep Archive
- Moving objects can be automated using a lifecycle configuration
Transition actions: It defines when objects are transitioned to another storage class:
- Move objects to Standard IA class 60 days after creation
- Move to Glacier for archiving after 6 months
Expiration actions: configure objects to expire (delete) after some time
- Access log files can be set to delete after 365 days
- Can be used to delete old versions of files (if versioning is enabled)
- Can be used to delete incomplete multi-part uploads
Rules can be created for a certain prefix
Rules can be created fr certain object tags

ANalytics:
- You can setup s3 Analytics to help determine when to transition objects from standard to standar_IA
- does not work for one zone IA or glacier
- Report is updated daily
- Takes about 24 hours to 48 hours to first start
- Good first step to put together Lifecycle Rules (or improve them)

Baseline Performance:
- Amazon s3 automatically scaled to high request rates, latency 100-200ms
- You application can achieve at least 3500 Put/Copy/Post/Delete and 5500 Get/Head requests per second per prefix in a bucket
- There are no limits to the number of prefixes in a bucket
- If you spread reads across four prefixes evenly, you can achienve 22,000 requests per second for Get and Head

KMS Limitation:
- if you use SSE-KMS you may be impacted by the KMS limits
- When you upload, it calls the GenerateDataKey KMS API
- When you ddownload, it calls the Decrypt KMS API
- Count towards the KMS quota per second (5500, 10000, 30000 req/s based on region)
- You can request a quote increase using the service quotas console

S3 Performance:
Multi part upload
- Recommended for files >100MB
- must use for files > 5GB
- Can help parallelize uploads (speed up transfers)
S3 transfer Acceleration
- Increase transfer speed by transferring file to an AWS edge location which will forward the data to the S3 bucket in the target region
- Compatible with multi-part upload

S3 Byte-Range Fetches
- Parallelize GETs by requesting specific Byte ranges
- Better resilience in case of failures
Can be used to speed up downloads
Can be used to retrieve only partial data (for example the head of a file)

S3 Select and Glacier Select
- Retrieve less data using SQL by performing server side filtering
- Can filter by rows and columns (simple SQL statements)
- Less network transfer, less CPU cost client-side

S3 Event Notifications:
- S3:ObjectCreated,...
- Object name filtering possible
- can create as many "S3 events" as desired
- S3 event notifications typically deliver events in seconds but can sometimes take a minute or longer
- If 2 writes are made to a single non-versioned object at the same time, it is possible that only a single event notifivation will be sent
- If you want to ensure that an event notification is sent for every successful write, you can enable versioning on your bucket

S3 Glacier Operations
Vault operations:
- Create and Delete: delete only when there's no archives in it
- Retrieving Metadata: creation date, number of archives, total size of all archives,...
- Download Inventory: - list of archives in the vault (archive ID, creation date, size)
Glacier Operations:
- Upload: single operation or by parts (MultiPart upload) for larger archives
- Download: first initiate a retrieval job for the archive, Glacier then prepares it for download. User then has a limited time to download the data from staging server. Optionally, specify a range or portion of bytes to retrieve
- Delete: use Glacier Rest API or AWS SDKs by specifying archinve ID
Restore links have an expiry date
Retrieval Options:
- Expedited (1 to 5 minutes) - $0.03 per GB and $10 per 1000 requests
- Standard (3 to 5 hours) - 0.01 per GB and 0.03 per 1000 requests
- Bulk (5 to 12 hours) - 0.0025 per GB and 0.025 per 1000 requests

Vault Policies and Vault Lock
Each vault has:
- one vault access policy
- one vault lock policy
Vault policies are written in JSON
Vault Access policy is like a bucket policy (restrict user/account permissions)
Vault Lock Policy is a policy you lock, for regulatory and compliance requirements
- The policy is immutable, it can never be changed (that's why it's called LOCK)
- Example 1: forbid deleting an archive if less than 1 year old
- Example 2: implement WORM policy (write once, read many)

Notifications for Restore Operations
Vault Notification Configuration:
- Configure a vault so that when a job completes, a message is sent to SNS
- Optionally, specify an SNs topic when you initiate a job
S3 event notifications
- S3 supports the restoration of objects archived to S3 Glacier storage classes
- s3:ObjectRestore:Post => notify when object restoration initiated
- s3:ObjectRestore:Completed => Notify when object restoration completed

## Athena
- Serverless service to perform analytics directly against S3 files
- Use SQL Language to query the files
- Has a JDBC / ODBC driver
- Charged per query and amount of data scanned
- Supports CSV, JSON, ORC, AVRO and Parquet (built on Presto)
- Use Cases: Business intelligence/analytics/ reporting, amalyze & query VPC Flow Logs, ELB Logs, CloudTrail trails, etc.
- Exam tip: Analyze dat adirectly on S3 => use Athena

S3 Access Points
Each Access pont gets it's own DNS and policy to limit who can access it
- A specific IAM user/group
- One policy per Access Point => Easier to manage than complex bucket policies
Can restrict to traffic from a specific VPC
Access points are linked to a specific bucket (unique name per acct/region)

S3 Bucket Policy:
use to:
- Grant public access to the bucket
- Force objects to be encrypted at upload
- Grant access to another account (Cross account)
Optional conditions on:
- Public IP or elastic IP (not on private IP)
- Source VPC or Source VPC Endpoint - only works with VPC Endpoints
- CloudFront Origin Identity
- MFA

Batch Operations:
Perfomr Bulk operations on existing S3 objects with a single request, example:
- Modify object metatdata & properties
- Copy objects between s# buckets
- Replace object tag sets
- Modify ACLs
- Restore objects from S3 Glacier
- Invoke Lambda function to perform custom action on each object
A job consists of a list of objects, the action to perform and optional parameters
S3 batch operations manages retries, tracks progress, sends completion notifications, generate reports, etc..
You can use s3 inventory to get object list and use s3 select to filter your objects

MultiPart Upload:
- Upload large objects in parts (in any order)
- Failures: restart uploading only failed parts (imporved performance)
- Use Lifecycle policy to automate old parts deletion of unfinished upload after x days 
- upload using aws cli or aws sdk

## Snow
- Highly- secure, portable devices to collect and process data at the edge and migrate data into and out of AWS
- Data migration: Snowcone, Snowball edge, snowmobile
- Edge computing: Snowcone, Snowball Edge
Challenges:
- Limited connectivity
- Limited bandwidth
- High network cost
- Shared bandwidth )can't maximize the line)
- Connection Stabability
Solve: AWS snow family: offline devices to perform data migrations
If it takes more than a week to transfer over the network, use snowball devices

Snowball Edge:
- Physical data transport solution: Move TBs or PBs of data in or out of AWS
- Alternative to moving data over the network (and paying network fees)
- Pay per data transfer job
- Provide block storage and Amazon S3-compatible object storage
Snowball Edge Storage Optimized
- 80TB of HDD capacity for block volume and S3 compatible object storage
Snowball Edge Compute Optimized
- 42 TB of HDD capacity for block volume and S3 compatible object storage
Use Cases: large data cloud migrations, DC decomission, disaster recovery

Snowcone:
- small, portable computing, anywhere, rugged & secure, withstands harsh environments
- light (4.5 pounds, 2.1 kg)
- devic used for edge computing, storage and data transfer
- 8 TB of usable storage
- Use snow cone where snowball does not fit (space-constrained environment)
- Must provide your own battery/cables
- Can be sent back to AWS offline, or connect it to internet and use AWS DataSync to send data 

Snowmobile:
- Transfer exabytes of data
- Each Snowmobile has 100 PB of capacity (use multiple in parallel)
- High security, temperature controlled, GPS, 24/7 video surveillance
- Better than Snowmobile if you transfer more than 10 PB

Usage Process:
1. Request snowball devices from the AWS console for delivery
2. Install the snowball client/AWS OpsHub on your servers
3. Connect the snowball to your servers and copy files using the client
4. Ship back the device when you're done (goes to the right AWS facility)
5. Data will be loaded into an S3 bucket
6. Snowball is completely wiped

Edge Computing:
Process data while it's being created on an edge location
- A truck on the road, a ship at sea, mining underground
These locations may include:
- Limited/no internet access
- Limited/no easy access to computing power
We setup a Snoball Edge/Snowcone device to do edge computing
Use cases of edge computing:
- Preprocess data
- Machine learning at the edge
- Transcoding media streams
Eventually we can ship back the device to AWS
Snowcone (smaller):
- 2 CPUs, 4GB of memory, wired or wireless access
- USB-C power using a cord or the optional battery
Snowball Edge - Compute Optimized
- 52 vCPUs, 208 GB of RAM
- Optional GPU
- 42 TB usable storage
Snowball Edge - Storage Optimized
- Up to 40 vCPUs, 80 GB of RAM
- Object storage clustering available
Can run EC2 Instances & AWS Lambda Functions (using AWS IoT Greengrass)

OpsHub:
- Historically to use Snow Family devices, you needed a CLI
Today, you can use AWS OpsHub ( a software you install on your computer/laptop) to manage your Snow Family Device
- Unlocking and configuring single or clustered devices
- Transferring files
- Launching and managing instances running on Snow Family devices
- Monitoring device metrics (storage capacity, active instances on your devices)
- Launch compatible AWS services on your devices (ex. Amazon EC2 instances, AWS DataSync, Network File System (NFS))

Hybrid Cloud Storage:
Aws is pushing for "hybrid cloud"
- part of your infrastructure is on the cloud
- part of your infrastructure is on-premises
This can be due to
- Long cloud migrations
- security requirements
- compliance requirements
- IT strategy
S3 is a proprietary technology (unlike EFS/NFS) so how do you expose the S3 data on-premises
AWS Storage Gateway

Storage gateway:
- Bridge between on-premises, data and cloud data in S3
- Use cases: disaster recovery, backup and restore, tiered storage
3 types of Storage Gateway:
- File Gateway
- Volume Gateway
- Tape Gateway

File Gateway
- Configred s3 buckets are accessible using NFS and SMB protocol
- Supports S3 standard, S3 IA, S3 One Zone IA
- Buckets access using IAM roles for each File Gateway
- Most recently used data is cached in the file gateway
- Can be mounted on many servers
- Integrated with Active Directory (AD) for user authentication 

Volume Gateway
- Block storage using iSCSI protocol backed by s3
- Backed by EBS snapshots which can help restore on-premises volumes
- Cached volumes: low latency access to most recent data
- Stored volumes: entire dataset is on premise, scheduled backups to S3

Tape Gateway
- Some companies have backup processes using physical tapes
- With tape gateway, companies use the same process, but in the cloud
- Virtual Tape Library (VTL) backed by Amazon S3 and glacier
- Back up data using existing tape-based processes (and iSCSI interface)
- Works with leading backup software vendors

Storage Gateway - Harware appliance
- Using storage gateway means you need on-premises virtualization
- Otherwise you can use a storage gateway harward appliance
- you can buy it on amazon.com
- Works with File Gateway, Volume Gateway, Tape Gateway
- Has the required CPU, memory, network, SSD cache resources
- Helpful for daily NFS backups in small data centers

Extras:
File gateway is POSIX compliant (Linux File system)
- POSIX metadata ownership, permissions and timestamps stored in objects' metadata in S3
Reboot Storage Gateway VM: (e.g. maintenance)
- File Gateway: simply restart the storage gateway VM
- Volume and tape Gateway
1. Stop storage Gateway Service (AWS console, VM local Console, Storage Gateway API)
2. Reboot the Storage Gateway VM
3. Start Storage Gateway Service (AWS console, VM local console, Storage Gateway API)
Activations
2 ways to get activation key:
- using the Gateway VM CLI
- Make a web request to the gateway VM (Port 80)
Troubleshooting Activation Failures
- Make sure the Gateway has port 80 opened
- Check that the Gateway VM has the correct time and synchronizing it's time automatically to a Network Time Protocol (NTP) server
Volume Gateway Cache
Cached mode: only the most recent data is stored
Looking at cache efficiency:
- look at the CacheHitPercent metric (you want it to be high)
- Look at the CachePercentUsed (you don't want it to be high)
Create a larger cache disk
- Use the cache volume to clone a new volume of a larger size
- Select the new disk as the cached volume

## Amazon FSx
problem to solve: EFS is a shared POSIX system for linux systems

FSx for Windows
- FSx for windows is a fully managed Windows file system share drive
- Supports SMB protocol & Windows NTFS
- Microsoft Active Directory integrations, ACL, user quotas
- Built on SSD, scale up to 10s of GB/s, millions of IOPS, 100s PB of data
- Can be accessed from your on-premise infrastructure
- Can be configured to be Multi-AZ (high availability)
- Data is backed-up daily to S3
Single AZ
- automatically replicates data winthin az
- 2 generations: single AZ1 (SSD), Single AZ 2 (SSD and HDD)
Multi-AZ
- automatically replicates data across AZ (synchronous)
- Standby file server in a different AZ (automatically failover)


FSx for Lustre
- Lustra is a type of parallel distributed file system for large-scale computing
- the name Lustre is derived from linux and cluster
- Machine learning, High performance computing (HPC)
- Video processing, Financial modeling, elctronic design automation
- Scales up to 100s GB/s, millions of IOPS, sub ms latencies
seamless integration with s3
- Can read s3 as a file system (through FSx)
- Can write the output of the computations back to S3 (through FSx)
Can be used from on-premises servers

FSx File system Deployment Options
Scratch File System:
- Temporary storage
- Data is not replicated (doesn't persist if file server fails)
- high burst (6x faster, 200MBps per TB)
- Usage: short term processing, optimize costs
Persistent File system:
- Long term storage
- data is replicated within same AZ
- Replace failed files within minutes
- Usage: long-term processing, sensitive data

## CloudFront
Content delivery network (CDN)
Improves read performance, content is cached at the edge
216 points of presence globally (edge locations)
DDoS protection, integration with Shield, AWS Web Application Firewall
Can expose external HTTPS and can talk to internal HTTPS backends

Cloudfront Origins:
S3 bucker;
- For distributed files and caching them at the edge
- Enhanced security with CloudFront Oirigin Access Identity (OAI)
- Cloudfront can be used as an ingress (to upload files to S3)
Custom Origin (HTTP)
- Application load balancer
- EC2 instance
- S3 website (must first enable the bucket as a static S3 website)
- Any HTTP backend you want

Geo Restriction
Wou can restrict who can access your distribution
- Whitelist: allow your users to access content only if they're in one of the countries on a list of approved countries
- Blacklist: Prevent your users from accessing your content if they're in one of the countries on a blacklist of banned countries
The "country" is determined using a 3rd party Geo-IP database
Use case: Copyright Laws to control access to content

vs. S3 CRR
Cloudfront:
- Global edge network
- Files are cached for a ttl (maybe a day)
- great for static content that be available anywhere
S3 cross region replication:
- Must be setup for each region you want replication to happen
- Files are updated in near real- time
- REad only
- Great for dynamic content that needs to be available at low latency in few regions

CloudFront Access Logs
- Logs every request made to Cloudfront into a logging S3 bucket

CloudFornt Reports:
Its possible to generate reports on:
- cache Statistics Report
- popular objects report
- Top Referrers Report
- Usage reports
- Viewers Reports
These reports are based on the data from the access logs

CloudFront troubleshooting
Cloudfront caches HTTP 4xx and 5xx status codes returned by s3

Cloudfront Caching:
Cache based on:
- headers
- session cookies
- query string parameters
The cache lives at each CloudFront Edge location
You want to maximize the cache hit rate to minimize requests on the origin
Control the TTL (0 seconds to 1 year), can be set by the origin using the Cache-Control header, Expires header...
You can invalidate part of the cache using the CreateInvalidation API
 
Headers:
1. Forward all headers to your origin
- no caching, every request to origin
- ttl must be set to 0
2. Forward a whitelist of headers
- caching based on values in all the specified headers
3. None == Forward only the default headers
- no caching based on request headers
- best caching performance
origin cutom headers
- origin-level setting
- set a constant header/header value for all requests to origin
Behaviour settings:
- cache related settings
- contains the whitelist of headers to forward

CloudFront Caching TTL
- "Cache-control: max age" is prefered to "Expires" header
- If the origin always sends back the header Cache-Control, then you can set the TTL to be controlled only by that header
- In case you want to set min/max boundaries, you choose "customize" for the object Caching setting
- In case the Cache-Control header is missing, it will default to "default value"

cloudFront cache behaviour for cookies
Cookies is a specific request header
1. Default do not process the cookies
- Caching is ot based on cookies
- cookies are not forwarded
2. Forward a whitelist of cookies
- Caching based on values in all the specified cookies
3. Forward all cookies
- worst caching performance

cloudFront cache behaviour for query strings
query strings parameters are in the url
1. default: do not process the query strings
-  Caching is not based on query strings
- Parameters are not forwarded
2. Forward a whitelist of query strings
- Caching based on the parameter whitelist
3. Forward all query strings
- Caching based on all parameters


Maximize cache hits:
maximize by seperating static and dynamic distributions

Increase Cache Ratio
- Monitor the CloudWAtch metric CacheHitRate
- Specify how long to cache your objects: Cache-Control max age header
- Specify none or the minimally required headers
- Specify none or the minimally required cookies
- Specify none or the minimally required query string parameters
- Seperate static and dynamic distributions (two origins)

Cloudfront with ALB sticky sessions:
- Must forward/whitelist the cookie that controls the session affinity to the origin to allow the session affinity to work
- Set a TTL to a value lesser than when the authentication cookie expires

## Databases

RDS
- RDS stands for Relational Database Service 
- It's a managed DB service for DB use SQL as a query language
- It allows you to create databases in the cloud that are managed by AWS
PostgreSQL, MySQL, MariaDB, Oracle, Microsoft SQL Server, Aurora

Advantages over using RDS vs deploying a DB on EC2
RDS is a managed service:
- Automated provisioning, OS patching
- Continuous backups and restore to specific timestamp ( Point in time restore)
- Monitoring dashboards
- Read replics
- Multi AZ setup for DR
- Maintenance windows for ungrades
- Scaling capability
- Storage backed by EBS
But you can't SSH into your instances

RDS storage auto Scaling
- helps you inscrease storage on your RDS DB instance dynamically
- When RDS detects you are running out of free database storage, it scales automatically
- Avoid manually scaling your DC storage
- You have to set Maximum Storage Threshold (maximum limit for DB storage)
Automaticall modify storage if
- free storage is less than 10% of allocated storage
- Low-storage lasts at least 5 minutes
- 6 hours have passed since last modificationo
Useful for applications with unpredictable workloads
Supports all RDS db engines (Mariadb, mysql postgres, sql server, oracle)

Read Replicas
- Up to 5 read replicas
- Within AZ, Cros AZ, Cross region
- Replication is ASYNC, so reads are eventually consistent
- Replicas can be promoted to their own DB
- Applications must update the connection string to leverage read replicas
Use Cases:
- You have a promduction databas that is taking on normal load
- you want to run a reporting application to run some snalytics
- you create a read replica to run the new workload there
- the production application is unaffected
- read replicas are used for select (=read) only kind of statements (not insert, update or delete)
Netowkr cost:
- In AWS there's a network cost when data goes from one az to another
- For RDS Read Replicas within the same region, you don't pay that fee

RDS Multi-AZ
- SYNC replication
- One DNS name - automatic app failover to standby 
- Increase availability
- Failover in case of loss of AZ, loss of network, instance or storage failure
- No manual intervention in apps
- Not used for scaling
- Multi- AZ replication is free
- Note: the Read Replicas can be setup as Multi AZ for disaster Recovery
From single AZ to Multi- AZ:
- Zero dwontime operation(no need to stop the DB)
- Just click on "modify" for the database
The following happens internally:
- A snapshot is taken
- A new db is restored from the snapshot in a new az
- synchronization is established between the 2 db
Failover Conditions:
The primary DB instance
- Failed
- OS is undergoing software patching
- Unreachable due to loss of network connectivity
- Modified (e.g. DB instance type changed)
- Busy and unresponsive
- Underlying storage failure
An AZ outage
A manual failover of the db instance was initated using reboot with failover
Encryption:
At rest:
- Possibility to encrypt the master and read replicas with AWS KMS - AES 256 encryption
- Encryption has to be defined at launch time
- If the master is not encrypted, the read replicas cannot be encrypted
- Transparent Data Encryption (TDE) available for Oracle and SQL Server
In-Flight encryption:
- SSL certificates to encryp data to RDS in flight
- Provide SSL options with trust certificate when connecting to DB
To enforce SSL:
PostgreSQL: rds.force_ssl=1 in the RDS console (Parameter group)
MySQL: Within the DB: Grant usage *.* to 'mysqluser'@'%' Require SSL;

RDS Encryption Operations
Encrypting RDS backups:
- Snapshots on un-encrypted RDS databases are un-encrypted
- Snapshots of encrypted RDS databases are encrypted
- Can copy aa snapshot into an encrypted one
Encrypt an un-encrypted RDS db
- Create a snapshot of the un-encrypted db
- Copy the snapshot and enable encryption for the snapshot
- Restore the db from the encrypted snapshot
- Migrate applications to the new db, and delete the old db

Security - IAM and Network
NetworkL
- RDS db are usually deployed within a private subnet, not in a public one
- RDS security works by leveraging security groups (the same concept as for the EC2 instances) - it controls which IP/ security group can communicate with RDS
Access Management:
- IAM policies help control who can manage AWS RDS (through the RDS API)
- Traditional Username and Passowrd can be used to login into the db
- IAM based authentication can be used to login into RDS MYSQL and PostgreSQL

IAM Authentication
- IAM db authentication works with MySQL and PostGReSQL
- You don't need a password, just an authentication token obtained through IAM & RDS API calls
- Auth Token has a lifetime of 15 minutes
Benefits: 
- Network in/out must be encrypted using SSL
- IAM to centrally manage users instad of DB
- Can leverage IAM and EC2 instance profiles for easy integration

Security - Summary:
encryption at rest:
- is done when you first create the db instance
- or unencrypted db ->snapshot->sopy snapshot as encrypted->create db from snapshot
your responsibility:
- check the ports/ip/security group inbound rules in DB's SG
- In-db user creation and permissions or manage through IAM
- Creating a db with or without public access
- Ensure parameter groups or db is configured to only allow SSL connections
AWS responsibility:
- No SSH access
- No manual DB patching
- No manual OS patching
- No way to audit the underlying instance

Lambda by default:
- 



























































































































