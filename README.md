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










