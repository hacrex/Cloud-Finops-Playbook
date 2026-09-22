# Sustainability in Cloud FinOps

Sustainability and FinOps are natural allies. The most cost-efficient cloud architectures are also typically the most energy-efficient. Eliminating waste, rightsizing resources, and optimizing workloads reduces both your cloud bill and your carbon footprint. As organizations face increasing pressure to report and reduce their environmental impact, sustainability has become a first-class concern in cloud strategy.

---

## Carbon-Aware Cloud Architecture

Carbon-aware architecture makes decisions based on the carbon intensity of compute, not just cost and performance.

### Carbon Intensity Basics

Carbon intensity measures the amount of CO₂ equivalent (CO₂e) emitted per unit of electricity consumed (gCO₂e/kWh). It varies significantly by:
- **Region:** Regions powered by renewables have lower carbon intensity
- **Time of day:** Grid carbon intensity fluctuates with renewable generation
- **Season:** Solar and wind availability varies seasonally

### Cloud Region Carbon Intensity

| Cloud Provider | Low Carbon Regions | High Carbon Regions |
|---|---|---|
| AWS | us-west-2 (Oregon), eu-west-1 (Ireland), eu-north-1 (Stockholm) | us-east-1 (Virginia), ap-southeast-1 (Singapore) |
| Azure | northeurope (Ireland), swedencentral, norwayeast | eastasia, southeastasia |
| GCP | us-west1 (Oregon), europe-north1 (Finland), europe-west1 (Belgium) | asia-east1 (Taiwan), asia-southeast1 (Singapore) |

**AWS Carbon Intensity by Region (approximate):**

| Region | Carbon Intensity (gCO₂e/kWh) | Renewable % |
|---|---|---|
| eu-north-1 (Stockholm) | ~8 | ~95% |
| eu-west-1 (Ireland) | ~316 | ~40% |
| us-west-2 (Oregon) | ~136 | ~70% |
| us-east-1 (Virginia) | ~415 | ~20% |
| ap-southeast-1 (Singapore) | ~493 | ~5% |

### Carbon-Aware Workload Placement

```python
# Carbon-aware workload scheduler
# Uses Electricity Maps API for real-time carbon intensity data

import requests
from dataclasses import dataclass
from typing import Optional

@dataclass
class RegionCarbonData:
    region: str
    zone: str
    carbon_intensity: float  # gCO2e/kWh
    renewable_percentage: float
    timestamp: str

class CarbonAwareScheduler:
    """
    Schedule workloads to minimize carbon footprint by choosing
    the lowest-carbon region or time window.
    """
    
    ELECTRICITY_MAPS_API = "https://api.electricitymap.org/v3"
    
    # Mapping of cloud regions to electricity grid zones
    REGION_TO_ZONE = {
        'us-east-1': 'US-MIDA-PJM',
        'us-west-2': 'US-NW-PACW',
        'eu-west-1': 'IE',
        'eu-north-1': 'SE',
        'ap-southeast-1': 'SG',
    }
    
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.headers = {'auth-token': api_key}
    
    def get_carbon_intensity(self, zone: str) -> Optional[float]:
        """Get current carbon intensity for a grid zone."""
        response = requests.get(
            f"{self.ELECTRICITY_MAPS_API}/carbon-intensity/latest",
            params={'zone': zone},
            headers=self.headers
        )
        if response.status_code == 200:
            return response.json()['carbonIntensity']
        return None
    
    def find_lowest_carbon_region(self, candidate_regions: list[str]) -> dict:
        """Find the lowest carbon intensity region from candidates."""
        results = []
        
        for region in candidate_regions:
            zone = self.REGION_TO_ZONE.get(region)
            if zone:
                intensity = self.get_carbon_intensity(zone)
                if intensity is not None:
                    results.append({
                        'region': region,
                        'zone': zone,
                        'carbon_intensity': intensity
                    })
        
        if not results:
            return {'error': 'No carbon data available'}
        
        results.sort(key=lambda x: x['carbon_intensity'])
        best = results[0]
        worst = results[-1]
        
        return {
            'recommended_region': best['region'],
            'carbon_intensity': best['carbon_intensity'],
            'carbon_savings_vs_worst': (
                (worst['carbon_intensity'] - best['carbon_intensity']) / 
                worst['carbon_intensity'] * 100
            ),
            'all_regions': results
        }
    
    def find_lowest_carbon_time_window(
        self,
        zone: str,
        duration_hours: int,
        within_hours: int = 24
    ) -> dict:
        """Find the best time window to run a workload in the next N hours."""
        response = requests.get(
            f"{self.ELECTRICITY_MAPS_API}/carbon-intensity/forecast",
            params={'zone': zone},
            headers=self.headers
        )
        
        if response.status_code != 200:
            return {'error': 'Forecast unavailable'}
        
        forecast = response.json()['forecast'][:within_hours]
        
        # Find window with lowest average carbon intensity
        best_window = None
        best_avg = float('inf')
        
        for i in range(len(forecast) - duration_hours + 1):
            window = forecast[i:i + duration_hours]
            avg_intensity = sum(w['carbonIntensity'] for w in window) / len(window)
            
            if avg_intensity < best_avg:
                best_avg = avg_intensity
                best_window = {
                    'start_time': window[0]['datetime'],
                    'end_time': window[-1]['datetime'],
                    'avg_carbon_intensity': avg_intensity
                }
        
        return best_window
```

