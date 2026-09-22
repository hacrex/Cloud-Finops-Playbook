# The FinOps Lifecycle

The FinOps lifecycle is a continuous, iterative loop across three phases: **Inform → Optimize → Operate**. Unlike a project with a start and end date, FinOps is a perpetual practice. Organizations cycle through these phases continuously, with each iteration building on the last.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   INFORM    │────▶│  OPTIMIZE   │────▶│   OPERATE   │
│             │     │             │     │             │
│ Visibility  │     │ Efficiency  │     │ Governance  │
│ Allocation  │     │ Rightsizing │     │ Budgets     │
│ Dashboards  │     │ Commitments │     │ Forecasting │
└─────────────┘     └─────────────┘     └─────────────┘
       ▲                                       │
       └───────────────────────────────────────┘
                  Continuous iteration
```

---

## Phase 1: Inform

The Inform phase is about **making cloud costs visible, accurate, and understandable**. You cannot optimize what you cannot see, and you cannot govern what you cannot measure.

### Cost Visibility

The foundation of the Inform phase is getting raw cost data into a usable form.

**Data sources:**
- **AWS:** Cost and Usage Report (CUR) — the most granular billing data available
- **Azure:** Cost Management exports to Storage Account or Event Hub
- **GCP:** Billing export to BigQuery

**CUR setup (AWS):**
```hcl
# Terraform: Enable AWS Cost and Usage Report
resource "aws_cur_report_definition" "main" {
  report_name                = "cur-hourly-parquet"
  time_unit                  = "HOURLY"
  format                     = "Parquet"
  compression                = "Parquet"
  additional_schema_elements = ["RESOURCES", "SPLIT_COST_ALLOCATION_DATA"]
  s3_bucket                  = aws_s3_bucket.cur_bucket.bucket
  s3_region                  = "us-east-1"
  s3_prefix                  = "cur"
  report_versioning          = "OVERWRITE_REPORT"
  refresh_closed_reports     = true
}
```

### Tagging and Cost Allocation

Tags are the mechanism that transforms raw billing data into business-meaningful cost allocation.

**Mandatory tag taxonomy:**

| Tag Key | Example Values | Purpose |
|---|---|---|
| `team` | platform, data, frontend | Team ownership |
| `product` | checkout, search, auth | Product/service |
| `environment` | prod, staging, dev | Environment |
| `cost-center` | CC-1234 | Finance allocation |
| `project` | migration-2024 | Project tracking |

**Tag enforcement with AWS Config:**
```json
{
  "ConfigRuleName": "required-tags",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "REQUIRED_TAGS"
  },
  "InputParameters": "{\"tag1Key\":\"team\",\"tag2Key\":\"environment\",\"tag3Key\":\"product\"}"
}
```

**Untagged spend handling:**
- Set up automated alerts when untagged spend exceeds a threshold
- Use AWS Tag Editor / Azure Tag Policies to bulk-tag existing resources
- Implement "tag or terminate" policies for resources older than 7 days without tags

### Dashboards

Effective dashboards serve different audiences with different needs.

**Executive dashboard (weekly):**
- Total cloud spend vs budget
- Month-over-month trend
- Top 5 cost drivers
- Efficiency score (cost per business unit)

**Engineering team dashboard (daily):**
- Team spend by service
- Budget utilization %
- Cost anomalies
- Top 10 most expensive resources

**FinOps practitioner dashboard (real-time):**
- Spend by account, region, service
- RI/SP coverage and utilization
- Untagged spend %
- Anomaly feed

**Tooling options:**

| Tool | Best For | Cost |
|---|---|---|
| AWS Cost Explorer | AWS-native, quick setup | Free |
| AWS CUDOS | Deep AWS analysis | Free (Athena/QuickSight costs) |
| Azure Cost Management | Azure-native | Free |
| GCP Billing + Looker Studio | GCP-native | Free |
| CloudHealth | Multi-cloud, enterprise | Paid |
| Apptio Cloudability | Enterprise FinOps | Paid |
| Grafana + CUR | Custom, engineering-friendly | Open source |

### Anomaly Detection

Cost anomalies — unexpected spikes or drops in spending — need to be caught within hours, not weeks.

**AWS Cost Anomaly Detection:**
```python
import boto3

