# EC2 Rightsizing

Rightsizing is the process of matching instance types and sizes to your workload performance and capacity requirements at the lowest possible cost.

## Why Rightsizing Matters

- **Cost Savings**: Typically 30-70% reduction in compute costs
- **Performance**: Better match between resources and actual needs
- **Sustainability**: Reduced energy consumption from unused capacity

## Prerequisites

Before rightsizing, ensure you have:
- At least 2 weeks of monitoring data
- CloudWatch metrics enabled (CPU, memory, network, disk)
- Understanding of application performance requirements
- Maintenance windows scheduled for changes

## Key Metrics to Monitor

| Metric | Source | Target Utilization | Notes |
|--------|--------|-------------------|-------|
| CPU Utilization | CloudWatch | 40-70% average | Watch for sustained peaks |
| Memory Utilization | CloudWatch/Agent | 50-80% | Install CW agent for OS metrics |
| Network I/O | CloudWatch | Varies by workload | Check for throttling |
| Disk I/O | CloudWatch | < 100% of burst balance | Monitor burst bucket depletion |
| Connection Count | Application metrics | Below instance limits | Important for web servers |

## Rightsizing Strategies

### 1. Downsize Overprovisioned Instances

**When to downsize:**
- Average CPU < 30% over 2 weeks
- Memory usage < 40% consistently
- No performance complaints from users

**Example:**
```bash
# Current: m5.2xlarge (8 vCPU, 32 GB RAM)
# Recommended: m5.xlarge (4 vCPU, 16 GB RAM)
# Savings: ~50% ($0.384/hr → $0.192/hr) = $140/month per instance
```

### 2. Change Instance Family

**When to change families:**
- Compute-optimized (C-series) for CPU-intensive workloads
- Memory-optimized (R/X-series) for databases and caching
- Storage-optimized (I-series) for high IOPS needs
- General-purpose (M/T-series) for balanced workloads

**Example Migration:**
```
Web Server: m5.xlarge → t3.xlarge (burstable, 20% savings)
Database:   m5.2xlarge → r5.xlarge (more RAM, same cost)
Batch Job:  m5.4xlarge → c5.2xlarge (more CPU, 30% savings)
```

### 3. Use Graviton Instances

**ARM-based Graviton processors offer:**
- Up to 40% better price-performance
- Lower power consumption
- Compatible with most modern applications

**Migration Path:**
```
m5.xlarge (x86) → m6g.xlarge (Graviton2): 20% savings
m5.2xlarge (x86) → m7g.2xlarge (Graviton3): 25% savings
```

**Check Compatibility:**
```bash
# Check if your runtime supports ARM
docker run --rm arm64v8/ubuntu uname -m
# Should output: aarch64
```

## Implementation Steps

### Step 1: Analyze Current State

**Using AWS Console:**
1. Go to EC2 Dashboard → Recommendations
2. Review "Right Sizing" recommendations
3. Filter by potential savings

**Using AWS CLI:**
```bash
# Get instance utilization (requires CloudWatch)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-15T00:00:00Z \
  --period 3600 \
  --statistics Average
```

**Using Cost Explorer:**
```bash
aws ce get-rightsizing-recommendation \
  --service EC2 \
  --configuration InclusionFilters='{"UtilizationTruncatedPercentage":["OVER_UTILIZED","UNDER_UTILIZED"]}'
```

### Step 2: Test Changes

**Best Practices:**
- Test in non-production first
- Use Auto Scaling groups for gradual rollout
- Monitor performance metrics closely
- Have rollback plan ready

**Testing Checklist:**
- [ ] Performance benchmarks pass
- [ ] Response times within SLA
- [ ] No increase in error rates
- [ ] Memory headroom for traffic spikes
- [ ] Application logs show no issues

### Step 3: Implement in Production

**Safe Rollout Strategy:**
```yaml
# Example: Blue-Green Deployment with ASG
Resources:
  RightSizedASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      LaunchTemplate:
        LaunchTemplateName: app-right-sized
      MinSize: 2
      MaxSize: 10
      TargetGroupARNs:
        - !Ref NewTargetGroup
```

**Rollback Plan:**
```bash
# Quick rollback to previous instance type
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateId=lt-old-version,Version=1
```

### Step 4: Monitor and Validate

**Key Dashboards:**
- CPU/Memory utilization trends
- Application response times
- Error rates
- Cost comparison (before/after)

**Alerts to Set:**
```yaml
# CloudWatch Alarm Example
CPUHighAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    MetricName: CPUUtilization
    Threshold: 80
    Period: 300
    EvaluationPeriods: 3
    ComparisonOperator: GreaterThanThreshold
```

