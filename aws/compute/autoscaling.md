# AWS Auto Scaling for Cost Optimization

Auto Scaling is the most direct lever for matching compute spend to actual demand. Over-provisioned static fleets are one of the top sources of cloud waste — Auto Scaling eliminates that by design.

---

## EC2 Auto Scaling Policies

### Target Tracking (Recommended Default)

Target tracking is the simplest and most effective policy for most workloads. You define a target metric value; AWS automatically adjusts capacity to maintain it.

```hcl
resource "aws_autoscaling_policy" "cpu_target_tracking" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value       = 60.0   # Keep CPU at 60% — leaves headroom for spikes
    disable_scale_in   = false  # Allow scale-in to reduce costs
    
    # Scale-in cooldown: wait 5 min before removing instances
    # Prevents thrashing during brief load drops
  }

  estimated_instance_warmup = 120  # seconds for new instance to contribute to metrics
}

# ALB request count target tracking
resource "aws_autoscaling_policy" "alb_target_tracking" {
  name                   = "alb-request-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
      resource_label         = "${aws_lb.app.arn_suffix}/${aws_lb_target_group.app.arn_suffix}"
    }
    target_value = 1000  # 1000 requests per instance
  }
}
```

### Step Scaling

Use step scaling when you need different scaling rates at different load levels.

```hcl
resource "aws_autoscaling_policy" "scale_out_step" {
  name                   = "scale-out-step"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "StepScaling"
  adjustment_type        = "PercentChangeInCapacity"

  step_adjustment {
    metric_interval_lower_bound = 0   # CPU 70–80%
    metric_interval_upper_bound = 10
    scaling_adjustment          = 20  # Add 20% capacity
  }

  step_adjustment {
    metric_interval_lower_bound = 10  # CPU 80–90%
    metric_interval_upper_bound = 20
    scaling_adjustment          = 40  # Add 40% capacity
  }

  step_adjustment {
    metric_interval_lower_bound = 20  # CPU > 90%
    scaling_adjustment          = 100 # Double capacity
  }
}
```

### Scheduled Scaling

Perfect for predictable patterns — dev environments, business-hours workloads, batch windows.

```hcl
# Scale down nights and weekends (dev environment)
resource "aws_autoscaling_schedule" "scale_down_night" {
  scheduled_action_name  = "scale-down-night"
  autoscaling_group_name = aws_autoscaling_group.dev.name
  min_size               = 0
  max_size               = 0
  desired_capacity       = 0
  recurrence             = "0 20 * * MON-FRI"  # 8 PM weekdays (UTC)
  time_zone              = "America/New_York"
}

resource "aws_autoscaling_schedule" "scale_up_morning" {
  scheduled_action_name  = "scale-up-morning"
  autoscaling_group_name = aws_autoscaling_group.dev.name
  min_size               = 1
  max_size               = 10
  desired_capacity       = 3
  recurrence             = "0 12 * * MON-FRI"  # 8 AM ET weekdays
  time_zone              = "America/New_York"
}

# Scale up before known traffic spike (e.g., daily report generation)
resource "aws_autoscaling_schedule" "pre_scale_batch" {
  scheduled_action_name  = "pre-scale-batch"
  autoscaling_group_name = aws_autoscaling_group.batch.name
  desired_capacity       = 20
  recurrence             = "45 1 * * *"  # 1:45 AM UTC, before 2 AM batch
}
```

---

## Application Auto Scaling

Application Auto Scaling covers non-EC2 resources: ECS services, DynamoDB tables, Lambda concurrency, Aurora replicas, and more.

```python
import boto3

def configure_ecs_autoscaling(
    cluster: str,
    service: str,
    min_capacity: int = 1,
    max_capacity: int = 50,
    target_cpu: float = 60.0,
):
    """Configure target tracking for an ECS service."""
    aas = boto3.client("application-autoscaling", region_name="us-east-1")
    
    resource_id = f"service/{cluster}/{service}"
    
    # Register scalable target
    aas.register_scalable_target(
        ServiceNamespace="ecs",
        ResourceId=resource_id,
        ScalableDimension="ecs:service:DesiredCount",
        MinCapacity=min_capacity,
        MaxCapacity=max_capacity,
    )
    
    # CPU target tracking
    aas.put_scaling_policy(
        PolicyName=f"{service}-cpu-tracking",
        ServiceNamespace="ecs",
        ResourceId=resource_id,
        ScalableDimension="ecs:service:DesiredCount",
        PolicyType="TargetTrackingScaling",
        TargetTrackingScalingPolicyConfiguration={
            "TargetValue": target_cpu,
            "PredefinedMetricSpecification": {
                "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
            },
            "ScaleInCooldown": 300,
            "ScaleOutCooldown": 60,
        },
    )
    print(f"Auto scaling configured for {cluster}/{service}")


def configure_dynamodb_autoscaling(table_name: str):
    """Configure DynamoDB read/write capacity auto scaling."""
    aas = boto3.client("application-autoscaling", region_name="us-east-1")
    
    for dimension, metric in [
        ("dynamodb:table:ReadCapacityUnits", "DynamoDBReadCapacityUtilization"),
        ("dynamodb:table:WriteCapacityUnits", "DynamoDBWriteCapacityUtilization"),
    ]:
        resource_id = f"table/{table_name}"
        
        aas.register_scalable_target(
            ServiceNamespace="dynamodb",
            ResourceId=resource_id,
            ScalableDimension=dimension,
            MinCapacity=5,
            MaxCapacity=1000,
        )
        
        aas.put_scaling_policy(
            PolicyName=f"{table_name}-{dimension.split(':')[-1]}-tracking",
            ServiceNamespace="dynamodb",
            ResourceId=resource_id,
            ScalableDimension=dimension,
            PolicyType="TargetTrackingScaling",
            TargetTrackingScalingPolicyConfiguration={
                "TargetValue": 70.0,  # 70% utilization target
                "PredefinedMetricSpecification": {
                    "PredefinedMetricType": metric
                },
            },
        )
```