ce = boto3.client('ce', region_name='us-east-1')

# Monitor by linked account
monitor = ce.create_anomaly_monitor(
    AnomalyMonitor={
        'MonitorName': 'LinkedAccountMonitor',
        'MonitorType': 'DIMENSIONAL',
        'MonitorDimension': 'LINKED_ACCOUNT'
    }
)

# Alert when anomaly impact > $500
ce.create_anomaly_subscription(
    AnomalySubscription={
        'MonitorArnList': [monitor['MonitorArn']],
        'Subscribers': [
            {'Address': 'finops@company.com', 'Type': 'EMAIL'},
            {'Address': 'arn:aws:sns:us-east-1:123456789012:finops-alerts', 'Type': 'SNS'}
        ],
        'ThresholdExpression': {
            'Dimensions': {
                'Key': 'ANOMALY_TOTAL_IMPACT_ABSOLUTE',
                'MatchOptions': ['GREATER_THAN_OR_EQUAL'],
                'Values': ['500']
            }
        },
        'Frequency': 'IMMEDIATE',
        'SubscriptionName': 'HighImpactAnomalyAlert'
    }
)
```

### Inform Phase KPIs

| KPI | Definition | Target |
|---|---|---|
| Cost allocation coverage | % of spend tagged to a team/product | > 95% |
| Tag compliance rate | % of resources with all mandatory tags | > 98% |
| Anomaly detection latency | Time from anomaly start to alert | < 4 hours |
| Dashboard adoption | % of engineers accessing cost dashboards monthly | > 60% |
| Unallocated spend | $ amount not attributed to any team | < 5% of total |

---

## Phase 2: Optimize

The Optimize phase uses the visibility from Inform to **reduce waste and improve cost efficiency** without sacrificing performance or reliability.

### Rightsizing

Rightsizing means matching resource capacity to actual workload requirements.

**Rightsizing workflow:**
1. Collect 2–4 weeks of CPU, memory, and network utilization data
2. Identify resources with consistently low utilization (< 20% CPU average)
3. Recommend a smaller instance type or configuration
4. Test in non-production, then apply to production
5. Monitor for performance regression

**AWS Compute Optimizer:**
```bash
# Get rightsizing recommendations for EC2
aws compute-optimizer get-ec2-instance-recommendations \
  --filters name=Finding,values=OVER_PROVISIONED \
  --query 'instanceRecommendations[*].{
    Instance:instanceArn,
    CurrentType:currentInstanceType,
    RecommendedType:recommendationOptions[0].instanceType,
    MonthlySavings:recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table
```

**Rightsizing targets by resource type:**

| Resource | Utilization Signal | Action |
|---|---|---|
| EC2 / VM | CPU < 10%, Mem < 20% | Downsize or terminate |
| RDS / Database | CPU < 10%, connections < 20% | Downsize instance class |
| Kubernetes pods | CPU request >> CPU usage | Reduce requests/limits |
| Storage volumes | < 20% used | Resize or delete |
| Load balancers | < 100 req/day | Consolidate or delete |

### Commitment Discounts

Commitment discounts (Reserved Instances, Savings Plans, Committed Use Discounts) are the highest-ROI optimization available — typically 30–72% savings with no architectural changes.

**Commitment strategy:**

```
Baseline (stable) compute → Savings Plans / Reserved Instances (1-year)
Variable compute          → On-demand
Batch / fault-tolerant    → Spot / Preemptible
```

**Coverage analysis:**
```bash
# AWS: Check Savings Plans coverage
aws ce get-savings-plans-coverage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity MONTHLY \
  --query 'SavingsPlansCoverages[*].{
    Coverage:Coverage.CoveragePercentage,
    OnDemandCost:Coverage.OnDemandCost,
    SpendCoveredBySavingsPlans:Coverage.SpendCoveredBySavingsPlans
  }'
