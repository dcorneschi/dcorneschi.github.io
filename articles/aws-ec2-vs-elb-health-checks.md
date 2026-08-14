# EC2 vs ELB Health Checks

AWS has two separate health check systems that are often confused. They serve different purposes, trigger different actions, and operate independently.

## The Two Systems

| Feature | EC2 Status Checks | ELB Health Checks |
|---------|-------------------|-------------------|
| **What checks** | Hardware/hypervisor (system) and OS (instance) | Application reachability (HTTP/TCP) |
| **Who performs it** | AWS infrastructure | Load balancer |
| **What it protects** | Instance availability | Traffic routing |
| **Action on failure** | Auto Scaling can terminate + replace | ELB stops sending traffic |
| **Scope** | Always runs on every instance | Only on instances registered with a target group/ELB |

## EC2 Status Checks

EC2 performs two built-in checks every minute on every instance:

### System Status Check

Tests the underlying AWS infrastructure (host, network, power):

- Loss of network connectivity
- Loss of system power
- Software issues on the physical host
- Hardware issues on the physical host

**You cannot fix this** — AWS must intervene. Recovery options:
- Wait for AWS to migrate the instance
- Stop and start the instance (moves to new hardware)
- Use auto-recovery

### Instance Status Check

Tests the instance's OS and networking:

- Failed system status check
- Incorrect networking or startup configuration
- Exhausted memory
- Corrupted filesystem
- Incompatible kernel

**You must fix this** — reboot, fix config, or replace the instance.

### Viewing EC2 Status Checks

```sh
# Check status for a specific instance
aws ec2 describe-instance-status --instance-ids i-0abc123 \
  --query "InstanceStatuses[0].{System:SystemStatus.Status, Instance:InstanceStatus.Status}" \
  --output table

# Check all instances with impaired status
aws ec2 describe-instance-status \
  --filters "Name=instance-status.status,Values=impaired" \
  --output table

# Check system status
aws ec2 describe-instance-status \
  --filters "Name=system-status.status,Values=impaired" \
  --output table
```

### EC2 Auto-Recovery

Automatically recover an instance when a system status check fails:

```sh
# Create a CloudWatch alarm that triggers recovery
aws cloudwatch put-metric-alarm \
  --alarm-name "recover-i-0abc123" \
  --namespace AWS/EC2 \
  --metric-name StatusCheckFailed_System \
  --dimensions Name=InstanceId,Value=i-0abc123 \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:automate:us-east-1:ec2:recover
```

Auto-recovery moves the instance to new hardware but keeps the same instance ID, private IP, Elastic IP, and EBS volumes.

## ELB Health Checks

The load balancer checks whether registered targets can serve traffic. If a target fails, the ELB stops routing requests to it.

### Health Check Types

| Type | How it works |
|------|-------------|
| HTTP/HTTPS | Sends a request to a path, expects a 200-399 status code |
| TCP | Opens a TCP connection, expects successful handshake |
| gRPC | Sends a gRPC request, expects a specific status code |

### Configuration Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| Protocol | HTTP | HTTP, HTTPS, TCP, gRPC |
| Path | `/` | Path for HTTP/HTTPS checks (e.g., `/health`) |
| Port | traffic port | Port to check (can differ from traffic port) |
| Healthy threshold | 5 (ALB), 3 (NLB) | Consecutive successes before marking healthy |
| Unhealthy threshold | 2 (ALB), 3 (NLB) | Consecutive failures before marking unhealthy |
| Timeout | 5s (ALB), 10s (NLB) | Time to wait for a response |
| Interval | 30s (ALB), 30s (NLB) | Time between checks |

### Target States

| State | Meaning |
|-------|---------|
| `initial` | Target is registering, health checks starting |
| `healthy` | Target is passing health checks, receiving traffic |
| `unhealthy` | Target is failing health checks, no traffic sent |
| `unused` | Target is not registered or is in a disabled AZ |
| `draining` | Target is deregistering, connections draining |
| `unavailable` | Health checks disabled or target is in an unused AZ |

### Viewing ELB Health Check Status

```sh
# Check target health in a target group
aws elbv2 describe-target-health --target-group-arn <tg-arn> \
  --output table

# Get target health with reason
aws elbv2 describe-target-health --target-group-arn <tg-arn> \
  --query "TargetHealthDescriptions[].{Target:Target.Id, Port:Target.Port, State:TargetHealth.State, Reason:TargetHealth.Reason}" \
  --output table

# Find all unhealthy targets
aws elbv2 describe-target-health --target-group-arn <tg-arn> \
  --query "TargetHealthDescriptions[?TargetHealth.State=='unhealthy']" \
  --output json
```

### Configuring ELB Health Checks

