# OpenStack FinOps

Manage quotas, chargeback, and private cloud resource optimization for OpenStack deployments.

## Why OpenStack FinOps Matters

Private clouds powered by OpenStack offer different cost dynamics than public clouds:

- **CapEx vs OpEx**: Upfront hardware investment vs pay-as-you-go
- **Fixed Capacity**: Limited by physical infrastructure
- **Shared Resources**: Multi-tenant cost allocation challenges
- **Operational Costs**: Power, cooling, datacenter space, staff

Effective FinOps practices help maximize ROI on your OpenStack investment.

## Key Cost Components

### 1. Infrastructure Costs

| Component | Cost Type | Considerations |
|-----------|----------|----------------|
| Compute Nodes | CapEx + OpEx | Server hardware, depreciation, power |
| Storage (Ceph) | CapEx + OpEx | Disk capacity, replication factor, SSD vs HDD |
| Network | CapEx + OpEx | Switches, NICs, bandwidth upgrades |
| Licensing | OpEx | Support contracts, enterprise features |
| Datacenter | OpEx | Rack space, power, cooling, network connectivity |

### 2. Operational Costs

- **Staff**: Cloud administrators, support engineers
- **Maintenance**: Hardware replacements, upgrades
- **Software Updates**: Version upgrades, security patches
- **Monitoring**: Observability tools and storage

## Quota Management

### Setting Effective Quotas

Prevent resource hoarding and ensure fair allocation:

```bash
# View current quotas for a project
openstack quota show my-project

# Set compute quotas
openstack quota set --instances 50 --cores 200 --ram 65536 my-project

# Set volume quotas
openstack quota set --volumes 20 --gigabytes 1000 my-project

# Set network quotas
openstack quota set --floating-ips 10 --networks 20 --ports 100 my-project
```

### Quota Best Practices

✅ **Do:**
- Set quotas based on actual needs, not maximum possible
- Review quotas quarterly and adjust based on utilization
- Implement quota hierarchy (project → department → organization)
- Monitor quota usage and alert before limits reached
- Allow quota borrowing/overcommit for burst workloads

❌ **Don't:**
- Set unlimited quotas (-1) except for special cases
- Forget to account for snapshot and backup storage
- Ignore floating IP quota (common source of waste)
- Set and forget - quotas need regular review

### Quota Optimization Script

```python
#!/usr/bin/env python3
"""
OpenStack Quota Analyzer
Identifies projects with low quota utilization
"""
from openstack import connection
import os

conn = connection.Connection(
    auth_url=os.environ['OS_AUTH_URL'],
    username=os.environ['OS_USERNAME'],
    password=os.environ['OS_PASSWORD'],
    project_name=os.environ['OS_PROJECT_NAME'],
    user_domain_name='Default',
    project_domain_name='Default'
)

def analyze_quota_utilization():
    """Check quota vs actual usage for all projects"""
    
    projects = list(conn.identity.projects())
    
    print(f"{'Project':<30} {'Resource':<15} {'Quota':<10} {'Used':<10} {'Util %':<10}")
    print("-" * 85)
    
    for project in projects:
        quotas = conn.compute.get_quota_set(project.id)
        servers = list(conn.compute.servers(all_projects=False, project_id=project.id))
        
        instance_util = (len(servers) / quotas.instances * 100) if quotas.instances > 0 else 0
        
        if instance_util < 30:  # Low utilization flag
            print(f"{project.name:<30} {'Instances':<15} "
                  f"{quotas.instances:<10} {len(servers):<10} {instance_util:.1f}% ⚠️")
        
        # Check RAM utilization
        total_ram = sum([
            conn.compute.get_server(server.id).flavor['original_name'] 
            for server in servers
        ])
        # Add RAM analysis logic here...

if __name__ == '__main__':
    analyze_quota_utilization()
```

## Chargeback & Showback

### Implementation Strategies

#### 1. Simple Showback (Visibility Only)

Show teams their resource consumption without actual billing:

```sql
-- Example: Monthly resource consumption report
SELECT 
    project.name as project_name,
    COUNT(instances.id) as instance_count,
    SUM(instances.vcpus) as total_vcpus,
    SUM(instances.ram_mb) as total_ram_mb,
    SUM(volumes.size_gb) as total_storage_gb
FROM instances
JOIN projects ON instances.project_id = projects.id
LEFT JOIN volumes ON instances.id = volumes.instance_id
WHERE instances.created_at >= DATE_SUB(NOW(), INTERVAL 1 MONTH)
GROUP BY projects.id;
```

