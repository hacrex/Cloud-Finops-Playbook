# AWS Savings Plans — Comprehensive Guide

Savings Plans are the most flexible commitment-based discount mechanism AWS offers. They apply automatically across eligible usage, making them easier to manage than Reserved Instances.

---

## The Three Types

### 1. Compute Savings Plans
- **Discount:** Up to 66% vs On-Demand
- **Applies to:** EC2 (any instance family, size, region, OS), Lambda, Fargate
- **Flexibility:** Highest — automatically applies wherever you run compute
- **Best for:** Organizations with diverse or evolving compute footprints

### 2. EC2 Instance Savings Plans
- **Discount:** Up to 72% vs On-Demand
- **Applies to:** Specific EC2 instance family in a specific region (any size, OS, tenancy)
- **Flexibility:** Medium — locked to family + region, flexible on size/OS
- **Best for:** Stable workloads on a known instance family (e.g., always running m5 in us-east-1)

### 3. SageMaker Savings Plans
- **Discount:** Up to 64% vs On-Demand
- **Applies to:** SageMaker ML instances (training, hosting, notebooks)
- **Flexibility:** Applies across instance families, sizes, and regions
- **Best for:** Teams with consistent SageMaker usage

| Feature | Compute SP | EC2 Instance SP | SageMaker SP |
|---|---|---|---|
| Max discount | 66% | 72% | 64% |
| Instance family lock | No | Yes | No |
| Region lock | No | Yes | No |
| Covers Lambda/Fargate | Yes | No | No |
| Covers SageMaker | No | No | Yes |

---

## How Savings Plans Apply

AWS applies Savings Plans in a specific billing order to maximize your benefit:

1. **Zonal RIs** (most specific) — applied first
2. **Regional RIs**
3. **EC2 Instance Savings Plans** (more specific)
4. **Compute Savings Plans** (broadest)
5. **On-Demand** — remainder billed at full price

The commitment is measured in **$/hour**. If you commit $10/hour, AWS applies that $10/hour of discount to your most eligible usage automatically — no manual assignment needed.

```
Example:
  Commitment: $5.00/hour Compute SP
  Usage hour:
    - 10x m5.xlarge @ $0.192/hr On-Demand = $1.92
    - 5x c5.2xlarge @ $0.340/hr On-Demand = $1.70
    - 20x Lambda GB-seconds = $0.40
    Total On-Demand equivalent: $4.02
  
  SP covers $4.02 at discounted rates → you pay ~$1.60 instead of $4.02
  Remaining $0.98 of commitment is unused (wasted)
```

---

## Savings Plans vs Reserved Instances

| Dimension | Savings Plans | Reserved Instances |
|---|---|---|
| Commitment unit | $/hour spend | Specific instance |
| Flexibility | High (Compute SP) | Low (Standard RI) |
| Applies to Lambda/Fargate | Yes (Compute SP) | No |
| Marketplace resale | No | Yes (Standard RI) |
| Modification | No | Yes (size/AZ) |
| Convertible exchange | N/A | Yes (Convertible RI) |
| Max discount | 72% | 72% |
| Management overhead | Low | High |
| Best for | Modern, flexible workloads | Specific, stable instances |

**Rule of thumb:** Start with Savings Plans. Only use RIs when you need the RI Marketplace (to sell unused capacity) or when you need the extra 0–5% discount from EC2 Instance SP vs Compute SP.

---

## Purchasing Strategy: Right-Size Before You Commit

Committing to the wrong baseline is the most expensive Savings Plans mistake. Always right-size first.

```python
import boto3
from datetime import datetime, timedelta

def get_rightsizing_recommendations():
    """Pull Compute Optimizer rightsizing recommendations before purchasing SPs."""
    ce = boto3.client("compute-optimizer", region_name="us-east-1")
    
    paginator = ce.get_paginator("get_ec2_instance_recommendations")
    recommendations = []
    
    for page in paginator.paginate():
        for rec in page["instanceRecommendations"]:
            if rec["finding"] in ["OVER_PROVISIONED", "UNDER_PROVISIONED"]:
                current = rec["currentInstanceType"]
                options = rec["recommendationOptions"]
                best = options[0] if options else None
                
                if best:
                    recommendations.append({
                        "instance_id": rec["instanceArn"].split("/")[-1],
                        "current_type": current,
                        "recommended_type": best["instanceType"],
                        "estimated_monthly_savings": best.get(
                            "estimatedMonthlySavings", {}).get("value", 0),
                        "finding": rec["finding"],
                    })
    
    return sorted(recommendations, key=lambda x: x["estimated_monthly_savings"], reverse=True)

recs = get_rightsizing_recommendations()
total_savings = sum(r["estimated_monthly_savings"] for r in recs)
print(f"Right-sizing opportunity: ${total_savings:.2f}/month before SP purchase")
```