## Automation Tools

### AWS Trusted Advisor
```bash
# Check Trusted Advisor recommendations
aws support describe-trusted-advisor-check-results \
  --check-id PjRxVlDXf
```

### AWS Compute Optimizer
```bash
# Get optimization recommendations
aws compute-optimizer get-ec2-recommendations \
  --instance-arn arn:aws:compute-optimizer:us-east-1:123456789012:instance/i-1234567890abcdef0
```

### Custom Rightsizing Script
```python
#!/usr/bin/env python3
"""
Simple EC2 rightsizing recommendation script
Analyzes CloudWatch metrics and suggests smaller instance types
"""
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')
ec2 = boto3.client('ec2')

def get_cpu_average(instance_id, days=14):
    """Get average CPU utilization over specified period"""
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(days=days)
    
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
        StartTime=start_time,
        EndTime=end_time,
        Period=3600,
        Statistics=['Average']
    )
    
    datapoints = response['Datapoints']
    if not datapoints:
        return None
    
    avg_cpu = sum(dp['Average'] for dp in datapoints) / len(datapoints)
    return avg_cpu

def recommend_downsize(instance_id, current_type, avg_cpu):
    """Recommend downsizing if CPU is consistently low"""
    if avg_cpu < 30:
        print(f"⚠️  {instance_id}: Consider downsizing from {current_type}")
        print(f"   Average CPU: {avg_cpu:.1f}%")
        print(f"   Potential savings: ~40-50%")
    elif avg_cpu > 80:
        print(f"📈 {instance_id}: Consider upsizing from {current_type}")
        print(f"   Average CPU: {avg_cpu:.1f}%")
        print(f"   Risk of performance issues")
    else:
        print(f"✅ {instance_id}: Well-sized ({current_type})")
        print(f"   Average CPU: {avg_cpu:.1f}%")

# Example usage
instances = ec2.describe_instances(
    Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
)

for reservation in instances['Reservations']:
    for instance in reservation['Instances']:
        instance_id = instance['InstanceId']
        instance_type = instance['InstanceType']
        avg_cpu = get_cpu_average(instance_id)
        
        if avg_cpu:
            recommend_downsize(instance_id, instance_type, avg_cpu)
```

## Common Scenarios

### Scenario 1: Development/Testing Environments
**Strategy:** Aggressive downsizing + scheduling
- Use T-series burstable instances
- Stop instances outside business hours
- Consider spot instances for non-critical testing

**Savings:** 60-80%

### Scenario 2: Production Web Servers
**Strategy:** Conservative rightsizing + Auto Scaling
- Maintain 20-30% headroom for traffic spikes
- Use ALB metrics to scale based on request count
- Implement warm pools for faster scaling

**Savings:** 30-40%

### Scenario 3: Batch Processing
**Strategy:** Spot instances + right-sized compute
- Use C-series for CPU-intensive jobs
- Leverage spot instances with checkpointing
- Scale to zero when no jobs running

**Savings:** 70-90%

### Scenario 4: Databases
**Strategy:** Memory-optimized + reserved pricing
- Prioritize memory over CPU for most databases
- Use Reserved Instances for stable workloads
- Consider Aurora Serverless for variable loads

**Savings:** 40-50%

## Cost-Benefit Analysis

**Example Calculation:**
```
Current State:
- 10 × m5.2xlarge @ $0.384/hr = $3.84/hr
- Monthly cost: $2,765

After Rightsizing:
- 10 × m5.xlarge @ $0.192/hr = $1.92/hr
- Monthly cost: $1,382

Monthly Savings: $1,383 (50%)
Annual Savings: $16,596

Effort: 8 hours engineering time
ROI: Immediate positive return
```

## Best Practices

✅ **Do:**
- Start with non-production environments
- Make one change at a time
- Monitor for at least 1 week before next change
- Document baseline metrics before changes
- Use Infrastructure as Code for consistency
- Combine with Reserved Instances/Savings Plans

❌ **Don't:**
- Rightsize based on peak usage only
- Ignore memory and network metrics
- Make changes during critical business periods
- Forget to update capacity reservations
- Skip performance testing after changes

## Related Resources

- [AWS EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [AWS Compute Optimizer](https://aws.amazon.com/compute-optimizer/)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Graviton Migration Guide](https://github.com/aws/aws-graviton-getting-started)

## See Also

- [Spot Instances](spot-instances.md)
- [Savings Plans](savings-plans.md)
- [Reserved Instances](reserved-instances.md)
- [Auto Scaling](autoscaling.md)
