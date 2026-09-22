# AWS Spot Instances — Deep Dive

Spot Instances let you use spare EC2 capacity at discounts of up to 90% vs On-Demand. The trade-off: AWS can reclaim them with a 2-minute warning. Designed correctly, this is a non-issue for most workloads.

---

## How Spot Pricing Works

Spot prices fluctuate based on supply and demand per instance type, per AZ. Unlike the old bidding model, you now pay the current Spot price — no bid required. Prices are generally stable and only spike during capacity crunches.

**Key concepts:**
- **Spot Pool**: A combination of instance type + OS + AZ. Each pool has its own price and interruption rate.
- **Spot Price History**: Available via console or API — use it to identify stable, low-interruption pools.
- **Interruption Rate**: AWS publishes frequency-of-interruption data per pool (< 5%, 5–10%, etc.).

```bash
# Get spot price history for a pool
aws ec2 describe-spot-price-history \
  --instance-types m5.xlarge m5.2xlarge \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -u +"%Y-%m-%dT%H:%M:%SZ" -d "1 hour ago") \
  --region us-east-1
```

---

## Spot Interruption Handling

When AWS needs capacity back, it sends a **2-minute interruption notice** via:
1. EC2 instance metadata endpoint
2. EventBridge event (`EC2 Spot Instance Interruption Warning`)
3. CloudWatch Events

### Metadata Endpoint Polling

```python
import requests
import time
import subprocess
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("spot-handler")

METADATA_URL = "http://169.254.169.254/latest/meta-data/spot/termination-time"
POLL_INTERVAL = 5  # seconds

def get_imds_token():
    """Get IMDSv2 token."""
    resp = requests.put(
        "http://169.254.169.254/latest/api/token",
        headers={"X-aws-ec2-metadata-token-ttl-seconds": "21600"},
        timeout=2
    )
    return resp.text

def check_spot_interruption(token: str) -> bool:
    """Returns True if termination notice is present."""
    try:
        resp = requests.get(
            METADATA_URL,
            headers={"X-aws-ec2-metadata-token": token},
            timeout=2
        )
        return resp.status_code == 200
    except requests.RequestException:
        return False

def graceful_shutdown():
    """Custom shutdown logic — checkpoint state, drain connections, etc."""
    logger.info("Spot interruption detected — starting graceful shutdown")
    # 1. Stop accepting new work
    subprocess.run(["systemctl", "stop", "my-worker"], check=False)
    # 2. Checkpoint current job state to S3
    subprocess.run(["python", "checkpoint.py", "--save"], check=False)
    # 3. Deregister from load balancer / service discovery
    subprocess.run(["python", "deregister.py"], check=False)
    logger.info("Graceful shutdown complete")

def main():
    token = get_imds_token()
    logger.info("Spot interruption monitor started")
    while True:
        if check_spot_interruption(token):
            graceful_shutdown()
            break
        time.sleep(POLL_INTERVAL)

if __name__ == "__main__":
    main()
```

