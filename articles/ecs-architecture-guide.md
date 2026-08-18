# ECS Cluster Architecture

Amazon Elastic Container Service (ECS) is AWS's fully managed container orchestration service. This guide covers cluster architecture, launch types, networking, service discovery, auto-scaling, and production patterns.

## Core Components

| Component | Description |
|-----------|-------------|
| Cluster | Logical grouping of tasks and services |
| Task Definition | Blueprint for running containers (image, CPU, memory, ports, volumes) |
| Task | Running instance of a task definition (one or more containers) |
| Service | Maintains a desired count of tasks, handles deployments and load balancing |
| Container Instance | EC2 instance registered to a cluster (EC2 launch type only) |
| ECS Agent | Software on EC2 instances that communicates with the ECS control plane |

## Launch Types

### Fargate (Serverless)

AWS manages the underlying infrastructure — no EC2 instances to provision or manage. Under the hood, each task runs in an AWS Firecracker microVM for strong isolation between tenants.

- Each task gets its own ENI (elastic network interface) with a private IP in your VPC
- Container runtime uses Firecracker microVMs — lightweight VM sandboxes (not shared kernels)
- You define CPU/memory, AWS provisions the sandbox per task

| Aspect | Detail |
|--------|--------|
| Infrastructure | Fully managed by AWS |
| Scaling | Per-task, no cluster capacity planning |
| Pricing | Pay per vCPU/memory per second |
| Patching | AWS handles OS and runtime |
| Best for | Microservices, batch jobs, teams without infra expertise |

### EC2 (Self-Managed)

You manage the EC2 instances that run your containers.

| Aspect | Detail |
|--------|--------|
| Infrastructure | You manage EC2 instances |
| Scaling | Cluster auto-scaling + service auto-scaling |
| Pricing | EC2 instance costs (can use Spot, RIs, Savings Plans) |
| Patching | You handle OS, Docker, ECS agent updates |
| Best for | GPU workloads, custom AMIs, cost optimization at scale, Windows containers |

### Fargate vs EC2 — Comparison

| Feature | Fargate | EC2 |
|---------|---------|-----|
| Cluster capacity management | No | Yes |
| Instance type selection | No | Yes |
| SSH into host | No | Yes |
| GPU support | No | Yes |
| Spot pricing | Fargate Spot | EC2 Spot |
| Startup latency | Higher (15-45s) | Lower (if capacity exists) |
| Cost at scale | Higher | Lower (with RIs/Spot) |
| Maintenance | None | OS patching, agent updates |
| daemonsets / sidecars | Limited | Full flexibility |
| Privileged containers | No | Yes |

## Cluster Architecture

### Single-Region Architecture

```text
┌─────────────────────────────────────────────────────────┐
│                        VPC                              │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Public       │  │ Public       │  │ Public       │   │
│  │ Subnet (AZ1) │  │ Subnet (AZ2) │  │ Subnet (AZ3) │   │
│  │ ALB          │  │ ALB          │  │ ALB          │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                 │           │
│  ┌──────┴───────┐  ┌──────┴───────┐  ┌──────┴───────┐   │
│  │ Private      │  │ Private      │  │ Private      │   │
│  │ Subnet (AZ1) │  │ Subnet (AZ2) │  │ Subnet (AZ3) │   │
│  │ ECS Tasks    │  │ ECS Tasks    │  │ ECS Tasks    │   │
│  └──────────────┘  └──────────────┘  └──────────────┘   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Network Configuration

- **Public subnets** — ALB/NLB, NAT Gateways
- **Private subnets** — ECS tasks/instances, RDS, ElastiCache
- **Security groups** — per-service (ALB → tasks only on container port)
- **NAT Gateway** — allows private tasks to pull images and reach AWS APIs

## Task Definition

### Key Parameters

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789012.dkr.ecr.eu-west-1.amazonaws.com/my-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "eu-west-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ],
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:eu-west-1:123456789012:secret:db-password"
        }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### IAM Roles

| Role | Purpose |
|------|---------|
| Execution Role | Pulls images from ECR, writes logs to CloudWatch, fetches secrets |
| Task Role | Permissions for the application code (S3, DynamoDB, SQS, etc.) |
| Instance Role (EC2 only) | Allows EC2 to register with ECS, pull images |

### Network Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `awsvpc` | Each task gets its own ENI and private IP | Fargate (required), EC2 (recommended) |
| `bridge` | Docker bridge networking (port mapping) | EC2 legacy workloads |
| `host` | Container uses host network directly | EC2 high-performance, single-task-per-port |
| `none` | No external networking | Batch processing with no network needs |

### Task Definition vs docker-compose.yml

Both describe how to run containers — same concepts, different orchestrator.

| Aspect | docker-compose.yml | ECS Task Definition |
|--------|-------------------|---------------------|
| Defines | Containers, images, ports, volumes, env vars | Same concepts |
| Multi-container | `services:` block | `containerDefinitions[]` array |
| Networking | Docker bridge / custom networks | `awsvpc` — real VPC IP per task |
| Volumes | Local bind mounts or named volumes | EFS, EBS, or ephemeral storage |
| Resource limits | `deploy.resources.limits` | `cpu` / `memory` at task level |
| Who runs it | Docker Engine on one machine | ECS scheduler on Fargate or EC2 |
| Scaling | `docker compose up --scale web=3` | Service `desired_count = 3` |
| Restart policy | `restart: always` | Service auto-replaces failed tasks |

Key distinction:
- **docker-compose** = "run these containers on this one machine"
- **Task Definition + Service** = "blueprint" + "keep N copies running across the cloud"

The Task Definition is the *what*, the Service is the *how many and where*.

## Services

### Service Configuration

```bash
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-app \
  --task-definition my-app:3 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa,subnet-bbb],securityGroups=[sg-123],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=app,containerPort=8080" \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100" \
  --deployment-controller "type=ECS"