---

## Measuring Cloud Carbon Footprint

### Cloud Provider Carbon Tools

**AWS Customer Carbon Footprint Tool:**
- Available in AWS Billing Console
- Shows Scope 1, 2, and 3 emissions
- Breaks down by service, region, and account
- Updated monthly with ~3-month lag

```python
import boto3

def get_aws_carbon_footprint(start_month: str, end_month: str) -> dict:
    """
    Retrieve AWS carbon footprint data.
    Requires AWS Customer Carbon Footprint Tool access.
    """
    ce = boto3.client('ce')
    
    response = ce.get_cost_and_usage(
        TimePeriod={'Start': start_month, 'End': end_month},
        Granularity='MONTHLY',
        Metrics=['CARBON_FOOTPRINT'],
        GroupBy=[
            {'Type': 'DIMENSION', 'Key': 'SERVICE'},
            {'Type': 'DIMENSION', 'Key': 'REGION'}
        ]
    )
    
    return response
```

**Azure Emissions Impact Dashboard:**
- Available in Azure Portal
- Shows carbon emissions by subscription, resource group, service
- Provides year-over-year comparison
- Integrates with Microsoft Sustainability Manager

**GCP Carbon Footprint:**
- Available in GCP Console under Billing
- Shows gross and net carbon emissions
- Breaks down by project, service, region
- Accounts for GCP's renewable energy purchases

### Carbon Calculation Methodology

```python
def estimate_carbon_footprint(
    compute_kwh: float,
    storage_kwh: float,
    network_kwh: float,
    region: str,
    carbon_intensity_gco2e_per_kwh: float,
    pue: float = 1.2  # Power Usage Effectiveness (cloud avg ~1.2)
) -> dict:
    """
    Estimate carbon footprint for cloud workloads.
    
    Formula: Carbon = Energy × PUE × Carbon Intensity
    """
    
    total_it_energy_kwh = compute_kwh + storage_kwh + network_kwh
    total_facility_energy_kwh = total_it_energy_kwh * pue
    
    # Gross emissions (before renewable energy credits)
    gross_emissions_kgco2e = total_facility_energy_kwh * (carbon_intensity_gco2e_per_kwh / 1000)
    
    # Cloud providers purchase renewable energy certificates (RECs)
    # AWS: 100% renewable energy matched (as of 2023)
    # GCP: 100% renewable energy matched (as of 2017)
    # Azure: 100% renewable energy matched (as of 2025 target)
    renewable_matching_pct = 1.0  # Assume 100% for major cloud providers
    
    net_emissions_kgco2e = gross_emissions_kgco2e * (1 - renewable_matching_pct)
    
    return {
        'region': region,
        'total_energy_kwh': total_facility_energy_kwh,
        'gross_emissions_kgco2e': gross_emissions_kgco2e,
        'gross_emissions_mtco2e': gross_emissions_kgco2e / 1000,
        'net_emissions_kgco2e': net_emissions_kgco2e,
        'renewable_matching_pct': renewable_matching_pct * 100,
        'carbon_intensity': carbon_intensity_gco2e_per_kwh
    }
```