```sh
# Modify health check settings
aws elbv2 modify-target-group \
  --target-group-arn <tg-arn> \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 10 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

# View current health check configuration
aws elbv2 describe-target-groups --target-group-arns <tg-arn> \
  --query "TargetGroups[0].{Protocol:HealthCheckProtocol, Path:HealthCheckPath, Interval:HealthCheckIntervalSeconds, Timeout:HealthCheckTimeoutSeconds, Healthy:HealthyThresholdCount, Unhealthy:UnhealthyThresholdCount}" \
  --output table
```

## How They Interact with Auto Scaling

This is where the two systems come together — and where confusion happens.

### Auto Scaling Health Check Types

Auto Scaling Groups (ASGs) can use one or both:

| ASG Health Check Type | What it checks | Default |
|-----------------------|----------------|---------|
| `EC2` | EC2 status checks only | Yes (always) |
| `ELB` | ELB target health | Must be explicitly enabled |

```sh
# Check what health check type an ASG uses
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg> \
  --query "AutoScalingGroups[0].HealthCheckType" --output text

# Enable ELB health checks on an ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg> \
  --health-check-type ELB \
  --health-check-grace-period 300
```

### What Happens on Failure

| Scenario | EC2 Health Check Type | ELB Health Check Type |
|----------|----------------------|----------------------|
| EC2 system check fails | ASG terminates + replaces | ASG terminates + replaces |
| EC2 instance check fails | ASG terminates + replaces | ASG terminates + replaces |
| ELB health check fails | ELB stops traffic, **ASG does nothing** | ELB stops traffic, **ASG terminates + replaces** |
| App returns 500 | Nothing (EC2 doesn't check apps) | ELB marks unhealthy, ASG replaces (if ELB type) |

### The Critical Difference

With ASG health check type set to `EC2` only:
- ELB stops routing traffic to unhealthy targets
- But the **instance stays running** — ASG thinks it's fine
- You get an instance that's registered but receiving no traffic
- Manual intervention required

With ASG health check type set to `ELB`:
- ELB stops routing traffic to unhealthy targets
- ASG detects the instance is unhealthy (via ELB)
- ASG terminates the instance and launches a replacement
- **Self-healing** — no manual intervention

### Health Check Grace Period

The grace period tells ASG to wait before checking ELB health after launch. This prevents ASG from terminating instances that are still booting:

```sh
# Set grace period to 5 minutes
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg> \
  --health-check-grace-period 300
```

If your app takes 3 minutes to boot, set the grace period to at least 180 seconds (add buffer).

## Timeline: What Happens When an App Crashes

```
T+0s    App crashes (stops responding on /health)
T+30s   ELB health check fails (1st failure, interval=30s)
T+60s   ELB health check fails again (2nd failure)
        → ELB marks target UNHEALTHY, stops sending traffic
T+60s   (If ASG type=ELB) ASG detects unhealthy instance
        → ASG terminates instance
        → ASG launches new instance
T+90s   New instance boots, app starts
T+120s  ELB health check passes on new instance
T+150s  ELB marks new target HEALTHY (after healthy threshold)
        → Traffic resumes to new instance
```

Total recovery time: ~2-3 minutes (depends on health check interval, thresholds, and app boot time).

## Best Practices

### Health Check Endpoint Design

```sh
# Good: dedicated health endpoint that checks dependencies
GET /health → 200 (app is ready to serve traffic)
GET /health → 503 (app is up but can't serve, e.g., DB is down)

# Bad: using the homepage as health check
GET / → 200 (might succeed even when the app is broken)
```

A good `/health` endpoint should:
- Return 200 only when the app can actually serve requests
- Check critical dependencies (database, cache, etc.)
- Be lightweight (no heavy computation)
- Not require authentication

### Recommended Settings

For most web applications:

```sh
aws elbv2 modify-target-group \
  --target-group-arn <tg-arn> \
  --health-check-path /health \
  --health-check-interval-seconds 10 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3
```

- **Interval 10s**: fast detection without overloading
- **Timeout 5s**: generous enough for cold responses
- **Healthy threshold 2**: don't send traffic too early
- **Unhealthy threshold 3**: avoid flapping on transient issues

### Always Enable ELB Health Checks on ASGs

```sh
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg> \
  --health-check-type ELB \
  --health-check-grace-period 300
```

Without this, your ASG won't replace instances with crashed applications.

## Gotchas

- **Grace period too short**: If your app takes 2 minutes to boot but the grace period is 60s, ASG will terminate instances in a loop (boot → mark unhealthy → terminate → boot → ...).
- **Health check path returns 301**: Redirects (e.g., HTTP→HTTPS) count as failures. Use the exact path that returns 200.
- **Security groups blocking health checks**: ELB health checks come from the ELB's IP range. The instance's security group must allow inbound traffic from the ELB (or the VPC CIDR) on the health check port.
- **NLB cross-zone disabled**: If cross-zone load balancing is off, health checks only reach targets in the same AZ as the NLB node. Targets in AZs without an NLB node won't be checked.
- **Target deregistration delay**: When a target is deregistered (or marked unhealthy), existing connections drain for 300s by default. Adjust with `deregistration_delay.timeout_seconds`.
- **ELB health ≠ EC2 health**: A target can be `healthy` in ELB but have an impaired EC2 status check (rare but possible during hardware degradation).
- **Multiple target groups**: If an instance is in multiple target groups, it can be healthy in one and unhealthy in another. ASG uses the combined result — unhealthy in any = unhealthy.

## Quick Reference

```sh
# EC2 status checks
aws ec2 describe-instance-status --instance-ids <id>

# ELB target health
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# ASG health check config
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names <asg> \
  --query "AutoScalingGroups[0].{Type:HealthCheckType, Grace:HealthCheckGracePeriod}"

# Enable ELB health checks on ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name <asg> \
  --health-check-type ELB \
  --health-check-grace-period 300

# Modify ELB health check
aws elbv2 modify-target-group \
  --target-group-arn <tg-arn> \
  --health-check-path /health \
  --health-check-interval-seconds 10 \
  --unhealthy-threshold-count 3
```


## Custom Health Check Endpoint Example

A good health endpoint checks critical dependencies and system resources:

```python
# Flask application with custom health endpoint
from flask import Flask, jsonify
import psutil

app = Flask(__name__)

@app.route('/health')
def health_check():
    try:
        db_status = check_database_connection()
        cpu_usage = psutil.cpu_percent(interval=1)
        memory_usage = psutil.virtual_memory().percent

        if db_status and cpu_usage < 80 and memory_usage < 85:
            return jsonify({
                "status": "healthy",
                "cpu": cpu_usage,
                "memory": memory_usage,
                "database": "connected"
            }), 200
        else:
            return jsonify({
                "status": "unhealthy",
                "cpu": cpu_usage,
                "memory": memory_usage,
                "database": "connected" if db_status else "disconnected"
            }), 503

    except Exception as e:
        return jsonify({"status": "unhealthy", "error": str(e)}), 503

def check_database_connection():
    try:
        # Your database connection logic here
        return True
    except:
        return False
```

Return 200 when healthy, 503 when degraded. The ELB accepts 200-399 as healthy by default.

## Common Scenarios

| Scenario | EC2 Check | ELB Check | Result |
|----------|-----------|-----------|--------|
| Database connection lost | Pass | Fail | Traffic stops, instance stays (EC2 type) or gets replaced (ELB type) |
| Hardware failure | Fail | Fail | Instance terminated and replaced |
| App deployment (restart) | Pass | Temporarily fail | Traffic shifts away, then back after healthy threshold |
| Memory exhaustion (OOM) | Fail | Fail | Instance terminated and replaced |
| App deadlock (hangs) | Pass | Fail | ELB stops traffic; ASG replaces only if health check type is ELB |

## Terraform Configuration

```hcl
# Auto Scaling Group with ELB health checks
resource "aws_autoscaling_group" "web" {
  name                = "web-servers-asg"
  vpc_zone_identifier = var.subnet_ids
  min_size            = 2
  max_size            = 10
  desired_capacity    = 4

  health_check_type         = "ELB"
  health_check_grace_period = 600

  target_group_arns = [aws_lb_target_group.web.arn]

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
}

# ALB Target Group with health check
resource "aws_lb_target_group" "web" {
  name     = "web-servers-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 10
    path                = "/health"
    matcher             = "200"
    protocol            = "HTTP"
    port                = "traffic-port"
  }
}

# CloudWatch alarm for EC2 status checks
resource "aws_cloudwatch_metric_alarm" "instance_status" {
  alarm_name          = "ec2-instance-status-check"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "StatusCheckFailed_Instance"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Maximum"
  threshold           = 0
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }
}
```

## CloudWatch Metrics to Monitor

### EC2 Health Metrics

| Metric | Description |
|--------|-------------|
| `StatusCheckFailed_Instance` | Instance-level check failed (OS/config issue) |
| `StatusCheckFailed_System` | System-level check failed (hardware/hypervisor) |
| `StatusCheckFailed` | Either check failed (combined) |

### ELB Health Metrics

| Metric | Description |
|--------|-------------|
| `HealthyHostCount` | Number of healthy targets |
| `UnHealthyHostCount` | Number of unhealthy targets |
| `TargetResponseTime` | Time to receive response from target |
| `HTTPCode_Target_5XX_Count` | 5xx errors from targets |
| `RequestCount` | Total requests routed |

### CloudWatch Dashboard Example

```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ApplicationELB", "HealthyHostCount", "TargetGroup", "my-tg", "LoadBalancer", "my-alb"],
          [".", "UnHealthyHostCount", ".", ".", ".", "."]
        ],
        "period": 60,
        "stat": "Average",
        "title": "ELB Target Health"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/EC2", "StatusCheckFailed_Instance", "AutoScalingGroupName", "my-asg"],
          [".", "StatusCheckFailed_System", ".", "."]
        ],
        "period": 60,
        "stat": "Maximum",
        "title": "EC2 Status Checks"
      }
    }
  ]
}
```
