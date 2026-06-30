# AWS Secure Three-Tier Web Application Deployment

A secure, scalable, and highly available three-tier web application architecture on AWS, built around the AWS Well-Architected Framework's security principles: least privilege, defense in depth, secure networking, identity management, encryption, centralized logging, and infrastructure automation.

## Overview

This project applies cloud security best practices by isolating resources across public and private subnets, implementing network segmentation, restricting inbound access with Security Groups, deploying an internet-facing Application Load Balancer (ALB), protecting the application with AWS WAF, securely managing database credentials with AWS Secrets Manager, encrypting data with AWS KMS, and monitoring infrastructure with CloudWatch, CloudTrail, and SNS.

The environment includes a custom VPC with public and private subnets, a Bastion Host for controlled administrative access, an internet-facing Application Load Balancer protected by AWS WAF, an Auto Scaling Group of Windows/IIS web servers built from a custom AMI, a Multi-AZ Amazon RDS for MySQL database, IAM least-privilege roles, AWS Secrets Manager for credential storage, and a full observability stack using CloudWatch, SNS, CloudTrail, and S3.



## Objectives

- Design a secure, production-style AWS infrastructure
- Implement a highly available web application architecture
- Separate public and private workloads using subnet isolation
- Protect the application against unauthorized access
- Implement secure IAM roles instead of hardcoded credentials
- Secure database communication
- Encrypt data at rest using AWS KMS
- Monitor infrastructure health and activity
- Follow the principle of least privilege across all IAM roles and security groups
- Demonstrate cloud security engineering concepts

## AWS Services Used

**Networking**
- Amazon VPC (public, private-web, and private-database subnets)
- Internet Gateway
- NAT Gateway
- Route Tables
- Network ACLs

**Compute**
- Amazon EC2 (Bastion Host and IIS web servers)
- Launch Template
- Auto Scaling Group
- Custom Windows Server AMI

**Security**
- AWS WAF
- IAM Roles
- Security Groups
- AWS Secrets Manager
- AWS KMS

**Database**
- Amazon RDS for MySQL (Multi-AZ)

**Monitoring & Logging**
- Amazon CloudWatch
- Amazon SNS
- AWS CloudTrail
- Amazon S3

| Service | Purpose |
|---|---|
| Amazon VPC | Network isolation across public and private subnets |
| Amazon EC2 | Hosts the Bastion Host and IIS web servers |
| Application Load Balancer (ALB) | Distributes incoming HTTP traffic across web servers |
| AWS WAF | Filters malicious traffic before it reaches the ALB |
| Auto Scaling Group (ASG) | Maintains web server availability and elasticity |
| IAM | Least-privilege roles and permissions for EC2 instances |
| AWS Secrets Manager | Securely stores and serves database credentials |
| Amazon RDS (MySQL) | Multi-AZ relational database for the application |
| AWS KMS | Encrypts RDS storage, Secrets Manager secrets, and SNS messages at rest |
| NAT Gateway | Provides outbound-only internet access for private subnets |
| Amazon CloudWatch | Monitors EC2 CPU utilization and raises alarms |
| Amazon SNS | Sends email notifications for CloudWatch alarms |
| AWS CloudTrail | Logs all API activity across the account |
| Amazon S3 | Stores CloudTrail audit logs |

## Network Design

**VPC:** `10.0.0.0/16`, DNS hostnames enabled, spanning `ap-south-1a` and `ap-south-1c` (Asia Pacific – Mumbai) for resilience against single-AZ failure.

**Subnets:**
- **Public Subnets** — host the Bastion Host, NAT Gateway, and Application Load Balancer
- **Private Web Subnets** — host the IIS web server EC2 instances, reachable only through the ALB and Bastion
- **Private DB Subnet** — hosts the Multi-AZ RDS MySQL instance, reachable only from the web tier

**Route Tables:**
| Route Table | Destination |
|---|---|
| Public | `0.0.0.0/0` → Internet Gateway |
| Private Web | `0.0.0.0/0` → NAT Gateway |
| Private DB | Local route only (no internet path) |

## Security Groups