---

## Predictive Scaling

Predictive scaling uses ML to forecast load and pre-provisions capacity before demand arrives — eliminating the lag of reactive scaling.

```hcl
resource "aws_autoscaling_policy" "predictive" {
  name                   = "predictive-scaling"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "PredictiveScaling"

  predictive_scaling_configuration {
    metric_specification {
      target_value = 60  # Target CPU %

      predefined_scaling_metric_specification {
        predefined_metric_type = "ASGAverageCPUUtilization"
      }

      predefined_load_metric_specification {
        predefined_metric_type = "ASGTotalCPUReservation"
      }
    }

    mode                          = "ForecastAndScale"  # vs ForecastOnly (dry run)
    scheduling_buffer_time        = 300  # Pre-scale 5 min before predicted peak
    max_capacity_breach_behavior  = "IncreaseMaxCapacity"
    max_capacity_buffer           = 10  # Allow 10% above max_size during peaks
  }
}
```

**Predictive scaling requirements:**
- ASG must have at least 24 hours of CloudWatch metric history
- Works best for workloads with recurring daily/weekly patterns
- Use `ForecastOnly` mode first to validate predictions before enabling scaling

---

## Scale-to-Zero for Dev/Test

Scale-to-zero is the highest-impact cost optimization for non-production environments.

```python
import boto3
import json
from datetime import datetime

def scale_to_zero_dev_resources():
    """Scale down all dev/test resources outside business hours."""
    ec2 = boto3.client("ec2", region_name="us-east-1")
    asg = boto3.client("autoscaling", region_name="us-east-1")
    rds = boto3.client("rds", region_name="us-east-1")
    
    # Stop EC2 instances tagged env=dev
    instances = ec2.describe_instances(
        Filters=[
            {"Name": "tag:Environment", "Values": ["dev", "test"]},
            {"Name": "instance-state-name", "Values": ["running"]},
        ]
    )
    
    instance_ids = [
        i["InstanceId"]
        for r in instances["Reservations"]
        for i in r["Instances"]
    ]
    
    if instance_ids:
        ec2.stop_instances(InstanceIds=instance_ids)
        print(f"Stopped {len(instance_ids)} dev/test instances")
    
    # Scale ASGs to 0
    asgs = asg.describe_auto_scaling_groups(
        Filters=[{"Name": "tag:Environment", "Values": ["dev", "test"]}]
    )
    
    for group in asgs["AutoScalingGroups"]:
        asg.update_auto_scaling_group(
            AutoScalingGroupName=group["AutoScalingGroupName"],
            MinSize=0,
            DesiredCapacity=0,
        )
        print(f"Scaled to 0: {group['AutoScalingGroupName']}")
    
    # Stop RDS instances
    dbs = rds.describe_db_instances()
    for db in dbs["DBInstances"]:
        tags = rds.list_tags_for_resource(ResourceName=db["DBInstanceArn"])["TagList"]
        env_tags = [t["Value"] for t in tags if t["Key"] == "Environment"]
        if any(e in ["dev", "test"] for e in env_tags):
            if db["DBInstanceStatus"] == "available":
                rds.stop_db_instance(DBInstanceIdentifier=db["DBInstanceIdentifier"])
                print(f"Stopped RDS: {db['DBInstanceIdentifier']}")
```

**Estimated savings from scale-to-zero:**
- Dev/test environments running 24/7 vs 10 hours/day weekdays: **~70% reduction**
- Example: 10x m5.xlarge ($0.192/hr) running 24/7 = $1,402/month → $420/month with scale-to-zero

---

## Warm Pools for Faster Scale-Out

Warm pools maintain pre-initialized instances in a stopped state, dramatically reducing scale-out latency without paying for running instances.