---

## Coverage and Utilization Analysis

```python
import boto3
from datetime import datetime, timedelta

def analyze_savings_plans(lookback_days: int = 30) -> dict:
    """Analyze SP coverage and utilization."""
    ce = boto3.client("ce", region_name="us-east-1")
    
    end = datetime.utcnow().date()
    start = end - timedelta(days=lookback_days)
    
    # Coverage: what % of eligible spend is covered by SPs
    coverage = ce.get_savings_plans_coverage(
        TimePeriod={"Start": str(start), "End": str(end)},
        Granularity="MONTHLY",
        Metrics=["SpendCoveredBySavingsPlans"],
    )
    
    # Utilization: what % of your SP commitment is being used
    utilization = ce.get_savings_plans_utilization(
        TimePeriod={"Start": str(start), "End": str(end)},
        Granularity="MONTHLY",
    )
    
    total = utilization["Total"]
    util_pct = float(total["Utilization"]["UtilizationPercentage"])
    net_savings = float(total["Savings"]["NetSavings"])
    unused = float(total["Savings"]["OnDemandCostEquivalent"]) - float(
        total["Savings"]["NetSavings"])
    
    coverage_pct = float(
        coverage["SavingsPlansCoverages"][0]["Coverage"]["CoveragePercentage"]
        if coverage["SavingsPlansCoverages"] else 0
    )
    
    return {
        "coverage_percentage": round(coverage_pct, 1),
        "utilization_percentage": round(util_pct, 1),
        "net_savings_usd": round(net_savings, 2),
        "unused_commitment_usd": round(unused, 2),
        "health": "good" if util_pct >= 95 else "warning" if util_pct >= 80 else "critical",
    }

result = analyze_savings_plans(lookback_days=30)
print(result)
```

**Target thresholds:**
- Utilization ≥ 95% — healthy, commitment is well-matched
- Coverage ≥ 70% — good coverage of eligible spend
- If utilization < 80% — you over-committed, consider not renewing at full amount

---

## Recommendations API

```python
def get_sp_recommendations(
    term: str = "ONE_YEAR",
    payment: str = "NO_UPFRONT",
    lookback: str = "30_DAYS",
    sp_type: str = "COMPUTE_SP",
) -> list:
    """Get AWS-generated SP purchase recommendations."""
    ce = boto3.client("ce", region_name="us-east-1")
    
    resp = ce.get_savings_plans_purchase_recommendation(
        SavingsPlansType=sp_type,
        TermInYears=term,
        PaymentOption=payment,
        LookbackPeriodInDays=lookback,
    )
    
    summary = resp["SavingsPlansPurchaseRecommendationSummary"]
    recs = resp["SavingsPlansPurchaseRecommendations"]
    
    print(f"Estimated annual savings: ${float(summary['EstimatedSavingsAmount']):.2f}")
    print(f"Estimated savings rate: {summary['EstimatedSavingsPercentage']}%")
    print(f"Recommended hourly commitment: ${float(summary['HourlyCommitmentToPurchase']):.4f}/hr")
    
    return recs

# Get recommendations for 1-year Compute SP, No Upfront
recs = get_sp_recommendations(term="ONE_YEAR", payment="NO_UPFRONT")
```

---

## 1-Year vs 3-Year Decision Framework

```
Decision tree:

Is your workload stable for 3+ years?
├── No → 1-year SP
└── Yes → Is the extra discount worth the lock-in risk?
    ├── 3-year discount is ~15% more than 1-year
    ├── Calculate: (extra_savings_3yr - extra_savings_1yr) vs flexibility_value
    └── If workload is core infrastructure (databases, always-on services) → 3-year
        If workload may change (new product, uncertain growth) → 1-year
```

