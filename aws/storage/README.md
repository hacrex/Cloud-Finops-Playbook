# AWS Storage Cost Optimization

Storage costs are often underestimated — S3 and EBS can quietly become top-5 cost drivers. The good news: storage optimization is largely automatable through lifecycle policies and right-sizing.

---

## S3 Storage Classes

| Storage Class | Use Case | Min Duration | Retrieval | Cost ($/GB/mo) |
|---|---|---|---|---|
| Standard | Frequently accessed data | None | Instant | ~$0.023 |
| Intelligent-Tiering | Unknown/changing access | None | Instant | ~$0.023 + monitoring fee |
| Standard-IA | Infrequent access, resilient | 30 days | Instant | ~$0.0125 |
| One Zone-IA | Infrequent, reproducible data | 30 days | Instant | ~$0.01 |
| Glacier Instant Retrieval | Archive, quarterly access | 90 days | Instant | ~$0.004 |
| Glacier Flexible Retrieval | Archive, hours retrieval OK | 90 days | Minutes–hours | ~$0.0036 |
| Glacier Deep Archive | Long-term archive, 12hr OK | 180 days | 12–48 hours | ~$0.00099 |

**Key cost considerations:**
- Standard-IA and One Zone-IA have a **per-GB retrieval fee** — high-read workloads can cost more than Standard
- Glacier classes have **minimum storage duration** — deleting early still charges for the minimum
- Intelligent-Tiering has a **per-object monitoring fee** (~$0.0025/1000 objects) — not worth it for small objects

---

## S3 Lifecycle Policies with Terraform

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id

  # Rule 1: Application logs — aggressive tiering
  rule {
    id     = "application-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"  # Glacier Instant Retrieval
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }

    expiration {
      days = 2555  # 7 years retention, then delete
    }

    noncurrent_version_transition {
      noncurrent_days = 7
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 30
    }
  }

  # Rule 2: ML training data — keep accessible longer
  rule {
    id     = "ml-training-data"
    status = "Enabled"

    filter {
      prefix = "training-data/"
    }

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 365
      storage_class = "GLACIER_IR"
    }
  }

  # Rule 3: Incomplete multipart uploads — often forgotten cost
  rule {
    id     = "abort-incomplete-multipart"
    status = "Enabled"

    filter {}  # Apply to entire bucket

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }

  # Rule 4: Delete old delete markers (versioned bucket cleanup)
  rule {
    id     = "cleanup-delete-markers"
    status = "Enabled"

    filter {}

    expiration {
      expired_object_delete_marker = true
    }
  }
}
```

---

## S3 Intelligent-Tiering: When It Makes Sense

Intelligent-Tiering automatically moves objects between access tiers based on usage patterns. It's not always the cheapest option.

```python
def should_use_intelligent_tiering(
    object_size_kb: float,
    access_frequency_per_month: float,
    objects_count: int,
) -> dict:
    """Determine if Intelligent-Tiering is cost-effective."""
    
    # Costs (approximate, us-east-1)
    standard_cost_per_gb = 0.023
    it_monitoring_per_1000_objects = 0.0025
    it_frequent_cost_per_gb = 0.023
    it_infrequent_cost_per_gb = 0.0125
    
    size_gb = (object_size_kb * objects_count) / (1024 * 1024)
    
    # Standard cost
    standard_monthly = size_gb * standard_cost_per_gb
    
    # IT cost: monitoring + storage (assume 50% moves to IA after 30 days)
    it_monitoring = (objects_count / 1000) * it_monitoring_per_1000_objects
    it_storage = (size_gb * 0.5 * it_frequent_cost_per_gb) + \
                 (size_gb * 0.5 * it_infrequent_cost_per_gb)
    it_monthly = it_monitoring + it_storage
    
    # IT makes sense when:
    # 1. Objects are > 128KB (smaller objects aren't eligible for tiering)
    # 2. Access patterns are genuinely mixed/unknown
    # 3. Monitoring fee doesn't exceed savings
    
    makes_sense = (
        object_size_kb >= 128 and
        it_monthly < standard_monthly and
        access_frequency_per_month < 1  # Less than monthly access
    )
    
    return {
        "standard_monthly_usd": round(standard_monthly, 4),
        "intelligent_tiering_monthly_usd": round(it_monthly, 4),
        "monthly_savings_usd": round(standard_monthly - it_monthly, 4),
        "use_intelligent_tiering": makes_sense,
        "reason": (
            "Object size < 128KB — IT won't tier these objects" if object_size_kb < 128
            else "IT saves money with mixed access patterns" if makes_sense
            else "Standard is cheaper for frequently accessed data"
        ),
    }

