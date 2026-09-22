# Cloud Budgeting Best Practices

Cloud budgeting is fundamentally different from traditional IT budgeting. In traditional IT, budgets were set annually based on procurement plans. In cloud, costs are variable and real-time — budgets must be dynamic, granular, and connected to automated alerting to be effective.

---

## Bottom-Up vs Top-Down Budgeting

### Top-Down Budgeting

Leadership sets a total cloud budget, which is then allocated down to business units, teams, and projects.

**Process:**
1. CFO/CTO sets annual cloud budget (e.g., $5M)
2. Budget allocated to business units based on headcount, revenue, or historical spend
3. Business units allocate to teams
4. Teams allocate to environments and projects

**Advantages:**
- Aligns with corporate financial planning
- Ensures total spend stays within financial constraints
- Simple to communicate

**Disadvantages:**
- Disconnected from actual workload requirements
- Can under-fund critical infrastructure
- Teams may game allocations

### Bottom-Up Budgeting

Teams estimate their cloud needs based on planned workloads, and these estimates are aggregated into a total budget.

**Process:**
1. Each team estimates cloud costs for planned workloads
2. FinOps team reviews and normalizes estimates
3. Estimates aggregated with growth assumptions
4. Finance reviews and approves total

**Advantages:**
- More accurate — based on actual workload plans
- Teams own their numbers
- Surfaces infrastructure investment needs

**Disadvantages:**
- Time-consuming
- Teams may over-estimate to create buffer
- Requires engineering teams to understand cloud pricing

### Recommended Approach: Hybrid

Use top-down for the total envelope, bottom-up for allocation within that envelope:

```
Step 1: Finance sets total cloud budget envelope ($5M)
Step 2: FinOps team runs bottom-up estimation with all teams
Step 3: Bottom-up total compared to top-down envelope
Step 4: Negotiate adjustments where gaps exist
Step 5: Finalize team-level budgets
Step 6: Review and adjust quarterly
```

---

## Budget Types

### Fixed Budgets

A fixed monthly or annual amount that doesn't change based on usage.

**Best for:**
- Teams with stable, predictable workloads
- Environments with known capacity requirements
- Projects with defined scope and timeline

```hcl
# Terraform: Fixed monthly budget
resource "aws_budgets_budget" "fixed_monthly" {
  name         = "team-platform-fixed-monthly"
  budget_type  = "COST"
  limit_amount = "15000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:team$platform"]
  }
}
```

### Variable / Flexible Budgets

Budgets that adjust based on business drivers (e.g., revenue, user count, transaction volume).

**Best for:**
- SaaS products where cloud cost scales with usage
- E-commerce with seasonal traffic patterns
- Any workload where cost should scale proportionally with business metrics

**Variable budget formula:**
```
Monthly Budget = Base Cost + (Variable Rate × Business Metric)

Example:
  Base Cost = $5,000 (fixed infrastructure)
  Variable Rate = $0.50 per 1,000 MAU
  Business Metric = 20,000 MAU
  Monthly Budget = $5,000 + ($0.50 × 20) = $15,000
```

### Anomaly-Based Budgets

Budgets that alert on statistical anomalies rather than fixed thresholds.

**Best for:**
- Workloads with highly variable but predictable patterns
- Detecting unexpected cost spikes without false positives from normal growth

**AWS Cost Anomaly Detection:**
```python
import boto3

ce = boto3.client('ce')

# Create anomaly monitor for a specific tag
monitor = ce.create_anomaly_monitor(
    AnomalyMonitor={
        'MonitorName': 'TeamPlatformMonitor',
        'MonitorType': 'CUSTOM',
        'MonitorSpecification': json.dumps({
            'Tags': {
                'Key': 'team',
                'Values': ['platform'],
                'MatchOptions': ['EQUALS']
            }
        })
    }
)

# Alert on anomalies > $200 impact
ce.create_anomaly_subscription(
    AnomalySubscription={
        'MonitorArnList': [monitor['MonitorArn']],
        'Subscribers': [
            {'Address': 'platform-team@company.com', 'Type': 'EMAIL'}
        ],
        'Threshold': 200,
        'Frequency': 'IMMEDIATE',
        'SubscriptionName': 'PlatformAnomalyAlert'
    }
)
```

