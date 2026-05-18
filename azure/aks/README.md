# Azure Kubernetes Service (AKS) Cost Optimization

**Comprehensive guide to optimizing costs on Azure Kubernetes Service with engineering-first strategies.**

## 🎯 Overview

Azure Kubernetes Service (AKS) offers powerful container orchestration, but without proper cost management, expenses can spiral quickly. This guide provides actionable strategies for optimizing AKS costs while maintaining performance and reliability.

## 💰 Key Cost Drivers in AKS

### 1. **Node Costs** (60-70% of total)
- VM size and family selection
- Node count and utilization
- OS disk type and size
- Ephemeral vs. managed disks

### 2. **Networking Costs** (15-20%)
- Load Balancer charges
- NAT Gateway egress
- Private Link endpoints
- VNet peering

### 3. **Storage Costs** (10-15%)
- Persistent Volume Claims
- Snapshot storage
- Backup solutions

### 4. **Add-ons & Services** (5-10%)
- Azure Monitor/Container Insights
- Policy enforcement
- Security Center integration

## 🚀 Optimization Strategies

### 1. Right-Sizing Node Pools

#### Analyze Current Utilization
```bash
# Install metrics-server if not already installed
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# View resource usage across pods
kubectl top pods --all-namespaces

# View node utilization
kubectl top nodes
```

#### Use Azure Advisor Recommendations
```bash
# Get AKS optimization recommendations
az advisor recommendation list --category cost --resource-type Microsoft.ContainerService/managedClusters
```

#### Implement Rightsizing with Kubectl
```yaml
# Example: Resource quotas for namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
```

### 2. Cluster Autoscaler Configuration

#### Enable and Configure Autoscaler
```bash
# Enable cluster autoscaler on existing node pool
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 10
```

#### Advanced Autoscaler Settings
```yaml
# Custom autoscaler profile
clusterAutoscalerProfile:
  balanceSimilarNodeGroups: "true"
  expander: least-waste
  maxEmptyBulkDelete: "10"
  maxGracefulTerminationSec: "600"
  scaleDownDelayAfterAdd: "10m"
  scaleDownUnneededTime: "10m"
  scaleDownUtilizationThreshold: "0.5"
  scanInterval: "10s"
```

### 3. Spot Instances for Workloads

#### Create Spot Node Pool
```bash
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name spotpool \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --priority Spot \
  --spot-max-price -1 \
  --enable-auto-scaling \
  --min-count 1 \
  --max-count 5
```

#### Taint Spot Nodes
```yaml
# Prevent critical workloads from scheduling on spot nodes
tolerations:
  - key: "kubernetes.azure.com/scalesetpriority"
    operator: "Equal"
    value: "spot"
    effect: "NoSchedule"
```

### 4. Reserved Instances & Savings Plans

#### Purchase Reserved VM Instances
```bash
# View VM sizes in your cluster
az vm list-skus --location eastus --size Standard_DS2_v2

# Calculate potential savings with reservations
# Up to 72% savings compared to pay-as-you-go
```

**Best Practices:**
- Commit to 1-year or 3-year reservations for stable workloads
- Use Azure Hybrid Benefit if you have Windows Server licenses
- Combine with Spot instances for mixed workload patterns

### 5. Storage Optimization

#### Choose Right Storage Tier
```yaml
# Use Standard SSD for most workloads
storageClassName: default

# Use Premium SSD only for I/O-intensive workloads
storageClassName: managed-premium

# Use Azure Files for shared storage needs
storageClassName: azurefile
```

#### Implement Storage Quotas
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
      storage: 100Gi
    min:
      storage: 1Gi
```

### 6. Network Cost Reduction

#### Optimize Egress Traffic
```bash
# Use Availability Zones in same region to reduce data transfer
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --zones 1 2 3
```

#### Implement NAT Gateway
```bash
# Create NAT Gateway for controlled egress
az network public-ip create \
  --resource-group myResourceGroup \
  --name myNATGatewayIP \
  --sku Standard

az network nat gateway create \
  --resource-group myResourceGroup \
  --name myNATGateway \
  --public-ip-addresses myNATGatewayIP
