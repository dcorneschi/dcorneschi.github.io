# The 7 R's of AWS Cloud Migration Strategies

When migrating applications to the cloud, AWS defines seven common strategies known as the 7 R's. Each strategy represents a different approach depending on your business needs, timeline, budget, and technical constraints.

## Strategy Overview

| # | Strategy | Also Known As | Effort | Speed |
|---|----------|--------------|:------:|:-----:|
| 1 | Retire | Decommission | None | Immediate |
| 2 | Retain | Revisit | None | N/A |
| 3 | Rehost | Lift and Shift | Low | Fast |
| 4 | Relocate | Hypervisor-Level Lift and Shift | Low | Fast |
| 5 | Repurchase | Drop and Shop | Medium | Medium |
| 6 | Replatform | Lift, Tinker, and Shift | Medium | Medium |
| 7 | Refactor | Re-architect | High | Slow |

## 1. Retire

Decommission applications that are no longer needed.

- Identify IT assets that are no longer useful and can be turned off
- Reduces costs by eliminating unnecessary infrastructure
- Common during portfolio discovery — many organizations find 10-20% of their portfolio is no longer needed

### When to Use

- Application has no active users
- Functionality has been replaced by another system
- Cost of maintaining exceeds the business value

## 2. Retain (Revisit)

Keep applications in the source environment for now.

- Not every application should be migrated immediately
- Some apps may require major refactoring that isn't justified yet
- Plan to revisit these applications in the future

### When to Use

- Application requires significant investment to migrate and has low priority
- Recent upgrades were made on-premises
- Regulatory or compliance constraints prevent moving to the cloud at this time
- Unresolved dependencies with other on-premises systems

## 3. Rehost (Lift and Shift)

Move applications to the cloud without making changes.

- Migrate servers and applications as-is to AWS (e.g., EC2 instances)
- Fastest migration path with minimal risk
- Can use automated tools like AWS Application Migration Service (MGN)
- Optimization can happen after migration

### When to Use

- Need to migrate quickly (data center lease expiring, etc.)
- Application works well and doesn't need architectural changes
- Want to realize cloud benefits (cost, scalability) without development effort
- Large-scale migrations where speed is critical

### AWS Services

- AWS Application Migration Service (MGN)
- AWS Server Migration Service (SMS)
- Amazon EC2

## 4. Relocate (Hypervisor-Level Lift and Shift)

Move infrastructure to the cloud without purchasing new hardware or rewriting applications.

- Migrate VMware-based workloads to VMware Cloud on AWS
- No changes to the application, OS, or networking
- Maintains existing operational tooling and processes

### When to Use

- Running VMware workloads on-premises
- Want to quickly scale capacity using AWS infrastructure
- Need to maintain VMware operational consistency
- Data center evacuation scenarios

### AWS Services

- VMware Cloud on AWS

## 5. Repurchase (Drop and Shop)

Replace the existing application with a different product, typically a SaaS solution.

- Move from a self-managed application to a cloud-native SaaS offering
- Involves changing licensing models (e.g., from perpetual to subscription)
- May require data migration and user retraining

### When to Use

- Current application is end-of-life or heavily customized legacy software
- A SaaS product provides equivalent or better functionality
- Want to reduce operational overhead of managing the application
- Moving from on-premises CRM to Salesforce, or from self-hosted email to Microsoft 365

### Examples

| From | To |
|------|-----|
| On-premises CRM | Salesforce |
| Self-hosted HR system | Workday |
| Custom CMS | Managed SaaS CMS |
| On-premises database | Amazon RDS or Aurora |

## 6. Replatform (Lift, Tinker, and Shift)

Make a few cloud optimizations without changing the core architecture.

- Migrate with minor adjustments to take advantage of cloud capabilities
- No major code changes — just targeted optimizations
- Balance between speed of migration and cloud-native benefits

### When to Use

- Want some cloud optimization without a full re-architecture
- Application can benefit from managed services with minimal changes
- Example: switching from a self-managed database to Amazon RDS

### Common Optimizations

- Move database to Amazon RDS or Aurora
- Replace local storage with Amazon S3
- Use Elastic Load Balancing instead of self-managed load balancers
- Migrate to managed containers (ECS/EKS) without rewriting the app

### AWS Services

- Amazon RDS / Aurora
- Amazon ElastiCache
- Amazon ECS / EKS
- Elastic Beanstalk

## 7. Refactor / Re-architect

Redesign the application using cloud-native features.

- Most expensive and time-consuming strategy
- Provides the greatest long-term benefits (scalability, performance, agility)
- Typically involves moving to microservices, serverless, or event-driven architectures

### When to Use

- Application needs features that are difficult to achieve in its current architecture
- Strong business need for scalability, performance, or agility
- Want to adopt DevOps practices and CI/CD pipelines
- Current architecture limits innovation

### Common Patterns

| From | To |
|------|-----|
| Monolith | Microservices |
| Self-managed servers | Serverless (AWS Lambda) |
| Relational database | Purpose-built databases (DynamoDB, Neptune) |
| Tightly coupled systems | Event-driven architecture (EventBridge, SNS, SQS) |

### AWS Services

- AWS Lambda
- Amazon API Gateway
- Amazon DynamoDB
- Amazon SQS / SNS / EventBridge
- AWS Step Functions
- Amazon ECS / EKS with Fargate

## Choosing the Right Strategy

| Factor | Recommended Strategy |
|--------|---------------------|
| Speed is critical | Rehost or Relocate |
| Reduce operational burden | Repurchase or Replatform |
| Maximize cloud benefits | Refactor |
| Application not needed | Retire |
| Not ready to migrate | Retain |
| Running VMware workloads | Relocate |
| Minor optimizations wanted | Replatform |

## Migration Decision Flow

1. Is the application still needed? → No → **Retire**
2. Is it ready to migrate now? → No → **Retain**
3. Can it be replaced by SaaS? → Yes → **Repurchase**
4. Does it need architectural changes? → Yes → **Refactor**
5. Can it benefit from managed services with small changes? → Yes → **Replatform**
6. Is it a VMware workload? → Yes → **Relocate**
7. Otherwise → **Rehost**
