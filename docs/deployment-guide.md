# Deployment guide

The order in which this environment was built in the AWS Console (region `ap-south-1`), with checks and teardown. Names match the screenshots, but you can choose your own.

> **Cost warning:** the NAT gateway, ALB and Multi-AZ RDS instance bill by the hour. Use a dedicated test account if you can, and finish with the [teardown](#teardown) section.

## Build order

Build bottom-up so each step has what it depends on.

### 1. Network

1. Create VPC `websec` with CIDR `10.0.0.0/16`.
2. Create six subnets (see the table in the [README](../README.md#network)): two public, two private web, two private database. The two database subnets must be in **different AZs** to form a valid RDS subnet group.
3. Create an Internet Gateway `websec-IGW` and attach it to the VPC.
4. Allocate an Elastic IP and create a NAT gateway `natgatewayforwebsec` in a **public** subnet.
5. Create three route tables and associate them:
   - public: `0.0.0.0/0` → Internet Gateway, associated with both public subnets
   - private web: `0.0.0.0/0` → NAT gateway, associated with both private web subnets
   - database: local route only, associated with **both** database subnets

### 2. Security groups

Create them in this order so each can reference the one before it:

1. `load-balancer-Sg`: inbound TCP 80 from `0.0.0.0/0`.
2. `bastion-SG`: inbound TCP 3389 from your admin IP as a `/32`. Do not add SSH.
3. `webser-SG`: inbound TCP 80 from `load-balancer-Sg`, and TCP 3389 from `bastion-SG`.
4. RDS security group: inbound TCP 3306 from `webser-SG`.

### 3. IAM role

Create an EC2 role with the trust policy for `ec2.amazonaws.com`. Attach `CloudWatchAgentServerPolicy` and the inline `GetSecretValue` policy from the [hardening guide](hardening.md#2-least-privilege-access-to-the-database-secret), not the broad managed Secrets Manager policies. Note the instance profile name.

### 4. Bastion host

Launch a Windows Server 2019 `t3.micro` in a public subnet with a public IPv4 address and `bastion-SG`.

### 5. Custom AMI

1. Launch a temporary Windows Server 2019 instance in a public subnet.
2. Install the IIS role in Server Manager and deploy the application.
3. Create an AMI (`project-web-ami`) from the instance.
4. **Terminate the temporary instance** once the AMI shows *Available*.

### 6. Database

1. Create an RDS DB subnet group from the two database subnets.
2. Create an RDS for MySQL instance: Multi-AZ, **Public access: No**, storage encryption enabled (`aws/rds`), and the RDS security group.
3. Create a Secrets Manager secret (`secret-manager-forwebsec`) holding the database credentials. Prefer letting RDS manage the master password in Secrets Manager.

### 7. Web tier

1. Create launch template `template-for-websec`: custom AMI, `t3.micro`, `webser-SG`, the IAM instance profile.
2. Create a target group `tgforwebsec` (instance targets, HTTP:80, health check on `/`).
3. Create an internet-facing ALB `LBforwebsec` across the two public subnets with `load-balancer-Sg`, listening on HTTP:80 and forwarding to `tgforwebsec`.
4. Create Auto Scaling group `ASG-for-websec1` from the launch template across the two private web subnets, attached to the target group, with ELB health checks. Set min 2, desired 2, max 2 (or higher if you want scale-out).

### 8. WAF

Create a web ACL `waf-for-websec` in `ap-south-1`, add AWS managed rule groups (for example Core rule set, Known bad inputs, SQL database), and associate it with the ALB.

### 9. Monitoring and audit

1. Create SNS topic `snsforwebsec` and subscribe your email. **Confirm the subscription** from the email you receive.
2. Create CloudWatch alarm `cloudwatch-websec`: EC2 `CPUUtilization >= 80%`, action → the SNS topic.
3. Create multi-region trail `websec-trails` with log file validation, delivering to a new S3 bucket.

## Validation checklist

- [ ] Target group shows **2 of 2 healthy** targets.
- [ ] Opening the ALB DNS name in a browser returns the IIS page.
- [ ] Stopping one web instance keeps the site up, and the ASG replaces the instance.
- [ ] RDP to the bastion works from your admin IP and fails from any other IP.
- [ ] From the bastion, RDP to a private web server works. Direct RDP to its private IP from the internet is impossible (it has no public IP).
- [ ] From a web server, the database port is reachable (`Test-NetConnection <rds-endpoint> -Port 3306`). From the bastion it is not, unless you allowed it.
- [ ] A private web server can reach the internet (Windows Update, AWS APIs) through the NAT gateway.
- [ ] The RDS instance shows *Publicly accessible: No* and *Encryption: Enabled*.
- [ ] Forcing CPU above 80% on a web instance sends an SNS email.
- [ ] A test of a WAF rule (for example a request with `<script>alert(1)</script>` in a query string) is blocked with a 403.
- [ ] CloudTrail event history shows your console activity and new log files appear in the S3 bucket.

## Teardown

Delete in reverse dependency order to avoid orphaned, billable resources:

1. Auto Scaling group (set desired capacity to 0, then delete), then the launch template.
2. ALB, then target group, then the WAF web ACL association and web ACL.
3. RDS instance (take a final snapshot only if you need one), then the DB subnet group.
4. Bastion host and any leftover AMI builder instance.
5. NAT gateway, then **release the Elastic IP**.
6. Custom AMI (deregister) and its snapshots.
7. Secrets Manager secret (it has a recovery window unless you force delete), SNS topic, CloudWatch alarm.
8. CloudTrail trail, then empty and delete the S3 bucket.
9. Route tables, subnets, Internet Gateway, security groups, then the VPC.
10. IAM role and instance profile.
