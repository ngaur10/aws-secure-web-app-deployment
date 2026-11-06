# Hardening guide

Fixes for the gaps listed in the [README](../README.md#known-gaps-and-hardening-roadmap), in priority order. Replace `<ACCOUNT_ID>` and other placeholders with your own values.

## 1. Encrypt traffic in transit (HTTPS)

1. Request a public certificate in AWS Certificate Manager (ACM) for your domain, in `ap-south-1`.
2. Add an HTTPS:443 listener on the ALB using that certificate and the `ELBSecurityPolicy-TLS13-1-2-2021-06` policy.
3. Change the existing HTTP:80 listener to redirect to HTTPS (301).
4. Add TCP 443 from `0.0.0.0/0` to the ALB security group.

## 2. Least-privilege access to the database secret

Detach `SecretsManagerReadWrite`, `AWSSecretsManagerClientReadOnlyAccess` and `AWSTransformSecretsManagerConnector…` from the instance role. Keep `CloudWatchAgentServerPolicy`, and add this inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadDatabaseSecretOnly",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-south-1:<ACCOUNT_ID>:secret:secret-manager-forwebsec-*"
    }
  ]
}
```

The trailing `-*` matches the random suffix Secrets Manager appends to the secret's ARN. Because the secret uses the AWS-managed key `aws/secretsmanager`, no extra KMS permission is needed. If you switch to a customer-managed key, also allow `kms:Decrypt` on that key.

Consider renaming the role from `iam-cloudwatch` to something that reflects both jobs, such as `ec2-web-server-role`.

## 3. Lock down administrative access

- On `bastion-SG`, confirm TCP 3389 allows only your current public IP as a `/32`, and remove the SSH (22) rule, which a Windows bastion doesn't need.
- On the web security group, confirm RDP is allowed only from the bastion security group (a security group reference, not a CIDR).
- Better still, replace the bastion with **Systems Manager Session Manager**: attach `AmazonSSMManagedInstanceCore` to the instance role, make sure the instances can reach the SSM endpoints (via the NAT gateway or VPC endpoints), then remove all inbound 3389 rules and terminate the bastion.

## 4. Strengthen CloudTrail

- Enable **SSE-KMS** on the trail with a customer-managed key (the key policy must allow `cloudtrail.amazonaws.com` to use it).
- Add a **CloudWatch Logs** log group and role so events can be searched and alarmed on.
- Create metric filters and alarms for high-signal events: root account use, console sign-in without MFA, IAM policy changes, security group changes and `StopLogging`.
- Turn on SNS notification delivery if you want trail-level notifications.
- Add S3 bucket protections on the log bucket: Block Public Access, versioning, and MFA delete or Object Lock if retention matters.

## 5. Log what the WAF sees

Enable WAF logging for `waf-for-websec` to a CloudWatch Logs group, an S3 bucket or Kinesis Firehose. The destination name must start with `aws-waf-logs-`. Redact the `authorization` and `cookie` headers in the logging configuration.

## 6. Remove single points of failure

- Create a second NAT gateway in a public subnet in ap-south-1a and a separate private route table for the 1a web subnet that points to it. Each private subnet should route to the NAT gateway in its own AZ.
- Set the Auto Scaling minimum to 2 (and desired to 2) if two-AZ redundancy is the goal.
- Associate `websec-rds-2` with the local-only `RDS-mysql` route table so both database subnets behave the same.

## 7. Detection and visibility

- **VPC Flow Logs** on the VPC, delivered to S3 or CloudWatch Logs.
- **GuardDuty** in `ap-south-1` (and ideally all regions).
- **AWS Config** with conformance packs or managed rules such as `restricted-ssh`, `restricted-common-ports`, `s3-bucket-public-read-prohibited`, `rds-storage-encrypted` and `iam-policy-no-statements-with-admin-access`.

## 8. Housekeeping

- Terminate the temporary AMI builder instance once the AMI is created.
- Enable detailed monitoring on the web instances if you want 1-minute CPU metrics for the alarm.
- Tag everything (`Project`, `Owner`, `Environment`) so cost and cleanup are easy.
- Recreate the environment in Terraform or CloudFormation so the whole stack can be reviewed, versioned and torn down in one command.
