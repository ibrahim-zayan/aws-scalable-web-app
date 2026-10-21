# Limitations and Future Improvements

This document consolidates what the baseline design does not cover and a prioritised roadmap. Return to the [README](../README.md) for the overview.

## 1. Limitations

| # | Limitation | Effect |
|---|---|---|
| 1 | Users can reach the ALB directly | CloudFront can be bypassed; WAF still applies at the ALB |
| 2 | CloudFront to ALB uses HTTP | No encryption in transit on that hop |
| 3 | No ACM certificate or custom domain | Users access the CloudFront default URL |
| 4 | Route 53 is optional and not enabled | When enabled, the alias targets the ALB rather than CloudFront |
| 5 | No VPC endpoints | AWS service traffic uses the NAT Gateways |
| 6 | Single Region | No cross-Region disaster recovery |
| 7 | Step scaling is scale-out only | Scale-in relies on target tracking |
| 8 | Minimum capacity of 2 instances | Losing an AZ leaves one instance until scale-out completes |
| 9 | RDS standby is not readable | Provides availability, not read scaling |
| 10 | RDS backup retention, deletion protection, and final snapshot are not in the baseline | Weaker data protection than a production target |
| 11 | Application does not read Secrets Manager at runtime | The diagram shows only the RDS-managed master password |
| 12 | Limited audit logging | No VPC Flow Logs, CloudTrail, or ALB/WAF logs in the baseline |
| 13 | Five alarms only | Other failure modes, such as NAT Gateway issues, are not alarmed |
| 14 | Not validated in a live account | No load test or deployment evidence is claimed |
| 15 | Diagram wording | The diagram subtitle and roadmap box mention infrastructure-as-code tooling from the design's origin; deployment code is outside this repository |

## 2. Roadmap

Ordered by security and reliability value.

| Priority | Improvement |
|---|---|
| 1 | Restrict the ALB to CloudFront: allow the CloudFront managed prefix list in `alb-sg`, require a secret origin header, and set the listener default action to 403 |
| 2 | HTTPS end to end: ACM certificates on CloudFront and the ALB, plus a Route 53 custom domain |
| 3 | RDS protection: deletion protection, final snapshot, and 7-day backup retention |
| 4 | Observability and audit: VPC Flow Logs, CloudTrail, ALB access logs, WAF logs |
| 5 | Network hardening: VPC endpoints for Systems Manager and other AWS services; KMS customer managed keys |
| 6 | Real application integration: read database credentials from Secrets Manager, run the CloudWatch agent, and use a pre-baked AMI |
| 7 | Environment separation: distinct dev and prod environments |
| 8 | Resilience: cross-Region disaster recovery and read replicas where read scaling is needed |
| 9 | Cost control: single NAT Gateway in non-production, Savings Plans for baseline capacity |