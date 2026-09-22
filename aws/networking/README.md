# AWS Networking Cost Optimization

Networking costs are notoriously opaque — they don't show up until you get the bill. Data transfer charges can easily reach 20–30% of total AWS spend for data-intensive workloads. Understanding the pricing model is the first step to controlling it.

---

## Data Transfer Cost Overview

```
AWS Data Transfer Pricing (approximate, us-east-1):

Inbound (ingress):
  Internet → AWS:           FREE
  AWS region → AWS region:  FREE (inbound)

Outbound (egress):
  AWS → Internet:           $0.09/GB (first 10TB), $0.085/GB (next 40TB)
  EC2 → S3 (same region):   FREE (via Gateway Endpoint)
  EC2 → S3 (cross-region):  $0.02/GB
  EC2 AZ-A → EC2 AZ-B:      $0.01/GB each way = $0.02/GB round trip
  EC2 → EC2 (same AZ):      FREE (private IP)
  EC2 → CloudFront:         FREE (origin fetch)
  CloudFront → Internet:    $0.0085–$0.085/GB (varies by region)
```

### Inter-AZ Transfer: The Hidden Cost

Inter-AZ traffic at $0.01/GB each way is one of the most common surprise costs.

```python
import boto3
from datetime import datetime, timedelta

def estimate_inter_az_costs(region: str = "us-east-1", days: int = 30) -> dict:
    """Estimate inter-AZ data transfer costs from VPC Flow Logs."""
    ce = boto3.client("ce", region_name="us-east-1")
    
    end = datetime.utcnow().date()
    start = end - timedelta(days=days)
    
    resp = ce.get_cost_and_usage(
        TimePeriod={"Start": str(start), "End": str(end)},
        Granularity="MONTHLY",
        Filter={
            "And": [
                {"Dimensions": {"Key": "SERVICE", "Values": ["Amazon Elastic Compute Cloud - Compute"]}},
                {"Dimensions": {"Key": "USAGE_TYPE", "Values": ["DataTransfer-Regional-Bytes"]}},
            ]
        },
        Metrics=["UnblendedCost", "UsageQuantity"],
    )
    
    total_cost = sum(
        float(p["Total"]["UnblendedCost"]["Amount"])
        for p in resp["ResultsByTime"]
    )
    total_gb = sum(
        float(p["Total"]["UsageQuantity"]["Amount"])
        for p in resp["ResultsByTime"]
    )
    
    return {
        "inter_az_cost_usd": round(total_cost, 2),
        "inter_az_gb": round(total_gb, 2),
        "monthly_rate": round(total_cost / (days / 30), 2),
    }
```

**Reducing inter-AZ costs:**
- Deploy services in the same AZ when latency allows (use AZ-aware routing)
- Use private IP addresses for EC2-to-EC2 communication (public IPs always charge)
- For EKS/ECS: use topology-aware routing to prefer same-AZ pod communication
- Consider single-AZ deployment for dev/test environments

---

## NAT Gateway Cost Optimization

NAT Gateway is frequently the #1 or #2 networking cost item. It charges both for the gateway itself and per-GB processed.

```
NAT Gateway pricing:
  Hourly: $0.045/hour per gateway (~$32.40/month)
  Data processing: $0.045/GB processed
  
Example: 1TB/month through NAT Gateway
  = $32.40 (hourly) + $46.08 (data) = $78.48/month per AZ
  3 AZs = $235/month just for NAT
```

### Option 1: VPC Gateway Endpoints (Free for S3 and DynamoDB)

```hcl
# S3 Gateway Endpoint — eliminates NAT charges for S3 traffic
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.s3"
  vpc_endpoint_type = "Gateway"

  route_table_ids = [
    aws_route_table.private_az1.id,
    aws_route_table.private_az2.id,
    aws_route_table.private_az3.id,
  ]

  tags = { Name = "s3-gateway-endpoint" }
}

# DynamoDB Gateway Endpoint — also free
resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.us-east-1.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private_az1.id]
}
```

### Option 2: NAT Instance (for low-traffic scenarios)

```hcl
# NAT Instance: ~$3.50/month (t4g.nano) vs $32.40/month (NAT Gateway)
# Trade-off: manual management, no auto-scaling, single point of failure
resource "aws_instance" "nat" {
  ami                    = data.aws_ami.nat.id  # amzn-ami-vpc-nat
  instance_type          = "t4g.nano"           # $0.0042/hr
  subnet_id              = aws_subnet.public.id
  source_dest_check      = false                # Required for NAT
  vpc_security_group_ids = [aws_security_group.nat.id]

  tags = { Name = "nat-instance" }
}

resource "aws_route" "private_nat" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  network_interface_id   = aws_instance.nat.primary_network_interface_id
}
```

