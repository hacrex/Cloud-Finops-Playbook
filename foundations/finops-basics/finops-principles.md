# FinOps Principles

The FinOps Foundation defines six core principles that guide a healthy FinOps practice. These principles are not aspirational slogans — they are operational commitments that shape how teams make decisions about cloud spending every day.

---

## Principle 1: Teams Need to Collaborate

### What It Means

Cloud cost management cannot be owned by a single team. Finance cannot optimize what they don't build. Engineering cannot budget what they don't understand financially. FinOps only works when finance, engineering, product, and business teams operate with shared context and shared accountability.

Collaboration means:
- Engineers understand the financial impact of their architectural choices
- Finance understands the technical constraints that drive costs
- Product managers connect features to their cost basis
- All teams use a common language for cloud economics

### How to Implement

**Structural changes:**
- Create a FinOps working group with representatives from engineering, finance, and product
- Hold monthly cloud cost reviews with cross-functional attendance
- Embed cost review into sprint retrospectives and architecture reviews
- Assign a FinOps champion in each engineering team

**Tooling:**
- Shared cost dashboards accessible to all teams (not just finance)
- Slack/Teams integrations that push cost anomalies to engineering channels
- PR-level cost estimation (Infracost, Terraform cost estimation)

**Process:**
- Include cost impact in architecture decision records (ADRs)
- Add "cost review" as a checklist item in PR templates
- Share monthly cost reports in engineering all-hands

### Common Pitfalls

- **Finance owns the dashboard, engineering never sees it** — cost data must be pushed to engineers, not pulled by finance
- **Blame culture** — cost overruns become finger-pointing exercises instead of learning opportunities
- **Siloed optimization** — finance buys Reserved Instances without telling engineering, engineering changes instance types without telling finance

### Success Metrics

| Metric | Target |
|---|---|
| % of engineering teams with cost visibility | > 90% |
| Cross-functional FinOps meeting attendance | > 80% |
| Time from cost anomaly to engineering awareness | < 24 hours |
| % of ADRs with cost impact section | > 75% |

---

## Principle 2: Everyone Takes Ownership of Their Cloud Usage

### What It Means

In a FinOps culture, cost ownership is **distributed**, not centralized. The team that provisions a resource is responsible for its cost. This is a fundamental shift from traditional IT, where a central ops team owned all infrastructure costs.

Ownership means:
- Each team can see their own costs in real time
- Teams are accountable for staying within their budgets
- Engineers consider cost as a first-class concern alongside performance and reliability

### How to Implement

**Tagging and allocation:**
- Every resource must be tagged with team, product, environment, and cost center
- Enforce tagging at provisioning time via IaC policies (OPA, Sentinel, Azure Policy)
- Implement automated tagging for resources that cannot be manually tagged

**Showback and chargeback:**
- Start with **showback**: share costs with teams for awareness, no financial consequences
- Evolve to **chargeback**: actual cost allocation to team budgets
- Use internal pricing to make costs tangible (e.g., "your Kubernetes namespace cost $12,400 last month")

**Team-level budgets:**
```yaml
# Example: AWS Budget per team
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "team-platform-monthly",
    "BudgetLimit": {"Amount": "15000", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "TagKeyValue": ["user:team$platform"]
    }
  }'
```

### Common Pitfalls

- **No tagging = no ownership** — without accurate tags, you cannot allocate costs to teams
- **Ownership without authority** — teams cannot be held accountable for costs they cannot control
- **Punitive chargeback** — charging teams for shared infrastructure they don't control breeds resentment

### Success Metrics

| Metric | Target |
|---|---|
| % of spend allocated to a team/product | > 95% |
| Tag compliance rate | > 98% |
| % of teams with active budgets | 100% |
| % of teams reviewing their costs monthly | > 85% |

---

## Principle 3: A Centralized Team Drives FinOps

### What It Means

While ownership is distributed, **coordination is centralized**. A dedicated FinOps team (or FinOps practitioner in smaller organizations) is responsible for:
- Maintaining the FinOps framework and tooling
- Producing consistent, accurate cost reporting
- Driving optimization initiatives
- Training and enabling other teams
- Managing commitment purchases (Reserved Instances, Savings Plans)

This is the "hub and spoke" model: a central FinOps hub enables distributed team spokes.

### How to Implement

**Team structure options:**

| Organization Size | FinOps Structure |
|---|---|
| < 50 engineers | 1 FinOps practitioner (often part-time) |
| 50–200 engineers | 1–2 dedicated FinOps practitioners |
| 200–1000 engineers | FinOps team of 3–5 |
| > 1000 engineers | FinOps Center of Excellence (CoE) |