```

### Deployment Strategies

| Strategy | Behavior |
|----------|----------|
| Rolling update | Replace tasks in batches (default) |
| Blue/Green (CodeDeploy) | Deploy to new target group, shift traffic |
| External | Third-party controller manages deployment |

### Rolling Update Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `minimumHealthyPercent` | Min % of tasks that must stay running | 100 |
| `maximumPercent` | Max % of tasks allowed during deployment | 200 |

With defaults (100/200): ECS starts new tasks first, then drains old ones — zero downtime.

## Load Balancing

### ALB Integration

```bash
# Create target group (IP type for awsvpc/Fargate)
aws elbv2 create-target-group \
  --name my-app-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-123 \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3
```

| Target Type | When to Use |
|-------------|-------------|
| `ip` | Fargate, awsvpc mode (required) |
| `instance` | EC2 with bridge/host mode |

### Multiple Target Groups

A single service can register with multiple target groups — useful for internal + external access or path-based routing.

## Auto Scaling

### Service Auto Scaling (Task Count)

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Target tracking policy (CPU)
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

### Predefined Metrics

| Metric | Description |
|--------|-------------|
| `ECSServiceAverageCPUUtilization` | Average CPU across all tasks |
| `ECSServiceAverageMemoryUtilization` | Average memory across all tasks |
| `ALBRequestCountPerTarget` | Requests per target from ALB |

### Cluster Auto Scaling (EC2 Launch Type)

Uses a Capacity Provider with an Auto Scaling Group:

```bash
aws ecs create-capacity-provider \
  --name my-capacity-provider \
  --auto-scaling-group-provider "autoScalingGroupArn=arn:aws:autoscaling:...,managedScaling={status=ENABLED,targetCapacity=80,minimumScalingStepSize=1,maximumScalingStepSize=5},managedTerminationProtection=ENABLED"
```

## Service Discovery (Cloud Map)

ECS integrates with AWS Cloud Map for DNS-based service discovery:

```bash
# Create namespace
aws servicediscovery create-private-dns-namespace \
  --name my-app.local \
  --vpc vpc-123

# Create service in Cloud Map
aws servicediscovery create-service \
  --name backend \
  --dns-config "NamespaceId=ns-123,DnsRecords=[{Type=A,TTL=10}]" \
  --health-check-custom-config "FailureThreshold=1"
