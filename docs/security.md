# Security Design

Security is applied in layers so that no single control is the only barrier. Return to the [README](../README.md) for the overview.

## 1. Security groups

| Security group | Inbound | Source |
|---|---|---|
| `alb-sg` | TCP 80 and 443 | `0.0.0.0/0` |
| `web-sg` | TCP 80 | `alb-sg` only |
| `rds-sg` | TCP 3306 | `web-sg` only |

Only the ALB is open to the internet. The web tier accepts traffic only from the ALB, and the database accepts traffic only from the web tier. Port 22 is not open anywhere.

## 2. Network ACLs

| Subnets | Inbound | Outbound |
|---|---|---|
| Public | 80, 443, ephemeral 1024-65535, VPC CIDR | All |
| Private (app and DB) | VPC CIDR and ephemeral ports (NAT return traffic) | All |

NACLs are stateless and act as a coarse second layer behind the stateful security groups.

## 3. Subnet isolation

- EC2 instances and RDS have no public IP addresses.
- App subnets reach the internet only outbound through a NAT Gateway in the same AZ.
- DB subnets have no route to the internet or a NAT Gateway.

## 4. Edge protection

- CloudFront terminates HTTPS from viewers.
- AWS WAF at the ALB applies the managed Common, Known Bad Inputs, SQL injection, and Linux rule sets, plus a rate limit of 2,000 requests per 5 minutes per IP.
- The ALB returns a fixed 403 for `/admin` and `/admin/*`.

## 5. Instance access and identity

- **Session Manager only:** no SSH, no key pair, no bastion host. Access is governed by IAM and can be logged by Systems Manager.
- **IMDSv2 required** on instances.
- The instance role carries `AmazonSSMManagedInstanceCore` and `CloudWatchAgentServerPolicy`, which are AWS managed policies scoped to their purpose.

## 6. Data protection

- EBS volumes for instances and RDS storage are encrypted.
- The RDS master password is managed by RDS through AWS Secrets Manager rather than stored in code.

## 7. Known gaps

These are disclosed so the documentation matches the design.

| Gap | Impact |
|---|---|
| CloudFront to ALB uses HTTP | Traffic between CloudFront and the ALB is not encrypted in transit |
| ALB reachable directly | `alb-sg` allows `0.0.0.0/0`, so clients can bypass CloudFront. WAF still applies at the ALB |
| Port 443 in `alb-sg` | The baseline origin flow uses HTTP; HTTPS on the ALB would require an ACM certificate |
| No VPC endpoints | Systems Manager and AWS API traffic leaves through the NAT Gateways |
| Default encryption keys | Customer managed KMS keys are not part of the baseline |
| Limited audit logging | VPC Flow Logs, CloudTrail, and ALB/WAF logs are not in the baseline |
| Secrets Manager use | The diagram shows the RDS-managed master password; it does not show an application reading secrets at runtime |

See [limitations-and-future-improvements.md](limitations-and-future-improvements.md) for the remediation roadmap.

## References

- [Security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html)
- [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)