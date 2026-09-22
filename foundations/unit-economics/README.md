# Unit Economics for Cloud

Unit economics transforms cloud cost data from abstract dollar amounts into engineering-meaningful metrics. Instead of "we spent $200K on cloud last month," unit economics tells you "we spent $0.0023 per API request" or "$1.40 per active user per month." This framing makes costs actionable for engineers and comparable across time, teams, and competitors.

---

## Defining Your Unit

The most important step in unit economics is choosing the right unit — the denominator that best represents the value your system delivers.

### Criteria for a Good Unit

1. **Directly tied to business value** — the unit should represent something customers pay for or care about
2. **Measurable and consistent** — you can count it reliably every month
3. **Controllable by engineering** — engineers can influence the cost-per-unit through their work
4. **Comparable over time** — the definition doesn't change, so you can track trends

### Unit Selection by System Type

| System Type | Primary Unit | Secondary Units |
|---|---|---|
| SaaS B2C | Monthly Active User (MAU) | Daily Active User, Feature Usage |
| SaaS B2B | Seat / License | API Call, Data Processed |
| API Service | API Request | Endpoint, Response Size |
| Data Pipeline | GB Processed | Record, Job Run |
| E-commerce | Transaction | Order, Order Line Item |
| Media/Streaming | Stream Hour | Content Minute, Viewer |
| Search Engine | Search Query | Index Document |
| AI/ML Inference | Inference | Token, Model Call |
| Storage Service | GB Stored | Object, Access Request |
| Communication | Message Sent | Notification, Email |

### Multi-Level Units

Complex systems often need multiple levels of unit economics:

```
Level 1 (Business): Cost per Monthly Active User
  → Answers: "What does it cost to serve one user for a month?"

Level 2 (Product): Cost per Feature Usage
  → Answers: "What does it cost to run the search feature per query?"

Level 3 (Engineering): Cost per Service Call
  → Answers: "What does it cost to call the recommendation service?"

Level 4 (Infrastructure): Cost per Compute Hour
  → Answers: "What does it cost to run one hour of our workload?"
```

---

## Cost Per Unit Calculation Methodology

### Simple Direct Calculation

```python
class UnitEconomics:
    """Calculate and track unit economics for cloud workloads."""
    
    def __init__(self, service_name: str):
        self.service_name = service_name
        self.history = []
    
    def calculate(
        self,
        period: str,
        total_cloud_cost: float,
        business_unit_count: float,
        unit_name: str
    ) -> dict:
        """Calculate unit economics for a given period."""
        
        if business_unit_count == 0:
            raise ValueError("Business unit count cannot be zero")
        
        cost_per_unit = total_cloud_cost / business_unit_count
        
        result = {
            'period': period,
            'service': self.service_name,
            'total_cost': total_cloud_cost,
            'unit_count': business_unit_count,
            'unit_name': unit_name,
            'cost_per_unit': cost_per_unit,
        }
        
        # Calculate trend if we have history
        if self.history:
            prev = self.history[-1]
            result['mom_change_pct'] = (
                (cost_per_unit - prev['cost_per_unit']) / prev['cost_per_unit'] * 100
            )
        
        self.history.append(result)
        return result
    
    def trend_report(self) -> str:
        """Generate a trend report for unit economics."""
        if len(self.history) < 2:
            return "Insufficient data for trend analysis"
        
        lines = [f"Unit Economics Trend: {self.service_name}"]
        lines.append("-" * 60)
        lines.append(f"{'Period':<12} {'Unit Count':>12} {'Total Cost':>12} {'Cost/Unit':>12} {'MoM':>8}")
        lines.append("-" * 60)
        
        for entry in self.history:
            mom = f"{entry.get('mom_change_pct', 0):+.1f}%" if 'mom_change_pct' in entry else "baseline"
            lines.append(
                f"{entry['period']:<12} "
                f"{entry['unit_count']:>12,.0f} "
                f"${entry['total_cost']:>11,.0f} "
                f"${entry['cost_per_unit']:>11.4f} "
                f"{mom:>8}"
            )
        
        return "\n".join(lines)


# Example usage
ue = UnitEconomics("checkout-service")
ue.calculate("2024-01", 85000, 2_500_000, "API request")
ue.calculate("2024-02", 88000, 2_800_000, "API request")
ue.calculate("2024-03", 90000, 3_200_000, "API request")
ue.calculate("2024-04", 87000, 3_500_000, "API request")  # Optimization deployed

print(ue.trend_report())
```