---

## Green Software Engineering Principles

The [Green Software Foundation](https://greensoftware.foundation) defines three pillars of green software:

### 1. Carbon Efficiency
Do more with less carbon. Minimize the carbon emitted per unit of work.

**Practices:**
- Choose energy-efficient programming languages and runtimes
- Optimize algorithms to reduce CPU cycles
- Use efficient data formats (Parquet vs CSV, Protocol Buffers vs JSON)
- Implement caching to avoid redundant computation

**Language energy efficiency (relative):**

| Language | Energy Efficiency | Notes |
|---|---|---|
| C/C++ | Highest | ~1x baseline |
| Rust | Very high | ~1.03x |
| Java | High | ~1.98x |
| Go | High | ~3.23x |
| Python | Lower | ~75x (use for orchestration, not compute) |
| JavaScript | Moderate | ~4.45x |

### 2. Energy Efficiency
Use the minimum energy required to perform a task.

```python
# Energy-efficient batch processing pattern
# Process in batches to maximize CPU utilization and minimize idle time

import asyncio
from typing import AsyncIterator

async def energy_efficient_batch_processor(
    items: list,
    batch_size: int = 100,
    max_concurrent_batches: int = 4
) -> AsyncIterator[list]:
    """
    Process items in batches to maximize CPU utilization.
    Avoids spinning up compute for each individual item.
    """
    semaphore = asyncio.Semaphore(max_concurrent_batches)
    
    async def process_batch(batch: list) -> list:
        async with semaphore:
            # Process batch — CPU stays busy, no idle cycles
            return await do_work(batch)
    
    # Create batches
    batches = [items[i:i+batch_size] for i in range(0, len(items), batch_size)]
    
    # Process concurrently with bounded parallelism
    tasks = [process_batch(batch) for batch in batches]
    results = await asyncio.gather(*tasks)
    
    for result in results:
        yield result
```

### 3. Hardware Efficiency
Use the minimum hardware required. Avoid over-provisioning.

**Practices:**
- Right-size instances to actual workload requirements
- Use serverless for variable workloads (no idle hardware)
- Prefer newer, more efficient instance generations
- Use ARM-based instances (Graviton, Ampere) — 20–40% better performance per watt

---

## Workload Scheduling for Carbon Efficiency

### Temporal Shifting

Move non-time-sensitive workloads to periods of lower carbon intensity:

```python
import boto3
from datetime import datetime, timedelta
import pytz

def schedule_carbon_aware_job(
    job_name: str,
    job_definition: str,
    job_queue: str,
    preferred_carbon_window: str = 'low',  # 'low', 'medium', 'any'
    deadline_hours: int = 24
) -> dict:
    """
    Schedule a batch job to run during low-carbon periods.
    Uses AWS Batch with EventBridge for scheduling.
    """
    
    batch = boto3.client('batch')
    events = boto3.client('events')
    
    # For simplicity, schedule during off-peak hours (typically lower carbon)
    # In production, integrate with Electricity Maps API for real-time data
    
    now = datetime.now(pytz.UTC)
    
    # Low carbon windows (approximate, region-dependent):
    # - Overnight (00:00-06:00 UTC) — lower demand, more renewable
    # - Midday (11:00-15:00 UTC) — peak solar generation
    
    if preferred_carbon_window == 'low':
        # Schedule for next overnight window
        next_low_carbon = now.replace(hour=2, minute=0, second=0, microsecond=0)
        if next_low_carbon < now:
            next_low_carbon += timedelta(days=1)
        
        schedule_time = next_low_carbon
    else:
        schedule_time = now + timedelta(minutes=5)
    
    # Create EventBridge rule for one-time execution
    rule_name = f"carbon-aware-{job_name}-{now.strftime('%Y%m%d%H%M%S')}"
    
    events.put_rule(
        Name=rule_name,
        ScheduleExpression=f"cron({schedule_time.minute} {schedule_time.hour} {schedule_time.day} {schedule_time.month} ? {schedule_time.year})",
        State='ENABLED',
        Description=f"Carbon-aware schedule for {job_name}"
    )
    
    return {
        'job_name': job_name,
        'scheduled_time': schedule_time.isoformat(),
        'carbon_window': preferred_carbon_window,
        'rule_name': rule_name
    }
```

### Spatial Shifting

Route workloads to lower-carbon regions when latency requirements allow:

```hcl
# Terraform: Multi-region deployment with carbon-aware routing
resource "aws_route53_health_check" "carbon_aware" {
  # Route traffic to lower-carbon region when both are healthy
  # Oregon (us-west-2) has lower carbon intensity than Virginia (us-east-1)
  
  fqdn              = "api.us-west-2.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 30
}

resource "aws_route53_record" "api_carbon_aware" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "api.example.com"
  type    = "A"

  # Weighted routing: prefer lower-carbon Oregon (70%) over Virginia (30%)
  set_identifier = "us-west-2-primary"
  weight         = 70

  alias {
    name                   = aws_lb.us_west_2.dns_name
    zone_id                = aws_lb.us_west_2.zone_id
    evaluate_target_health = true
  }
}
```

---

## Cloud Provider Sustainability Tools

### AWS Sustainability Tools

| Tool | Purpose | Access |
|---|---|---|
| Customer Carbon Footprint Tool | View emissions by service/region | AWS Billing Console |
| AWS Graviton | ARM-based instances, 20-40% better perf/watt | EC2 instance selection |
| AWS Spot Instances | Utilize spare capacity, reduce waste | EC2 pricing |
| AWS Compute Optimizer | Rightsize to reduce energy waste | AWS Console |
| Amazon CodeGuru Profiler | Identify inefficient code | Developer tools |

### Azure Sustainability Tools

| Tool | Purpose | Access |
|---|---|---|
| Emissions Impact Dashboard | View carbon emissions | Azure Portal |
| Microsoft Sustainability Manager | Enterprise sustainability reporting | Microsoft Cloud |
| Azure Carbon Optimization | Recommendations to reduce emissions | Azure Advisor |
| Azure Spot VMs | Utilize spare capacity | VM pricing |

### GCP Sustainability Tools

| Tool | Purpose | Access |
|---|---|---|
| Carbon Footprint | View emissions by project/service | GCP Console |
| Active Assist | Sustainability recommendations | GCP Console |
| Carbon-free energy percentage | Real-time renewable % by region | GCP documentation |
| Spot VMs | Utilize spare capacity | Compute Engine |

---

## Reporting Scope 2 and Scope 3 Emissions

### GHG Protocol Scopes

| Scope | Definition | Cloud Relevance |
|---|---|---|
| Scope 1 | Direct emissions from owned sources | Data center generators (on-prem only) |
| Scope 2 | Indirect emissions from purchased electricity | Cloud compute energy consumption |
| Scope 3 | All other indirect emissions in value chain | Cloud provider manufacturing, employee travel |

### Scope 2 Reporting for Cloud

Cloud compute falls under **Scope 2** emissions for your organization:

```python
def calculate_scope2_emissions(
    monthly_cloud_costs: dict,  # {service: cost}
    emission_factors: dict,     # {service: kgCO2e per $}
    reporting_period: str
) -> dict:
    """
    Calculate Scope 2 emissions from cloud usage.
    
    Note: Use cloud provider's official carbon footprint tools
    for accurate reporting. This is an estimation approach.
    """
    
    total_emissions_kgco2e = 0
    service_breakdown = {}
    
    for service, cost in monthly_cloud_costs.items():
        # Emission factor: approximate kgCO2e per $ of cloud spend
        # These vary significantly by region and service
        factor = emission_factors.get(service, 0.1)  # Default: 0.1 kgCO2e/$
        emissions = cost * factor
        
        service_breakdown[service] = {
            'cost': cost,
            'emission_factor': factor,
            'emissions_kgco2e': emissions,
            'emissions_mtco2e': emissions / 1000
        }
        total_emissions_kgco2e += emissions
    
    return {
        'reporting_period': reporting_period,
        'scope': 'Scope 2',
        'total_emissions_kgco2e': total_emissions_kgco2e,
        'total_emissions_mtco2e': total_emissions_kgco2e / 1000,
        'service_breakdown': service_breakdown,
        'methodology': 'Spend-based estimation (use provider tools for accuracy)'
    }
```

### CDP and ESG Reporting

For formal sustainability reporting (CDP, GRI, TCFD):

1. **Use official provider data** — AWS Customer Carbon Footprint Tool, Azure Emissions Impact Dashboard, GCP Carbon Footprint
2. **Apply market-based method** — accounts for renewable energy certificates (RECs) purchased by cloud providers
3. **Document methodology** — auditors require clear methodology documentation
4. **Include uncertainty ranges** — cloud carbon data has inherent uncertainty

---

## Sustainability KPIs

Track these metrics to measure and improve your cloud sustainability performance:

### Primary KPIs

| KPI | Definition | Target | Frequency |
|---|---|---|---|
| Carbon intensity | kgCO₂e per $1,000 cloud spend | Trending down | Monthly |
| Carbon per business unit | kgCO₂e per MAU / transaction | Trending down | Monthly |
| Renewable energy % | % of compute in renewable-powered regions | > 80% | Monthly |
| Carbon-efficient region usage | % of workloads in low-carbon regions | > 60% | Monthly |
| Idle resource carbon waste | kgCO₂e from idle/unused resources | Trending to zero | Weekly |

### Secondary KPIs

| KPI | Definition | Target |
|---|---|---|
| Graviton/ARM adoption | % of compute on ARM instances | > 40% |
| Spot instance usage | % of eligible workloads on spot | > 40% |
| Storage tiering | % of cold data in archive tiers | > 70% |
| Carbon-aware scheduling | % of batch jobs scheduled for low-carbon windows | > 50% |
| Rightsizing coverage | % of resources at optimal size | > 80% |

### Sustainability Dashboard

```
┌─────────────────────────────────────────────────────────────┐
│  Cloud Sustainability Dashboard — Q1 2024                   │
├─────────────────┬───────────────────┬───────────────────────┤
│ Total Emissions │ Carbon/MAU        │ Renewable Energy %    │
│ 45.2 tCO₂e     │ 0.82 kgCO₂e      │ 78%                   │
│ ↓ -12% YoY     │ ↓ -18% YoY       │ ↑ +8% YoY            │
├─────────────────┴───────────────────┴───────────────────────┤
│  Emissions by Region                                        │
│  us-west-2 (Oregon):    12.1 tCO₂e  ████████ (27%)        │
│  eu-west-1 (Ireland):   18.4 tCO₂e  ████████████ (41%)    │
│  us-east-1 (Virginia):  14.7 tCO₂e  ██████████ (32%)      │
├─────────────────────────────────────────────────────────────┤
│  Optimization Opportunities                                 │
│  → Move 30% of us-east-1 batch workloads to us-west-2      │
│    Estimated savings: 4.2 tCO₂e/month                      │
│  → Enable Graviton for 15 eligible EC2 instances           │
│    Estimated savings: 1.8 tCO₂e/month                      │
│  → Schedule ML training for overnight low-carbon windows   │
│    Estimated savings: 2.1 tCO₂e/month                      │
└─────────────────────────────────────────────────────────────┘
```

---

## The FinOps-Sustainability Virtuous Cycle

Cost optimization and carbon reduction reinforce each other:

```
Eliminate waste          → Less compute running → Less energy consumed
Rightsize resources      → Less over-provisioning → Less energy wasted
Use spot instances       → Utilize spare capacity → Better grid efficiency
Adopt serverless         → Scale to zero → No idle energy consumption
Optimize data transfer   → Less network traffic → Less energy in transit
Use efficient regions    → Lower carbon intensity → Same work, less carbon
```

**Key insight:** A 20% reduction in cloud spend typically corresponds to a 15–25% reduction in carbon emissions. FinOps and sustainability are not competing priorities — they are the same priority expressed in different currencies.
