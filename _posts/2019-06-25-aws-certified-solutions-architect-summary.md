---
layout: post
title: AWS Certified Solutions Architect Summary
date: 2019-06-25
tags: [ aws ]
---

I've preparing [the AWS Certified Solutions Architect exam](https://aws.amazon.com/es/certification/certified-solutions-architect-associate/) running [the Udemy course](https://www.udemy.com/aws-certified-solutions-architect-associate) driven by [Ryan Kroonenburg](https://twitter.com/acloudguru).

AWS cloud stack is the most popular cloud infrastructure and since I've using AWS stack for some years already, I decided to prepare myself this exam to go deeper my specialization. 

# The exam composition

- 130 minutes in length
- 60 questions (scenario based questions)
- Multiple choice
- Results between 100 - 1000 with a passing score of 720
- Qualification is valid for 2 years

# AWS - 10000 Foot Overview

![AWS Stack]({{ site.url }}{{ site.baseurl }}/images/aws_stack.png)

Where the exam focuses on:
- AWS Global Infrastructure
- Security, Identity & Compliance
- Compute: ec2, lambda
- Storage: s3
- Databases: RDS, DynamoDB
- Network & Content Delivery: CloudFront, Route 53, VPC

## Regions, Availability Zones, Edge Locations

Number of Edge Locations > Number of Availability zones > Number of regions

**A region** is a physical location spread across globe to host your data to reduce latency. In each region there will be at least two **availability zones**.

**An availability zone** is a datacenter that does not need to be separated by multiple kilometers physically but by meters with in a physical compound which are completely isolated from each other failure such as power, network in a given AZ.

**An edge location** is where end users access services located at AWS. A site that CloudFront uses to cache copies of your content for faster delivery to users at any location. Edge locations serve requests for CloudFront and Route 53. Requests going to either one of these services will be routed to the nearest edge location automatically. This allows for low latency no matter where the end user is located. 

# IAM: Identity Access Management

Manage users and their level of access to AWS Console. The basics:

- Allow very granular permissions to leverage the user access.
- Identify Federation (including Activate Directory, Facebook, Linkedin, etc).
- Multifactor Authentication
- Password policies
- IAM is universal. It does not apply to regions at this time. 
- The "root account" is simply the account created when first setup your AWS ccount. It has complete Admin access. We should setup the Multifactor authentication here for security.
- New Users have no premissions at the beginning.
- New Users can be configured to access via web or programmatically or both. When programmatically, they need to be assigned an Access Key ID & Secret Access Keys.

**Users** can be organised in **Groups** of people.
**Policy** is a set of JSON documents with the permissions. Policies can be grouped in **Roles**. We can create new roles as needed. 
Then, either **Users** or **Groups** can be attached to **Policies** or **Roles**. 

## Create a billing alarm

1. Go to My Account > Billing Dashboard > Billing Preferences
2. Enable Receive Billing Alarts
3. Then, go to Services > Cloud Watch > Billing
4. Fill the Billing alarm section (with the amount of dollars)

## Roles

1. IAM > Roles > Create Role
2. Choose the service that will use this role
3. Attach policies

- Roles are more secure than storing your access key and secret access key on individual EC2 instances.
- Roles are easier to manage.
- Roles can be assigned to an EC2 instance after it is created using both the console & command line.
- Roles are universal - you can use them in any region.

# AWS Command Line

```bash
> sudo su
> aws configue
AWS Access Key ID: xxx
AWS Secret Access Key: YYY
Default region name: zzz
Default output format: 
> aws s3 ls -> see the list of buckets in S3
> aws s3 mb s3://testbucket -> make bucket
```

# S3