### Fully-Loaded Cost Calculation

For accurate unit economics, include all cost components:

```python
def calculate_fully_loaded_cost(
    direct_compute: float,      # EC2, Lambda, containers
    direct_storage: float,      # S3, EBS, RDS storage
    direct_database: float,     # RDS, DynamoDB, ElastiCache
    direct_networking: float,   # Data transfer, load balancers
    shared_platform_cost: float, # Kubernetes, monitoring, logging
    shared_platform_allocation_pct: float,  # % of shared costs to allocate
    support_cost: float,        # AWS support plan allocation
    tooling_cost: float         # FinOps tools, observability
) -> dict:
    """Calculate fully-loaded cloud cost for a service."""
    
    direct_cost = direct_compute + direct_storage + direct_database + direct_networking
    allocated_shared = shared_platform_cost * (shared_platform_allocation_pct / 100)
    overhead = support_cost + tooling_cost
    
    total = direct_cost + allocated_shared + overhead
    
    return {
        'direct_cost': direct_cost,
        'allocated_shared_cost': allocated_shared,
        'overhead': overhead,
        'total_fully_loaded': total,
        'breakdown': {
            'compute_pct': direct_compute / total * 100,
            'storage_pct': direct_storage / total * 100,
            'database_pct': direct_database / total * 100,
            'networking_pct': direct_networking / total * 100,
            'shared_pct': allocated_shared / total * 100,
            'overhead_pct': overhead / total * 100
        }
    }
```

---

## Unit Economics Dashboards

### Dashboard Design

A unit economics dashboard should show:

1. **Current cost-per-unit** — the headline metric
2. **Trend over time** — is it improving or degrading?
3. **Cost breakdown** — what's driving the cost?
4. **Comparison to target** — are we on track?
5. **Anomalies** — unexpected spikes in unit cost

### Sample Dashboard Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Unit Economics Dashboard — Checkout Service                │
│  Period: April 2024                                         │
├─────────────────┬───────────────────┬───────────────────────┤
│ Cost/API Request│ Cost/Transaction  │ Cost/MAU              │
│ $0.0023         │ $0.0340           │ $1.42                 │
│ ↓ -8.3% MoM    │ ↓ -5.1% MoM      │ ↓ -12.4% MoM         │
├─────────────────┴───────────────────┴───────────────────────┤
│  Cost Per Request Trend (12 months)                         │
│  $0.0040 ┤                                                  │
│  $0.0035 ┤  ●                                               │
│  $0.0030 ┤    ●  ●                                          │
│  $0.0025 ┤          ●  ●  ●                                 │
│  $0.0020 ┤                    ●  ●  ●  ●                    │
│          └──────────────────────────────────────────────    │
│           Jan Feb Mar Apr May Jun Jul Aug Sep Oct Nov Dec   │
├─────────────────────────────────────────────────────────────┤
│  Cost Breakdown (April 2024)                                │
│  Compute:    $45,000 (52%)  ████████████████████████        │
│  Database:   $22,000 (25%)  ████████████                    │
│  Networking: $12,000 (14%)  ███████                         │
│  Storage:     $5,000 (6%)   ███                             │
│  Shared:      $3,000 (3%)   ██                              │
└─────────────────────────────────────────────────────────────┘
```

### Grafana Dashboard Query (Prometheus + CloudWatch)

```yaml
# Grafana dashboard panel: Cost per API request
panels:
  - title: "Cost per API Request"
    type: stat
    targets:
      - expr: |
          # Cost per request = monthly cost / monthly request count
          (
            sum(aws_billing_estimated_charges{service="checkout"}) 
            / 
            sum(rate(http_requests_total{service="checkout"}[30d])) * 86400 * 30
          )
    fieldConfig:
      defaults:
        unit: currencyUSD
        decimals: 4
        thresholds:
          steps:
            - value: 0
              color: green
            - value: 0.003
              color: yellow
            - value: 0.005
              color: red
