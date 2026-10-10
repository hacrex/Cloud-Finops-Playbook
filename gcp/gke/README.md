# Google Kubernetes Engine (GKE) Cost Optimization

**Engineering-first strategies for optimizing costs on Google Kubernetes Engine.**

## 🎯 Overview

Google Kubernetes Engine (GKE) provides powerful managed Kubernetes capabilities, but costs can quickly escalate without proper governance. This guide covers practical optimization techniques for GKE from an engineering perspective.

## 💰 Key Cost Drivers in GKE

### 1. **Compute Costs** (65-75% of total)
- Node VM instances (vCPU, memory)
- Node pool configuration
- OS disk type and size
- GPUs for ML workloads

### 2. **Networking** (10-15%)
- Cloud NAT egress charges
- Load Balancer costs
- Inter-zone traffic
- External IP addresses

### 3. **Storage** (10-15%)
- Persistent Disk (SSD vs HDD)
- Snapshot storage
- Filestore instances

### 4. **Add-ons & Services** (5-10%)
- Cloud Monitoring & Logging
- Anthos features
- Security scanning

## 🚀 Optimization Strategies

### 1. Cluster Autoscaler & Node Auto-Provisioning

#### Enable Cluster Autoscaler
```bash
gcloud container clusters update my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --region us-central1
```

#### Configure Node Auto-Provisioning (NAP)
```bash
gcloud container clusters update my-cluster \
  --enable-autoprovisioning \
  --min-cpu=1 \
  --max-cpu=100 \
  --min-memory=1 \
  --max-memory=200 \
  --autoprovisioning-service-account=nap-sa@project.iam.gserviceaccount.com
```

#### Custom NAP Profile
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: nap-quota
  namespace: default
spec:
  hard:
    requests.cpu: "50"
    requests.memory: 100Gi
    limits.cpu: "100"
    limits.memory: 200Gi
```

### 2. Preemptible & Spot VMs

#### Create Preemptible Node Pool
```bash
gcloud container node-pools create preemptible-pool \
  --cluster=my-cluster \
  --preemptible \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --region=us-central1
```

#### Taint Preemptible Nodes
```yaml
# Prevent critical workloads from scheduling on preemptible nodes
tolerations:
  - key: "preemptible"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

**Cost Savings**: Preemptible VMs offer up to **91% discount** compared to standard VMs.

### 3. Committed Use Discounts (CUDs)

#### Purchase CUDs for Stable Workloads
```bash
# View current machine type usage
gcloud compute machine-types list \
  --filter="zone:us-central1-a" \
  --format="table(name,memoryGb,guestCpus)"

# Calculate potential savings with CUDs
# 1-year commitment: ~37% discount
# 3-year commitment: ~52% discount
```

**Best Practices:**
- Analyze 30-day usage patterns before committing
- Start with 1-year commitments for uncertain workloads
- Combine with preemptible VMs for hybrid approach
- Use committed use contracts for baseline capacity

### 4. Rightsizing Recommendations

#### Enable Recommendation Hub
```bash
# Get GKE rightsizing recommendations
gcloud recommender recommendations list \
  --location=global \
  --filter="category=COST" \
  --format="table(name,state,priority,content.operationGroups.operations)"
```

#### Apply Recommendations Programmatically
```python
#!/usr/bin/env python3
# apply-rightsizing.py

from google.cloud import recommender_v1

def apply_rightsizing_recommendations(project_id):
    client = recommender_v1.RecommenderClient()
    parent = f"projects/{project_id}/locations/global/recommenders/google.compute.instance.MachineTypeRecommender"
    
    recommendations = client.list_recommendations(parent=parent)
    
    for rec in recommendations:
        if rec.state == recommender_v1.Recommendation.State.ACTIVE:
            # Apply the recommendation
            operation = rec.content.operation_groups[0].operations[0]
            print(f"Applying: {operation.action}")
            
            # Mark as claimed
            client.mark_recommendation_as_claimed(
                name=rec.name
            )
```

### 5. Storage Optimization

#### Choose Right Storage Class
```yaml
# Standard PD (balanced cost/performance)
storageClassName: standard

# SSD PD (high performance, higher cost)
storageClassName: premium-rwo

# Regional PD (high availability)
storageClassName: regional-pd

# Use ephemeral storage for temporary data
ephemeral-storage-local-ssd: true
```

#### Implement Storage Limits
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limits
  namespace: production
spec:
  limits:
  - type: PersistentVolumeClaim
    max:
      storage: 500Gi
    min:
      storage: 1Gi
```

### 6. Network Cost Reduction

#### Optimize Cross-Zone Traffic
```bash
# Use regional clusters to reduce cross-zone costs
gcloud container clusters create my-regional-cluster \
  --region=us-central1 \
  --num-nodes=3 \
  --enable-ip-alias
```

#### Configure Private Google Access
```bash
# Reduce egress costs with private access
gcloud compute networks subnets update my-subnet \
  --region=us-central1 \
  --enable-private-ip-google-access
```

#### Use Cloud NAT Efficiently
```bash
# Create optimized NAT configuration
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

## 📊 Monitoring & Cost Allocation

### Cloud Billing Export to BigQuery

#### Enable Billing Export
```bash
# Enable billing export (run from billing account)
gcloud beta billing accounts enable-billing-export \
  --billing-account=XXXXXX-YYYYYY-ZZZZZZ \
  --bigquery-dataset=gke_costs \
  --bigquery-table=exports
```

#### Query GKE Costs
```sql
-- Monthly cost by namespace
SELECT
  project.id AS project,
  labels.key AS label_key,
  labels.value AS label_value,
  SUM(cost) AS total_cost
FROM `project.gke_costs.exports`
WHERE
  service.description = 'Kubernetes Engine'
  AND usage_start_time >= TIMESTAMP('2024-01-01')
GROUP BY project, label_key, label_value
ORDER BY total_cost DESC;
```

