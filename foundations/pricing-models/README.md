# Cloud Pricing Models Deep Dive

Cloud providers offer a spectrum of pricing models, each with different trade-offs between cost, flexibility, and commitment. Understanding these models — and how to combine them strategically — is one of the highest-ROI activities in FinOps.

---

## On-Demand Pricing: The Flexibility Premium

On-demand is the default pricing model: pay for compute by the hour or second, with no upfront cost and no commitment.

### Pricing Structure

```
AWS EC2 on-demand (us-east-1, Linux):
  t3.micro:    $0.0104/hour  →  $7.59/month
  t3.medium:   $0.0416/hour  →  $30.37/month
  m6i.xlarge:  $0.192/hour   →  $140.16/month
  r6i.4xlarge: $1.008/hour   →  $735.84/month
  p3.2xlarge:  $3.06/hour    →  $2,233.80/month
```

### The Flexibility Premium

On-demand pricing includes a significant premium for the flexibility of no commitment. This premium is the "insurance" you pay for the ability to stop at any time.

**Flexibility premium by commitment type:**

| Pricing Model | Monthly Cost (m6i.xlarge) | Premium vs 3yr All-Upfront |
|---|---|---|
| On-demand | $140.16 | +178% |
| 1yr No-Upfront | $91.98 | +83% |
| 1yr All-Upfront | $84.53 | +68% |
| 3yr No-Upfront | $63.51 | +26% |
| 3yr All-Upfront | $50.22 | baseline |

### When On-Demand Makes Sense

- Workloads running **< 30% of the time** (on-demand is cheaper than 1yr RI at < 30% utilization)
- **New workloads** where usage patterns are unknown
- **Spiky, unpredictable** traffic that can't be baselined
- **Short-lived environments** (dev, testing, demos)
- **Burst capacity** above your committed baseline

---

## Reserved Instances and Committed Use Discounts

### AWS Reserved Instances (RIs)

Reserved Instances are a billing discount applied to on-demand usage that matches the RI attributes. You're not reserving capacity — you're reserving a discount.

**RI dimensions:**
- **Instance type:** e.g., m6i.xlarge
- **Region:** e.g., us-east-1
- **OS:** Linux, Windows, RHEL, SUSE
- **Tenancy:** Shared (default) or Dedicated
- **Term:** 1 year or 3 years
- **Payment:** All Upfront, Partial Upfront, No Upfront

**RI types:**

| RI Type | Flexibility | Discount |
|---|---|---|
| Standard RI | Fixed instance type, region | Highest (up to 72%) |
| Convertible RI | Can exchange for different type | Lower (up to 54%) |
| Scheduled RI | Reserved for specific time windows | Moderate |

**Standard vs Convertible RI decision:**
```
Use Standard RI when:
  - Instance type is stable and unlikely to change
  - You want maximum discount
  - Workload is well-understood

Use Convertible RI when:
  - You may need to change instance types (e.g., new generation releases)
  - Workload requirements may evolve
  - You value flexibility over maximum discount
```

### Payment Options: 1yr vs 3yr, Upfront vs No-Upfront

**AWS m6i.xlarge RI pricing comparison:**

| Term | Payment | Effective Hourly | Monthly | Savings vs On-Demand |
|---|---|---|---|---|
| On-demand | — | $0.192 | $140.16 | — |
| 1yr | No Upfront | $0.126 | $91.98 | 34% |
| 1yr | Partial Upfront | $0.116 | $84.53 | 40% |
| 1yr | All Upfront | $0.113 | $82.49 | 41% |
| 3yr | No Upfront | $0.087 | $63.51 | 55% |
| 3yr | Partial Upfront | $0.073 | $53.29 | 62% |
| 3yr | All Upfront | $0.069 | $50.22 | 64% |

**Upfront payment analysis:**

The "all upfront" option provides the highest discount but requires capital outlay. Calculate the effective annual return:

```python
def ri_upfront_roi(
    on_demand_monthly: float,
    ri_monthly_equivalent: float,
    upfront_payment: float,
    term_years: int
) -> dict:
    """Calculate ROI of paying upfront for Reserved Instances."""
    
    monthly_savings = on_demand_monthly - ri_monthly_equivalent
    total_savings = monthly_savings * (term_years * 12)
    net_savings = total_savings - upfront_payment
    
    # Annualized return on the upfront investment
    annual_return = (monthly_savings * 12) / upfront_payment * 100
    
    return {
        'monthly_savings': monthly_savings,
        'total_savings_over_term': total_savings,
        'upfront_cost': upfront_payment,
        'net_savings': net_savings,
        'annual_return_pct': annual_return,
        'payback_months': upfront_payment / monthly_savings
    }

# Example: 1yr All-Upfront m6i.xlarge
result = ri_upfront_roi(
    on_demand_monthly=140.16,
    ri_monthly_equivalent=82.49,
    upfront_payment=989.00,  # 1yr all-upfront price
    term_years=1
)
# annual_return_pct ≈ 70% — excellent return on capital
```