```

Tasks register automatically and are reachable at `backend.my-app.local`.

## Secrets and Configuration

### Secrets Manager Integration

```json
"secrets": [
  {
    "name": "DB_PASSWORD",
    "valueFrom": "arn:aws:secretsmanager:eu-west-1:123456789012:secret:prod/db-password"
  },
  {
    "name": "API_KEY",
    "valueFrom": "arn:aws:secretsmanager:eu-west-1:123456789012:secret:prod/api-key"
  }
]
```

### SSM Parameter Store

```json
"secrets": [
  {
    "name": "CONFIG_VALUE",
    "valueFrom": "arn:aws:ssm:eu-west-1:123456789012:parameter/prod/config"
  }
]
```

### Environment Files (S3)

```json
"environmentFiles": [
  {
    "value": "arn:aws:s3:::my-bucket/config/prod.env",
    "type": "s3"
  }
]
```

## Logging

### CloudWatch Logs (awslogs)

```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group": "/ecs/my-app",
    "awslogs-region": "eu-west-1",
    "awslogs-stream-prefix": "ecs",
    "awslogs-create-group": "true"
  }
}
```

### FireLens (Fluent Bit/Fluentd)

For routing logs to multiple destinations (S3, Elasticsearch, Datadog):

```json
"logConfiguration": {
  "logDriver": "awsfirelens",
  "options": {
    "Name": "datadog",
    "Host": "http-intake.logs.datadoghq.eu",
    "TLS": "on",
    "apikey": "${DD_API_KEY}",
    "dd_service": "my-app",
    "dd_source": "ecs"
  }
}
```

## ECS CLI Commands

```bash
# Cluster management
aws ecs list-clusters
aws ecs describe-clusters --clusters my-cluster
aws ecs create-cluster --cluster-name my-cluster
aws ecs delete-cluster --cluster my-cluster

# Task definitions
aws ecs list-task-definitions
aws ecs describe-task-definition --task-definition my-app:3
aws ecs register-task-definition --cli-input-json file://task-def.json
aws ecs deregister-task-definition --task-definition my-app:2

# Services
aws ecs list-services --cluster my-cluster
aws ecs describe-services --cluster my-cluster --services my-app
aws ecs update-service --cluster my-cluster --service my-app --desired-count 5
aws ecs update-service --cluster my-cluster --service my-app --task-definition my-app:4 --force-new-deployment
aws ecs delete-service --cluster my-cluster --service my-app --force

# Tasks
aws ecs list-tasks --cluster my-cluster --service-name my-app
aws ecs describe-tasks --cluster my-cluster --tasks task-id
aws ecs run-task --cluster my-cluster --task-definition my-app:3 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-aaa],securityGroups=[sg-123],assignPublicIp=DISABLED}"
aws ecs stop-task --cluster my-cluster --task task-id --reason "Manual stop"

# Container instances (EC2 launch type)
aws ecs list-container-instances --cluster my-cluster
aws ecs describe-container-instances --cluster my-cluster --container-instances instance-id
aws ecs update-container-instances-state --cluster my-cluster --container-instances instance-id --status DRAINING

# Execute command (ECS Exec)
aws ecs execute-command --cluster my-cluster --task task-id --container app --interactive --command "/bin/sh"
```

## ECS Exec (Debugging)

Interactive shell into running containers (like `kubectl exec`):

```bash
# Enable on service
aws ecs update-service --cluster my-cluster --service my-app --enable-execute-command

# Or enable in task definition
# Requires SSM agent and task role with ssmmessages permissions

# Exec into container
aws ecs execute-command \
  --cluster my-cluster \
  --task abc123 \
  --container app \
  --interactive \
  --command "/bin/sh"
```

Required task role permissions:

```json
{
  "Effect": "Allow",
  "Action": [
    "ssmmessages:CreateControlChannel",
    "ssmmessages:CreateDataChannel",
    "ssmmessages:OpenControlChannel",
    "ssmmessages:OpenDataChannel"
  ],
  "Resource": "*"
}
```

## Capacity Providers

### Fargate Capacity Providers

```bash
aws ecs put-cluster-capacity-providers \
  --cluster my-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy "capacityProvider=FARGATE,weight=1,base=2" "capacityProvider=FARGATE_SPOT,weight=3"
```

This runs 2 tasks on regular Fargate (base), then spreads additional tasks 75% Spot / 25% On-Demand.

### EC2 Capacity Providers

Link an ASG to ECS for managed scaling:

```bash
aws ecs create-capacity-provider \
  --name ec2-spot-provider \
  --auto-scaling-group-provider "autoScalingGroupArn=arn:aws:autoscaling:...,managedScaling={status=ENABLED,targetCapacity=80},managedTerminationProtection=ENABLED"