---

## AWS Budgets

AWS Budgets is the native AWS tool for setting cost and usage budgets with automated alerts.

### Budget Types in AWS

| Budget Type | Tracks | Use Case |
|---|---|---|
| Cost budget | Dollar spend | Most common — track total spend |
| Usage budget | Service usage units | Track specific resource consumption |
| RI utilization budget | % of RI used | Ensure commitments are being used |
| RI coverage budget | % of compute covered by RI | Ensure adequate commitment coverage |
| Savings Plans utilization | % of SP used | Ensure SP commitments are used |
| Savings Plans coverage | % of compute covered by SP | Ensure adequate SP coverage |

### AWS Budget Configuration

```hcl
# Terraform: Comprehensive AWS budget setup
resource "aws_budgets_budget" "team_budget" {
  name              = "team-${var.team}-${var.environment}-monthly"
  budget_type       = "COST"
  limit_amount      = var.monthly_limit
  limit_unit        = "USD"
  time_unit         = "MONTHLY"
  time_period_start = "2024-01-01_00:00"

  # Filter to team's resources
  cost_filter {
    name   = "TagKeyValue"
    values = ["user:team$${var.team}"]
  }

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:environment$${var.environment}"]
  }

  # 80% alert — informational
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.team_lead_email]
  }

  # 90% alert — warning
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 90
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.team_lead_email, var.finops_email]
    subscriber_sns_topic_arns  = [var.alerts_topic_arn]
  }

  # 100% alert — exceeded
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.team_lead_email, var.finops_email, var.eng_manager_email]
    subscriber_sns_topic_arns  = [var.alerts_topic_arn]
  }

  # Forecasted 110% alert — proactive warning
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 110
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = [var.team_lead_email, var.finops_email]
  }
}
```

### AWS Budget Actions

AWS Budgets can automatically take actions when thresholds are breached:

```hcl
# Budget action: Apply SCP to restrict spending when budget exceeded
resource "aws_budgets_budget_action" "restrict_on_exceed" {
  budget_name        = aws_budgets_budget.team_budget.name
  action_type        = "APPLY_SCP_POLICY"
  approval_model     = "AUTOMATIC"
  notification_type  = "ACTUAL"

  action_threshold {
    action_threshold_type  = "PERCENTAGE"
    action_threshold_value = 110
  }

  definition {
    scp_action_definition {
      policy_id  = aws_organizations_policy.restrict_new_resources.id
      target_ids = [var.team_account_id]
    }
  }

  subscriber {
    address           = var.finops_email
    subscription_type = "EMAIL"
  }
}
```

---

## Azure Cost Management Budgets

```bash
# Azure CLI: Create a budget for a resource group
az consumption budget create \
  --budget-name "team-platform-monthly" \
  --amount 15000 \
  --time-grain Monthly \
  --start-date 2024-01-01 \
  --end-date 2024-12-31 \
  --resource-group rg-platform-prod \
  --notifications '[
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["platform-lead@company.com"],
      "thresholdType": "Actual"
    },
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "contactEmails": ["platform-lead@company.com", "finops@company.com"],
      "thresholdType": "Actual"
    },
    {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 110,
      "contactEmails": ["platform-lead@company.com"],
      "thresholdType": "Forecasted"
    }
  ]'
```

---

## GCP Budgets