### Deploy OpenCost for Real-Time Visibility
```bash
# Install OpenCost via Helm
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost \
  --namespace opencost \
  --create-namespace \
  --set opencost.cloudProvider=GCP \
  --set opencost.prometheus.external.enabled=true
```

### Set Up Budget Alerts
```bash
gcloud billing budgets create \
  --billing-account=XXXXXX-YYYYYY-ZZZZZZ \
  --display-name="GKE Monthly Budget" \
  --amount=5000 \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=80 \
  --threshold-rule=percent=100 \
  --notification-email=team@example.com
```

## 🔧 Automation Scripts

### Automated Idle Pod Detection
```python
#!/usr/bin/env python3
# detect-idle-pods.py

from kubernetes import client, config
from datetime import datetime, timedelta

def find_idle_pods(namespace='default', cpu_threshold=0.1, memory_threshold=0.2):
    """Find pods with low resource utilization"""
    config.load_kube_config()
    metrics_api = client.CustomObjectsApi()
    
    idle_pods = []
    try:
        metrics = metrics_api.list_namespaced_custom_object(
            group="metrics.k8s.io",
            version="v1beta1",
            namespace=namespace,
            plural="pods"
        )
        
        for pod in metrics['items']:
            cpu_usage = float(pod.get('usage', {}).get('cpu', '0').replace('m', '')) / 1000
            memory_usage = int(pod.get('usage', {}).get('memory', '0').replace('Ki', '')) / (1024 * 1024)
            
            if cpu_usage < cpu_threshold and memory_usage < memory_threshold:
                idle_pods.append({
                    'name': pod['metadata']['name'],
                    'namespace': namespace,
                    'cpu': cpu_usage,
                    'memory': memory_usage
                })
        
        return idle_pods
    except Exception as e:
        print(f"Error: {e}")
        return []

if __name__ == "__main__":
    idle = find_idle_pods()
    print(f"Found {len(idle)} idle pods:")
    for pod in idle:
        print(f"  - {pod['namespace']}/{pod['name']} (CPU: {pod['cpu']}v, Mem: {pod['memory']}Gi)")
```

### Scheduled Cluster Shutdown for Dev Environments
```bash
#!/bin/bash
# shutdown-dev-clusters.sh

# Stop dev clusters outside business hours
DEV_CLUSTERS=("dev-cluster-1" "dev-cluster-2" "staging-cluster")
REGION="us-central1"

for cluster in "${DEV_CLUSTERS[@]}"; do
  echo "Stopping $cluster..."
  gcloud container clusters resize $cluster \
    --num-nodes=0 \
    --region=$REGION \
    --quiet
done

echo "All dev clusters stopped."
```

## 📈 Case Study: SaaS Platform Migration

### Before Optimization
- **Cluster**: 3 regional node pools, 45 nodes total
- **Machine Types**: All n1-standard-4 (over-provisioned)
- **Preemptible Usage**: 0%
- **Monthly Cost**: $28,000
- **Average Utilization**: 28%

### After Optimization
- Implemented Node Auto-Provisioning
- Migrated 60% of workloads to preemptible VMs
- Rightsized remaining nodes to e2-standard-2
- Purchased 1-year CUDs for baseline (20% of capacity)
- Optimized storage classes

### Results
- **New Cluster Size**: 15-35 nodes (auto-provisioned)
- **Monthly Cost**: $11,200 (**60% reduction**)
- **Average Utilization**: 72%
- **Preemptible Usage**: 60%
- **ROI**: Achieved in first month

## ✅ Optimization Checklist

- [ ] Enable cluster autoscaler with appropriate bounds
- [ ] Configure Node Auto-Provisioning (NAP)
- [ ] Use preemptible VMs for fault-tolerant workloads
- [ ] Purchase Committed Use Discounts for stable baseline
- [ ] Implement pod resource requests/limits
- [ ] Deploy OpenCost for cost visibility
- [ ] Enable billing export to BigQuery
- [ ] Set up budget alerts
- [ ] Optimize storage classes per workload
- [ ] Use regional clusters for HA requirements only
- [ ] Schedule dev/staging cluster shutdowns
- [ ] Review and apply rightsizing recommendations monthly

## 🔗 Related Resources

- [AWS EKS Cost Optimization](../../aws/compute/eks-cost-optimization.md)
- [Azure AKS FinOps](../../azure/aks/README.md)
- [Kubernetes Cost Optimization](../../kubernetes/kube-cost/README.md)
- [FinOps Basics](../../foundations/finops-basics.md)
- [GCP Pricing Calculator](https://cloud.google.com/products/calculator)
- [GKE Best Practices](https://cloud.google.com/kubernetes-engine/docs/best-practices)

## 🛠️ Tools Mentioned

| Tool | Purpose | Link |
|------|---------|------|
| Cloud Billing Reports | Native GCP cost tracking | [Console](https://console.cloud.google.com/billing) |
| OpenCost | Kubernetes cost monitoring | [GitHub](https://github.com/opencost/opencost) |
| Kubecost | Commercial cost optimization | [Website](https://www.kubecost.com/) |
| Recommender API | Rightsizing recommendations | [Docs](https://cloud.google.com/recommender/docs) |
| Goldilocks | VPA recommendations | [GitHub](https://github.com/FairwindsPolicies/goldilocks) |

---

**💡 Pro Tip**: GKE's Node Auto-Provisioning is a game-changer—let GCP automatically create optimal node pools based on your actual workload requirements instead of pre-defining them.