```

---

## Using Unit Economics for Capacity Planning

Unit economics enables data-driven capacity planning:

```python
def capacity_plan(
    current_unit_cost: float,
    current_unit_volume: float,
    projected_unit_volume: float,
    efficiency_improvement_pct: float = 5.0,
    commitment_discount_pct: float = 30.0
) -> dict:
    """
    Plan cloud capacity and budget based on unit economics.
    
    Args:
        current_unit_cost: Current cost per unit ($)
        current_unit_volume: Current monthly unit volume
        projected_unit_volume: Projected monthly unit volume
        efficiency_improvement_pct: Expected engineering efficiency gain
        commitment_discount_pct: Expected discount from RI/SP purchases
    """
    
    current_total = current_unit_cost * current_unit_volume
    
    # Raw projection (no optimization)
    raw_projected_cost = current_unit_cost * projected_unit_volume
    
    # With engineering efficiency improvements
    efficiency_factor = 1 - (efficiency_improvement_pct / 100)
    optimized_unit_cost = current_unit_cost * efficiency_factor
    
    # With commitment discounts on baseline
    baseline_volume = current_unit_volume  # Commit to current baseline
    variable_volume = max(0, projected_unit_volume - baseline_volume)
    
    committed_cost = (baseline_volume * optimized_unit_cost) * (1 - commitment_discount_pct / 100)
    variable_cost = variable_volume * optimized_unit_cost
    
    total_projected = committed_cost + variable_cost
    
    return {
        'current_monthly_cost': current_total,
        'raw_projected_cost': raw_projected_cost,
        'optimized_projected_cost': total_projected,
        'projected_unit_cost': total_projected / projected_unit_volume,
        'savings_vs_raw': raw_projected_cost - total_projected,
        'growth_factor': projected_unit_volume / current_unit_volume,
        'cost_growth_factor': total_projected / current_total,
        'efficiency_gain': (current_unit_cost - (total_projected / projected_unit_volume)) / current_unit_cost * 100
    }

# Example: Planning for 2x user growth
plan = capacity_plan(
    current_unit_cost=1.50,      # $1.50 per MAU
    current_unit_volume=50_000,  # 50K MAU
    projected_unit_volume=100_000,  # 100K MAU (2x growth)
    efficiency_improvement_pct=10,
    commitment_discount_pct=35
)