### GCP Committed Use Discounts (CUDs)

GCP offers two types of committed use discounts:

**Resource-based CUDs:**
- Commit to specific vCPU and memory amounts
- 1-year: ~37% discount, 3-year: ~55% discount
- Applies to N1, N2, N2D, C2, C2D, M1, M2 machine types

**Spend-based CUDs:**
- Commit to a minimum monthly spend on specific services
- Available for Cloud SQL, Cloud Spanner, Cloud Run, and others
- 1-year: ~20% discount, 3-year: ~40% discount

```bash
# GCP: Create a committed use discount
gcloud compute commitments create my-commitment \
  --plan=12-month \
  --region=us-central1 \
  --resources=vcpu=100,memory=400GB
```

### Azure Reserved VM Instances

Azure Reserved VM Instances work similarly to AWS RIs:

```bash
# Azure CLI: Purchase a reservation
az reservations reservation-order purchase \
  --reservation-order-id "$(uuidgen)" \
  --sku Standard_D4s_v3 \
  --location eastus \
  --reserved-resource-type VirtualMachines \
  --billing-scope /subscriptions/your-subscription-id \
  --term P1Y \
  --billing-plan Upfront \
  --quantity 10 \
  --display-name "prod-web-tier-reservation"
```

---

## Spot / Preemptible Instances

### How Spot Pricing Works

Spot instances use spare cloud provider capacity. Pricing fluctuates based on supply and demand, but in practice AWS Spot prices are relatively stable (within 20–30% of the floor price most of the time).

**AWS Spot pricing example (us-east-1):**
```
m6i.xlarge on-demand:  $0.192/hour
m6i.xlarge spot:       $0.038–0.058/hour  (70–80% discount)

c6i.4xlarge on-demand: $0.680/hour
c6i.4xlarge spot:      $0.120–0.180/hour  (74–82% discount)
```

### Interruption Handling

AWS provides a 2-minute interruption notice before reclaiming a Spot instance. Design workloads to handle this gracefully:

```python
# Spot interruption handler for a worker process
import boto3
import requests
import signal
import sys
import threading
import time

class SpotWorker:
    def __init__(self):
        self.running = True
        self.current_job = None
        self._start_interruption_monitor()
    
    def _start_interruption_monitor(self):
        """Monitor for spot interruption notice in background thread."""
        def monitor():
            while self.running:
                try:
                    # Check instance metadata service v2
                    token_response = requests.put(
                        'http://169.254.169.254/latest/api/token',
                        headers={'X-aws-ec2-metadata-token-ttl-seconds': '21600'},
                        timeout=1
                    )
                    token = token_response.text
                    
                    response = requests.get(
                        'http://169.254.169.254/latest/meta-data/spot/termination-time',
                        headers={'X-aws-ec2-metadata-token': token},
                        timeout=1
                    )
                    
                    if response.status_code == 200:
                        print(f"Spot interruption at: {response.text}")
                        self._handle_interruption()
                        return
                        
                except Exception:
                    pass
                
                time.sleep(5)
        
        thread = threading.Thread(target=monitor, daemon=True)
        thread.start()
    
    def _handle_interruption(self):
        """Gracefully handle spot interruption."""
        print("Spot interruption received. Checkpointing and draining...")
        
        if self.current_job:
            # Save job state to S3 for resumption on another instance
            self._checkpoint_job(self.current_job)
        
        # Signal the main loop to stop
        self.running = False
        
        # Deregister from SQS / load balancer
        self._deregister()
    
    def _checkpoint_job(self, job):
        """Save job state to S3."""
        s3 = boto3.client('s3')
        s3.put_object(
            Bucket='my-job-checkpoints',
            Key=f'checkpoints/{job["id"]}.json',
            Body=json.dumps(job['state'])
        )
    
    def process_jobs(self):
        """Main job processing loop."""
        sqs = boto3.client('sqs')
        
        while self.running:
            messages = sqs.receive_message(
                QueueUrl='https://sqs.us-east-1.amazonaws.com/123456789012/jobs',
                MaxNumberOfMessages=1,
                WaitTimeSeconds=20
            ).get('Messages', [])
            
            for message in messages:
                self.current_job = json.loads(message['Body'])
                self._process(self.current_job)
                sqs.delete_message(
                    QueueUrl='...',
                    ReceiptHandle=message['ReceiptHandle']
                )
                self.current_job = None
```