```

## ALB vs Kubernetes Ingress (Traefik)

Conceptually the same role: L7 reverse proxy routing external traffic to backend containers.

| Aspect | ALB (AWS) | Traefik (K8s Ingress) |
|--------|-----------|----------------------|
| Role | L7 load balancer / reverse proxy | L7 load balancer / reverse proxy |
| Routing | Host-based, path-based rules | Host, path, headers, regex, middleware |
| TLS termination | Yes (via ACM certs) | Yes (via cert-manager / Let's Encrypt) |
| Service discovery | Target groups (ECS registers tasks automatically) | Kubernetes API (watches Ingress/IngressRoute CRDs) |
| Auto-scaling | AWS-managed, infinite capacity | You scale Traefik pods yourself |
| Configuration | Terraform / Console / listeners+rules | Annotations, IngressRoute CRDs, file/labels |
| Health checks | Built-in, per target group | Built-in per backend |
| Cost model | Pay per LCU + hourly | Free (runs as pods on your cluster) |

### What ALB Cannot Do That Traefik Can

- Middleware chains (rate limiting, circuit breakers, retry policies built-in)
- Automatic Let's Encrypt cert provisioning (ALB uses ACM instead)
- TCP/UDP routing on the same listener (need NLB for raw TCP)

### What ALB Gives You That Traefik Doesn't

- Fully managed — zero pods to maintain, no OOM risk, no patching
- Infinite horizontal scaling handled by AWS
- Native integration with WAF, Shield, Cognito auth

## Traffic Flow Pattern

```text
Client (HTTPS:443)
  → ALB (public subnets, terminates TLS via ACM)
    → Target Group (forwards to container port)
      → ECS Task (private subnet, Fargate, awsvpc)
        → Application container
          → EFS / EBS (persistent storage)
          → RDS / DynamoDB (database)
```

## Security Group Chaining

Each layer only accepts traffic from the layer above — defense in depth:

```text
ALB SG        → allows 443/80 inbound from internet (0.0.0.0/0)
ECS Task SG   → allows container port inbound only from ALB SG
RDS SG        → allows 5432/3306 inbound only from ECS Task SG
EFS SG        → allows 2049 inbound only from ECS Task SG
```

This ensures:
- The database is never directly reachable from the internet
- ECS tasks are only reachable through the load balancer
- Storage (EFS) is only accessible from application tasks

## Production Best Practices

| Area | Recommendation |
|------|----------------|
| High availability | Spread tasks across 3 AZs minimum |
| Networking | Use `awsvpc` mode with private subnets |
| Security | Separate execution role (ECR/logs) from task role (app permissions) |
| Secrets | Use Secrets Manager or SSM Parameter Store, never env vars |
| Images | Use ECR with immutable tags, scan for vulnerabilities |
| Health checks | Configure both container-level and ALB health checks |
| Logging | Use awslogs or FireLens, set retention policies |
| Auto scaling | Target tracking on CPU + ALB request count |
| Deployments | Use minimumHealthyPercent=100 for zero-downtime |
| Cost | Use Fargate Spot for fault-tolerant workloads, RIs for steady state |
| Monitoring | CloudWatch Container Insights for metrics and logs |
| ECS Exec | Enable for debugging but restrict with IAM conditions |

## Troubleshooting

| Issue | Investigation |
|-------|---------------|
| Task fails to start | Check `stoppedReason` in describe-tasks output |
| Image pull failure | Verify ECR permissions in execution role, check NAT/VPC endpoints |
| Health check failures | Check security group allows ALB → container port |
| Service stuck deploying | Check `events` in describe-services, verify health check passes |
| Insufficient capacity | Check capacity provider target capacity, verify subnet IPs available |
| Container OOMKilled | Increase memory in task definition, check for memory leaks |
| ECS Exec not working | Verify SSM permissions in task role, ensure `--enable-execute-command` |
| Slow scaling | Reduce scale-out cooldown, check if ALB slow to register targets |

```bash
# View stopped task reason
aws ecs describe-tasks --cluster my-cluster --tasks task-id \
  --query 'tasks[].{status:lastStatus,reason:stoppedReason,code:containers[].exitCode}'

# View service events (last 10)
aws ecs describe-services --cluster my-cluster --services my-app \
  --query 'services[].events[:10].[createdAt,message]' --output table

# Check container instance resources (EC2)
aws ecs describe-container-instances --cluster my-cluster --container-instances id \
  --query 'containerInstances[].{remaining:remainingResources,registered:registeredResources}'
```