**Decision guide:**
- < 100GB/month through NAT: NAT Instance (t4g.nano) saves ~$25/month
- 100GB–1TB/month: NAT Gateway (reliability + managed)
- > 1TB/month: Audit what's going through NAT — likely S3/DynamoDB traffic that should use Gateway Endpoints

---

## VPC Endpoints: Gateway vs Interface

| Type | Services | Cost | Use Case |
|---|---|---|---|
| Gateway | S3, DynamoDB | FREE | Always use these |
| Interface | 100+ AWS services | $0.01/hr + $0.01/GB | When NAT cost exceeds endpoint cost |

```hcl
# Interface endpoint for ECR (eliminates NAT charges for container pulls)
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# Interface endpoint cost: $0.01/hr × 3 AZs = $21.60/month
# Break-even: if ECR traffic through NAT > 21.60/0.045 = 480GB/month
```

**High-value interface endpoints for EKS clusters:**
- `ecr.api` + `ecr.dkr` — container image pulls
- `sts` — IAM token requests
- `logs` — CloudWatch Logs
- `monitoring` — CloudWatch metrics
- `secretsmanager` — secrets retrieval

---

## CloudFront for Egress Cost Reduction

CloudFront origin fetch from S3/EC2 is free. CloudFront → Internet is significantly cheaper than EC2 → Internet.

```
Cost comparison for 10TB/month public content:
  EC2 → Internet:        10,000 GB × $0.09 = $900/month
  CloudFront → Internet: 10,000 GB × $0.0085 = $85/month (US/EU)
  Savings: $815/month (90% reduction)
```

```hcl
resource "aws_cloudfront_distribution" "app" {
  origin {
    domain_name = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id   = "S3-assets"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-assets"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true  # Reduces transfer size

    cache_policy_id = "658327ea-f89d-4fab-a63d-7e88639e58f6"  # CachingOptimized
  }

  price_class = "PriceClass_100"  # US, Canada, Europe only — cheapest
  # PriceClass_200: adds Asia/Middle East
  # PriceClass_All: all edge locations (most expensive)

  enabled = true
}
```

---

## AWS Global Accelerator vs CloudFront

| Feature | Global Accelerator | CloudFront |
|---|---|---|
| Protocol | TCP/UDP | HTTP/HTTPS |
| Caching | No | Yes |
| Use case | Non-HTTP, gaming, IoT | Web content, APIs |
| Pricing | $0.025/hr + $0.015/GB | $0.0085–$0.085/GB |
| Cost for 10TB/month | ~$168 | ~$85 |
| Best for | Low-latency TCP apps | Web/API acceleration |

Use CloudFront for web workloads. Use Global Accelerator only for non-HTTP protocols or when you need static anycast IPs.

---

## Direct Connect vs VPN Cost Analysis

```
Direct Connect pricing (1 Gbps, us-east-1):
  Port fee: $0.30/hr = $216/month
  Data transfer out: $0.02/GB (vs $0.09/GB internet)
  
  Break-even: 216 / (0.09 - 0.02) = 3,086 GB/month (~3TB)
  
VPN pricing:
  VPN connection: $0.05/hr = $36/month
  Data transfer: standard internet rates ($0.09/GB)
  
  Best for: < 3TB/month, or when Direct Connect isn't available
```

```python
def direct_connect_roi(monthly_gb_transfer: float) -> dict:
    """Calculate Direct Connect ROI vs VPN."""
    
    # Direct Connect
    dx_port_monthly = 216  # 1 Gbps port
    dx_transfer_rate = 0.02  # per GB
    dx_total = dx_port_monthly + (monthly_gb_transfer * dx_transfer_rate)
    
    # VPN
    vpn_monthly = 36
    vpn_transfer_rate = 0.09
    vpn_total = vpn_monthly + (monthly_gb_transfer * vpn_transfer_rate)
    
    savings = vpn_total - dx_total
    
    return {
        "monthly_gb": monthly_gb_transfer,
        "direct_connect_cost": round(dx_total, 2),
        "vpn_cost": round(vpn_total, 2),
        "dx_saves_usd": round(savings, 2),
        "recommendation": "Direct Connect" if savings > 0 else "VPN",
        "break_even_gb": round(dx_port_monthly / (vpn_transfer_rate - dx_transfer_rate), 0),
    }

print(direct_connect_roi(5000))   # 5TB/month → DX saves $170/month
print(direct_connect_roi(1000))   # 1TB/month → VPN saves $146/month
```