print(f"Current monthly cost: ${plan['current_monthly_cost']:,.0f}")
print(f"Raw projected cost (2x users): ${plan['raw_projected_cost']:,.0f}")
print(f"Optimized projected cost: ${plan['optimized_projected_cost']:,.0f}")
print(f"Projected cost per MAU: ${plan['projected_unit_cost']:.2f}")
print(f"Savings vs raw: ${plan['savings_vs_raw']:,.0f}")
```

---

## Unit Economics for AI/ML Workloads

AI/ML workloads have unique unit economics driven by model size, inference complexity, and training frequency.

### Key AI/ML Units

| Workload | Primary Unit | Cost Drivers |
|---|---|---|
| LLM inference | Token (input + output) | Model size, GPU type, batch size |
| Image generation | Image | Resolution, steps, GPU type |
| Embedding generation | Embedding / 1K tokens | Model size, batch efficiency |
| Model training | Training run / epoch | GPU hours, data volume |
| Fine-tuning | Fine-tuning job | Base model, dataset size |
| Vector search | Query | Index size, recall requirements |

### Cost Per Token Calculation

```python
class LLMCostCalculator:
    """Calculate cost per token for LLM inference workloads."""
    
    # AWS Bedrock pricing (approximate, verify current pricing)
    BEDROCK_PRICING = {
        'claude-3-haiku': {'input': 0.00025, 'output': 0.00125},   # per 1K tokens
        'claude-3-sonnet': {'input': 0.003, 'output': 0.015},
        'claude-3-opus': {'input': 0.015, 'output': 0.075},
        'llama3-70b': {'input': 0.00265, 'output': 0.0035},
        'titan-text-lite': {'input': 0.0003, 'output': 0.0004},
    }
    
    def __init__(self, model: str, provider: str = 'bedrock'):
        self.model = model
        self.provider = provider
        self.pricing = self.BEDROCK_PRICING.get(model, {})
    
    def cost_per_request(
        self,
        avg_input_tokens: int,
        avg_output_tokens: int
    ) -> float:
        """Calculate cost per API request."""
        input_cost = (avg_input_tokens / 1000) * self.pricing['input']
        output_cost = (avg_output_tokens / 1000) * self.pricing['output']
        return input_cost + output_cost
    
    def monthly_cost_estimate(
        self,
        monthly_requests: int,
        avg_input_tokens: int,
        avg_output_tokens: int
    ) -> dict:
        """Estimate monthly cost for LLM workload."""
        cost_per_req = self.cost_per_request(avg_input_tokens, avg_output_tokens)
        total_monthly = cost_per_req * monthly_requests
        
        total_input_tokens = avg_input_tokens * monthly_requests
        total_output_tokens = avg_output_tokens * monthly_requests
        
        return {
            'model': self.model,
            'monthly_requests': monthly_requests,
            'cost_per_request': cost_per_req,
            'total_monthly_cost': total_monthly,
            'total_input_tokens': total_input_tokens,
            'total_output_tokens': total_output_tokens,
            'cost_per_1k_total_tokens': total_monthly / ((total_input_tokens + total_output_tokens) / 1000)
        }

# Compare models for a chatbot workload
models = ['claude-3-haiku', 'claude-3-sonnet', 'claude-3-opus']
for model in models:
    calc = LLMCostCalculator(model)
    estimate = calc.monthly_cost_estimate(
        monthly_requests=1_000_000,
        avg_input_tokens=500,
        avg_output_tokens=200
    )
    print(f"{model}: ${estimate['total_monthly_cost']:,.0f}/month "
          f"(${estimate['cost_per_request']:.4f}/request)")
```

### GPU Cost Optimization for Training

```python
def compare_training_options(
    training_hours: float,
    model_size_b_params: float
) -> dict:
    """Compare cost of different GPU options for model training."""
    
    # Approximate hourly costs (us-east-1, on-demand)
    gpu_options = {
        'p3.2xlarge (V100 16GB)': {'hourly': 3.06, 'memory_gb': 16},
        'p3.8xlarge (4x V100)': {'hourly': 12.24, 'memory_gb': 64},
        'p4d.24xlarge (8x A100 40GB)': {'hourly': 32.77, 'memory_gb': 320},
        'p4de.24xlarge (8x A100 80GB)': {'hourly': 40.96, 'memory_gb': 640},
        'g5.xlarge (A10G 24GB)': {'hourly': 1.006, 'memory_gb': 24},
        'g5.12xlarge (4x A10G)': {'hourly': 5.672, 'memory_gb': 96},
    }
    
    results = {}
    for instance, specs in gpu_options.items():
        on_demand_cost = training_hours * specs['hourly']
        spot_cost = on_demand_cost * 0.3  # ~70% spot discount
        
        results[instance] = {
            'on_demand_cost': on_demand_cost,
            'spot_cost': spot_cost,
            'memory_gb': specs['memory_gb'],
            'fits_model': specs['memory_gb'] >= model_size_b_params * 2  # Rough estimate
        }
    
    return results
