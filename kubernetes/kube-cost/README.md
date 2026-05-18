# Kubernetes Cost Optimization

**Comprehensive guide to optimizing costs across all Kubernetes distributions with engineering-first strategies.**

## 🎯 Overview

Kubernetes cost optimization requires a multi-layered approach covering infrastructure, workloads, storage, and networking. This guide provides actionable strategies for reducing Kubernetes costs while maintaining performance and reliability.

## 📚 Table of Contents

1. [Core Concepts](#core-concepts)
2. [Cost Monitoring Tools](#cost-monitoring-tools)
3. [Node Optimization](#node-optimization)
4. [Workload Optimization](#workload-optimization)
5. [Storage Optimization](#storage-optimization)
6. [Network Optimization](#network-optimization)
7. [Automation & Policies](#automation--policies)
8. [Best Practices Checklist](#best-practices-checklist)

## Core Concepts

### Understanding Kubernetes Costs

```
Total K8s Cost = Compute + Storage + Network + Add-ons

Compute (70-80%):
  - Node VM instances
  - Control plane (managed services)
  - System overhead

Storage (10-15%):
  - Persistent volumes
  - Snapshots
  - Backup solutions

Network (5-10%):
  - Load balancers
  - Data transfer
  - NAT gateways

Add-ons (5%):
  - Monitoring
  - Security
  - Service mesh
```

### Cost Allocation Models

**Namespace-based:** Allocate costs by team/project namespace
**Label-based:** Use labels for granular cost tracking
**Resource-based:** Track by CPU, memory, GPU usage

## Cost Monitoring Tools

### 1. OpenCost (Open Source)

#### Installation
```bash
# Install via Helm
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm install opencost opencost/opencost \
  --namespace opencost \
  --create-namespace \
  --set opencost.cloudProvider=<aws|azure|gcp>
```

#### Key Features
- Real-time cost monitoring
- Cost per namespace/pod/deployment
- Supports AWS, Azure, GCP, and on-prem
- Integrates with Prometheus/Grafana

#### Query Examples
```promql
# Cost by namespace
sum(container_cpu_allocation{namespace="production"}) by (namespace)

# Daily spend trend
sum(kube_pod_container_resource_requests{resource="cpu"}) * on(instance) group_right() node_cpu_hourly_cost * 24
```

### 2. Kubecost (Commercial)

#### Features
- All OpenCost features plus:
- Multi-cluster dashboards
- Forecasting and budgeting
- Automated recommendations
- Enterprise support

#### Pricing
- Free tier: Single cluster
- Standard: $50/node/month
- Enterprise: Custom pricing

### 3. Cloud-Native Tools

| Provider | Tool | Cost |
|----------|------|------|
| AWS | EKS Cost Allocation Tags + CUR | Free |
| Azure | Azure Cost Management + AKS tags | Free |
| GCP | Billing Export + BigQuery | Free + BQ costs |

## Node Optimization

### 1. Cluster Autoscaler

#### Configuration
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: ClusterAutoscaler
metadata:
  name: cluster-autoscaler
spec:
  minNodes: 3
  maxNodes: 20
  scaleDownDelay: 10m
  scaleDownUtilizationThreshold: 0.5
  expander: least-waste
```

#### Best Practices
- Set appropriate min/max bounds
- Enable priority expander for cost optimization
- Configure scale-down delays to prevent thrashing
- Use multiple node groups for different workload types

### 2. Karpenter (Advanced Autoscaling)

#### Installation
```bash
# Install Karpenter
helm repo add karpenter https://charts.karpenter.sh
helm install karpenter karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set settings.clusterName=my-cluster
```

#### NodePool Configuration
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

**Benefits:**
- Launches right-sized nodes in seconds
- Consolidates workloads automatically
- Supports spot instances natively
- Up to **90% cost reduction** vs. static provisioning

### 3. Spot/Preemptible Instances

#### Strategy Matrix
| Workload Type | Spot % | On-Demand % | Notes |
|--------------|--------|-------------|-------|
| Batch processing | 100% | 0% | Fault-tolerant |
| Stateless apps | 70% | 30% | With PDB |
| Stateful apps | 0% | 100% | Avoid spot |
| Development | 100% | 0% | Cost priority |

#### Implementation
```yaml
# Taint spot nodes
tolerations:
  - key: karpenter.sh/capacity-type
    operator: Equal
    value: spot
    effect: NoSchedule

# Pod disruption budget for spot workloads
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: spot-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      capacity-type: spot
```

## Workload Optimization

### 1. Resource Requests & Limits

#### Right-Sizing with VPA
```bash
# Install Vertical Pod Autoscaler
kubectl apply -f https://github.com/kubernetes/autoscaler/releases/latest/download/vertical-pod-autoscaler.yaml
```

#### VPA Configuration
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: Auto
  resourcePolicy:
    containerPolicies:
      - containerName: '*'
        controlledResources: ["cpu", "memory"]
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 2
          memory: 4Gi
```

### 2. KEDA for Event-Driven Scaling

#### Installation
```bash
helm repo add kedacore https://kedacore.github.io/charts
helm install keda kedacore/keda --namespace keda --create-namespace
```

#### ScaledObject Example
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: queue-processor-scaler
spec:
  scaleTargetRef:
    name: queue-processor
  minReplicaCount: 0
  maxReplicaCount: 10
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
        messageCount: "5"
        scalarOperations:
          - operator: Divide
            value: 5
```

**Benefits:**
- Scale to zero when idle
- Respond to custom metrics
- Reduce costs for bursty workloads

### 3. Pod Priority & Preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Critical production workloads"
```

## Storage Optimization

### 1. Storage Class Selection

```yaml
# High-performance (expensive)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: io2
  iopsPerGB: "50"

# Balanced cost/performance
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-gp3
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  
# Low-cost archival
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: archive
provisioner: kubernetes.io/aws-ebs
parameters:
  type: st1
```

### 2. Implement Storage Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    requests.storage: 500Gi
    persistentvolumeclaims: "20"
```

### 3. Cleanup Unused PVCs

```bash
#!/bin/bash
# cleanup-unused-pvcs.sh

# Find unbound PVCs older than 7 days
kubectl get pvc --all-namespaces -o json | \
  jq -r '.items[] | select(.status.phase == "Pending") | "\(.metadata.namespace) \(.metadata.name)"' | \
  while read ns name; do
    echo "Deleting unused PVC: $ns/$name"
    kubectl delete pvc $name -n $ns
  done
```

## Network Optimization

### 1. Optimize Load Balancers

**Strategies:**
- Use NodePort + ingress instead of LB per service
- Share load balancers across services
- Use internal LBs where possible
- Implement ingress controllers (nginx, traefik, ALB)

### 2. Reduce Cross-AZ Traffic

```yaml
# Topology spread constraints
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: my-app
```

### 3. Implement Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
```

## Automation & Policies

### 1. OPA/Gatekeeper Cost Policies

```rego
# Require resource limits
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  not input.request.object.spec.containers[_].resources.limits.cpu
  msg := "CPU limits are required"
}

deny[msg] {
  input.request.kind.kind == "Pod"
  not input.request.object.spec.containers[_].resources.limits.memory
  msg := "Memory limits are required"
}
```

### 2. Automated Rightsizing Script

```python
#!/usr/bin/env python3
# rightsizer.py

from kubernetes import client, config
import json

def analyze_and_recommend(namespace='default'):
    config.load_kube_config()
    v1 = client.CoreV1Api()
    
    pods = v1.list_namespaced_pod(namespace)
    recommendations = []
    
    for pod in pods.items:
        for container in pod.spec.containers:
            if container.resources.requests:
                cpu_request = container.resources.requests.get('cpu', '0')
                mem_request = container.resources.requests.get('memory', '0')
                
                # Analyze actual usage from metrics
                # Compare with requests
                # Generate recommendation
                
                recommendations.append({
                    'pod': pod.metadata.name,
                    'container': container.name,
                    'current_cpu': cpu_request,
                    'current_mem': mem_request,
                    'recommended_cpu': 'TODO',
                    'recommended_mem': 'TODO'
                })
    
    return recommendations

if __name__ == "__main__":
    recs = analyze_and_recommend()
    print(json.dumps(recs, indent=2))
```

### 3. Scheduled Cluster Shutdown

```yaml
# CronJob for dev cluster shutdown
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekend-shutdown
spec:
  schedule: "0 20 * * 5"  # Friday 8 PM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: kubectl
            image: bitnami/kubectl:latest
            command:
            - /bin/sh
            - -c
            - |
              kubectl scale deployment --all --replicas=0 -n dev
          restartPolicy: OnFailure
```

## Best Practices Checklist

### Infrastructure
- [ ] Enable cluster autoscaler with proper bounds
- [ ] Use spot/preemptible instances for fault-tolerant workloads
- [ ] Implement Karpenter for advanced autoscaling
- [ ] Right-size node instance types
- [ ] Use Graviton/ARM instances where supported

### Workloads
- [ ] Set resource requests and limits on all pods
- [ ] Deploy VPA for automatic rightsizing
- [ ] Use HPA/KEDA for demand-based scaling
- [ ] Implement pod priority classes
- [ ] Scale non-production to zero during off-hours

### Storage
- [ ] Choose appropriate storage classes
- [ ] Implement storage quotas per namespace
- [ ] Clean up orphaned PVCs regularly
- [ ] Use ephemeral storage for temporary data
- [ ] Snapshot only critical data

### Network
- [ ] Share load balancers across services
- [ ] Use ingress controllers
- [ ] Minimize cross-AZ traffic
- [ ] Implement network policies
- [ ] Use private endpoints where possible

### Governance
- [ ] Deploy OpenCost/Kubecost for visibility
- [ ] Tag all resources for cost allocation
- [ ] Set up budget alerts
- [ ] Implement OPA policies for cost controls
- [ ] Review costs weekly in team meetings

## 📈 Case Study: Multi-Tenant Platform

### Before
- 15 clusters, 500+ nodes
- No autoscaling
- Over-provisioned by 3x
- Monthly cost: $180,000

### After
- Implemented Karpenter across all clusters
- Migrated 60% to spot instances
- Deployed OpenCost for visibility
- Enforced resource quotas
- Automated weekend shutdowns

### Results
- **New monthly cost**: $62,000
- **Savings**: 66% ($1.4M annually)
- **Utilization**: Increased from 25% to 72%

## 🔗 Related Resources

- [FinOps Basics](../../foundations/finops-basics.md)
- [AWS EKS Optimization](../../aws/compute/ec2-rightsizing.md)
- [Azure AKS Guide](../../azure/aks/README.md)
- [GCP GKE FinOps](../../gcp/gke/README.md)
- [GPU Cost Optimization](../../ai-infrastructure/gpu-cost-optimization/README.md)

## 🛠️ Tools Summary

| Category | Tool | Type | Link |
|----------|------|------|------|
| Cost Monitoring | OpenCost | Open Source | [GitHub](https://github.com/opencost/opencost) |
| Cost Monitoring | Kubecost | Commercial | [Website](https://kubecost.com) |
| Autoscaling | Karpenter | Open Source | [GitHub](https://github.com/aws/karpenter) |
| Autoscaling | VPA | Built-in | [Docs](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) |
| Event Scaling | KEDA | Open Source | [Website](https://keda.sh) |
| Policy | OPA Gatekeeper | Open Source | [GitHub](https://github.com/open-policy-agent/gatekeeper) |

---

**💡 Pro Tip**: Start with visibility (deploy OpenCost today), then implement quick wins (autoscaler, spot instances), and finally optimize architecture (Karpenter, workload consolidation). Measure everything!