### Spot Use Cases and Suitability

| Use Case | Spot Suitability | Notes |
|---|---|---|
| Batch data processing | ✅ Excellent | Checkpointing handles interruptions |
| CI/CD build agents | ✅ Excellent | Jobs can be retried |
| ML training (with checkpointing) | ✅ Good | Save model checkpoints frequently |
| Stateless web tier | ✅ Good | Use with on-demand fallback |
| Stateful databases | ❌ Poor | Data loss risk on interruption |
| Real-time APIs (SLA-bound) | ❌ Poor | Interruption violates SLA |
| Long-running interactive jobs | ⚠️ Risky | Interruption loses progress |

### Spot Fleet / Auto Scaling with Spot

```hcl
# Terraform: Mixed instance policy with spot
resource "aws_autoscaling_group" "mixed" {
  name                = "mixed-spot-ondemand"
  vpc_zone_identifier = var.subnet_ids
  min_size            = 2
  max_size            = 20
  desired_capacity    = 6

  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = 2    # Always keep 2 on-demand
      on_demand_percentage_above_base_capacity = 20   # 20% on-demand above base
      spot_allocation_strategy                 = "capacity-optimized"
    }

    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.app.id
        version            = "$Latest"
      }

      # Diversify across instance types for better spot availability
      override {
        instance_type = "m6i.xlarge"
      }
      override {
        instance_type = "m6a.xlarge"
      }
      override {
        instance_type = "m5.xlarge"
      }
      override {
        instance_type = "m5a.xlarge"
      }
    }
  }
}
```

---

## AWS Savings Plans

Savings Plans are a more flexible alternative to Reserved Instances, introduced by AWS in 2019.

### Savings Plan Types

**Compute Savings Plans:**
- Most flexible — applies to any EC2 instance (any family, size, region, OS, tenancy), Lambda, and Fargate
- Discount: up to 66% vs on-demand
- Best for: organizations with diverse or changing compute needs

**EC2 Instance Savings Plans:**
- Applies to a specific instance family in a specific region (e.g., M family in us-east-1)
- Discount: up to 72% vs on-demand
- Best for: stable workloads with predictable instance family usage

**SageMaker Savings Plans:**
- Applies to SageMaker ML instances
- Discount: up to 64% vs on-demand
- Best for: organizations with significant ML workloads

### Savings Plans vs Reserved Instances

| Dimension | Savings Plans | Reserved Instances |
|---|---|---|
| Flexibility | High (Compute SP) to Medium (EC2 SP) | Low (Standard) to Medium (Convertible) |
| Maximum discount | 66–72% | 72% |
| Applies to | EC2 + Lambda + Fargate | EC2 only (per type) |
| Commitment unit | $/hour spend | Specific instance |
| Marketplace | Cannot sell unused | Can sell on RI Marketplace |
| Recommendation | Preferred for most use cases | Better for very stable, specific workloads |

### Purchasing Savings Plans

```python
import boto3

ce = boto3.client('ce')

# Get Savings Plans purchase recommendations
recommendations = ce.get_savings_plans_purchase_recommendation(
    SavingsPlansType='COMPUTE_SP',
    TermInYears='ONE_YEAR',
    PaymentOption='NO_UPFRONT',
    LookbackPeriodInDays='THIRTY_DAYS'
)

for rec in recommendations['SavingsPlansPurchaseRecommendation']['SavingsPlansPurchaseRecommendationDetails']:
    print(f"Recommended hourly commitment: ${rec['HourlyCommitmentToPurchase']}")
    print(f"Estimated monthly savings: ${rec['EstimatedMonthlySavingsAmount']}")
    print(f"Estimated savings rate: {rec['EstimatedSavingsPercentage']}%")
    print(f"Estimated ROI: {rec['EstimatedROI']}%")
```

---

## GCP Sustained Use Discounts

GCP automatically applies **Sustained Use Discounts (SUDs)** to N1, N2, N2D, C2, C2D, M1, and M2 machine types when they run for more than 25% of a month — no commitment required.

**SUD discount schedule:**

| Usage in Month | Discount |
|---|---|
| 0–25% | 0% (full price) |
| 25–50% | 20% |
| 50–75% | 40% |
| 75–100% | 60% |