Secure, durable, highly-scalable object storage. The basics:
- S3 is Object-based. An object is a key (the name), value (the content of the file in bytes) and a version ID (important for versioning), some metadata, other subresources (like access control lists and torrent).
- Files can be from 0 Bytes to 5 TB.
- There is unlimited storage.
- The objects are organised in buckets, that are like folders. The bucket name must be unique globally. 
- The objects are directly available when adding new objects, but there will be eventual consistencies on updates (PUTS and DELETES).
- 99.9(...)% availability and durability.
- Tiered Storage available: Store objects in different tiers with different availability configuration (to save up money).
- Lifecycle Management: it allows to move your objects in different tiers depending on the activity and time windows to save up money. It also can be used to configure expiration.
- Versioning: Have multiple versions of objects. Once enabled versioning, it cannot be disabled.
- Encryption: SSL/TLS (transport connection). SS3-S3 (server side encryption)
- MultiFactor authentitcation for deleting objects
- Secure your data using access control lists or using bucket policies.

## Storage Classes
- S3 Standard
- S3 IA: Infrequently Accessed but requires rapid access. Lower fee then standard, but you're charged a retrieval fee.
- S3 One Zone - IA: Infrequently Accessed and do not require the multiple availability zones. Lowest option.
- S3 Intelligent Tiering: Use machine learning to configure the objects around storage classes to the most cost-effective option.
- S3 Glacier: For data archiving. Secure, durable, low-cost storage. Retrieval times configurable from minutes to hours. 
- S3 Glacier Deep Archive: Same as above but it allows retrieval times of 12 hours. 

![AWS Storage Classes]({{ site.url }}{{ site.baseurl }}/images/aws-s3-storage-classes.png)

## Transfer Acceleration

Amazon S3 Transfer Acceleration enables fast, easy, and secure transfer of files over long distances between your end users and an S3 bucket. It takes advantage of Amazon CloudFront's globally distributed edge locations: as the data arrives at an edge location, data is routed to Amazon S3 over an optimized network path.

## Security

We can configure S3 to log who is accessing the objects. Secure your data using **access control lists** or using **bucket policies**. 

We can encrypt by:
- Encryption In Transit: SSL/TLS
- Encryption At Rest (Server Side) via (1) S3 Managed Keys - SSE-S3, (2) AWS Key Management Service - SS3-KMS, or (3) Server Side Encryption with customer - SS3-C (you provide the keys).
- Client Side Encryption

## Cross Region Replication

1. Create bucket
2. Go to Management and Replication
3. Enable versioning (it must be enabled in both source and destination)
4. Set Source and Destination
5. Configuration options: select/create the role

Files in an existing bucket and delete markers (or versions) are not replicated automatically. It only works for new files.

## Pricing
- By Storage class
- By Requests
- By Storage Management Pricing
- By Data Transfer Pricing
- By Transfer Acceleration
- By Cross Region Replication Pricing

# CloudFront

A content delivery network (CDN) is a system of distributed servers (network) that deliver webpages and other web content to a user based on the geographic locations of the user, the origin of the webpage, and a content delivery server.
- Edge Location: where content will be cached (using a TTL - Time To Live). We can clear cached objects, but we will be charged.
- Origin: This is the origin of all the files that the CDN will distribute. This can be an S3 bucket, an EC2 instance, an Elastic Load Balancer, or Route53.
- Distribution: This is the name given the CDN which consists of a collection of Edge Locations.
- Types: (1) **Web Distribution**: To route whole websites; and (2) **RTMP**: Used for Media Streaming.

# Storage Gateway

Service that connects an on-premises software appliance with cloud-based storage to provide seamless and secure integration between an organization's on-premises IT environment and AWS's storage infrastructure. The service enables you to securely store data to the AWS cloud for scalable and cost-effective storage. 

| Type  | Description | 
| ------------- | ------------- | 
| File Gateway (NFS) | Files are stored as objects in your S3 buckets, accessed throught a NFS mount point. | 
| Volume Gateway (iSCSI) | Same using virtual directories via iSCSI block protocol. Files are stored in the cloud as Amazon EBS snapshots. Two types: (1) Stored volumnes and (2) Cached volumes. |
| Type Gateway (VTL) | It offers durable, cost-effective solution to archive your data in the AWS Cloud (same mecanism as Volume Gateway). |

# EC2: Elastic Compute Cloud