| Security Group | Inbound Rules |
|---|---|
| sg-bastion | TCP 3389 (RDP) from administrator IP only |
| sg-alb | TCP 80 (HTTP) from `0.0.0.0/0` |
| sg-web | TCP 80 from sg-alb; TCP 3389 (RDP) from sg-bastion |
| sg-rds | TCP 3306 (MySQL) from sg-web |

## IAM Role

EC2 web servers run under a dedicated IAM role (`ec2-web-server-role`) instead of embedded credentials, with two managed policies:
- **CloudWatchAgentServerPolicy** — publish metrics/logs to CloudWatch
- **SecretsManagerReadWrite** — retrieve database credentials at runtime

## Encryption (AWS KMS)

AWS-managed KMS keys are used to encrypt resources at rest:

| Key | Purpose |
|---|---|
| `aws/rds` | Encrypts RDS storage volumes and snapshots |
| `aws/secretsmanager` | Encrypts secrets stored in Secrets Manager |
| `aws/sns` | Encrypts SNS messages when encryption is enabled |
| `aws/lambda` | Available for Lambda encryption if Lambda functions are added |

These AWS-managed keys eliminate the need for manual key administration while maintaining secure encryption across services.

## Traffic Flow

**Application Request Flow**
1. Client sends an HTTP request over the internet
2. Request enters the VPC through the Internet Gateway
3. AWS WAF inspects and filters the request
4. The Application Load Balancer receives the filtered request
5. The request is forwarded to the Target Group
6. An EC2 Web Server processes the request via IIS
7. The web server queries Amazon RDS MySQL over port 3306 if needed
8. The response travels back through the same path to the client

**Administrative Access Flow**
1. Administrator connects via RDP to the Bastion Host using its public IP
2. From the Bastion Host, the administrator RDPs into a private web server

## Security Best Practices Applied

- Network segmentation across public, private-web, and private-database subnets
- Least-privilege IAM roles instead of long-lived access keys on instances
- Database deployed in a private subnet with no public endpoint
- Credentials stored in AWS Secrets Manager rather than hard-coded in application config
- AWS WAF protecting the application from common web exploits
- AWS KMS encryption for data at rest on RDS
- Multi-AZ RDS deployment for automatic failover
- CloudTrail auditing of all API activity
- CloudWatch monitoring with SNS alerting
- Auto Scaling for resilience and elasticity
- Bastion Host as the sole path for administrative access

## Challenges Faced

- Understanding how route tables determine traffic flow between public and private subnets
- Correctly scoping security group rules so each tier only accepts traffic from the tier above it
- Attaching the right IAM permissions to the EC2 role without over-granting access
- Configuring the Launch Template to consistently produce working instances from the custom AMI

## Future Improvements

- [ ] Add HTTPS using an AWS Certificate Manager (ACM) certificate and an HTTPS listener
- [ ] Enable AWS Shield Advanced for enhanced DDoS protection
- [ ] Replace the Bastion Host with AWS Systems Manager Session Manager for keyless administrative access
- [ ] Enable AWS Config for continuous compliance monitoring
- [ ] Enable Amazon GuardDuty for threat detection
- [ ] Enable VPC Flow Logs for deeper network visibility
- [ ] Migrate the manual setup to Infrastructure as Code using Terraform or AWS CloudFormation
- [ ] Build a CI/CD pipeline with GitHub Actions for automated deployment

## Skills Demonstrated

AWS Cloud Architecture, Cloud Security, Network Segmentation, VPC Design, EC2 Administration, Windows Server, IIS Deployment, IAM, Auto Scaling, Application Load Balancer, AWS WAF, Amazon RDS, Secrets Management, Encryption with AWS KMS, Infrastructure Monitoring, Cloud Logging, Security Hardening, High Availability Design, Infrastructure Documentation.

## Conclusion

This project delivered a secure, highly available, three-tier web application on AWS, built around network segmentation, least-privilege access, encryption at rest, and continuous monitoring. Working through VPC design, security group scoping, IAM permissions, Auto Scaling, and credential management provided hands-on experience with the core practices of cloud security engineering, producing an architecture that reflects real-world production patterns rather than a simplified lab exercise.

## Author

*Add your name and links (LinkedIn, portfolio, email) here.*

## License

This project is for educational/portfolio purposes.
