# 3-Tier Web Application Architecture on AWS

## Overview

In this project i implemented a highly available, secure 3-tier web application architecture on AWS consisting of a Web tier, an Application tier and a Database tier. The infrastructure runs inside a custom VPC (`192.168.0.0/22`) spanning two Availability Zones, with the Web tier deployed in public subnets behind an internet-facing Application Load Balancer and the App and Database tiers isolated in private subnets with no direct internet exposure. Outbound internet access for the private tiers is provided through a single NAT Gateway. Traffic flows from an external Application Load Balancer to a Web tier Auto Scaling Group, then to an internal Application Load Balancer that fronts the App tier, which in turn communicates with a MySQL/Aurora RDS instance in the Data tier. Access control between tiers is enforced entirely through security groups, each scoped to allow only the traffic it needs from the specific tier or CIDR range in front of it. Application code is pulled from an S3 bucket via an IAM role i attached to the EC2 instances and instance access is managed exclusively through AWS Systems Manager (SSM), removing the need for SSH or bastion hosts.

## AWS Services that i used

- **VPC** — Custom network (`192.168.0.0/22`) with 2 public and 4 private subnets across 2 Availability Zones
- **EC2** — Web tier and App tier compute instances
- **Elastic Load Balancing (ALB)** — External (internet-facing) and internal Application Load Balancers
- **Auto Scaling** — Web tier Auto Scaling Group with a custom AMI and launch template
- **RDS (MySQL/Aurora)** — Managed database in the Data tier
- **IAM** — Roles for EC2-to-S3 access and SSM connectivity
- **S3** — Storage and deployment source for application code
- **Systems Manager (SSM)** — Secure, keyless access to private EC2 instances
- **NAT Gateway** — Outbound internet access for private subnets
- **CloudWatch** — Monitoring for the Auto Scaling Group

## What Was Configured and Why

### Subnet Design
While creating the VPC,i created 2 public subnets (Web tier) and 4 private subnets (2 for the App tier, 2 for the Data tier), spread across 2 Availability Zones(AZ). This separation keeps the App and Database tiers completely unreachable from the internet while still letting the Web tier serve public traffic and the multi-AZ layout provides fault tolerance if one AZ becomes unavailable. A single NAT Gateway (in one AZ) gives the private subnets outbound internet access , for example, to pull OS or package updates  without exposing them to inbound traffic.

### Security Groups
I created five(5) security groups so as to enforce least-privilege access between the tiers:
- **External LB SG** — allows inbound HTTP from anywhere (public entry point)
- **Web Tier SG** — allows inbound HTTP only from the External LB SG and the VPC CIDR range(192.168.0.0/22)
- **Internal LB SG** — allows inbound HTTP only from the VPC CIDR range
- **App Tier SG** — allows inbound HTTP only from the VPC CIDR range
- **Database Tier SG** — allows inbound MySQL/Aurora traffic only from the VPC CIDR range

This chained design means each tier only accepts traffic from the tier immediately in front of it, rather than being broadly open, significantly reducing the attack surface.

### IAM Roles and Deployment
I created an IAM role which i names, (`3-Tier-EC2-Role`) and attched the `AmazonEC2RoleforSSM` policy to the EC2 instances and S3 bucket. This served two purposes: it let application code be pulled from the S3 bucket (`3-tier-webarch-project`) into the EC2 instances without hardcoding credentials and it enabled Session Manager access to instances in private subnets, removing the need for SSH keys or a bastion host.

### Load Balancing and App Configuration
I created an internal ALB which was placed in front of the App tier (target group on port 4000) so the Web tier could reach the App tier reliably without knowing individual instance addresses. The internal ALB's DNS name was written into `nginx.conf` and pushed back to the S3 bucket, so any Web tier instance pulling that config automatically routes application traffic to the App tier correctly.

### Scaling Policy
I don't want my public IP to be visible to everyone so i converted the Web tier into a launch template and AMI (`WebTier-AMI`), then attached them to an Auto Scaling Group (`WebTier-ASG`) with a minimum capacity of 2 and a maximum of 4. This ensures the Web tier always has redundancy across AZs and can scale out under load, while an external ALB (`WebTier-TG`) distributes incoming traffic evenly across the running instances.My app is then accessed through the External load balancer's DNS

### Monitoring
CloudWatch monitoring was enabled on the Auto Scaling Group to track instance health and performance, providing visibility needed to support scaling decisions and catch issues with Web tier instances early.

## Key Learnings / Results

- I was able to successfully deploy a fully functional 3-tier architecture with strict network segmentation , the App and Database tiers have no direct internet exposure.
- I used SSM instead of SSH/bastion hosts significantly simplified secure access management for private instances.
- I chained security groups (tier-to-tier rather than broad CIDR-only rules) which made it a clean way to enforce least-privilege access without manual IP management.
- I was able to pull the  application code from S3 via an IAM role simplified deployment and made it easy to update configuration (e.g., the internal ALB DNS name in `nginx.conf`) without rebuilding instances from scratch.
- The Auto Scaling Group + external ALB combination confirmed the app was reachable only through the load balancer's DNS, not directly via public IP, validating the security design.
- I performed an end-to-end application test by accessing the external ALB DNS name from a browser. The application loaded successfully, confirming connectivity between the web, application, and database tiers.
- I tested Auto Scaling resilience by terminating the running EC2 instances and observing new instances being automatically launched by the Auto Scaling Group. This demonstrated the architecture's ability to automatically recover from instance failures and maintain application availability.
- After completing the project and testing the architecture,i terminated the AWS resources and services that were no longer required to avoid unnecessary ongoing charges.
