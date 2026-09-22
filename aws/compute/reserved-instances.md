# AWS Reserved Instances — Complete Guide

Reserved Instances (RIs) provide significant discounts (up to 72%) in exchange for a 1- or 3-year commitment to a specific instance configuration. While Savings Plans have largely superseded RIs for new purchases, RIs remain relevant for specific use cases — particularly when you need the RI Marketplace or maximum discount on a stable, known instance type.

---

## Standard vs Convertible RIs

| Feature | Standard RI | Convertible RI |
|---|---|---|
| Max discount (3yr All Upfront) | 72% | 66% |
| Modify instance size | Yes (same family) | Yes |
| Change instance family | No | Yes (via exchange) |
| Change OS | No | Yes (via exchange) |
| Change tenancy | No | Yes (via exchange) |
| RI Marketplace (sell unused) | Yes | No |
| Best for | Known, stable instance type | Uncertain future needs |

**When to choose Standard:** You're confident in the instance family and region for 3 years (e.g., always running m5 in us-east-1 for your database tier).

**When to choose Convertible:** You expect instance family changes (e.g., migrating from x86 to Graviton, or unsure about future sizing).

---

## Regional vs Zonal RIs

| Feature | Regional RI | Zonal RI |
|---|---|---|
| AZ flexibility | Any AZ in region | Specific AZ only |
| Capacity reservation | No | Yes (guaranteed capacity) |
| Instance size flexibility | Yes (same family) | No (exact size) |
| Applies to | Any matching instance in region | Exact AZ + size match |
| Best for | Most workloads | When capacity guarantee is critical |

**Recommendation:** Use Regional RIs unless you specifically need capacity reservation (e.g., for compliance or disaster recovery scenarios).

---

## RI Pricing Tiers and Discount Percentages

Approximate discounts for Linux/UNIX, us-east-1 (vary by instance type):

| Payment Option | 1-Year Discount | 3-Year Discount |
|---|---|---|
| No Upfront | ~30–40% | ~45–55% |
| Partial Upfront | ~35–45% | ~50–60% |
| All Upfront | ~38–48% | ~55–72% |

```bash
# Get current RI pricing for a specific instance type
aws pricing get-products \
  --service-code AmazonEC2 \
  --filters \
    "Type=TERM_MATCH,Field=instanceType,Value=m5.xlarge" \
    "Type=TERM_MATCH,Field=location,Value=US East (N. Virginia)" \
    "Type=TERM_MATCH,Field=operatingSystem,Value=Linux" \
    "Type=TERM_MATCH,Field=tenancy,Value=Shared" \
    "Type=TERM_MATCH,Field=preInstalledSw,Value=NA" \
  --region us-east-1 \
  --query 'PriceList[0]' \
  --output text | python -m json.tool | grep -A5 "Reserved"
```

---

## RI Marketplace: Buying and Selling

The RI Marketplace lets you sell unused Standard RIs or buy shorter-term RIs from other customers.

**Selling unused RIs:**
- List via Console → EC2 → Reserved Instances → Sell Reserved Instances
- AWS takes a 12% service fee on the sale price
- Useful when you've over-committed or are migrating away from an instance type
- Only Standard RIs can be sold (not Convertible)

**Buying from Marketplace:**
- Can find RIs with < 1 year remaining — useful for short-term commitments
- Prices set by sellers, often competitive
- Good for testing RI strategy before full commitment

```python
import boto3

def list_marketplace_offerings(instance_type: str, region: str = "us-east-1") -> list:
    """Browse RI Marketplace offerings for a specific instance type."""
    ec2 = boto3.client("ec2", region_name=region)
    
    resp = ec2.describe_reserved_instances_offerings(
        InstanceType=instance_type,
        ProductDescription="Linux/UNIX",
        IncludeMarketplace=True,
        OfferingClass="standard",
        OfferingType="No Upfront",
        MinDuration=1,
        MaxDuration=31536000,  # 1 year in seconds
    )
    
    marketplace = [
        o for o in resp["ReservedInstancesOfferings"]
        if o.get("Marketplace", False)
    ]
    
    return sorted(marketplace, key=lambda x: x["UsagePrice"])
```

---

## RI Modification and Exchange

### Modifying Standard RIs (size within family)