```

### Waste Elimination

Waste is the easiest money to recover. Common waste categories:

| Waste Type | Detection Method | Typical Savings |
|---|---|---|
| Idle EC2 instances | CPU < 1% for 14 days | 100% of instance cost |
| Unattached EBS volumes | No attachment for 7+ days | 100% of volume cost |
| Unused Elastic IPs | Not associated with running instance | $3.65/IP/month |
| Old snapshots | Age > 90 days, no policy | Varies |
| Unused load balancers | 0 healthy targets | 100% of LB cost |
| Oversized NAT Gateways | Low data transfer | Varies |
| Unused RDS instances | 0 connections for 7+ days | 100% of DB cost |

**Automated waste detection script:**
```python
import boto3
from datetime import datetime, timedelta

ec2 = boto3.client('ec2')
cloudwatch = boto3.client('cloudwatch')

def find_idle_instances():
    instances = ec2.describe_instances(
        Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
    )
    
    idle = []
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            instance_id = instance['InstanceId']
            
            # Check average CPU over last 14 days
            metrics = cloudwatch.get_metric_statistics(
                Namespace='AWS/EC2',
                MetricName='CPUUtilization',
                Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
                StartTime=datetime.utcnow() - timedelta(days=14),
                EndTime=datetime.utcnow(),
                Period=86400,
                Statistics=['Average']
            )
            
            if metrics['Datapoints']:
                avg_cpu = sum(d['Average'] for d in metrics['Datapoints']) / len(metrics['Datapoints'])
                if avg_cpu < 1.0:
                    idle.append({'InstanceId': instance_id, 'AvgCPU': avg_cpu})
    
    return idle
```

### Architecture Optimization

Beyond rightsizing and waste, architectural changes can deliver step-change cost reductions.

**High-impact architectural patterns:**
- **Serverless migration:** Move event-driven workloads from always-on EC2 to Lambda/Cloud Run
- **Storage tiering:** Move infrequently accessed data to cheaper storage tiers (S3 Glacier, Azure Cool)
- **Data transfer optimization:** Use VPC endpoints, CDNs, and regional data locality to reduce egress costs
- **Database right-sizing:** Move from RDS to Aurora Serverless for variable workloads
- **Caching:** Add ElastiCache/Redis to reduce database query costs

### Optimize Phase KPIs

| KPI | Definition | Target |
|---|---|---|
| Rightsizing savings | Monthly $ saved from rightsizing | Growing |
| Waste elimination | Monthly $ recovered from waste | Trending to zero |
| RI/SP coverage | % of eligible compute covered | > 70% |
| RI/SP utilization | % of purchased commitments used | > 90% |
| Spot adoption | % of eligible workloads on spot | > 40% |

---

## Phase 3: Operate

The Operate phase **embeds cost management into day-to-day engineering and business processes**. This is where FinOps becomes self-sustaining rather than a periodic exercise.

### Budget Management

Budgets create accountability and early warning systems.

**Budget hierarchy:**
```
Organization Budget
├── Business Unit Budget
│   ├── Team Budget
│   │   ├── Production Environment Budget
│   │   ├── Staging Environment Budget
│   │   └── Development Environment Budget
│   └── Shared Services Budget
└── Platform/Infrastructure Budget
```

**Budget alert escalation policy:**

| Alert Threshold | Action | Recipient |
|---|---|---|
| 80% of budget | Informational notification | Team lead |
| 90% of budget | Warning — review required | Team lead + FinOps |
| 100% of budget | Budget exceeded — escalate | Engineering manager + Finance |
| 120% of budget | Critical — executive notification | VP Engineering + CFO |

### Forecasting

Accurate forecasting enables proactive cost management rather than reactive firefighting.

**Forecasting methods:**

| Method | Best For | Accuracy |
|---|---|---|
| Trend-based | Stable, growing workloads | ±10–15% |
| Driver-based | Workloads tied to business metrics | ±5–10% |
| ML-based | Complex, seasonal patterns | ±5–8% |

**Driver-based forecast example:**
```python
# Cost forecast based on user growth
def forecast_cost(current_cost, current_users, projected_users, efficiency_improvement=0.05):
    """
    Project future cost based on user growth and efficiency improvements.
    
    Args:
        current_cost: Current monthly cost ($)
        current_users: Current monthly active users
        projected_users: Projected monthly active users
        efficiency_improvement: Expected cost efficiency gain per month (default 5%)
    """
    cost_per_user = current_cost / current_users
    raw_projected_cost = cost_per_user * projected_users
    
    # Apply efficiency improvement (FinOps optimization effect)
    optimized_cost = raw_projected_cost * (1 - efficiency_improvement)
    
    return {
        'cost_per_user': cost_per_user,
        'raw_projected_cost': raw_projected_cost,
        'optimized_projected_cost': optimized_cost,
        'efficiency_savings': raw_projected_cost - optimized_cost
    }