**Central team responsibilities:**
- Own the cost allocation taxonomy and tagging standards
- Manage cloud provider accounts and billing data pipelines
- Produce and distribute monthly cost reports
- Manage Reserved Instance and Savings Plan portfolio
- Run the FinOps working group
- Maintain FinOps tooling (CUDOS, CloudHealth, Apptio, etc.)

**Governance without gatekeeping:**
The central team should be an **enabler**, not a bottleneck. Avoid creating a situation where every cost decision requires central approval.

### Common Pitfalls

- **FinOps as a side job** — assigning FinOps responsibilities to someone already at 100% capacity
- **Central team as cost police** — creates adversarial relationships with engineering
- **No executive sponsorship** — FinOps initiatives stall without leadership support

### Success Metrics

| Metric | Target |
|---|---|
| Time to produce monthly cost report | < 3 business days |
| RI/SP coverage rate | > 70% |
| RI/SP utilization rate | > 90% |
| Number of teams actively engaged with FinOps | Growing month-over-month |

---

## Principle 4: Reports Should Be Accessible and Timely

### What It Means

Cost data has a short shelf life. A report showing last month's costs, delivered two weeks into the current month, is nearly useless for operational decisions. FinOps requires **near-real-time** cost visibility delivered to the people who can act on it.

Accessible means:
- Available to engineers, not just finance
- Understandable without a finance degree
- Actionable — showing what to do, not just what happened

Timely means:
- Daily cost data (not monthly)
- Anomaly alerts within hours
- Forecasts updated continuously

### How to Implement

**Dashboard tiers:**

| Audience | Refresh Rate | Key Metrics |
|---|---|---|
| Executive | Weekly | Total spend, budget variance, efficiency trend |
| FinOps team | Daily | Cost by service, anomalies, optimization opportunities |
| Engineering team | Daily | Team spend, cost per service, budget status |
| Individual engineer | On-demand | Resource-level costs, tag compliance |

**Tooling options:**
- **AWS:** Cost Explorer, CUDOS (Cost and Usage Dashboard for Operations), CUR + Athena + QuickSight
- **Azure:** Cost Management + Power BI
- **GCP:** Billing export to BigQuery + Looker Studio
- **Multi-cloud:** CloudHealth, Apptio Cloudability, Spot.io Eco

**Anomaly detection setup:**
```python
# Example: AWS Cost Anomaly Detection via boto3
import boto3

ce = boto3.client('ce')

# Create anomaly monitor for a specific service
response = ce.create_anomaly_monitor(
    AnomalyMonitor={
        'MonitorName': 'EC2AnomalyMonitor',
        'MonitorType': 'DIMENSIONAL',
        'MonitorDimension': 'SERVICE'
    }
)

# Create subscription for alerts
ce.create_anomaly_subscription(
    AnomalySubscription={
        'MonitorArnList': [response['MonitorArn']],
        'Subscribers': [{'Address': 'finops-team@company.com', 'Type': 'EMAIL'}],
        'Threshold': 100,  # Alert on anomalies > $100
        'Frequency': 'DAILY',
        'SubscriptionName': 'DailyAnomalyAlert'
    }
)
```

### Common Pitfalls

- **Monthly-only reporting** — by the time the report arrives, the damage is done
- **Finance-only access** — engineers need cost data to make better decisions
- **Vanity metrics** — reporting total spend without context (vs budget, vs last month, vs unit cost)

### Success Metrics

| Metric | Target |
|---|---|
| Cost data latency | < 24 hours |
| Time to detect cost anomaly | < 4 hours |
| % of engineers with dashboard access | > 90% |
| Dashboard active users / total engineers | > 60% |

---

## Principle 5: Decisions Are Driven by Business Value of Cloud

### What It Means

The goal of FinOps is not to minimize cloud spend — it is to **maximize the business value delivered per dollar spent**. Sometimes the right decision is to spend more. A $50K/month increase in cloud spend that enables $500K/month in new revenue is an excellent investment.

This principle reframes the conversation from "how do we cut costs?" to "are we getting the right return on our cloud investment?"

Business value framing:
- Cost per transaction, cost per user, cost per API call
- Cloud spend as a % of revenue
- Cost efficiency trend (same output for less cost over time)
- Time-to-market enabled by cloud elasticity

### How to Implement

**Unit economics framework:**

Define your business unit and track cost against it:

