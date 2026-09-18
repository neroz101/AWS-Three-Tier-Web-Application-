# AWS Three-Tier Application - Setup Notes

## VPC Configuration

| Configuration | Value |
|---|---|
| VPC CIDR | 192.168.0.0/22 |
| Availability Zones | 2 |
| Architecture | Three-tier |
| Internet-facing component | External Application Load Balancer |

## Subnet Design

### Availability Zone 1

- Public Subnet
- Private App Subnet
- Private Data Subnet

### Availability Zone 2

- Public Subnet
- Private App Subnet
- Private Data Subnet

The application and database tiers were placed in private subnets
and were not directly exposed to the internet.

## Load Balancers

### External ALB

- Internet-facing Application Load Balancer
- Security Group: `External-LB-SG`
- Routes external traffic to the Web Tier

### Internal ALB

- Internal Application Load Balancer
- Security Group: `Internal-LB-SG`
- Target Group Port: `4000`
- Routes traffic from the Web Tier to the App Tier

## EC2 Web Tier

- Auto Scaling Group
- Security Group: `Web-Tier-SG`
- Minimum instances: 2
- Maximum instances: 4
- Instances deployed across two Availability Zones

## EC2 App Tier

- Security Group: `App-Tier-SG`
- Deployed in private App subnets
- No direct internet exposure
- Application traffic received through the Internal ALB

## Database Tier

- Amazon RDS MySQL/Aurora
- Security Group: `DB-Tier-SG`
- Subnet Group: `rds-sub-grp`
- Database deployed in private Data subnets
- No direct internet access

## NAT Gateway

- NAT Gateway deployed in the public subnet
- Used to provide outbound internet access for private resources when required
- Private App subnets route outbound traffic through the NAT Gateway

## Security Groups

Security groups were configured using tier-to-tier access rather than
broad CIDR-based rules.

### External-LB-SG

Allows:

- HTTP/HTTPS from the internet

### Web-Tier-SG

Allows:

- Application traffic from `External-LB-SG`

### Internal-LB-SG

Allows:

- Application traffic from `Web-Tier-SG`

### App-Tier-SG

Allows:

- Application traffic from `Internal-LB-SG`

### DB-Tier-SG

Allows:

- Database traffic from `App-Tier-SG`

This approach avoids exposing application and database resources
directly to the internet and follows least-privilege network access.

## Secure Instance Access

AWS Systems Manager (SSM) was used to access private EC2 instances
instead of SSH or a bastion host.

This avoided the need to expose SSH port 22 to the internet.

## IAM

EC2 IAM roles were used to allow instances to retrieve application
code from Amazon S3 without storing AWS access keys on the instances.

## Application Deployment

Application code was stored in:

`3-tier-webarch-project`

EC2 instances retrieved the application code from S3 using their
assigned IAM role.

The internal ALB DNS name was configured in `nginx.conf`.

## Monitoring

Amazon CloudWatch was used to monitor the AWS infrastructure and
application resources.

## Resilience Testing

EC2 instances were manually terminated to test the Auto Scaling Group.

New instances were automatically launched by the Auto Scaling Group,
demonstrating automatic recovery from instance failures.

## Cleanup

After completing the project and testing the architecture, AWS
resources that were no longer required were deleted to avoid
unnecessary ongoing charges.
