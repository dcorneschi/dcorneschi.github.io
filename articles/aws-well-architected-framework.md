# AWS Well-Architected Framework

The AWS Well-Architected Framework helps cloud architects build secure, high-performing, resilient, and efficient infrastructure. It provides a consistent approach for evaluating architectures and implementing designs that scale over time.

The framework is organized around six pillars, each representing a key area of architectural excellence.

## Pillar Summary

| Pillar | Key Question | Goal |
|--------|-------------|------|
| Operational Excellence | How do you manage and automate changes? | Automate operations, respond to events, learn and improve |
| Security | How do you protect your data and systems? | Protect information and systems with least privilege |
| Reliability | How do you recover from failures? | Recover from disruptions, meet demand dynamically |
| Performance Efficiency | How do you select the right resources? | Use resources efficiently, adopt new technologies |
| Cost Optimization | How do you manage spending? | Deliver value at the lowest cost |
| Sustainability | How do you reduce environmental impact? | Minimize resource waste and carbon footprint |

## 1. Operational Excellence

Run and monitor systems to deliver business value and continually improve processes and procedures.

### Design Principles

- Perform operations as code (Infrastructure as Code)
- Make frequent, small, reversible changes
- Refine operations procedures frequently
- Anticipate failure
- Learn from all operational failures

### Key Focus Areas

- **Organization** — Understand business priorities and organize teams accordingly
- **Prepare** — Design workloads for observability, implement deployment pipelines
- **Operate** — Monitor health, respond to events, manage routine operations
- **Evolve** — Learn from experience, share learnings, improve iteratively

### AWS Services

| Area | Services |
|------|----------|
| IaC | AWS CloudFormation, AWS CDK, Terraform |
| Monitoring | Amazon CloudWatch, AWS X-Ray, CloudTrail |
| Automation | AWS Systems Manager, AWS Config |
| Deployment | AWS CodePipeline, CodeDeploy, CodeBuild |

## 2. Security

Protect information, systems, and assets while delivering business value through risk assessments and mitigation strategies.

### Design Principles

- Implement a strong identity foundation (least privilege)
- Maintain traceability
- Apply security at all layers
- Automate security best practices
- Protect data in transit and at rest
- Keep people away from data
- Prepare for security events

### Key Focus Areas

- **Identity and Access Management** — Control who can do what
- **Detection** — Identify security misconfigurations and anomalies
- **Infrastructure Protection** — Protect network and compute resources
- **Data Protection** — Encrypt and classify data
- **Incident Response** — Respond to and recover from security events

### AWS Services

| Area | Services |
|------|----------|
| Identity | IAM, AWS Organizations, AWS SSO (Identity Center) |
| Detection | AWS CloudTrail, Amazon GuardDuty, AWS Security Hub |
| Infrastructure | AWS WAF, AWS Shield, VPC, Security Groups |
| Data Protection | AWS KMS, AWS Certificate Manager, Amazon Macie |
| Incident Response | Amazon Detective, AWS Config Rules |

## 3. Reliability

Ensure a workload performs its intended function correctly and consistently when expected.

### Design Principles

- Automatically recover from failure
- Test recovery procedures
- Scale horizontally to increase aggregate availability
- Stop guessing capacity
- Manage change in automation

### Key Focus Areas

- **Foundations** — Service quotas, network topology
- **Workload Architecture** — Distributed systems design, service-oriented architecture
- **Change Management** — Monitor resources, adapt to demand changes
- **Failure Management** — Withstand and recover from component failures

### AWS Services

| Area | Services |
|------|----------|
| Foundations | AWS Service Quotas, VPC, AWS Trusted Advisor |
| Scaling | Auto Scaling, Elastic Load Balancing |
| Backup | AWS Backup, S3 Cross-Region Replication |
| Recovery | Route 53 (DNS failover), Multi-AZ, Multi-Region |
| Monitoring | CloudWatch, AWS Health Dashboard |

## 4. Performance Efficiency

Use computing resources efficiently to meet system requirements and maintain efficiency as demand changes and technologies evolve.