# Example: 1M objects, 50KB average, accessed weekly
result = should_use_intelligent_tiering(
    object_size_kb=50,
    access_frequency_per_month=4,
    objects_count=1_000_000,
)
print(result)
```

**Rule of thumb:** Use Intelligent-Tiering for objects > 128KB with unpredictable access patterns. Use explicit lifecycle rules for predictable patterns (logs, backups, archives).

---

## EBS Volume Optimization: gp2 → gp3 Migration

gp3 is **20% cheaper** than gp2 and provides better baseline performance. This is one of the easiest wins in AWS cost optimization.

| Feature | gp2 | gp3 |
|---|---|---|
| Price | $0.10/GB/month | $0.08/GB/month |
| Baseline IOPS | 3 IOPS/GB (max 16,000) | 3,000 IOPS (always) |
| Max IOPS | 16,000 | 16,000 |
| Baseline throughput | Up to 250 MB/s | 125 MB/s |
| Max throughput | 250 MB/s | 1,000 MB/s |
| IOPS burst | Yes (credit-based) | N/A (consistent) |

```python
import boto3

def migrate_gp2_to_gp3(region: str = "us-east-1", dry_run: bool = True) -> dict:
    """Find and migrate all gp2 volumes to gp3."""
    ec2 = boto3.client("ec2", region_name=region)
    
    paginator = ec2.get_paginator("describe_volumes")
    gp2_volumes = []
    
    for page in paginator.paginate(
        Filters=[{"Name": "volume-type", "Values": ["gp2"]}]
    ):
        for vol in page["Volumes"]:
            monthly_cost_gp2 = vol["Size"] * 0.10
            monthly_cost_gp3 = vol["Size"] * 0.08
            savings = monthly_cost_gp2 - monthly_cost_gp3
            
            gp2_volumes.append({
                "volume_id": vol["VolumeId"],
                "size_gb": vol["Size"],
                "state": vol["State"],
                "monthly_savings_usd": round(savings, 2),
                "instance_id": (
                    vol["Attachments"][0]["InstanceId"]
                    if vol["Attachments"] else "unattached"
                ),
            })
    
    total_savings = sum(v["monthly_savings_usd"] for v in gp2_volumes)
    print(f"Found {len(gp2_volumes)} gp2 volumes")
    print(f"Total monthly savings from migration: ${total_savings:.2f}")
    print(f"Annual savings: ${total_savings * 12:.2f}")
    
    if not dry_run:
        for vol in gp2_volumes:
            if vol["state"] == "in-use":
                ec2.modify_volume(
                    VolumeId=vol["volume_id"],
                    VolumeType="gp3",
                    # gp3 defaults: 3000 IOPS, 125 MB/s throughput
                    # Increase if workload needs more (still often cheaper than gp2)
                )
                print(f"Migrated {vol['volume_id']} to gp3")
    
    return {"volumes": gp2_volumes, "total_monthly_savings": total_savings}

# Dry run first
result = migrate_gp2_to_gp3(dry_run=True)
```

---

## EBS Snapshot Management and Cleanup

Orphaned and aged snapshots are a common source of hidden storage costs.

```python
def cleanup_ebs_snapshots(
    account_id: str,
    region: str = "us-east-1",
    retention_days: int = 90,
    dry_run: bool = True,
) -> dict:
    """Find and optionally delete old EBS snapshots."""
    ec2 = boto3.client("ec2", region_name=region)
    from datetime import timezone
    
    paginator = ec2.get_paginator("describe_snapshots")
    old_snapshots = []
    cutoff = datetime.now(timezone.utc) - timedelta(days=retention_days)
    
    for page in paginator.paginate(OwnerIds=[account_id]):
        for snap in page["Snapshots"]:
            if snap["StartTime"] < cutoff:
                # Check if snapshot is used by an AMI
                images = ec2.describe_images(
                    Filters=[{"Name": "block-device-mapping.snapshot-id",
                              "Values": [snap["SnapshotId"]]}]
                )
                
                if not images["Images"]:  # Not backing any AMI
                    old_snapshots.append({
                        "snapshot_id": snap["SnapshotId"],
                        "size_gb": snap["VolumeSize"],
                        "age_days": (datetime.now(timezone.utc) - snap["StartTime"]).days,
                        "estimated_monthly_cost": snap["VolumeSize"] * 0.05,
                    })
    
    total_cost = sum(s["estimated_monthly_cost"] for s in old_snapshots)
    print(f"Found {len(old_snapshots)} deletable snapshots")
    print(f"Estimated monthly savings: ${total_cost:.2f}")
    
    if not dry_run:
        for snap in old_snapshots:
            ec2.delete_snapshot(SnapshotId=snap["snapshot_id"])
    
    return {"snapshots": old_snapshots, "monthly_savings": total_cost}
