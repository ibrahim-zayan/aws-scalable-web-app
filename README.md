# aws-scalable-web-app
# Scalable Web Application on AWS: EC2, ALB and Auto Scaling

A highly available, scalable, and secure three-tier web application architecture on AWS, built from an Application Load Balancer, an EC2 Auto Scaling group in private subnets, and a Multi-AZ Amazon RDS for MySQL database, fronted by Amazon CloudFront and AWS WAF.

> **Scope note:** This repository is an architecture design and documentation deliverable. It documents the design only. It does not contain deployment code, and it makes no claim that the architecture has been deployed or load-tested in a live AWS account.

> **Report:** A full project report is available in [docs/project-report.pdf](docs/project-report.pdf).

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Business Problem and Objectives](#2-business-problem-and-objectives)
3. [Solution Architecture Diagram](#3-solution-architecture-diagram)
4. [Architecture Overview](#4-architecture-overview)
5. [AWS Services and Responsibilities](#5-aws-services-and-responsibilities)
6. [Application Request Flow](#6-application-request-flow)
7. [Network Design and Subnet Placement](#7-network-design-and-subnet-placement)
8. [High Availability and Scalability](#8-high-availability-and-scalability)
9. [Security Design](#9-security-design)
10. [Monitoring and Reliability](#10-monitoring-and-reliability)
11. [Cost Considerations](#11-cost-considerations)
12. [Design Assumptions and Limitations](#12-design-assumptions-and-limitations)
13. [Potential Future Improvements](#13-potential-future-improvements)
14. [Project Directory Structure](#14-project-directory-structure)
15. [Documentation Navigation](#15-documentation-navigation)
16. [References](#16-references)

---

## 1. Project Overview

This project designs a production-style web application platform on AWS. All compute and database resources run in private subnets across **two Availability Zones** in `us-east-1`. Public traffic enters through Amazon CloudFront and an Application Load Balancer (ALB). Web servers run in an EC2 Auto Scaling group, and persistent data is stored in an Amazon RDS for MySQL instance with a synchronous Multi-AZ standby.

Administrators reach instances through AWS Systems Manager Session Manager, so there is no bastion host, no SSH port, and no key pair.

## 2. Business Problem and Objectives

**Problem.** A single-server web application is a single point of failure. It cannot absorb traffic spikes, it exposes the server and database to the internet, and it gives operators little visibility.

**Objectives.**

| Objective | How the design addresses it |
|---|---|
| Stay available if an Availability Zone fails | Resources in two AZs, ALB across both, RDS Multi-AZ, one NAT Gateway per AZ |
| Scale with demand | Auto Scaling group (min 2, desired 2, max 6) with target tracking policies |
| Reduce latency for static content | CloudFront caches `/static/*` |
| Protect the application | AWS WAF, layered security groups and NACLs, private subnets, no public IPs on instances or database |
| Remove bastion hosts | Session Manager for instance access |
| Detect problems early | Five CloudWatch alarms notifying an SNS email topic, plus a dashboard |

## 3. Solution Architecture Diagram

![AWS Scalable Web App Architecture](architecture/architecture.png)

*Diagram colour legend: dark lines = request and database flow; pink = management, alerts, monitoring; purple = NAT egress and edge; red = security (WAF, credentials); dashed = optional, alternative, or replication path.*

## 4. Architecture Overview

The design has five layers:

1. **Edge:** CloudFront (HTTPS to viewers, caches `/static/*`) and an optional Route 53 alias record.
2. **Entry:** An internet-facing ALB spanning two public subnets, protected by an AWS WAF web ACL.
3. **Application:** EC2 web servers running nginx, managed by an Auto Scaling group across two private app subnets.
4. **Data:** RDS for MySQL 8.0 with a primary in one private DB subnet and a standby in the other. The DB subnets have no internet route.
5. **Operations:** Systems Manager, CloudWatch, SNS, IAM, and Secrets Manager.

The VPC uses CIDR `10.0.0.0/16` with DNS support and DNS hostnames enabled.

## 5. AWS Services and Responsibilities

| Service | Responsibility in this design |
|---|---|
| **Amazon VPC** | Isolated network with public, private app, and private DB subnets in two AZs; route tables, security groups, and NACLs |
| **Internet Gateway** | Internet entry and exit for the public subnets |
| **NAT Gateway (one per AZ)** | Outbound-only internet access for private app instances (for example OS updates and Systems Manager connectivity) |
| **Amazon CloudFront** | HTTPS termination for viewers, caching of `/static/*`, price class 100; uses the ALB as an HTTP origin |
| **AWS WAF (WAFv2)** | Web ACL associated with the ALB: AWS managed Common, Known Bad Inputs, SQL injection, and Linux rule sets, plus a rate limit of 2,000 requests per 5 minutes per IP |
| **Application Load Balancer** | Layer 7 load balancing across both AZs; health check on `/health`; listener rule returning a fixed 403 for `/admin` and `/admin/*` |
| **Amazon EC2** | nginx web servers on Amazon Linux 2023, IMDSv2 enforced, encrypted gp3 volumes, no public IP |
| **EC2 Auto Scaling** | Launch template, min 2 / desired 2 / max 6, target tracking policies, scale-out step policy, rolling instance refresh on template change |
| **Amazon RDS for MySQL 8.0** | Multi-AZ database with automatic failover, encrypted gp3 storage (20 GB, autoscaling up to 100 GB) |
| **AWS Secrets Manager** | Stores the RDS master password (managed by RDS) |
| **AWS Systems Manager** | Session Manager for shell access without SSH |
| **AWS IAM** | Instance role and profile with `AmazonSSMManagedInstanceCore` and `CloudWatchAgentServerPolicy` |
| **Amazon CloudWatch** | Five alarms and one dashboard with four widgets |
| **Amazon SNS** | Email subscription that receives alarm notifications |
| **Amazon Route 53 (optional)** | Alias A record to the ALB with a health check; shown as optional and not enabled in the baseline |

## 6. Application Request Flow

1. The user resolves the CloudFront URL. (Route 53 is optional and not enabled in the baseline.)
2. The user connects to **CloudFront** over HTTPS. Requests for `/static/*` can be served from the cache.
3. For cache misses and dynamic paths, CloudFront forwards the request over **HTTP** to the **ALB** through the Internet Gateway.
4. **AWS WAF** inspects the request at the ALB. Requests to `/admin` and `/admin/*` receive a fixed 403 response from an ALB listener rule.
5. The ALB forwards allowed requests on port 80 to **EC2 instances** in the private app subnets.
6. The application on EC2 connects to **RDS MySQL** on port 3306 in the private DB subnets.

Outbound traffic from the instances (for example package updates) leaves through the NAT Gateway in the same AZ.

## 7. Network Design and Subnet Placement

**VPC:** `10.0.0.0/16`, region `us-east-1`.

| Tier | AZ-a | AZ-b | Contents | Default route |
|---|---|---|---|---|
| Public | `10.0.0.0/24` | `10.0.1.0/24` | ALB nodes, NAT Gateways (with Elastic IPs) | Internet Gateway |
| Private app | `10.0.10.0/24` | `10.0.11.0/24` | EC2 web servers (Auto Scaling group) | NAT Gateway in the same AZ |
| Private DB | `10.0.20.0/24` | `10.0.21.0/24` | RDS primary and standby | None (local VPC traffic only) |

**Network ACLs**

- Public subnets: inbound 80, 443, ephemeral ports 1024-65535, and VPC CIDR traffic; all outbound allowed.
- Private subnets (app and DB): inbound from the VPC CIDR plus ephemeral ports (for NAT return traffic); all outbound allowed.

## 8. High Availability and Scalability

**High availability**

- The ALB runs nodes in both public subnets and routes only to healthy targets.
- The Auto Scaling group spreads instances over two private app subnets in different AZs.
- RDS Multi-AZ keeps a synchronous standby in the second AZ and fails over automatically. The standby is not readable.
- Each AZ has its own NAT Gateway, so a NAT or AZ problem does not remove outbound access for the other AZ.

**Scalability**

- Target tracking: average CPU at 50 percent, and 500 ALB requests per target.
- A step scaling policy adds capacity (scale-out only).
- RDS storage autoscaling grows from 20 GB up to 100 GB.
- CloudFront offloads cacheable static requests from the origin.

**Capacity caveat:** with a minimum of two instances, losing one AZ leaves one instance serving traffic until the Auto Scaling group launches replacements.

Details: [docs/availability-and-scalability.md](docs/availability-and-scalability.md).

## 9. Security Design

| Layer | Controls |
|---|---|
| Edge | CloudFront with HTTPS to viewers; WAF managed rule sets and rate limiting at the ALB |
| Network | Three-tier subnets; no public IPs on instances or database; DB subnets with no internet route; NACLs as a second layer |
| Security groups | `alb-sg` (80/443 from `0.0.0.0/0`) to `web-sg` (80 from `alb-sg` only) to `rds-sg` (3306 from `web-sg` only) |
| Access | Session Manager only: no port 22, no key pair, no bastion |
| Instance | IMDSv2 enforced; encrypted gp3 volumes; least-privilege managed policies on the instance role |
| Data | RDS storage encrypted; master password managed by Secrets Manager |
| Application | `/admin` and `/admin/*` blocked at the ALB with a fixed 403 |

Details and known gaps: [docs/security.md](docs/security.md).

## 10. Monitoring and Reliability

Five CloudWatch alarms send notifications to an SNS topic with an email subscription:

1. Auto Scaling group CPU
2. ALB HTTP 5xx errors
3. Unhealthy host count
4. RDS CPU
5. RDS free storage

A CloudWatch dashboard with four widgets gives a single view of the stack. The instance role includes `CloudWatchAgentServerPolicy` so the CloudWatch agent can publish additional metrics and logs. The ALB health check path is `/health`.

## 11. Cost Considerations

No dollar figures are published here because current prices were not verified for this document. The main cost drivers are:

- **NAT Gateways** (two): hourly charge plus data processing, running continuously.
- **EC2 instances:** at least two running at all times.
- **RDS Multi-AZ:** a provisioned standby increases cost compared with a Single-AZ instance.
- **ALB and AWS WAF:** hourly or per-resource charges plus usage-based charges.
- **CloudFront:** usage-based; price class 100 limits delivery to lower-cost edge locations.
- **Data transfer, CloudWatch, and Secrets Manager:** usage-based.

See [docs/cost-estimation.md](docs/cost-estimation.md) for the method to produce a verified estimate with the AWS Pricing Calculator.

## 12. Design Assumptions and Limitations

- **Region:** single Region (`us-east-1`); there is no cross-Region disaster recovery.
- **ALB is directly reachable:** the ALB security group allows `0.0.0.0/0`, so users can bypass CloudFront. WAF still applies at the ALB.
- **No end-to-end HTTPS:** CloudFront connects to the ALB over HTTP. There is no ACM certificate or custom domain in the baseline.
- **Route 53 is optional** and not enabled in the baseline. When enabled, the alias record points at the ALB, not CloudFront.
- **No VPC endpoints:** Systems Manager and package traffic uses the NAT Gateways.
- **Scaling policy:** the step policy handles scale-out only; scale-in relies on target tracking.
- **Secrets Manager:** the diagram shows the RDS-managed master password, not an application reading it at runtime.
- **Alarms:** five alarms are defined; coverage of other failure modes (for example NAT Gateway) is not included.
- **Validation:** the design has not been load-tested or verified in a live AWS account for this submission.
- **Diagram text:** the diagram's subtitle and roadmap box mention infrastructure-as-code tooling from the design's origin. Deployment code is out of scope for this repository.

Full list: [docs/limitations-and-future-improvements.md](docs/limitations-and-future-improvements.md).

## 13. Potential Future Improvements

1. Restrict the ALB to CloudFront only (CloudFront managed prefix list plus a secret origin header, default action 403).
2. HTTPS end to end with ACM certificates and a Route 53 custom domain.
3. VPC endpoints, KMS customer managed keys, VPC Flow Logs, CloudTrail, and ALB/WAF access logs.
4. RDS deletion protection, final snapshot, and a 7-day backup retention period.
5. Application that reads credentials from Secrets Manager at runtime, with the CloudWatch agent and a pre-baked AMI.
6. Separate dev and prod environments.
7. Cross-Region disaster recovery.

## 14. Project Directory Structure

```text
aws-scalable-web-app/
├── README.md
├── architecture/
│   └── architecture.png
└── docs/
    ├── architecture.md
    ├── security.md
    ├── availability-and-scalability.md
    ├── cost-estimation.md
    ├── limitations-and-future-improvements.md
    └── project-report.pdf

```

## 15. Documentation Navigation

| Document | Contents |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Component-by-component configuration reference |
| [docs/security.md](docs/security.md) | Security controls by layer, traffic rules, and known gaps |
| [docs/availability-and-scalability.md](docs/availability-and-scalability.md) | Failure scenarios and scaling behaviour |
| [docs/cost-estimation.md](docs/cost-estimation.md) | Cost drivers and how to build a verified estimate |
| [docs/limitations-and-future-improvements.md](docs/limitations-and-future-improvements.md) | Limitations and prioritised roadmap |
| [docs/project-report.pdf](docs/project-report.pdf) | Full project report (PDF) |

## 16. References

Official AWS documentation:

- [Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [Network ACLs](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
- [NAT gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html)
- [Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- [Target tracking scaling policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- [Amazon RDS Multi-AZ deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html)
- [AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/what-is-aws-waf.html)
- [Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [Amazon CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)
- [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)

Repository structure and presentation were inspired by the official AWS Solutions repository [Dynamic Image Transformation for Amazon CloudFront](https://github.com/aws-solutions/dynamic-image-transformation-for-amazon-cloudfront). No content was copied from it.