```python
# Python: Create a GCP budget using the Billing Budgets API
from google.cloud import billing_budgets_v1

def create_budget(
    billing_account: str,
    project_id: str,
    budget_amount_usd: float,
    team_label: str
):
    client = billing_budgets_v1.BudgetServiceClient()
    
    budget = billing_budgets_v1.Budget(
        display_name=f"team-{team_label}-monthly",
        budget_filter=billing_budgets_v1.Filter(
            projects=[f"projects/{project_id}"],
            labels={"team": [team_label]},
            calendar_period=billing_budgets_v1.CalendarPeriod.MONTH
        ),
        amount=billing_budgets_v1.BudgetAmount(
            specified_amount=billing_budgets_v1.Money(
                currency_code="USD",
                units=int(budget_amount_usd)
            )
        ),
        threshold_rules=[
            billing_budgets_v1.ThresholdRule(
                threshold_percent=0.8,
                spend_basis=billing_budgets_v1.ThresholdRule.Basis.CURRENT_SPEND
            ),
            billing_budgets_v1.ThresholdRule(
                threshold_percent=1.0,
                spend_basis=billing_budgets_v1.ThresholdRule.Basis.CURRENT_SPEND
            ),
            billing_budgets_v1.ThresholdRule(
                threshold_percent=1.1,
                spend_basis=billing_budgets_v1.ThresholdRule.Basis.FORECASTED_SPEND
            )
        ],
        notifications_rule=billing_budgets_v1.NotificationsRule(
            pubsub_topic=f"projects/{project_id}/topics/budget-alerts",
            monitoring_notification_channels=[
                f"projects/{project_id}/notificationChannels/your-channel-id"
            ]
        )
    )
    
    parent = f"billingAccounts/{billing_account}"
    return client.create_budget(parent=parent, budget=budget)
```

---

## Budget Alert Escalation Policies

A well-designed escalation policy ensures the right people are notified at the right time:

### Escalation Matrix

| Threshold | Alert Type | Recipients | Action Required |
|---|---|---|---|
| 70% actual | Informational | Team lead | Review spend trend |
| 80% actual | Warning | Team lead | Identify optimization opportunities |
| 90% actual | Urgent | Team lead + FinOps | Create optimization plan |
| 100% actual | Critical | Team lead + FinOps + Eng Manager | Immediate review, possible freeze |
| 110% forecasted | Proactive | Team lead + FinOps | Adjust forecast, plan response |
| 120% actual | Executive | VP Eng + Finance | Executive review required |

### Slack Integration for Budget Alerts

```python
# Lambda function: Route budget alerts to Slack
import json
import os
import urllib.request

def lambda_handler(event, context):
    """Route SNS budget alert to appropriate Slack channel."""
    
    message = json.loads(event['Records'][0]['Sns']['Message'])
    
    budget_name = message.get('budgetName', 'Unknown')
    actual_spend = float(message.get('actualSpend', {}).get('amount', 0))
    budget_limit = float(message.get('budgetLimit', {}).get('amount', 0))
    utilization = (actual_spend / budget_limit) * 100 if budget_limit > 0 else 0
    
    # Determine severity and channel
    if utilization >= 100:
        channel = '#finops-critical'
        color = '#FF0000'
        emoji = '🚨'
    elif utilization >= 90:
        channel = '#finops-alerts'
        color = '#FF8C00'
        emoji = '⚠️'
    else:
        channel = '#finops-alerts'
        color = '#FFA500'
        emoji = '📊'
    
    slack_message = {
        "channel": channel,
        "attachments": [{
            "color": color,
            "title": f"{emoji} Budget Alert: {budget_name}",
            "fields": [
                {"title": "Actual Spend", "value": f"${actual_spend:,.2f}", "short": True},
                {"title": "Budget Limit", "value": f"${budget_limit:,.2f}", "short": True},
                {"title": "Utilization", "value": f"{utilization:.1f}%", "short": True},
                {"title": "Remaining", "value": f"${budget_limit - actual_spend:,.2f}", "short": True}
            ],
            "footer": "AWS Budgets | FinOps Platform"
        }]
    }
    
    webhook_url = os.environ['SLACK_WEBHOOK_URL']
    req = urllib.request.Request(
        webhook_url,
        data=json.dumps(slack_message).encode('utf-8'),
        headers={'Content-Type': 'application/json'}
    )
    urllib.request.urlopen(req)
```

---

## Team-Level vs Project-Level Budgets

### Budget Hierarchy Design