```hcl
resource "aws_autoscaling_group" "app_with_warm_pool" {
  name             = "app-with-warm-pool"
  min_size         = 2
  max_size         = 20
  desired_capacity = 5

  # ... launch template, subnets, etc.

  warm_pool {
    pool_state                  = "Stopped"  # Stopped = ~$0 (only EBS cost)
    min_size                    = 2          # Keep 2 warm instances ready
    max_group_prepared_capacity = 8          # Total warm + running = 8 max

    instance_reuse_policy {
      reuse_on_scale_in = true  # Return instances to warm pool on scale-in
    }
  }
}
```

**Cost impact:** Stopped warm pool instances cost only EBS storage (~$0.10/GB/month) vs full instance cost. A warm pool of 5x m5.xlarge saves ~$700/month vs keeping them running.

---

## Karpenter vs Cluster Autoscaler for EKS

| Feature | Karpenter | Cluster Autoscaler |
|---|---|---|
| Provisioning speed | ~45 seconds | 2–5 minutes |
| Instance selection | Any EC2 type dynamically | Pre-defined node groups |
| Spot diversification | Automatic, broad | Limited to node group config |
| Bin packing | Yes (consolidation) | Limited |
| Node consolidation | Yes (automatic) | No |
| Graviton support | Automatic (arm64) | Manual node groups |
| Cost optimization | Higher (dynamic selection) | Lower (static groups) |
| Complexity | Medium | Low |

```yaml
# Karpenter consolidation — automatically removes underutilized nodes
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 30s  # Consolidate quickly for cost savings
    expireAfter: 720h      # Recycle nodes every 30 days (security + fresh AMIs)
  limits:
    cpu: "500"
    memory: 2000Gi
```

---

## Cost-Optimized Scaling Policies

```python
def create_cost_optimized_scaling_config(
    asg_name: str,
    workload_type: str = "web",  # web, batch, ml
) -> dict:
    """Generate scaling policy recommendations based on workload type."""
    
    configs = {
        "web": {
            "target_cpu": 60,
            "scale_out_cooldown": 60,
            "scale_in_cooldown": 300,  # Slow scale-in to avoid thrashing
            "warmup_seconds": 120,
            "min_size": 2,  # HA minimum
            "notes": "ALB request count often better than CPU for web",
        },
        "batch": {
            "target_cpu": 80,  # Higher utilization OK for batch
            "scale_out_cooldown": 30,
            "scale_in_cooldown": 60,
            "warmup_seconds": 60,
            "min_size": 0,  # Scale to zero when no jobs
            "notes": "Use SQS queue depth as scaling metric",
        },
        "ml": {
            "target_cpu": 85,  # GPU utilization preferred
            "scale_out_cooldown": 300,  # ML instances take longer to warm
            "scale_in_cooldown": 600,
            "warmup_seconds": 300,
            "min_size": 0,
            "notes": "Consider Spot + checkpointing for training jobs",
        },
    }
    
    return configs.get(workload_type, configs["web"])
```

---

## Monitoring Scaling Events and Costs

```python
import boto3
from datetime import datetime, timedelta

def get_scaling_cost_impact(asg_name: str, days: int = 7) -> dict:
    """Correlate scaling events with cost changes."""
    cw = boto3.client("cloudwatch", region_name="us-east-1")
    
    end = datetime.utcnow()
    start = end - timedelta(days=days)
    
    # Get desired capacity over time
    metrics = cw.get_metric_statistics(
        Namespace="AWS/AutoScaling",
        MetricName="GroupDesiredCapacity",
        Dimensions=[{"Name": "AutoScalingGroupName", "Value": asg_name}],
        StartTime=start,
        EndTime=end,
        Period=3600,  # Hourly
        Statistics=["Average"],
    )
    
    datapoints = sorted(metrics["Datapoints"], key=lambda x: x["Timestamp"])
    
    # Calculate instance-hours
    total_instance_hours = sum(dp["Average"] for dp in datapoints)
    
    print(f"ASG: {asg_name}")
    print(f"Period: {days} days")
    print(f"Total instance-hours: {total_instance_hours:.0f}")
    print(f"Average capacity: {total_instance_hours / (days * 24):.1f} instances")
    
    if datapoints:
        max_cap = max(dp["Average"] for dp in datapoints)
        min_cap = min(dp["Average"] for dp in datapoints)
        print(f"Peak capacity: {max_cap:.0f}, Min capacity: {min_cap:.0f}")
        print(f"Scaling efficiency: {(total_instance_hours / (max_cap * days * 24)) * 100:.1f}%")
    
    return {"total_instance_hours": total_instance_hours, "datapoints": datapoints}
```

**Key metrics to monitor:**
- `GroupDesiredCapacity` — track over time to spot over-provisioning
- `GroupInServiceInstances` — actual running instances
- Scaling activity frequency — too frequent = thrashing, too rare = under-scaling
- Cost per scaling event — use Cost Explorer with ASG tag filtering