```
Cost per active user = Total cloud spend / Monthly active users
Cost per transaction = Total cloud spend / Monthly transactions
Cost per GB processed = Total cloud spend / GB of data processed
```

**Value-based decision framework:**

Before approving or rejecting a cost optimization, ask:
1. What business outcome does this resource support?
2. What is the cost of the alternative (on-prem, different architecture)?
3. What is the risk of the optimization (performance, reliability)?
4. What is the payback period?

**Cloud spend as % of revenue benchmark:**

| Industry | Cloud Spend / Revenue |
|---|---|
| SaaS (early stage) | 15–25% |
| SaaS (mature) | 8–15% |
| E-commerce | 2–5% |
| Financial services | 3–8% |
| Media/streaming | 10–20% |

### Common Pitfalls

- **Cost cutting for its own sake** — optimizing away infrastructure that supports revenue-generating features
- **No unit economics** — tracking total spend without connecting it to business output
- **Ignoring opportunity cost** — the cost of *not* using cloud (slower time-to-market, less reliability)

### Success Metrics

| Metric | Target |
|---|---|
| Cloud spend as % of revenue | Trending down over time |
| Cost per primary business unit | Trending down over time |
| % of optimization decisions with ROI analysis | > 80% |

---

## Principle 6: Take Advantage of the Variable Cost Model of the Cloud

### What It Means

Cloud's greatest financial advantage is that you only pay for what you use. FinOps maximizes this advantage by:
- Scaling down (or off) resources when not needed
- Using spot/preemptible instances for fault-tolerant workloads
- Purchasing commitments strategically to get discounts without sacrificing flexibility
- Designing architectures that scale to zero

This principle is about **embracing elasticity** as a financial tool, not just an operational one.

### How to Implement

**Elasticity patterns:**

```hcl
# Terraform: Auto Scaling Group with scheduled scaling
resource "aws_autoscaling_schedule" "scale_down_nights" {
  scheduled_action_name  = "scale-down-nights"
  min_size               = 0
  max_size               = 2
  desired_capacity       = 1
  recurrence             = "0 20 * * MON-FRI"  # 8 PM weekdays
  autoscaling_group_name = aws_autoscaling_group.app.name
}

resource "aws_autoscaling_schedule" "scale_up_mornings" {
  scheduled_action_name  = "scale-up-mornings"
  min_size               = 2
  max_size               = 10
  desired_capacity       = 4
  recurrence             = "0 7 * * MON-FRI"  # 7 AM weekdays
  autoscaling_group_name = aws_autoscaling_group.app.name
}
```

**Commitment strategy:**
- Use **on-demand** for unpredictable, spiky workloads
- Use **Savings Plans / Reserved Instances** for stable baseline (60–70% of compute)
- Use **Spot / Preemptible** for batch, CI/CD, and fault-tolerant workloads (up to 90% discount)

**Serverless for variable workloads:**
- AWS Lambda, Azure Functions, GCP Cloud Run
- Pay only for actual invocations — true pay-per-use
- Ideal for event-driven, unpredictable traffic patterns

### Common Pitfalls

- **Over-committing** — buying too many Reserved Instances for workloads that change
- **Ignoring spot** — leaving 60–90% savings on the table for batch workloads
- **Static infrastructure** — running dev/test environments 24/7 when they're only used 8 hours/day

### Success Metrics

| Metric | Target |
|---|---|
| Spot/preemptible instance usage for eligible workloads | > 50% |
| Dev/test environment uptime during off-hours | < 20% |
| Savings Plan / RI coverage for stable compute | > 70% |
| Savings Plan / RI utilization | > 90% |

---

## Applying the Principles Together

The six principles are mutually reinforcing. A team that collaborates (P1) and takes ownership (P2), supported by a central team (P3) with timely data (P4), making value-based decisions (P5) while leveraging cloud elasticity (P6) — that is a mature FinOps organization.

Use this maturity checklist to assess your current state:

| Principle | Crawl | Walk | Run |
|---|---|---|---|
| Collaborate | Ad-hoc cost conversations | Monthly cross-functional reviews | Cost embedded in all engineering processes |
| Ownership | Central team owns costs | Teams see their costs | Teams own and act on their costs |
| Centralized team | No dedicated FinOps | Part-time FinOps practitioner | Dedicated FinOps CoE |
| Timely reports | Monthly reports | Weekly dashboards | Real-time alerts and daily dashboards |
| Business value | Cost reduction focus | Some unit economics | Full unit economics, value-based decisions |
| Variable model | On-demand only | Some commitments | Optimized commitment + spot portfolio |