#### 2. Full Chargeback (Internal Billing)

Assign costs to departments based on usage:

**Pricing Model Example:**

| Resource | Unit | Internal Rate | Notes |
|----------|------|---------------|-------|
| vCPU | per hour | $0.02 | Based on hardware depreciation |
| RAM | GB-hour | $0.005 | Memory cost allocation |
| Storage (SSD) | GB-month | $0.15 | Faster storage tier |
| Storage (HDD) | GB-month | $0.05 | Standard tier |
| Floating IP | per hour | $0.01 | Encourages release when unused |
| Network Egress | GB | $0.02 | External traffic only |

**Monthly Chargeback Report:**

```yaml
Project: data-science
Department: R&D
Period: January 2024

Compute:
  vCPU Hours: 144,000 × $0.02 = $2,880
  RAM GB-Hours: 921,600 × $0.005 = $4,608
  
Storage:
  SSD (2 TB): 2,000 × $0.15 = $300
  HDD (10 TB): 10,000 × $0.05 = $500
  
Network:
  Floating IPs (20 × 720 hrs): 14,400 × $0.01 = $144
  Egress (500 GB): 500 × $0.02 = $10

Total: $8,442
```

### Metering Configuration

Enable comprehensive metering in OpenStack:

```ini
# /etc/ceilometer/pipeline.yaml
publishers:
  - gnocchi://

sources:
  - name: cpu_meter
    interval: 60
    meters:
      - cpu
      - cpu.util
  - name: memory_meter
    interval: 60
    meters:
      - memory.usage
      - memory.util
  - name: disk_meter
    interval: 300
    meters:
      - disk.root.size
      - disk.ephemeral.size
```

## Resource Optimization

### 1. Instance Rightsizing

Analyze actual usage vs allocated resources:

```bash
# Get instance list with flavors
openstack server list --long -c ID -c Name -c Flavor -c Project

# Check instance utilization via metrics
openstack metrics measure --resource-id $INSTANCE_ID
```

**Rightsizing Recommendations:**

| Current Flavor | Actual Usage | Recommended | Savings |
|---------------|--------------|-------------|---------|
| m1.xlarge (8 vCPU, 16GB) | 2 vCPU, 4GB | m1.small (2 vCPU, 4GB) | 75% |
| m1.2xlarge (16 vCPU, 32GB) | 4 vCPU, 8GB | m1.medium (4 vCPU, 8GB) | 75% |

### 2. Volume Cleanup

Identify and remove unused volumes:

```bash
# List available (unattached) volumes
openstack volume list --status available

# Show volume details including creation date
openstack volume show $VOLUME_ID -c CreatedAt -c Size -c Status

# Delete old unattached volumes (careful!)
openstack volume delete $VOLUME_ID
```

**Automated Cleanup Policy:**
- Volumes unattached for > 30 days: Warning notification
- Volumes unattached for > 60 days: Scheduled for deletion
- Volumes unattached for > 90 days: Auto-delete (with backup)

### 3. Floating IP Optimization

Floating IPs are often wasted:

```bash
# Find unassociated floating IPs
openstack floating ip list --status DOWN

# Show floating IP usage by project
openstack floating ip list -c 'Project ID' -c 'Floating IP Address' -c Status
```

**Best Practices:**
- Use security groups instead of floating IPs where possible
- Implement auto-release for unused floating IPs
- Charge premium rates for floating IPs to encourage efficiency
- Use NAT gateways for outbound connectivity

### 4. Image Management

Old images consume significant storage:

```bash
# List images sorted by size
openstack image list --sort key=size:desc

# Find images not used in last 6 months
openstack image list --sort key=created_at:asc

# Delete deprecated images
openstack image delete $IMAGE_ID
```

**Image Lifecycle Policy:**
- Development images: Retain for 30 days
- Production images: Retain last 3 versions
- Base images: Review quarterly
- Mark deprecated images before deletion

## Capacity Planning

### Monitoring Cluster Utilization

Track overall cluster health and predict capacity needs:

```bash
# View compute node resource usage
openstack hypervisor show $HYPERVISOR_ID

# Check total cluster capacity
openstack hypervisor stats show

# View service status
openstack compute service list
```

### Capacity Planning Dashboard

Key metrics to track:

| Metric | Current | Threshold | Action |
|--------|---------|-----------|--------|
| CPU Utilization | 65% | > 80% | Plan expansion |
| RAM Utilization | 72% | > 85% | Add compute nodes |
| Storage Utilization | 58% | > 75% | Add Ceph OSDs |
| Network Utilization | 45% | > 70% | Upgrade switches |

### Growth Forecasting

```python
#!/usr/bin/env python3
"""
Simple capacity forecasting based on historical growth
"""
import pandas as pd
from datetime import datetime, timedelta

def forecast_capacity(current_usage, growth_rate_monthly, months_ahead=6):
    """
    Predict future capacity needs
    
    Args:
        current_usage: Current resource usage (e.g., vCPUs in use)
        growth_rate_monthly: Monthly growth rate (e.g., 0.05 for 5%)
        months_ahead: How many months to forecast
    
    Returns:
        DataFrame with monthly projections
    """
    months = pd.date_range(start=datetime.now(), periods=months_ahead+1, freq='M')
    projections = [current_usage * (1 + growth_rate_monthly) ** i 
                   for i in range(months_ahead+1)]
    
    df = pd.DataFrame({
        'Month': months,
        'Projected Usage': projections,
        'Growth': [0] + [projections[i] - projections[i-1] 
                        for i in range(1, len(projections))]
    })
    
    return df

# Example: Forecast vCPU needs
current_vcpus = 1000
monthly_growth = 0.08  # 8% monthly growth

forecast = forecast_capacity(current_vcpus, monthly_growth)
print(forecast)

# Output tells you when to order new hardware
```

## Cost Comparison: OpenStack vs Public Cloud

### TCO Analysis Example

**Workload:** 100 VMs (4 vCPU, 16GB RAM each), 50TB storage

**OpenStack (3-year TCO):**
```
Hardware (compute nodes):     $180,000
Storage (Ceph cluster):       $120,000
Networking:                   $40,000
Datacenter (power/cooling):   $90,000
Staff (2 FTE):                $360,000
Support contract:             $60,000
------------------------------------
Total 3-year TCO:             $850,000
Annual cost:                  $283,333
Cost per VM/month:            ~$236
```

**AWS Equivalent (3-year):**
```
m5.xlarge × 100:              $56,064/year × 3 = $168,192
EBS Storage 50TB:             $4,000/month × 36 = $144,000
Data transfer:                $2,000/month × 36 = $72,000
------------------------------------
Total 3-year:                 $384,192
Annual cost:                  $128,064
Cost per VM/month:            ~$107
```

**Break-even Analysis:**
- OpenStack becomes cheaper at ~200+ VMs
- Better economics for predictable, steady workloads
- Public cloud better for variable/bursty workloads

## Automation & Best Practices

### Automated Reporting

Set up monthly cost reports:

```bash
#!/bin/bash
# monthly-report.sh

# Generate resource usage report
openstack server list --project $PROJECT_ID -f json > instances.json
openstack volume list --project $PROJECT_ID -f json > volumes.json
openstack floating ip list --project $PROJECT_ID -f json > floating_ips.json

# Calculate costs using custom script
python3 calculate_costs.py \
  --instances instances.json \
  --volumes volumes.json \
  --floating-ips floating_ips.json \
  --output report-$MONTH.json

# Send to stakeholders
sendmail finance@company.com < report-$MONTH.txt
```

### Tagging Strategy

Implement consistent tagging for cost allocation:

```bash
# Required tags for all resources
openstack server set \
  --property cost_center=engineering \
  --property environment=production \
  --property owner=john.doe \
  --property project=myapp \
  $SERVER_ID
```

**Standard Tags:**
- `cost_center`: Department or budget code
- `environment`: production/staging/development
- `owner`: Responsible person/team
- `project`: Application or initiative name
- `retention`: Data retention policy (for volumes)

## See Also

- [OpenStack Capacity Planning](./capacity-planning/)
- [Ceph Storage Optimization](./ceph/)
- [Private Cloud FinOps Best Practices](../../foundations/private-cloud-finops.md)
- [Chargeback Models](../chargeback/)

## Resources

- [OpenStack Operations Guide](https://docs.openstack.org/operations-guide/)
- [OpenStack Administrator Guide](https://docs.openstack.org/admin-guide/)
- [Ceph Performance Tuning](https://docs.ceph.com/en/latest/performance/)
- [OpenStack Cost Calculator](https://github.com/openstack/oslo.config)
