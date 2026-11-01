---
tags:
  - AWS
  - Troubleshooting
---

Knowing these problems helps you isolate AWS identity, networking, compute, storage, and permissions failures.

---

## Main idea

Most AWS problems involve credentials, IAM permissions, VPC routing/security, service limits, or region/resource references.

Replace placeholders before running commands. Some commands require additional arguments or permissions.

For each problem below:
- **Why it happens** = root cause
- **Quick sign** = how to recognize it fast
- **Solution** = how to fix it
- **Useful commands** = where to start

---

## 1. Unable to locate credentials

- **Why it happens:** No credentials are configured or the selected profile is invalid.
- **Quick sign:** AWS CLI reports that it cannot locate credentials.
- **Solution:** Verify the active profile and configured credential source; use short-lived roles where possible and never commit keys.
- **Useful commands:** `aws configure list`, `aws sts get-caller-identity`, `aws configure list-profiles`

---

## 2. AccessDenied error

- **Why it happens:** The identity lacks an allowed IAM action, resource policy denies access, or a permission boundary restricts it.
- **Quick sign:** API calls return AccessDenied.
- **Solution:** Identify the exact action and resource, inspect identity and resource policies, then grant only the required permission.
- **Useful commands:** `aws sts get-caller-identity`, `aws iam simulate-principal-policy --policy-source-arn <principal-arn> --action-names <action> --resource-arns <resource-arn>`, CloudTrail event history

---

## 3. Expired token or credentials

- **Why it happens:** Temporary credentials expired or the clock is significantly wrong.
- **Quick sign:** Requests report expired token.
- **Solution:** Refresh credentials through the approved identity mechanism and verify system time synchronization.
- **Useful commands:** `aws sts get-caller-identity`, `timedatectl`, `chronyc sources`

---

## 4. Wrong AWS region

- **Why it happens:** The CLI, SDK, or console is pointed to a different region than the resources.
- **Quick sign:** Resources appear missing even though the account is correct.
- **Solution:** Confirm the intended region and list resources explicitly with that region.
- **Useful commands:** `aws configure get region`, `aws ec2 describe-instances --region <region>`, `aws s3 ls --region <region>`

---

## 5. EC2 instance cannot reach the internet

- **Why it happens:** The subnet lacks an internet gateway route, the instance has no public IP, or security/NACL rules block traffic.
- **Quick sign:** Package downloads and external API calls fail while internal VPC access may work.
- **Solution:** Trace route tables, internet/NAT gateway configuration, security groups, and NACLs in order.
- **Useful commands:** `aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=<subnet-id>`, `aws ec2 describe-instances --instance-ids <instance-id>`, `aws ec2 describe-security-groups --group-ids <sg-id>`

---

## 6. Security group blocks a required port

- **Why it happens:** Inbound rules do not allow the application port or expected source.
- **Quick sign:** Connections time out despite the service running.
- **Solution:** Verify listener, source IP, protocol, and security-group rules; use the narrowest practical source range.
- **Useful commands:** `aws ec2 describe-security-groups --group-ids <sg-id>`, `ss -lntp`, `curl -v <endpoint>`

---

## 7. Security group or NACL misconfiguration

- **Why it happens:** Security groups are stateful, while NACLs are stateless and require explicit return-traffic rules.
- **Quick sign:** One direction works while return traffic fails, especially with NACLs.
- **Solution:** Check both inbound and outbound rules and distinguish security-group stateful behavior from NACL rules.
- **Useful commands:** `aws ec2 describe-security-groups --group-ids <sg-id>`, `aws ec2 describe-network-acls --network-acl-ids <nacl-id>`

---

## 8. VPC peering or transit gateway route missing

- **Why it happens:** A route, attachment, or return route is absent.
- **Quick sign:** Private IP connectivity fails between VPCs.
- **Solution:** Verify attachments, route tables, CIDR overlap, and security controls on both sides.
- **Useful commands:** `aws ec2 describe-route-tables`, `aws ec2 describe-vpc-peering-connections`, `aws ec2 describe-transit-gateways`

---

## 9. DNS resolution fails in a VPC

- **Why it happens:** VPC DNS support, private hosted zones, resolver endpoints, or DHCP options are misconfigured.
- **Quick sign:** Private names fail while IP-based access works.
- **Solution:** Verify VPC DNS attributes, hosted zones, resolver endpoints, and security-group access for DNS traffic.
- **Useful commands:** `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`, `aws route53 list-hosted-zones`, `dig <name>`

---

## 10. S3 access denied

- **Why it happens:** IAM policy, bucket policy, KMS key policy, or Block Public Access settings conflict.
- **Quick sign:** GetObject or PutObject returns AccessDenied.
- **Solution:** Identify the exact operation and principal, then review identity policy, bucket policy, and encryption settings.
- **Useful commands:** `aws s3api get-object --bucket <bucket> --key <key> <output-file>`, `aws s3api get-bucket-policy --bucket <bucket>`, `aws s3api get-bucket-encryption --bucket <bucket>`

---

## 11. S3 upload fails or is slow

- **Why it happens:** Object size, multipart configuration, network throughput, permissions, or encryption requirements cause failure.
- **Quick sign:** Large uploads fail or remain incomplete.
- **Solution:** Verify object size and permissions, use multipart upload where appropriate, and inspect incomplete multipart uploads.
- **Useful commands:** `aws s3 cp <file> s3://<bucket>/<key>`, `aws s3api list-multipart-uploads --bucket <bucket>`