---

## Transit Gateway Cost Optimization

```
Transit Gateway pricing:
  Attachment: $0.05/hr per attachment = $36/month per VPC
  Data processing: $0.02/GB
  
  10 VPCs × $36 = $360/month in attachment fees alone
```

**Optimization strategies:**
- Use VPC Peering for simple hub-spoke topologies (free, no per-GB charge)
- Consolidate VPCs where possible to reduce attachment count
- Use Transit Gateway only when you need transitive routing (VPC Peering doesn't support this)
- Enable Transit Gateway multicast only if needed (adds cost)

```hcl
# VPC Peering (free) vs Transit Gateway ($36/VPC/month)
resource "aws_vpc_peering_connection" "app_to_shared" {
  vpc_id        = aws_vpc.app.id
  peer_vpc_id   = aws_vpc.shared_services.id
  auto_accept   = true

  tags = { Name = "app-to-shared-peering" }
}
```

---

## Load Balancer Cost Optimization

| Load Balancer | Hourly | LCU/hour | Best For |
|---|---|---|---|
| ALB | $0.008 | $0.008 | HTTP/HTTPS, microservices |
| NLB | $0.006 | $0.006 | TCP/UDP, extreme performance |
| CLB (Classic) | $0.025 | $0.008/GB | Legacy — migrate away |
| Gateway LB | $0.004 | $0.004 | Network appliances |

```python
def find_idle_load_balancers(region: str = "us-east-1") -> list:
    """Find load balancers with no healthy targets or minimal traffic."""
    elbv2 = boto3.client("elbv2", region_name=region)
    cw = boto3.client("cloudwatch", region_name=region)
    
    idle_lbs = []
    lbs = elbv2.describe_load_balancers()["LoadBalancers"]
    
    for lb in lbs:
        # Check request count over last 7 days
        metrics = cw.get_metric_statistics(
            Namespace="AWS/ApplicationELB",
            MetricName="RequestCount",
            Dimensions=[{"Name": "LoadBalancer", "Value": lb["LoadBalancerArn"].split(":")[-1]}],
            StartTime=datetime.utcnow() - timedelta(days=7),
            EndTime=datetime.utcnow(),
            Period=604800,
            Statistics=["Sum"],
        )
        
        total_requests = sum(dp["Sum"] for dp in metrics["Datapoints"])
        
        if total_requests < 100:  # Essentially idle
            idle_lbs.append({
                "name": lb["LoadBalancerName"],
                "arn": lb["LoadBalancerArn"],
                "type": lb["Type"],
                "requests_7d": total_requests,
                "monthly_cost_estimate": 0.008 * 24 * 30,  # ALB base cost
            })
    
    return idle_lbs
```

---

## Data Transfer Cost Monitoring

```python
def get_data_transfer_costs_by_type(months: int = 1) -> dict:
    """Break down data transfer costs by type."""
    ce = boto3.client("ce", region_name="us-east-1")
    
    end = datetime.utcnow().date()
    start = end - timedelta(days=months * 30)
    
    resp = ce.get_cost_and_usage(
        TimePeriod={"Start": str(start), "End": str(end)},
        Granularity="MONTHLY",
        Filter={
            "Dimensions": {
                "Key": "USAGE_TYPE_GROUP",
                "Values": ["EC2: Data Transfer - Internet (Out)",
                           "EC2: Data Transfer - Region to Region (Out)",
                           "EC2: Data Transfer - CloudFront (Out)",
                           "EC2: Data Transfer - Inter AZ"],
            }
        },
        GroupBy=[{"Type": "DIMENSION", "Key": "USAGE_TYPE_GROUP"}],
        Metrics=["UnblendedCost", "UsageQuantity"],
    )
    
    breakdown = {}
    for period in resp["ResultsByTime"]:
        for group in period["Groups"]:
            transfer_type = group["Keys"][0]
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            gb = float(group["Metrics"]["UsageQuantity"]["Amount"])
            breakdown[transfer_type] = {"cost_usd": round(cost, 2), "gb": round(gb, 2)}
    
    return breakdown
```

**Monthly networking cost review checklist:**
- [ ] Check NAT Gateway data processing costs — are Gateway Endpoints deployed?
- [ ] Review inter-AZ transfer costs — can services be co-located?
- [ ] Audit idle load balancers
- [ ] Verify CloudFront is in front of public S3 buckets
- [ ] Check for cross-region data transfer — is it intentional?
- [ ] Review VPC endpoint coverage for high-traffic AWS services