```bash
# Modify a regional RI: split one m5.2xlarge RI into two m5.xlarge RIs
aws ec2 modify-reserved-instances \
  --reserved-instances-ids ri-0abc123def456789 \
  --target-configurations \
    "AvailabilityZone=us-east-1a,InstanceCount=2,InstanceType=m5.xlarge,Scope=Region"
```

### Exchanging Convertible RIs

```python
def exchange_convertible_ri(
    source_ri_id: str,
    target_offering_id: str,
    region: str = "us-east-1"
) -> dict:
    """Exchange a Convertible RI for a different configuration."""
    ec2 = boto3.client("ec2", region_name=region)
    
    # First, get the exchange quote
    quote = ec2.get_reserved_instances_exchange_quote(
        ReservedInstanceIds=[source_ri_id],
        TargetConfigurations=[{
            "OfferingId": target_offering_id,
            "InstanceCount": 1,
        }]
    )
    
    print(f"Exchange valid until: {quote['ValidationFailureReason'] or 'Valid'}")
    print(f"Output reserved instances: {quote['OutputReservedInstancesWillExpireAt']}")
    
    # If the quote looks good, execute the exchange
    if not quote.get("IsValidExchange", False):
        raise ValueError(f"Invalid exchange: {quote.get('ValidationFailureReason')}")
    
    result = ec2.accept_reserved_instances_exchange_quote(
        ReservedInstanceIds=[source_ri_id],
        TargetConfigurations=[{
            "OfferingId": target_offering_id,
            "InstanceCount": 1,
        }]
    )
    
    return result
```

**Exchange rules:**
- New RI must be of equal or greater value
- If greater value, you pay the difference
- Exchange resets the term (you get a fresh 1 or 3-year RI)
- Can exchange multiple source RIs into one target RI

---

## RI Coverage Analysis with boto3

```python
import boto3
from datetime import datetime, timedelta

def analyze_ri_coverage(lookback_days: int = 30) -> dict:
    """Analyze RI coverage across your EC2 fleet."""
    ce = boto3.client("ce", region_name="us-east-1")
    
    end = datetime.utcnow().date()
    start = end - timedelta(days=lookback_days)
    
    # Coverage by instance type
    coverage = ce.get_reservation_coverage(
        TimePeriod={"Start": str(start), "End": str(end)},
        GroupBy=[{"Type": "DIMENSION", "Key": "INSTANCE_TYPE"}],
        Granularity="MONTHLY",
        Metrics=["Hour"],
    )
    
    results = []
    for group in coverage["CoveragesByTime"][0]["Groups"]:
        instance_type = group["Attributes"]["INSTANCE_TYPE"]
        cov = group["Coverage"]["CoverageHours"]
        results.append({
            "instance_type": instance_type,
            "covered_hours": float(cov["CoveredHours"]),
            "on_demand_hours": float(cov["OnDemandHours"]),
            "coverage_pct": float(cov["CoverageHoursPercentage"]),
        })
    
    # Sort by uncovered On-Demand hours (biggest opportunity)
    results.sort(key=lambda x: x["on_demand_hours"], reverse=True)
    
    print("\nRI Coverage by Instance Type:")
    print(f"{'Instance Type':<20} {'Coverage %':<12} {'OD Hours':<12} {'Opportunity'}")
    print("-" * 60)
    for r in results[:10]:
        opportunity = "HIGH" if r["coverage_pct"] < 50 else "MED" if r["coverage_pct"] < 80 else "LOW"
        print(f"{r['instance_type']:<20} {r['coverage_pct']:<12.1f} {r['on_demand_hours']:<12.0f} {opportunity}")
    
    return results
```

---

## RI Utilization Monitoring

```python
def analyze_ri_utilization(lookback_days: int = 30) -> dict:
    """Find underutilized RIs — wasted commitment."""
    ce = boto3.client("ce", region_name="us-east-1")
    
    end = datetime.utcnow().date()
    start = end - timedelta(days=lookback_days)
    
    utilization = ce.get_reservation_utilization(
        TimePeriod={"Start": str(start), "End": str(end)},
        GroupBy=[{"Type": "DIMENSION", "Key": "SUBSCRIPTION_ID"}],
        Granularity="MONTHLY",
    )
    
    underutilized = []
    for group in utilization["UtilizationsByTime"][0]["Groups"]:
        util = group["Utilization"]
        util_pct = float(util["UtilizationPercentage"])
        unused_cost = float(util.get("UnusedAmortizedUpfrontFeeForBillingPeriod", 0)) + \
                      float(util.get("UnusedRecurringFee", 0))
        
        if util_pct < 80:
            underutilized.append({
                "subscription_id": group["Attributes"]["SUBSCRIPTION_ID"],
                "utilization_pct": util_pct,
                "unused_cost_usd": unused_cost,
            })
    
    total_waste = sum(r["unused_cost_usd"] for r in underutilized)
    print(f"Underutilized RIs: {len(underutilized)}, Total waste: ${total_waste:.2f}/month")
    return underutilized
```