# Example: 20% user growth, 5% efficiency improvement
result = forecast_cost(
    current_cost=100000,
    current_users=50000,
    projected_users=60000,
    efficiency_improvement=0.05
)
```

### Governance

Governance policies prevent cost problems before they occur.

**Policy categories:**

| Policy Type | Example | Enforcement |
|---|---|---|
| Tagging | All resources must have team tag | AWS Config, Azure Policy |
| Instance types | No GPU instances in dev | Service Control Policy (SCP) |
| Regions | Resources only in approved regions | SCP / Azure Policy |
| Budget gates | Deployments blocked if budget exceeded | CI/CD pipeline check |
| Idle resource cleanup | Auto-terminate instances idle > 7 days | Lambda + CloudWatch Events |

**SCP example — restrict expensive instance types in dev:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyExpensiveInstancesInDev",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringLike": {
          "ec2:InstanceType": ["p3.*", "p4.*", "x1.*", "x2.*"]
        },
        "StringEquals": {
          "aws:ResourceTag/environment": "dev"
        }
      }
    }
  ]
}
```

### Automation

Mature FinOps practices automate repetitive cost management tasks.

**Automation opportunities:**

| Task | Automation Approach | Savings |
|---|---|---|
| Dev environment shutdown | Lambda + EventBridge schedule | 60–70% of dev costs |
| Snapshot cleanup | Lambda + lifecycle policies | Storage cost reduction |
| RI purchase recommendations | AWS Cost Explorer API + approval workflow | Commitment optimization |
| Tag remediation | Lambda triggered by Config rule | Improved allocation |
| Rightsizing execution | Compute Optimizer + SSM automation | Compute cost reduction |

### Operate Phase KPIs

| KPI | Definition | Target |
|---|---|---|
| Budget variance | Actual vs forecasted spend | < ±10% |
| Forecast accuracy | Forecast vs actual (30-day) | < ±10% |
| Policy compliance | % of resources meeting governance policies | > 95% |
| Automation coverage | % of optimization tasks automated | Growing |
| Time to remediate anomaly | From detection to resolution | < 48 hours |

---

## How the Phases Interact

The three phases are not sequential — they run **concurrently and feed each other**:

```
Inform feeds Optimize:
  New cost visibility reveals rightsizing opportunities

Optimize feeds Operate:
  Optimization results inform budget adjustments and forecasts

Operate feeds Inform:
  Governance policies generate new tagging requirements
  Budget alerts trigger new dashboard views

Operate feeds Optimize:
  Anomaly detection triggers optimization investigations
```

**Maturity progression:**

| Maturity Level | Inform | Optimize | Operate |
|---|---|---|---|
| Crawl | Basic cost reports | Manual rightsizing | Manual budgets |
| Walk | Tagged allocation, dashboards | RI/SP purchases, waste cleanup | Automated alerts, forecasting |
| Run | Real-time anomaly detection | Continuous automated optimization | Policy-as-code, full automation |

---

## Team Responsibilities by Phase

| Phase | FinOps Team | Engineering | Finance | Product |
|---|---|---|---|---|
| Inform | Build dashboards, maintain taxonomy | Tag resources, review team costs | Validate allocations | Review product costs |
| Optimize | Identify opportunities, manage commitments | Execute rightsizing, adopt spot | Approve commitment purchases | Prioritize optimization work |
| Operate | Manage budgets, run governance | Respond to alerts, fix anomalies | Budget planning, variance analysis | Set unit cost targets |