### Design Principles

- Democratize advanced technologies (use managed services)
- Go global in minutes
- Use serverless architectures
- Experiment more often
- Consider mechanical sympathy (use the right tool for the job)

### Key Focus Areas

- **Selection** — Choose the right resource types (compute, storage, database, network)
- **Review** — Continually evaluate new services and features
- **Monitoring** — Monitor performance and remediate issues
- **Trade-offs** — Use caching, partitioning, compression where appropriate

### AWS Services

| Area | Services |
|------|----------|
| Compute | EC2 (instance types), Lambda, Fargate, Graviton |
| Storage | S3, EBS (gp3, io2), EFS, FSx |
| Database | RDS, Aurora, DynamoDB, ElastiCache, MemoryDB |
| Network | CloudFront, Global Accelerator, VPC endpoints |
| Monitoring | CloudWatch, AWS Compute Optimizer |

## 5. Cost Optimization

Run systems to deliver business value at the lowest price point.

### Design Principles

- Implement cloud financial management
- Adopt a consumption model (pay only for what you use)
- Measure overall efficiency
- Stop spending money on undifferentiated heavy lifting
- Analyze and attribute expenditure

### Key Focus Areas

- **Practice Cloud Financial Management** — Build cost-awareness culture
- **Expenditure and Usage Awareness** — Understand where money is going
- **Cost-Effective Resources** — Right-size, use appropriate pricing models
- **Manage Demand and Supply** — Match supply to demand dynamically
- **Optimize Over Time** — Review and adopt new cost-efficient services

### AWS Services

| Area | Services |
|------|----------|
| Visibility | AWS Cost Explorer, AWS Budgets, Cost and Usage Report |
| Right-sizing | AWS Compute Optimizer, Trusted Advisor |
| Pricing Models | Reserved Instances, Savings Plans, Spot Instances |
| Governance | AWS Organizations (consolidated billing), AWS Control Tower |
| Storage Optimization | S3 Intelligent-Tiering, S3 Glacier, EBS Snapshots Lifecycle |

## 6. Sustainability

Minimize the environmental impacts of running cloud workloads.

### Design Principles

- Understand your impact
- Establish sustainability goals
- Maximize utilization
- Anticipate and adopt new, more efficient hardware and software
- Use managed services
- Reduce the downstream impact of your cloud workloads

### Key Focus Areas

- **Region Selection** — Choose Regions with lower carbon intensity
- **Alignment to Demand** — Scale infrastructure with actual user demand
- **Software and Architecture** — Optimize code patterns and resource usage
- **Data Management** — Reduce unnecessary data movement and storage
- **Hardware and Services** — Use efficient hardware (Graviton) and managed services

### AWS Services

| Area | Services |
|------|----------|
| Measurement | AWS Customer Carbon Footprint Tool |
| Efficient Compute | Graviton instances, Lambda, Fargate |
| Storage | S3 Intelligent-Tiering, lifecycle policies |
| Optimization | Compute Optimizer, Auto Scaling |

## Well-Architected Reviews

AWS provides the Well-Architected Tool in the AWS Console to conduct reviews of your workloads against the framework.

### How It Works

1. Define your workload in the Well-Architected Tool
2. Answer questions for each pillar
3. Review identified high-risk issues (HRIs) and medium-risk issues (MRIs)
4. Create an improvement plan
5. Track progress and re-evaluate periodically

### Benefits

- Identify architectural risks before they become incidents
- Build shared understanding across teams
- Learn AWS best practices specific to your workload type
- Create a roadmap for continuous improvement

## Well-Architected Lenses

In addition to the general framework, AWS offers specialized lenses for specific workload types:

| Lens | Focus |
|------|-------|
| Serverless | Best practices for serverless applications |
| SaaS | Multi-tenant SaaS architectures |
| Data Analytics | Analytics and data lake workloads |
| Machine Learning | ML workloads |
| IoT | Internet of Things workloads |
| Financial Services | Regulated financial workloads |
| Games Industry | Gaming workloads |
| Hybrid Networking | Hybrid and multi-cloud networking |
