cloud-finops-playbook/
│
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── CODE_OF_CONDUCT.md
├── ROADMAP.md
├── CHANGELOG.md
├── SECURITY.md
├── AWESOME.md
├── FINOPS-MATURITY.md
├── TAGGING-STANDARDS.md
├── COST-ALLOCATION.md
├── SHOWBACK-CHARGEBACK.md
├── CLOUD-GOVERNANCE.md
├── GREEN-CLOUD-COMPUTING.md
│
├── assets/
│   ├── diagrams/
│   │   ├── hyperscaler-vs-neocloud.png
│   │   ├── kubernetes-cost-flow.png
│   │   ├── gpu-utilization.png
│   │   ├── ai-inference-architecture.png
│   │   ├── multi-cloud-finops.png
│   │   └── storage-tiering.png
│   │
│   ├── dashboards/
│   ├── images/
│   ├── logos/
│   └── presentations/
│
├── foundations/
│   ├── finops-basics/
│   │   ├── what-is-finops.md
│   │   ├── finops-principles.md
│   │   ├── finops-lifecycle.md
│   │   └── engineering-driven-finops.md
│   │
│   ├── cloud-economics/
│   ├── pricing-models/
│   ├── governance/
│   ├── tagging-strategy/
│   ├── budgeting/
│   ├── forecasting/
│   ├── unit-economics/
│   ├── cost-allocation/
│   ├── shared-responsibility/
│   ├── cloud-billing/
│   └── sustainability/
│
├── aws/
│   ├── compute/
│   │   ├── ec2-rightsizing.md
│   │   ├── spot-instances.md
│   │   ├── savings-plans.md
│   │   ├── reserved-instances.md
│   │   └── autoscaling.md
│   │
│   ├── storage/
│   ├── networking/
│   ├── databases/
│   ├── kubernetes/
│   ├── eks/
│   ├── serverless/
│   ├── cloudfront/
│   ├── observability/
│   ├── ai-ml/
│   ├── architecture-patterns/
│   ├── security-costs/
│   └── automation/
│
├── azure/
│   ├── vm/
│   ├── aks/
│   ├── storage/
│   ├── networking/
│   ├── databases/
│   ├── reserved-capacity/
│   ├── defender-costs/
│   ├── ai-services/
│   ├── observability/
│   └── automation/
│
├── gcp/
│   ├── compute-engine/
│   ├── gke/
│   ├── cloud-storage/
│   ├── networking/
│   ├── bigquery/
│   ├── vertex-ai/
│   ├── committed-use/
│   ├── observability/
│   └── automation/
│
├── oracle-cloud/
│   ├── compute/
│   ├── networking/
│   ├── storage/
│   ├── databases/
│   ├── kubernetes/
│   ├── ai-services/
│   └── cost-analysis/
│
├── alibaba-cloud/
│   ├── ecs/
│   ├── ack/
│   ├── networking/
│   ├── storage/
│   ├── databases/
│   ├── observability/
│   └── automation/
│
├── linode/
│   ├── compute/
│   ├── kubernetes/
│   ├── object-storage/
│   ├── networking/
│   ├── backups/
│   └── cost-analysis/
│
├── neoclouds/
│   ├── overview/
│   │   ├── what-are-neoclouds.md
│   │   ├── ai-native-clouds.md
│   │   └── gpu-cloud-landscape.md
│   │
│   ├── economics/
│   │   ├── gpu-economics.md
│   │   ├── inference-economics.md
│   │   ├── token-economics.md
│   │   ├── llm-serving-costs.md
│   │   └── ai-training-costs.md
│   │
│   ├── hyperscaler-vs-neocloud/
│   │   ├── aws-vs-coreweave.md
│   │   ├── azure-vs-nebius.md
│   │   ├── gcp-vs-lambda.md
│   │   └── economics-comparison.md
│   │
│   ├── gpu-clouds/
│   ├── ai-infrastructure/
│   ├── bare-metal-ai/
│   ├── inference-economics/
│   ├── storage-architecture/
│   ├── networking/
│   │   ├── infiniband-vs-ethernet.md
│   │   ├── rdma.md
│   │   ├── nvlink.md
│   │   └── east-west-traffic.md
│   │
│   ├── kubernetes/
│   ├── scheduling/
│   ├── sovereignty/
│   ├── observability/
│   ├── sustainability/
│   ├── multi-region/
│   ├── startup-case-studies/
│   └── future-trends/
│
├── kubernetes/
│   ├── cluster-rightsizing/
│   ├── autoscaling/
│   ├── kube-cost/
│   ├── opencost/
│   ├── karpenter/
│   ├── keda/
│   ├── storage-optimization/
│   ├── networking/
│   ├── observability/
│   ├── multi-tenancy/
│   ├── gpu-sharing/
│   ├── node-pools/
│   ├── cluster-api/
│   ├── service-mesh/
│   ├── platform-engineering/
│   └── security-costs/
│
├── ai-infrastructure/
│   ├── gpu-cost-optimization/
│   ├── llmops/
│   ├── inference-optimization/
│   ├── vector-databases/
│   ├── model-serving/
│   ├── distributed-training/
│   ├── ai-storage/
│   ├── ai-networking/
│   ├── model-caching/
│   ├── batching/
│   ├── rag-costs/
│   ├── hybrid-ai-infra/
│   └── observability/
│
├── openstack/
│   ├── nova/
│   ├── neutron/
│   ├── cinder/
│   ├── ceph/
│   ├── quotas/
│   ├── chargeback/
│   ├── metering/
│   ├── private-cloud-finops/
│   └── capacity-planning/
│
├── apache-cloudstack/
│   ├── compute/
│   ├── storage/
│   ├── networking/
│   ├── metering/
│   ├── chargeback/
│   └── observability/
│
├── opennebula/
│   ├── virtualization/
│   ├── networking/
│   ├── storage/
│   ├── capacity-planning/
│   ├── quotas/
│   └── private-cloud-economics/
│
├── vmware/
│   ├── vsphere/
│   ├── nsx/
│   ├── vsan/
│   ├── aria-operations/
│   ├── rightsizing/
│   ├── private-cloud-finops/
│   ├── virtualization-overhead/
│   └── hybrid-cloud/
│
├── observability/
│   ├── prometheus/
│   ├── grafana/
│   ├── opentelemetry/
│   ├── jaeger/
│   ├── cloudwatch/
│   ├── gpu-observability/
│   ├── cost-monitoring/
│   ├── ai-observability/
│   └── distributed-tracing/
│
├── automation/
│   ├── terraform/
│   ├── opentofu/
│   ├── ansible/
│   ├── policies/
│   ├── cleanup-scripts/
│   ├── serverless/
│   ├── kubernetes/
│   ├── event-driven/
│   └── ai-automation/
│
├── tools/
│   ├── kubecost/
│   ├── opencost/
│   ├── infracost/
│   ├── checkov/
│   ├── trivy/
│   ├── falco/
│   ├── karpenter/
│   ├── keda/
│   ├── crossplane/
│   ├── terraform/
│   ├── opentofu/
│   └── cloud-custodian/
│
├── networking/
│   ├── cdn-economics/
│   ├── data-transfer-costs/
│   ├── edge-computing/
│   ├── load-balancing/
│   ├── hybrid-networking/
│   ├── service-mesh/
│   └── private-connectivity/
│
├── storage/
│   ├── object-storage/
│   ├── block-storage/
│   ├── cold-storage/
│   ├── lifecycle-policies/
│   ├── deduplication/
│   ├── replication/
│   └── backup-economics/
│
├── databases/
│   ├── sql/
│   ├── nosql/
│   ├── vector-databases/
│   ├── caching/
│   ├── replication/
│   ├── serverless-databases/
│   └── database-finops/
│
├── security/
│   ├── cloud-security-costs/
│   ├── cnapp/
│   ├── runtime-security/
│   ├── compliance/
│   ├── policy-as-code/
│   ├── secrets-management/
│   └── zero-trust/
│
├── bare-metal/
│   ├── bare-metal-vs-cloud.md
│   ├── gpu-bare-metal.md
│   ├── hybrid-infrastructure.md
│   ├── numa.md
│   ├── sriov.md
│   ├── pcie-passthrough.md
│   └── metal-economics.md
│
├── platform-engineering/
│   ├── internal-developer-platforms/
│   ├── self-service-infrastructure/
│   ├── golden-paths/
│   ├── cost-aware-platforms/
│   ├── kubernetes-platforms/
│   └── developer-experience/
│
├── sustainability/
│   ├── carbon-aware-scheduling/
│   ├── renewable-energy/
│   ├── green-datacenters/
│   ├── gpu-power-efficiency/
│   ├── thermal-density/
│   └── energy-economics/
│
├── templates/
│   ├── cost-review-template/
│   ├── finops-dashboard/
│   ├── architecture-review/
│   ├── optimization-checklist/
│   ├── governance-template/
│   └── monthly-review-template/
│
├── scripts/
│   ├── aws/
│   ├── azure/
│   ├── gcp/
│   ├── kubernetes/
│   ├── cleanup/
│   ├── gpu-monitoring/
│   ├── cost-reporting/
│   └── automation/
│
├── dashboards/
│   ├── grafana/
│   ├── prometheus/
│   ├── kubecost/
│   ├── cloudwatch/
│   ├── azure-monitor/
│   └── gcp-monitoring/
│
├── case-studies/
│   ├── startups/
│   ├── enterprise/
│   ├── ai-workloads/
│   ├── kubernetes/
│   ├── hybrid-cloud/
│   ├── neoclouds/
│   └── bare-metal/
│
├── research/
│   ├── cloud-trends/
│   ├── ai-infrastructure-trends/
│   ├── neocloud-research/
│   ├── hyperscaler-analysis/
│   └── future-of-finops/
│
└── .github/
    ├── ISSUE_TEMPLATE/
    ├── PULL_REQUEST_TEMPLATE.md
    ├── workflows/
    │   ├── markdown-lint.yml
    │   ├── link-checker.yml
    │   └── docs-build.yml
    │
    └── FUNDING.yml