```

---

## EFS vs FSx Cost Comparison

| Service | Use Case | Price | Performance |
|---|---|---|---|
| EFS Standard | Shared POSIX filesystem, multi-AZ | $0.30/GB/month | Elastic |
| EFS Standard-IA | Infrequently accessed shared files | $0.025/GB/month + retrieval | Elastic |
| EFS One Zone | Single-AZ shared filesystem | $0.16/GB/month | Elastic |
| FSx for Lustre | HPC, ML training, scratch | $0.14/GB/month (scratch) | Very high |
| FSx for Windows | Windows workloads, SMB | $0.13/GB/month | High |
| FSx for NetApp ONTAP | Enterprise NAS migration | $0.22/GB/month | High |

**EFS cost optimization:**
- Enable EFS Intelligent-Tiering (moves files to IA after 30 days of no access)
- Use EFS One Zone for dev/test (47% cheaper, single-AZ)
- Set lifecycle policy to transition to IA after 14 days for most workloads

```hcl
resource "aws_efs_file_system" "app" {
  lifecycle_policy {
    transition_to_ia                    = "AFTER_14_DAYS"
    transition_to_primary_storage_class = "AFTER_1_ACCESS"  # Move back on access
  }

  lifecycle_policy {
    transition_to_archive = "AFTER_90_DAYS"  # EFS Archive tier (even cheaper)
  }
}
```

---

## S3 Request Cost Optimization

S3 request costs can exceed storage costs for high-throughput workloads.

```python
def analyze_s3_request_costs(bucket_name: str, days: int = 30) -> dict:
    """Analyze S3 request costs using CloudWatch metrics."""
    cw = boto3.client("cloudwatch", region_name="us-east-1")
    
    end = datetime.utcnow()
    start = end - timedelta(days=days)
    
    metrics = {}
    for metric_name in ["GetRequests", "PutRequests", "ListRequests", "HeadRequests"]:
        resp = cw.get_metric_statistics(
            Namespace="AWS/S3",
            MetricName=metric_name,
            Dimensions=[
                {"Name": "BucketName", "Value": bucket_name},
                {"Name": "FilterId", "Value": "EntireBucket"},
            ],
            StartTime=start,
            EndTime=end,
            Period=86400,
            Statistics=["Sum"],
        )
        total = sum(dp["Sum"] for dp in resp["Datapoints"])
        metrics[metric_name] = total
    
    # Pricing: GET $0.0004/1000, PUT $0.005/1000, LIST $0.005/1000
    costs = {
        "get_cost": metrics.get("GetRequests", 0) * 0.0004 / 1000,
        "put_cost": metrics.get("PutRequests", 0) * 0.005 / 1000,
        "list_cost": metrics.get("ListRequests", 0) * 0.005 / 1000,
    }
    
    return {**metrics, **costs, "total_request_cost": sum(costs.values())}
```

**Request cost reduction strategies:**
- Use S3 Transfer Acceleration only when needed (adds cost)
- Batch small objects into larger ones where possible
- Use CloudFront in front of S3 to cache GET requests
- Avoid `LIST` operations in hot paths — cache directory listings

---

## Data Transfer Cost Reduction

```
S3 data transfer pricing:
  - S3 → Internet: $0.09/GB (first 10TB/month)
  - S3 → CloudFront: FREE
  - S3 → EC2 (same region): FREE
  - S3 → EC2 (different region): $0.02/GB
  - S3 → S3 (cross-region replication): $0.02/GB
```

**Key optimizations:**
1. Use S3 Gateway Endpoints (free) for EC2 → S3 traffic within VPC
2. Use CloudFront for public S3 content (eliminates egress charges)
3. Ensure EC2 and S3 are in the same region
4. Use S3 Batch Operations instead of per-object API calls for bulk operations

---

## Storage Cost Monitoring with Cost Explorer

```python
def get_storage_cost_breakdown(months: int = 3) -> dict:
    """Break down storage costs by service and usage type."""
    ce = boto3.client("ce", region_name="us-east-1")
    
    end = datetime.utcnow().date().replace(day=1)
    start = (end - timedelta(days=months * 31)).replace(day=1)
    
    resp = ce.get_cost_and_usage(
        TimePeriod={"Start": str(start), "End": str(end)},
        Granularity="MONTHLY",
        Filter={
            "Dimensions": {
                "Key": "SERVICE",
                "Values": ["Amazon Simple Storage Service", "Amazon Elastic Block Store",
                           "Amazon Elastic File System", "Amazon FSx"],
            }
        },
        GroupBy=[
            {"Type": "DIMENSION", "Key": "SERVICE"},
            {"Type": "DIMENSION", "Key": "USAGE_TYPE"},
        ],
        Metrics=["UnblendedCost"],
    )
    
    costs = {}
    for period in resp["ResultsByTime"]:
        month = period["TimePeriod"]["Start"][:7]
        costs[month] = {}
        for group in period["Groups"]:
            service = group["Keys"][0]
            usage_type = group["Keys"][1]
            cost = float(group["Metrics"]["UnblendedCost"]["Amount"])
            if cost > 0.01:
                costs[month][f"{service} | {usage_type}"] = round(cost, 2)
    
    return costs
```
