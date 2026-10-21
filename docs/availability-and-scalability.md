# Availability and Scalability

This document explains how the design behaves under failure and load. Return to the [README](../README.md) for the overview.

## 1. Availability design

| Layer | Mechanism |
|---|---|
| Load balancing | ALB nodes in both public subnets; traffic goes only to healthy targets (health check `/health`) |
| Compute | Auto Scaling group across two private app subnets in two AZs |
| Database | RDS Multi-AZ with a synchronous standby and automatic failover |
| Outbound access | One NAT Gateway per AZ, each used by the private app subnet in that AZ |
| Edge | CloudFront serves cacheable `/static/*` content |

## 2. Failure scenarios

| Event | Expected behaviour | Residual risk |
|---|---|---|
| One web instance fails | The ALB health check marks it unhealthy and stops sending traffic to it. The Auto Scaling group can replace it. | Replacement relies on the group's health check configuration (see assumptions) |
| One AZ fails | The ALB nodes and instances in the other AZ keep serving. RDS fails over to the standby in the other AZ. | Only one instance remains until the group launches replacements. A short database interruption occurs during failover. |
| RDS primary fails | RDS promotes the standby automatically. | Applications must reconnect; connections are interrupted during failover |
| One NAT Gateway or its AZ fails | Instances in the other AZ keep outbound access through their own NAT Gateway. | Instances in the affected AZ lose outbound access |
| Traffic spike | Target tracking adds instances up to the maximum of 6. CloudFront absorbs cacheable static requests. | Scale-out takes time to launch and register instances; requests above capacity in that window may be slow |
| Full Region failure | Not handled | Single-Region design |

## 3. Scalability design

**Auto Scaling group:** minimum 2, desired 2, maximum 6.

| Policy | Type | Setting |
|---|---|---|
| CPU | Target tracking | Average CPU utilisation 50 percent |
| Requests | Target tracking | 500 ALB requests per target |
| Step | Step scaling | Scale-out only |

Target tracking policies also scale in when load drops. When several policies apply, Auto Scaling follows the one that gives the larger capacity for scale-out.

**Deployments:** an instance refresh with a rolling strategy replaces instances when the launch template changes.

**Database storage:** RDS storage autoscaling grows storage from 20 GB up to 100 GB. The database compute tier does not scale automatically, and the standby is not readable, so it adds availability but not read capacity.

## 4. Assumptions

- The Auto Scaling group uses ELB health checks so that instances failing the ALB check are replaced. The diagram does not state the health check type; confirm this setting when implementing.
- The maximum of 6 instances is within the account's EC2 service quotas for the Region.

## References

- [Amazon EC2 Auto Scaling](https://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
- [Target tracking scaling policies](https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html)
- [Application Load Balancer health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)
- [RDS Multi-AZ deployments](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)