```
Organization ($5M annual)
├── Platform Team ($1.2M)
│   ├── Production ($800K)
│   ├── Staging ($200K)
│   └── Development ($200K)
├── Data Team ($1.5M)
│   ├── Production ($1M)
│   ├── ML Training ($300K)
│   └── Development ($200K)
├── Product Teams ($1.8M)
│   ├── Checkout ($600K)
│   ├── Search ($500K)
│   └── Auth ($700K)
└── Shared Services ($500K)
    ├── Monitoring ($200K)
    ├── Security ($150K)
    └── Networking ($150K)
```

### Project Budgets for Time-Bounded Work

For projects with defined timelines (migrations, new feature launches):

```hcl
resource "aws_budgets_budget" "project_budget" {
  name              = "project-migration-2024"
  budget_type       = "COST"
  limit_amount      = "50000"  # Total project budget
  limit_unit        = "USD"
  time_unit         = "QUARTERLY"
  time_period_start = "2024-01-01_00:00"
  time_period_end   = "2024-06-30_23:59"

  cost_filter {
    name   = "TagKeyValue"
    values = ["user:project$migration-2024"]
  }
}
```

---

## Budget vs Actual Tracking

### Monthly Budget Review Template

```markdown
## Cloud Budget Review — [Month Year]

### Summary
| Team | Budget | Actual | Variance | Variance % |
|------|--------|--------|----------|------------|
| Platform | $15,000 | $14,200 | -$800 | -5.3% |
| Data | $25,000 | $27,800 | +$2,800 | +11.2% ⚠️ |
| Backend | $12,000 | $11,500 | -$500 | -4.2% |
| **Total** | **$52,000** | **$53,500** | **+$1,500** | **+2.9%** |

### Variance Analysis
**Data Team (+11.2%):**
- Root cause: ML training job ran 3x longer than estimated
- Action: Implement spot instances for training jobs (saves ~$2,000/month)
- Owner: Data team lead
- Due date: [Date]

### Next Month Forecast
| Team | Forecast | Budget | Expected Variance |
|------|----------|--------|-------------------|
| Platform | $14,500 | $15,000 | -3.3% |
| Data | $24,000 | $25,000 | -4.0% |
| Backend | $11,800 | $12,000 | -1.7% |
```

---

## Handling Budget Overruns

### Response Playbook

**Tier 1: Minor overrun (< 10% over budget)**
1. Identify root cause (new resource? traffic spike? pricing change?)
2. Determine if overrun is one-time or recurring
3. If recurring: create optimization plan with 30-day timeline
4. Document in monthly budget review

**Tier 2: Significant overrun (10–25% over budget)**
1. Immediate root cause analysis (within 24 hours)
2. Implement quick wins to reduce spend (idle resources, rightsizing)
3. Escalate to engineering manager
4. Adjust next month's forecast
5. Consider budget reallocation from underspending teams

**Tier 3: Major overrun (> 25% over budget)**
1. Immediate escalation to VP Engineering and Finance
2. Freeze non-critical resource provisioning
3. Emergency optimization sprint
4. Root cause report within 48 hours
5. Budget reforecast for remainder of year

### Budget Freeze Implementation

```python
# Automated budget freeze: Apply restrictive SCP when budget exceeded
import boto3
import json

def apply_budget_freeze(account_id: str, team: str):
    """Apply a restrictive SCP to an account when budget is exceeded."""
    
    orgs = boto3.client('organizations')
    
    # Create restrictive policy
    policy_document = {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "DenyNewResourceCreation",
                "Effect": "Deny",
                "Action": [
                    "ec2:RunInstances",
                    "rds:CreateDBInstance",
                    "elasticache:CreateCacheCluster"
                ],
                "Resource": "*",
                "Condition": {
                    "StringNotEquals": {
                        "aws:RequestedRegion": []  # Block all regions
                    }
                }
            }
        ]
    }
    
    policy = orgs.create_policy(
        Content=json.dumps(policy_document),
        Description=f"Budget freeze for team {team}",
        Name=f"budget-freeze-{team}",
        Type="SERVICE_CONTROL_POLICY"
    )
    
    orgs.attach_policy(
        PolicyId=policy['Policy']['PolicySummary']['Id'],
        TargetId=account_id
    )
    
    print(f"Budget freeze applied to account {account_id} for team {team}")
```