EC2 is a web service that provides resizable compute capacity in the cloud. It reduces the time required to obtain and boot new server instances to minutes, allowing you to quickly scale capacity, both up and down, as your computing requirements change.

- Termination Protection: Disabled by default. We need to enable it in order to prevent our EC2 instance is accidentally shutdown. 
- By default, the EC2 is linked to a EBS volume that will be deleted when the instance is terminated. This default volume can't be encrypted. But we can add additional volumes that can be encrypted. 
- About security groups: All Inbound traffic is blocked by default. All outbound traffic is allowed. Changes to Security Groups take effect immediately. The security groups can be shared by different EC2 instances. We cannot configure with deny rules, only allow rules (in order to block IP addresses, we need to use Network Access Control Lists). 
- Metadata: useful to get information about an instance (curl http://ec-ip/latest/meta-data). Also to get user data (curl http://ec-ip/latest/user-data). 

## Pricing

| Type  | Description | Use Cases |
| ------------- | ------------- | ------------- |
| On Demand | Fixed rate by the hour (or by the second) | Low cost and flexibility. (1) Applications with short term, spiky or unpredictable workloads that cannot be interrupted. (2) Applications being developed or tested for the first time. |
| Reserved: Standard or Convertible or Scheduled | Provides you with a capacity reservation, and offer a significant discount on the hourly charge for an instance. Contract terms are 1 year or 3 years. | (1) Applications with steady state or predictable usage. (2) Applications that require reserved capacity. (3) Users able to male upfront payments to reduce their total computing costs even further. |
| Spot | Enables you to bid whatever price you want for instance capacity, providing for even greater savings if your applications have flexible start and end times. | (1) Applications that have flexible start and end times. (2) Applications that are only feasible at very low compute prices. (3) Users with urgent computing needs for large amounts of additional capacity. |
| Dedicated hosts | Physical EC2 server dedicated. It can help you reduce costs by allowing you to use your existing server-bound software licenses. It can be purchased On-Demand (hourly) or as a reservation for up to 70% off the On-Demand price. | (1) Useful for regulatory requirements that may not support multi-tenant virtualization. (2) Great for licensing which does not support multi-tenancy or cloud deployments. |

## Instance Types

![AWS EC2 Instance Types]({{ site.url }}{{ site.baseurl }}/images/aws-ec2-instance-types.png)

This won't be part of the exam. The summary is:
- F: For FPGA
- I: For IOPS
- G: Graphics
- H: High Disk Throughput
- T: Cheap general purpose
- D: For Density
- R: For RAM
- M: Main choice for general purpose apps
- C: For Compute
- P: Graphics (think Pics)
- X: Extreme Memory
- Z: Extreme Memory and CPU
- A: ARM-Based workloads
- U: Bare Metal

## Placement Groups

The name of placement groups must be unique within your AWS account. Only certain types of instances can be launched in a placement group: compute optimized, GPU, memory optimized and storage optimized. We can't move existing instances into a placement group (they must be selected when are being created).

- Clustered Placement Group: group instances within a single availability zone. This is recommended for applications that need low network latency, high network throughput or both. Only certain instances can be launched in this mode.

- Spread Placement Group: This is the opposite. Instances that are each placed on distinct underlying hardware. This is recommended for applications that have a small number of critical instances that should be kept separate from each other. 

# EBS: Elastic Block Store

EBS provides persistent block storage volumes for use with EC2 instances. EBS volume is like a virtual hard disk in the cloud. Each EBS volume is automatically replicated within its availability zone to protect you from component failure, offering high availability and durability. 

- We can create snapshots on S3: point in time of Volumes. Snapshots are also incremental which means that only the blocks that have changed since your last snapshot are moved to S3. In order to create a snapshot for EBS that serve as root devices, you should stop the instance before taking the snapshot. We can create AMI's from both volumes and snapshots. 
- Snapshots of encrypted volumes are encrypted automatically. Volumes restored from encrypted snapshots are encrypted automatically. Only unencrypted snapshots can be shared. These snapshots can be shared with other AWS accounts or made public.
- We can change the EBS volume size or the storage type on the fly. 
- EBS Volumes will always be in the same availability zone as the EC2 instance. 
- Storage Types:

| Type | Description | Use Cases | API Name | Volume Size | Max. IOPS / Volume |
| ------------- | ------------- | ------------- | ------------- | ------------- |
| General Purpose (SSD) | General purpose that balances price and performance | Most work loads | gp2 | 1GB-16TB | 16000 |
| Provisioned IOPS (SSD) | Highest-performance SSD for mission-critical applications | Databases | io1 | 4GB-16TB | 64000 |
| Throughput Optimised Hard Disk Drive | Low cost HDD for freq. accessed, throughput-intensive workloads | Big Data & Data warehouses | st1 | 500GB-16TB | 500 |
| Cold HDD | Lowest cost HDD for less freq. accessed workloads | File Servers | sc1 | 500GB - 16TB | 250 |
| EBS Magnetic | Previous generation HDD | Workloads where data is infreq. accessed | Standard | 1GB-1TB | 40-200 |

## EBS-Backed Versus Instance Store

**An instance store** provides temporary block-level storage for your instance. Instance Store Volumes are sometimes called Ephemeral Storage. This storage is located on disks that are physically attached to the host computer. Instance store is ideal for temporary storage of information that changes frequently, such as buffers, caches, scratch data, and other temporary content, or for data that is replicated across a fleet of instances, such as a load-balanced pool of web servers.

- Instance store volumes cannot be stopped. If the underlying host fails, you will lose your data.

**An "EBS-backed" instance** is an EC2 instance which uses an EBS volume as it’s root device. EBS volumes are redundant, "virtual" drives, which are not tied to any particular hardware, however they are restricted to a particular EC2 availability zone. This means that an EBS volume can move from one piece of hardware to another within the same availability zone. You can think of EBS volumes as a kind of Network Attached Storage.

If the virtual machine’s hardware fails, the EBS volume can simply be moved to another virtual machine and re-launched. In theory, you won’t lose any data.

Another benefit, is that EBS volumes can easily be backed up and duplicated. So you can take easy backup snapshots of your volumes, create new volumes and launch new EC2 instances based on those duplicate volumes.
- EBS backed instances can be stopped. You'll not lose the data on this instance if it is stopped.
- By default, both root volumes will be deleted on termination. However, with EBS volumes, you can tell AWS to keep the root device volume.

# EFS: Elastic File System

Elastic File System is a file storage service for EC2 instances. EFS is easy to use and provides a simple interface that allows you to create and configure file systems quickly and easily. With EFS, storage capacity is elastic, growing and shrinking automatically as you add and remove files, so your applications have the storage they need, when they need it.

Once EC2 is created, we need to mount the EFS disks:
- TLS mount for encrypted option

```bash
mount -t efs -o tls fs-xxxx:/ /target
```

- It also supports network file system version 4 (NFSv4).
- We only pay for the storage you use (no pre-provisioning required).
- Can scale up to petabytes.
- Can support thousands of concurrent NFS connections.
- Data is stored across multiple AZ's within a region.
- Read After Write Consistency. (No eventual consistency)

# CloudWatch

CloudWatch is a monitoring service to monitor your AWS resources, as well as the applications that you run on AWS. Metrics like CPU, network, disk, status check. 

- CloudWatch is about monitoring performance.
- CloudWatch with EC2 will monitor events every 5 minutes by default. 
- You can have 1 minute intervals by turning on detailed monitoring.
- You can create CloudWatch alams which trigger notifications. 
- **Dashboards:** Creates awesome dashboards to see what is happening with your AWS environment.
- **Alamrs:** Allows you to set Alarms that notify you when particular thresholds are hit.
- **Events:** Helps you to responde to state changes in your AWS resources.
- **Logs:** Aggregates log data. 

# CloudTrail

AWS CloudTrail increases visibility into your user and resource activity by recording AWS Management Console actions and API calls. You can identify which users and accounts called AWS, the source IP address from which the calls were made, and when the calls occurred.

- CloudTrail is about auditing.