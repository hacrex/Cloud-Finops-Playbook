# FinOps Basics

FinOps (Financial Operations) combines engineering, finance, and operations to optimize cloud spending and maximize business value.

## What is FinOps?

FinOps is a cultural practice and operating model that brings financial accountability to the variable spending model of cloud computing. It enables organizations to understand cloud costs, make trade-offs, and align spending with business value.

### Core Principles

1. **Collaboration**: Engineers, finance, and business teams work together
2. **Ownership**: Everyone takes ownership of their cloud usage
3. **Accessibility**: Cost data is available to all stakeholders in a timely manner
4. **Centralized Team**: A central FinOps team drives best practices and tooling
5. **Variable Rate Model**: Leverage the variable cost model of cloud
6. **Value-Driven**: Focus on business outcomes, not just cost reduction

## The FinOps Lifecycle

### 1. Inform (Visibility & Allocation)
- **Goal**: Complete visibility into cloud spend
- **Key Activities**:
  - Implement comprehensive tagging strategy
  - Allocate costs to teams, products, and services
  - Create dashboards and reports
  - Establish unit economics (cost per customer, transaction, etc.)

### 2. Optimize (Rate & Usage Optimization)
- **Goal**: Reduce waste and optimize rates
- **Key Activities**:
  - Rightsizing resources (compute, storage, databases)
  - Purchasing committed use discounts (RIs, Savings Plans, CUDs)
  - Eliminating unused resources
  - Architectural optimization for cost efficiency

### 3. Operate (Continuous Improvement)
- **Goal**: Embed FinOps into operations
- **Key Activities**:
  - Set budgets and forecasts
  - Implement anomaly detection
  - Establish governance policies
  - Continuous optimization culture

## Engineering-Driven FinOps

Traditional FinOps focuses on finance teams tracking costs. **Engineering-driven FinOps** empowers engineers to:

- Make cost-aware architectural decisions
- Implement automated cost controls in CI/CD
- Build cost visibility into developer workflows
- Optimize infrastructure as code before deployment
- Own the cost implications of their designs

### Key Practices for Engineers

1. **Tag Everything**: Every resource should have owner, environment, and cost center tags
2. **Right-Size Proactively**: Use monitoring data to match resources to actual needs
3. **Automate Cleanup**: Implement automated cleanup of dev/test environments
4. **Cost in Code Reviews**: Include cost impact in architecture reviews
5. **Unit Economics**: Understand cost per user, transaction, or feature

## Getting Started

### Week 1-2: Visibility
- [ ] Enable billing exports and cost explorer
- [ ] Implement mandatory tagging policy
- [ ] Create basic cost dashboards by team/service

### Week 3-4: Optimization Quick Wins
- [ ] Identify and eliminate unused resources
- [ ] Review largest cost centers for rightsizing opportunities
- [ ] Set up budget alerts

### Month 2-3: Culture & Automation
- [ ] Establish FinOps guild/champions in each team
- [ ] Integrate cost checks into CI/CD pipelines
- [ ] Implement showback/chargeback model
- [ ] Regular cost review meetings

## Key Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| Cloud Waste % | Unused or overprovisioned resources | < 15% |
| Coverage % | Resources with proper tags | > 95% |
| Reserved Coverage | Workloads on committed pricing | 60-80% for stable workloads |
| Unit Cost Trend | Cost per business metric | Flat or decreasing |

## Common Pitfalls

❌ **Only focusing on cost reduction** - FinOps is about maximizing value, not just cutting costs  
❌ **Finance-owned only** - Engineers must be empowered and accountable  
❌ **Perfect tags before starting** - Start with basics, iterate  
❌ **One-time optimization** - FinOps is continuous, not a project  
❌ **Ignoring speed/cost tradeoffs** - Sometimes higher cost enables faster delivery  

## Next Steps

- Read about [Cloud Economics](../foundations/cloud-economics/)
- Implement [Tagging Standards](../TAGGING-STANDARDS.md)
- Set up [Cost Allocation](../COST-ALLOCATION.md)
- Explore platform-specific guides: [AWS](../aws/), [Azure](../azure/), [GCP](../gcp/)
- Learn about [Kubernetes Cost Optimization](../kubernetes/)

## References

- [FinOps Foundation](https://www.finops.org/)
- [State of FinOps Report](https://www.finops.org/reports/)
- [FinOps Framework](https://www.finops.org/framework/)