```

## 📊 Monitoring & Cost Allocation

### Azure Cost Management Integration

#### Set Up Budget Alerts
```bash
az consumption budget create \
  --resource-group myResourceGroup \
  --budget-name AKS-Monthly-Budget \
  --amount 5000 \
  --time-grain monthly \
  --start-date 2024-01-01 \
  --end-date 2024-12-31 \
  --notification actual-gt-80-percent \
  --notification-thresholds 80 100 \
  --contact-emails team@example.com
```

#### Tag Resources for Cost Allocation
```bash
# Apply tags to AKS cluster
az resource tag \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --resource-type Microsoft.ContainerService/managedClusters \
  --tags Environment=Production Team=Platform CostCenter=IT123
```

### OpenCost Deployment
```yaml
# Deploy OpenCost for Kubernetes-native cost monitoring
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost \
  --namespace opencost \
  --create-namespace \
  --set opencost.cloudProvider=Azure
```

## 🔧 Automation Scripts

### Automated Idle Resource Cleanup
```bash
#!/bin/bash
# cleanup-idle-nodes.sh

RESOURCE_GROUP="myResourceGroup"
CLUSTER_NAME="myAKSCluster"

# Get nodes with <10% CPU utilization for 1 hour
IDLE_NODES=$(kubectl get nodes \
  -o jsonpath='{range .items[?(@.status.allocatable.cpu<"1")]}{.metadata.name}{"\n"}{end}')

for node in $IDLE_NODES; do
  echo "Scaling down idle node: $node"
  az aks nodepool scale \
    --resource-group $RESOURCE_GROUP \
    --cluster-name $CLUSTER_NAME \
    --name default \
    --node-count $(($(az aks nodepool show \
      --resource-group $RESOURCE_GROUP \
      --cluster-name $CLUSTER_NAME \
      --name default \
      --query nodeCount -o tsv) - 1))
done
```

## 📈 Case Study: E-Commerce Platform

### Before Optimization
- **Cluster Size**: 20 nodes (Standard_D4s_v3)
- **Monthly Cost**: $12,000
- **Average Utilization**: 35%
- **Spot Usage**: 0%

### After Optimization
- Implemented cluster autoscaler (min: 8, max: 25)
- Migrated 40% of workloads to Spot instances
- Right-sized over-provisioned pods
- Used Reserved Instances for baseline capacity

### Results
- **New Cluster Size**: 8-15 nodes (auto-scaled)
- **Monthly Cost**: $5,800 (**52% reduction**)
- **Average Utilization**: 68%
- **Spot Usage**: 40%
- **Payback Period**: Immediate

## ✅ Optimization Checklist

- [ ] Enable cluster autoscaler with appropriate min/max
- [ ] Implement pod resource requests/limits
- [ ] Use Spot instances for fault-tolerant workloads
- [ ] Purchase Reserved Instances for baseline capacity
- [ ] Apply cost allocation tags
- [ ] Set up budget alerts
- [ ] Deploy OpenCost for visibility
- [ ] Review storage classes and remove unused PVCs
- [ ] Optimize network egress with NAT Gateway
- [ ] Schedule non-production cluster shutdowns

## 🔗 Related Resources

- [AWS EKS Cost Optimization](../../aws/compute/eks-cost-optimization.md)
- [GCP GKE FinOps](../../gcp/gke/README.md)
- [Kubernetes Cost Optimization](../../kubernetes/kube-cost/README.md)
- [FinOps Basics](../../foundations/finops-basics.md)
- [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/)
- [AKS Best Practices](https://learn.microsoft.com/azure/aks/best-practices)

## 🛠️ Tools Mentioned

| Tool | Purpose | Link |
|------|---------|------|
| Azure Cost Management | Native cost tracking | [Link](https://azure.microsoft.com/services/cost-management/) |
| OpenCost | Kubernetes cost monitoring | [GitHub](https://github.com/opencost/opencost) |
| Kubecost | Commercial cost optimization | [Website](https://www.kubecost.com/) |
| Azure Advisor | Optimization recommendations | [Portal](https://portal.azure.com/#blade/Microsoft_Azure_Advisor/AdvisorMenuBlade/overview) |

---

**💡 Pro Tip**: Start with visibility (OpenCost + Azure Cost Management), then implement quick wins (autoscaler, rightsizing), and finally optimize long-term (reserved instances, architecture changes).