| Term | Compute SP Discount | EC2 Instance SP Discount | Flexibility |
|---|---|---|---|
| 1-year No Upfront | ~40% | ~45% | Renew or change annually |
| 1-year Partial Upfront | ~43% | ~48% | Renew or change annually |
| 1-year All Upfront | ~45% | ~50% | Renew or change annually |
| 3-year No Upfront | ~55% | ~60% | Locked for 3 years |
| 3-year Partial Upfront | ~60% | ~65% | Locked for 3 years |
| 3-year All Upfront | ~66% | ~72% | Locked for 3 years |

---

## Payment Options: ROI Analysis

```python
def sp_payment_roi(
    hourly_commitment: float,
    term_years: int,
    no_upfront_rate: float,
    partial_upfront_rate: float,
    all_upfront_rate: float,
    discount_rate: float = 0.05,  # Annual cost of capital
) -> dict:
    """Compare payment options on NPV basis."""
    hours = term_years * 8760
    
    # No Upfront: pay monthly
    no_upfront_total = hourly_commitment * hours * no_upfront_rate
    
    # Partial Upfront: ~50% upfront + monthly
    partial_upfront_upfront = hourly_commitment * hours * partial_upfront_rate * 0.5
    partial_upfront_monthly = hourly_commitment * hours * partial_upfront_rate * 0.5
    # NPV of monthly payments
    partial_npv = partial_upfront_upfront + partial_upfront_monthly / (
        1 + discount_rate) ** (term_years / 2)
    
    # All Upfront: pay everything now
    all_upfront_total = hourly_commitment * hours * all_upfront_rate
    
    return {
        "no_upfront_total": round(no_upfront_total, 2),
        "partial_upfront_npv": round(partial_npv, 2),
        "all_upfront_total": round(all_upfront_total, 2),
        "partial_vs_no_savings": round(no_upfront_total - partial_npv, 2),
        "all_vs_no_savings": round(no_upfront_total - all_upfront_total, 2),
        "recommendation": (
            "All Upfront" if discount_rate < 0.03
            else "Partial Upfront" if discount_rate < 0.07
            else "No Upfront"
        ),
    }
```

**General guidance:**
- **No Upfront**: Best cash flow, ~5% more expensive. Use when capital is constrained.
- **Partial Upfront**: Best balance for most organizations.
- **All Upfront**: Maximum savings (~5–10% more than No Upfront). Use when you have idle capital.

---

## Monitoring SP Utilization with CloudWatch

```python
import boto3

def create_sp_utilization_alarm(threshold_pct: float = 80.0):
    """Alert when SP utilization drops below threshold."""
    cw = boto3.client("cloudwatch", region_name="us-east-1")
    
    cw.put_metric_alarm(
        AlarmName="SavingsPlans-LowUtilization",
        AlarmDescription=f"SP utilization below {threshold_pct}% — possible over-commitment",
        MetricName="UtilizationPercentage",
        Namespace="AWS/BillingConductor",
        Statistic="Average",
        Period=86400,  # Daily
        EvaluationPeriods=3,
        Threshold=threshold_pct,
        ComparisonOperator="LessThanThreshold",
        AlarmActions=["arn:aws:sns:us-east-1:123456789:finops-alerts"],
        TreatMissingData="notBreaching",
    )
    print(f"Alarm created: SP utilization < {threshold_pct}%")
```

---

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Buying SPs before right-sizing | Committing to over-provisioned baseline | Right-size first, then commit |
| Over-committing (chasing 100% coverage) | Wasted spend on unused commitment | Target 70–80% coverage, use Spot/OD for rest |
| Under-committing (fear of lock-in) | Missing 40–66% discounts | Use 1-year No Upfront to start conservatively |
| Ignoring utilization alerts | Paying for unused commitment | Set CloudWatch alarm at 80% utilization |
| Buying 3-year for new workloads | Locked into wrong commitment | Use 1-year until workload stabilizes |
| Not staggering purchases | Cliff-edge renewals | Spread purchases across months |
| Forgetting SageMaker SP | Missing 64% SageMaker discount | Analyze SageMaker spend separately |