**Effective discount for always-on instances:** ~30% vs on-demand

This makes GCP inherently more cost-effective for always-on workloads compared to AWS on-demand, even without purchasing commitments.

---

## Azure Hybrid Benefit

Azure Hybrid Benefit allows organizations with existing Windows Server and SQL Server licenses (with Software Assurance) to use those licenses in Azure, significantly reducing costs.

**Windows Server Hybrid Benefit:**
- Reduces Windows VM cost by ~40%
- Each 2-core Windows Server license covers 1 vCPU in Azure
- Can be applied to Azure VMs, AKS nodes, and Azure Stack HCI

**SQL Server Hybrid Benefit:**
- Reduces Azure SQL Database cost by up to 55%
- Each SQL Server Enterprise core license covers 4 vCores in Azure SQL
- Applies to Azure SQL Database, SQL Managed Instance, SQL on VMs

```bash
# Azure CLI: Apply Hybrid Benefit to existing VM
az vm update \
  --resource-group myResourceGroup \
  --name myVM \
  --license-type Windows_Server

# Apply to all VMs in a resource group
az vm list --resource-group myResourceGroup --query "[].name" -o tsv | \
  xargs -I {} az vm update --resource-group myResourceGroup --name {} \
  --license-type Windows_Server
```

---

## Pricing Calculators and Estimation Tools

| Tool | Provider | Best For |
|---|---|---|
| [AWS Pricing Calculator](https://calculator.aws) | AWS | New architecture estimates |
| [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) | Azure | Azure workload estimates |
| [GCP Pricing Calculator](https://cloud.google.com/products/calculator) | GCP | GCP workload estimates |
| [Infracost](https://www.infracost.io) | Multi-cloud | Terraform cost estimation in CI/CD |
| [CloudPrice.net](https://cloudprice.net) | Multi-cloud | Instance type price comparison |
| [ec2instances.info](https://ec2instances.info) | AWS | EC2 instance comparison |
| [instances.vantage.sh](https://instances.vantage.sh) | AWS | EC2 pricing with RI/SP comparison |

---

## Commitment Strategy: Portfolio Approach

The optimal commitment strategy treats your cloud compute like an investment portfolio — balancing risk, return, and liquidity.

### The Three-Layer Model

```
Layer 1: Committed baseline (60–70% of compute)
  → 1-year Savings Plans / Reserved Instances
  → Stable, predictable workloads
  → Highest discount, lowest flexibility

Layer 2: On-demand buffer (15–25% of compute)
  → On-demand instances
  → Variable workloads, new services
  → Full flexibility, full price

Layer 3: Spot opportunistic (10–20% of compute)
  → Spot / Preemptible instances
  → Batch, CI/CD, fault-tolerant workloads
  → Maximum discount, interruption risk
```

### Commitment Sizing Process

```python
def calculate_commitment_size(
    daily_usage_hours: list[float],  # 30 days of hourly usage data
    commitment_coverage_target: float = 0.70,
    percentile: float = 0.20  # Cover the 20th percentile of usage
) -> dict:
    """
    Calculate optimal Savings Plan commitment size.
    
    Strategy: Cover the stable baseline (low percentile of usage)
    to avoid over-committing on variable workloads.
    """
    import statistics
    
    sorted_usage = sorted(daily_usage_hours)
    baseline_usage = sorted_usage[int(len(sorted_usage) * percentile)]
    
    return {
        'recommended_hourly_commitment': baseline_usage * commitment_coverage_target,
        'p20_usage': baseline_usage,
        'average_usage': statistics.mean(daily_usage_hours),
        'max_usage': max(daily_usage_hours),
        'coverage_at_recommendation': commitment_coverage_target
    }
```

### Commitment Review Cadence

| Activity | Frequency | Owner |
|---|---|---|
| RI/SP utilization review | Weekly | FinOps team |
| Coverage analysis | Monthly | FinOps team |
| New purchase recommendations | Monthly | FinOps team + Finance |
| Commitment portfolio rebalancing | Quarterly | FinOps team + Finance |
| 3-year commitment review | Annually | FinOps + Engineering + Finance |

### Key Metrics

| Metric | Definition | Target |
|---|---|---|
| RI/SP Coverage | % of eligible compute covered by commitments | 70–80% |
| RI/SP Utilization | % of purchased commitments actually used | > 90% |
| Effective savings rate | Actual discount vs on-demand equivalent | > 35% |
| Commitment waste | $ of unused commitments | < 5% of commitment spend |