```

---

## Benchmarking Unit Costs Against Industry

### Industry Benchmarks

| Metric | Early Stage | Growth Stage | Mature |
|---|---|---|---|
| Cloud cost as % of revenue | 15–25% | 10–18% | 5–12% |
| Cost per MAU/month | $2–5 | $1–3 | $0.50–1.50 |
| Cost per API request | $0.001–0.01 | $0.0005–0.005 | $0.0001–0.002 |
| Cost per GB stored | $0.02–0.05 | $0.015–0.03 | $0.01–0.02 |
| Cost per transaction | $0.05–0.20 | $0.02–0.10 | $0.005–0.05 |

**Sources for benchmarking:**
- [FinOps Foundation State of FinOps Report](https://data.finops.org)
- [Andreessen Horowitz Cloud Cost Benchmarks](https://a16z.com/cloud-cost-benchmarks)
- Peer group comparisons via FinOps Foundation community

### Competitive Benchmarking Framework

```python
def benchmark_unit_cost(
    your_cost_per_unit: float,
    industry_p25: float,  # 25th percentile (best performers)
    industry_p50: float,  # Median
    industry_p75: float   # 75th percentile (laggards)
) -> dict:
    """Benchmark your unit cost against industry percentiles."""
    
    if your_cost_per_unit <= industry_p25:
        tier = "Top Quartile (Best-in-class)"
        recommendation = "Maintain current efficiency, focus on growth"
    elif your_cost_per_unit <= industry_p50:
        tier = "Second Quartile (Above average)"
        recommendation = "Target 10-15% improvement to reach top quartile"
    elif your_cost_per_unit <= industry_p75:
        tier = "Third Quartile (Below average)"
        recommendation = "Significant optimization opportunity — prioritize FinOps"
    else:
        tier = "Bottom Quartile (Lagging)"
        recommendation = "Critical — cloud costs are a competitive disadvantage"
    
    improvement_to_median = max(0, (your_cost_per_unit - industry_p50) / your_cost_per_unit * 100)
    
    return {
        'your_cost': your_cost_per_unit,
        'industry_p25': industry_p25,
        'industry_p50': industry_p50,
        'industry_p75': industry_p75,
        'tier': tier,
        'recommendation': recommendation,
        'improvement_to_median_pct': improvement_to_median
    }
```

---

## Unit Economics as an Engineering KPI

### Integrating Unit Economics into Engineering Metrics

Unit economics should be tracked alongside traditional engineering KPIs:

| KPI Category | Traditional Metrics | FinOps Addition |
|---|---|---|
| Reliability | Uptime, error rate, latency | Cost per 9 of availability |
| Performance | P99 latency, throughput | Cost per request at P99 |
| Velocity | Deployment frequency, lead time | Cost per deployment |
| Quality | Bug rate, test coverage | Cost of incidents |
| **Efficiency** | CPU utilization | **Cost per business unit** |

### Engineering OKR Example

```
Objective: Improve platform cost efficiency while supporting 2x user growth

Key Results:
  KR1: Reduce cost per MAU from $1.50 to $1.20 by Q4 (20% improvement)
  KR2: Achieve 80% Savings Plan coverage for production compute
  KR3: Reduce untagged spend from 8% to < 2%
  KR4: Implement cost estimation in 100% of IaC pull requests

Initiatives:
  - Migrate batch workloads to Spot instances (saves ~$15K/month)
  - Implement Redis caching for top 10 database queries (saves ~$8K/month)
  - Right-size 20 over-provisioned EC2 instances (saves ~$5K/month)
  - Purchase 1-year Compute Savings Plan for baseline compute (saves ~$12K/month)
```

### Unit Cost Alerting

```python
# Alert when unit cost degrades beyond threshold
def check_unit_cost_regression(
    current_cost_per_unit: float,
    baseline_cost_per_unit: float,
    regression_threshold_pct: float = 10.0
) -> dict:
    """
    Alert when unit cost increases beyond acceptable threshold.
    Useful for detecting cost regressions after deployments.
    """
    change_pct = (current_cost_per_unit - baseline_cost_per_unit) / baseline_cost_per_unit * 100
    
    is_regression = change_pct > regression_threshold_pct
    
    return {
        'current': current_cost_per_unit,
        'baseline': baseline_cost_per_unit,
        'change_pct': change_pct,
        'is_regression': is_regression,
        'severity': 'critical' if change_pct > 25 else 'warning' if is_regression else 'ok',
        'message': (
            f"Unit cost regression detected: +{change_pct:.1f}% above baseline"
            if is_regression else
            f"Unit cost within acceptable range: {change_pct:+.1f}%"
        )
    }
```