### EventBridge Rule for Interruption

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Spot Instance Interruption Warning"],
  "detail": {
    "instance-action": ["terminate"]
  }
}
```

---

## Spot Fleet and EC2 Auto Scaling with Mixed Instances

### Mixed Instances Policy (Terraform)

```hcl
resource "aws_autoscaling_group" "spot_mixed" {
  name                = "spot-mixed-asg"
  vpc_zone_identifier = var.subnet_ids
  min_size            = 2
  max_size            = 20
  desired_capacity    = 5

  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = 2      # Always keep 2 On-Demand
      on_demand_percentage_above_base_capacity = 0      # Rest are Spot
      spot_allocation_strategy                 = "capacity-optimized"
      spot_instance_pools                      = 0      # Not used with capacity-optimized
    }

    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.app.id
        version            = "$Latest"
      }

      # Instance type diversification — critical for Spot availability
      override {
        instance_type     = "m5.xlarge"
        weighted_capacity = "4"
      }
      override {
        instance_type     = "m5a.xlarge"
        weighted_capacity = "4"
      }
      override {
        instance_type     = "m4.xlarge"
        weighted_capacity = "4"
      }
      override {
        instance_type     = "m5.2xlarge"
        weighted_capacity = "8"
      }
      override {
        instance_type     = "m5a.2xlarge"
        weighted_capacity = "8"
      }
    }
  }

  tag {
    key                 = "Name"
    value               = "spot-mixed-worker"
    propagate_at_launch = true
  }
}
```

**Allocation strategies:**
| Strategy | Best For |
|---|---|
| `capacity-optimized` | Lowest interruption rate — recommended default |
| `price-capacity-optimized` | Balance of price and availability |
| `lowest-price` | Maximum savings, higher interruption risk |
| `diversified` | Spot Fleet only — spread across pools |

---

## Spot for Kubernetes

### Karpenter (Recommended)

Karpenter is the modern approach — it provisions nodes directly without node groups, enabling fine-grained Spot diversification.

```yaml
# NodePool with Spot + On-Demand fallback
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["m", "c", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["4"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

```yaml
# Pod with Spot preference
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-worker
spec:
  template:
    spec:
      nodeSelector:
        karpenter.sh/capacity-type: spot
      tolerations:
        - key: "karpenter.sh/capacity-type"
          operator: "Equal"
          value: "spot"
          effect: "NoSchedule"
      terminationGracePeriodSeconds: 120  # Match 2-min notice
```

### Managed Node Groups with Spot

```hcl
resource "aws_eks_node_group" "spot" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "spot-workers"
  node_role_arn   = aws_iam_role.node.arn
  subnet_ids      = var.private_subnet_ids
  capacity_type   = "SPOT"

  instance_types = [
    "m5.xlarge", "m5a.xlarge", "m5d.xlarge",
    "m4.xlarge", "m5.2xlarge", "m5a.2xlarge"
  ]

  scaling_config {
    desired_size = 3
    min_size     = 1
    max_size     = 20
  }
}
```

---

## Spot for ECS and Fargate Spot

### ECS Capacity Provider Strategy

```json
{
  "capacityProviders": ["FARGATE_SPOT", "FARGATE"],
  "defaultCapacityProviderStrategy": [
    {
      "capacityProvider": "FARGATE_SPOT",
      "weight": 4,
      "base": 0
    },
    {
      "capacityProvider": "FARGATE",
      "weight": 1,
      "base": 1
    }
  ]
}
```

Fargate Spot saves ~70% vs standard Fargate. Use it for batch tasks, background workers, and non-critical services.

---

## SageMaker Managed Spot Training

```python
import sagemaker
from sagemaker.pytorch import PyTorch

estimator = PyTorch(
    entry_point="train.py",
    role=sagemaker.get_execution_role(),
    instance_type="ml.p3.2xlarge",
    instance_count=1,
    framework_version="2.0",
    py_version="py310",
    # Spot training config
    use_spot_instances=True,
    max_run=86400,           # 24 hours max total
    max_wait=90000,          # 25 hours — allows for interruption + resume
    checkpoint_s3_uri="s3://my-bucket/checkpoints/",
    checkpoint_local_path="/opt/ml/checkpoints",
)

estimator.fit({"training": "s3://my-bucket/data/"})

# Savings report
print(f"Training seconds: {estimator.training_job_analytics()}")
```

Managed Spot Training can save up to **90%** on training costs. SageMaker handles checkpointing and resumption automatically.

---

## Instance Diversification Strategy

The #1 rule for Spot reliability: **never depend on a single instance type or AZ**.

```python
import boto3

def get_spot_recommendations(vcpus_needed: int, memory_gib_needed: float) -> list:
    """Find instance types suitable for Spot diversification."""
    ec2 = boto3.client("ec2", region_name="us-east-1")
    
    paginator = ec2.get_paginator("describe_instance_types")
    candidates = []
    
    for page in paginator.paginate(
        Filters=[
            {"Name": "vcpu-info.default-vcpus", "Values": [str(vcpus_needed)]},
            {"Name": "supported-usage-class", "Values": ["spot"]},
            {"Name": "current-generation", "Values": ["true"]},
        ]
    ):
        for itype in page["InstanceTypes"]:
            mem_gib = itype["MemoryInfo"]["SizeInMiB"] / 1024
            if mem_gib >= memory_gib_needed:
                candidates.append({
                    "type": itype["InstanceType"],
                    "vcpus": itype["VCpuInfo"]["DefaultVCpus"],
                    "memory_gib": mem_gib,
                })
    
    return sorted(candidates, key=lambda x: x["type"])

# Get 10+ instance types for diversification
candidates = get_spot_recommendations(vcpus_needed=4, memory_gib_needed=16)
print(f"Found {len(candidates)} candidate instance types for diversification")
```

**Diversification checklist:**
- [ ] Minimum 5–10 instance types per ASG/fleet
- [ ] Span multiple instance families (m5, m5a, m5d, m4, m6i)
- [ ] Include multiple sizes (xlarge + 2xlarge with weighted capacity)
- [ ] Deploy across 3+ AZs
- [ ] Use `capacity-optimized` allocation strategy

---

## Spot Savings Calculator

```python
def calculate_spot_savings(
    instance_type: str,
    region: str,
    hours_per_month: float,
    on_demand_price: float,
    spot_price: float,
    interruption_rate_pct: float = 5.0,
) -> dict:
    """Calculate expected Spot savings accounting for interruption overhead."""
    
    # Effective hours (accounting for interruptions and restart time)
    restart_overhead_hours = (interruption_rate_pct / 100) * hours_per_month * 0.1
    effective_hours = hours_per_month + restart_overhead_hours
    
    on_demand_cost = on_demand_price * hours_per_month
    spot_cost = spot_price * effective_hours
    savings = on_demand_cost - spot_cost
    savings_pct = (savings / on_demand_cost) * 100
    
    return {
        "instance_type": instance_type,
        "region": region,
        "on_demand_monthly": round(on_demand_cost, 2),
        "spot_monthly": round(spot_cost, 2),
        "monthly_savings": round(savings, 2),
        "savings_percentage": round(savings_pct, 1),
        "annual_savings": round(savings * 12, 2),
    }

# Example: m5.xlarge in us-east-1
result = calculate_spot_savings(
    instance_type="m5.xlarge",
    region="us-east-1",
    hours_per_month=730,
    on_demand_price=0.192,
    spot_price=0.058,
    interruption_rate_pct=5,
)
print(result)
# {'monthly_savings': 97.44, 'savings_percentage': 69.5, 'annual_savings': 1169.28}
```

---

## Use Cases

| Workload | Spot Suitability | Strategy |
|---|---|---|
| CI/CD build agents | ✅ Excellent | Stateless, fast restart, Jenkins/GitHub Actions |
| Batch data processing | ✅ Excellent | Checkpoint to S3, use Step Functions |
| ML training | ✅ Excellent | SageMaker managed spot or checkpoint-aware code |
| Stateless web tier | ✅ Good | Behind ALB, graceful drain on interruption |
| Stateful databases | ❌ Poor | Use RDS/Aurora with RIs instead |
| Real-time APIs (strict SLA) | ⚠️ Risky | Mix with On-Demand base capacity |
| Long-running jobs (no checkpoint) | ⚠️ Risky | Add checkpointing first |

---

## Decision Matrix: Spot vs On-Demand vs Savings Plans

| Factor | Spot | On-Demand | Savings Plans |
|---|---|---|---|
| Discount | Up to 90% | 0% | 20–66% |
| Commitment | None | None | 1 or 3 years |
| Interruption risk | Yes (2-min notice) | None | None |
| Flexibility | High (any type) | High | Compute SP: high |
| Best for | Fault-tolerant batch/ML | Unpredictable spiky loads | Steady-state baseline |
| Combine with | On-Demand base | Spot for burst | Spot for overflow |

**Recommended architecture:** Savings Plans for baseline (60–70% of compute) + Spot for burst and batch (20–30%) + On-Demand for spiky/unpredictable remainder.
