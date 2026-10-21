# Cost Estimation

**Status of numerical estimates: unavailable.** Current AWS prices were not verified when this document was written, so no dollar figures are given. This document lists the cost drivers of the design and explains how to produce a verified estimate. Return to the [README](../README.md) for the overview.

- **AWS Region:** us-east-1
- **Document date:** 2026-10-21
- **Pricing basis:** none; qualitative only

## 1. Cost drivers

| Component | Quantity in the design | How it is charged | Notes |
|---|---|---|---|
| NAT Gateway | 2 (one per AZ) | Hourly charge plus per-GB data processing | Runs continuously; often one of the largest fixed costs in this pattern |
| EC2 | 2 to 6 instances | Per instance-hour by instance type, plus EBS storage | Minimum of 2 runs at all times |
| Application Load Balancer | 1 | Hourly charge plus usage-based capacity units | |
| RDS for MySQL Multi-AZ | 1 primary and 1 standby | Instance-hours, storage, and backup storage beyond the free allowance | Multi-AZ costs more than Single-AZ because a standby is provisioned |
| AWS WAF | 1 web ACL, managed rule groups, 1 rate rule | Per web ACL, per rule, and per request | |
| CloudFront | 1 distribution, price class 100 | Data transfer out and requests | Price class 100 limits delivery to lower-cost edge locations |
| Elastic IPs | 2 (NAT Gateways) | Public IPv4 address charges | |
| CloudWatch | 5 alarms, 1 dashboard | Per alarm, per dashboard, plus metrics and logs | |
| Secrets Manager | 1 secret | Per secret per month plus API calls | |
| SNS | 1 topic | Per request and per notification | |
| Data transfer | | Charged for traffic leaving AWS and between AZs | Depends on actual traffic |
| Route 53 | Optional, not enabled | Hosted zone, queries, health checks | |

## 2. Cost levers

- Use a single NAT Gateway in non-production environments (trading AZ resilience for cost).
- Choose instance and database sizes from measured utilisation.
- Increase cache hit ratio for static content to reduce origin load.
- Use Savings Plans or Reserved Instances for steady baseline capacity.
- Review whether the WAF rule sets and request volume justify their charges.

## 3. How to build a verified estimate

1. Open the [AWS Pricing Calculator](https://calculator.aws/).
2. Select Region US East (N. Virginia).
3. Add each service from the table above with your chosen instance types, storage, and expected traffic.
4. Record the date and the assumptions used (hours per month, GB transferred, request counts).
5. Add the resulting figures to this document with that date.

## 4. Pricing pages

- [Amazon EC2](https://aws.amazon.com/ec2/pricing/)
- [Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/pricing/)
- [Amazon VPC (NAT Gateway)](https://aws.amazon.com/vpc/pricing/)
- [Amazon RDS for MySQL](https://aws.amazon.com/rds/mysql/pricing/)
- [AWS WAF](https://aws.amazon.com/waf/pricing/)
- [Amazon CloudFront](https://aws.amazon.com/cloudfront/pricing/)
- [Amazon CloudWatch](https://aws.amazon.com/cloudwatch/pricing/)
- [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/pricing/)