---

## Convertible RI Exchange Strategy

Use Convertible RIs as a hedge against architectural change:

```
Strategy: "Ladder Exchange"

Year 1: Buy Convertible RIs for current instance family (e.g., m5)
Year 2: If migrating to Graviton (m6g), exchange Convertible RIs
        - m5.xlarge → m6g.xlarge (same or better performance, lower cost)
        - Exchange resets term, you get fresh 3-year discount on new family
Year 3: Exchange again if needed (e.g., m6g → m7g)

Result: Always on latest generation with RI discounts, no stranded commitment
```

---

## RI vs Savings Plans Decision Guide

```
Should I buy RIs or Savings Plans?

1. Do you need to sell unused capacity on the RI Marketplace?
   YES → Standard RI (only option for Marketplace)
   NO  → Continue

2. Do you need capacity reservation in a specific AZ?
   YES → Zonal RI
   NO  → Continue

3. Is your instance family/region stable for 3 years?
   YES → EC2 Instance SP (slightly higher discount than Compute SP)
   NO  → Compute SP (maximum flexibility)

4. Do you run Lambda or Fargate?
   YES → Compute SP (covers Lambda + Fargate, RIs don't)
   NO  → Either works

5. Do you run SageMaker?
   YES → SageMaker SP (separate from EC2 SPs)
```

---

## Terraform RI Purchase Example

```hcl
# Purchase a 1-year Standard RI for m5.xlarge
resource "aws_reserved_instance" "app_server" {
  # Note: Terraform uses aws_reserved_instance for purchasing
  # In practice, most teams purchase via Console or AWS CLI
  # and import into Terraform state
}

# More practical: document RI inventory as data source
data "aws_ec2_reserved_instances" "current" {
  filter {
    name   = "state"
    values = ["active"]
  }
  filter {
    name   = "instance-type"
    values = ["m5.xlarge", "m5.2xlarge"]
  }
}

output "active_ri_count" {
  value = length(data.aws_ec2_reserved_instances.current.ids)
}
```

```bash
# Purchase via AWS CLI (more common for RIs)
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id <offering-id> \
  --instance-count 5 \
  --limit-price Amount=0.05,CurrencyCode=USD
```

---

## RI Portfolio Management

**Monthly RI review checklist:**

```python
def monthly_ri_review():
    """Automated monthly RI portfolio health check."""
    ec2 = boto3.client("ec2", region_name="us-east-1")
    
    # 1. Find expiring RIs (within 60 days)
    resp = ec2.describe_reserved_instances(
        Filters=[{"Name": "state", "Values": ["active"]}]
    )
    
    from datetime import timezone
    now = datetime.now(timezone.utc)
    expiring_soon = []
    
    for ri in resp["ReservedInstances"]:
        end = ri.get("End")
        if end:
            days_remaining = (end - now).days
            if days_remaining <= 60:
                expiring_soon.append({
                    "id": ri["ReservedInstancesId"],
                    "type": ri["InstanceType"],
                    "count": ri["InstanceCount"],
                    "days_remaining": days_remaining,
                    "end_date": str(end.date()),
                })
    
    print(f"\nRIs expiring within 60 days: {len(expiring_soon)}")
    for ri in sorted(expiring_soon, key=lambda x: x["days_remaining"]):
        print(f"  {ri['type']} x{ri['count']} — expires in {ri['days_remaining']} days ({ri['end_date']})")
    
    return expiring_soon

# Run monthly via Lambda or scheduled job
monthly_ri_review()
```

**Portfolio management principles:**
- Review expiring RIs 60–90 days before expiry — renewal decisions take time
- Track RI utilization weekly; act on < 80% utilization immediately
- Maintain a spreadsheet/CMDB of all RIs with owner, workload, and renewal decision
- Stagger purchases across months to avoid cliff-edge renewals
- Consider Savings Plans for new commitments; use RIs only where they provide unique value
