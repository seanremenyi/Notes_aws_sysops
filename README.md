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
- When you crea a placement group, you specify one of the following strategies for the group:
Cluster - clusterrs instance into a lot latency group in a single Availabiliy zone
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










































































