---

## 12. EBS volume will not attach

- **Why it happens:** AZ mismatch, attachment limits, instance state, or an existing attachment prevents mounting.
- **Quick sign:** Attach-volume fails or the device does not appear inside the OS.
- **Solution:** Confirm AZ, instance state, attachment status, and device naming; never assume a device name without checking the OS.
- **Useful commands:** `aws ec2 describe-volumes --volume-ids <volume-id>`, `aws ec2 describe-instances --instance-ids <instance-id>`, `lsblk`

---

## 13. RDS connection fails

- **Why it happens:** Security groups, VPC/subnet groups, endpoint/DNS, credentials, or database engine settings are wrong.
- **Quick sign:** The application cannot connect even though the instance is available.
- **Solution:** Test from the intended network location, verify security-group ingress, endpoint resolution, and database authentication.
- **Useful commands:** `aws rds describe-db-instances --db-instance-identifier <identifier>`, `nslookup <rds-endpoint>`, database client connection test

---

## 14. Lambda times out

- **Why it happens:** The function exceeds its configured timeout, waits on a downstream service, or has insufficient memory/CPU allocation.
- **Quick sign:** Logs show Task timed out after the configured duration.
- **Solution:** Inspect duration and memory metrics, identify the slow dependency, and adjust timeout or resources based on measured behavior.
- **Useful commands:** `aws lambda get-function-configuration --function-name <name>`, CloudWatch Logs, `aws lambda invoke --function-name <name> <output-file>`

---

## 15. Lambda cannot reach a VPC resource

- **Why it happens:** The function is attached to private subnets without a NAT route, or security-group rules block the destination.
- **Quick sign:** Internet or VPC-resource calls time out.
- **Solution:** Verify subnet routing, NAT/internet gateway design, security groups, and DNS configuration. Separate private-destination failures from internet-destination failures.
- **Useful commands:** `aws lambda get-function-configuration --function-name <name>`, `aws ec2 describe-route-tables`, `aws ec2 describe-security-groups --group-ids <sg-id>`

---

## 16. CloudFormation stack fails

- **Why it happens:** Invalid template syntax, IAM permissions, dependency ordering, resource limits, or an existing resource conflict.
- **Quick sign:** Stack events show CREATE_FAILED or UPDATE_FAILED.
- **Solution:** Read the first failing event, correct the specific resource or permission, then update or roll back deliberately.
- **Useful commands:** `aws cloudformation describe-stack-events --stack-name <stack>`, `aws cloudformation validate-template --template-body file://template.yaml`

---

## 17. Elastic IP or public IP is missing

- **Why it happens:** The instance uses a private subnet, the Elastic IP was released, or the network interface configuration changed.
- **Quick sign:** External systems cannot reach the instance.
- **Solution:** Verify subnet routing, public-IP settings, and Elastic IP association before changing network configuration.
- **Useful commands:** `aws ec2 describe-instances --instance-ids <instance-id>`, `aws ec2 describe-addresses`, `aws ec2 describe-subnets --subnet-ids <subnet-id>`

---

## 18. Service limit or quota exceeded

- **Why it happens:** Account or regional quotas restrict instances, IPs, volumes, Lambdas, or API rates.
- **Quick sign:** API errors mention limit exceeded, throttling, or quota.
- **Solution:** Identify the specific quota, reduce waste, and request an increase only where justified.
- **Useful commands:** `aws service-quotas list-service-quotas --service-code <service-code>`, `aws ec2 describe-account-attributes`

---

## 19. API throttling

- **Why it happens:** Too many requests, retry storms, or bursty automation exceed service rate limits.
- **Quick sign:** Requests return Throttling or TooManyRequestsException.
- **Solution:** Add exponential backoff and jitter, reduce request volume, and batch operations where supported.
- **Useful commands:** `aws sts get-caller-identity`, CloudWatch metrics, application retry logs

---

## 20. Unexpected AWS charges

- **Why it happens:** Unused resources, data transfer, snapshots, NAT processing, or forgotten test environments continue running.
- **Quick sign:** Billing rises without an obvious workload change.
- **Solution:** Use Cost Explorer and tagged resource reports to identify the driver, then stop, delete, or right-size only verified resources.
- **Useful commands:** AWS Billing console, Cost Explorer, `aws resourcegroupstaggingapi get-resources`

---

## Common mistakes

- granting broad admin permissions to fix AccessDenied
- committing long-lived access keys
- opening security groups to 0.0.0.0/0 unnecessarily
- assuming a resource exists in every region
- deleting resources without checking backups or dependencies

---

## Good practices

- use IAM roles and short-lived credentials where possible
- apply least privilege and test with policy simulation
- tag resources for ownership, environment, and cost tracking
- verify region before diagnosing missing resources
- use CloudTrail and CloudWatch for permission and performance evidence

---

## Must memorize

```bash
aws sts get-caller-identity
aws configure list
aws ec2 describe-instances --region <region>
aws ec2 describe-security-groups --group-ids <sg-id>
aws ec2 describe-route-tables
aws s3 ls
aws cloudformation describe-stack-events --stack-name <stack>
```

---

## Key ideas

- Identity, network, and service configuration can each cause the same symptom.
- Security groups are stateful; NACLs are stateless.
- Region and account context must be verified first.
- Least privilege is safer and easier to troubleshoot than broad access.
