# Tagging Strategy for Cost Allocation

Tags are the backbone of cloud cost allocation. Without a consistent, enforced tagging strategy, you cannot answer the most basic FinOps questions: "Which team owns this resource?" and "What is this resource for?" A well-designed tagging strategy enables accurate cost allocation, governance enforcement, and operational automation.

---

## Tag Taxonomy Design

A tag taxonomy is the structured set of tag keys and allowed values that your organization uses across all cloud resources.

### Design Principles

1. **Start with cost allocation needs** — what dimensions do you need to slice costs by?
2. **Keep it simple** — 5–8 mandatory tags is better than 20 optional ones
3. **Use consistent naming** — decide on `camelCase`, `kebab-case`, or `snake_case` and stick to it
4. **Define allowed values** — free-text tags lead to inconsistency (`prod`, `Prod`, `production`, `PROD`)
5. **Plan for automation** — tags should be settable by IaC without human intervention

### Recommended Core Taxonomy

| Tag Key | Description | Example Values | Required |
|---|---|---|---|
| `team` | Owning team | `platform`, `data`, `frontend`, `backend` | ✅ Mandatory |
| `product` | Product or service | `checkout`, `search`, `auth`, `analytics` | ✅ Mandatory |
| `environment` | Deployment environment | `prod`, `staging`, `dev`, `sandbox` | ✅ Mandatory |
| `cost-center` | Finance cost center code | `CC-1001`, `CC-2034` | ✅ Mandatory |
| `project` | Project or initiative | `migration-2024`, `ml-platform` | ✅ Mandatory |
| `owner` | Resource owner (email) | `alice@company.com` | ✅ Mandatory |
| `managed-by` | Provisioning method | `terraform`, `pulumi`, `manual` | ✅ Mandatory |
| `data-classification` | Data sensitivity | `public`, `internal`, `confidential` | ⚠️ Conditional |
| `backup` | Backup policy | `daily`, `weekly`, `none` | ⚠️ Conditional |
| `auto-shutdown` | Scheduled shutdown | `true`, `false` | ⚠️ Optional |
| `expiry-date` | Resource expiry | `2024-12-31` | ⚠️ Optional |

### Naming Convention

Choose one convention and enforce it across all clouds:

```
Recommended: kebab-case for keys, lowercase for values

Keys:   team, cost-center, managed-by, data-classification
Values: platform, CC-1001, terraform, confidential

Avoid:
  Mixed case keys:  Team, CostCenter, ManagedBy
  Mixed case values: Platform, Terraform, Confidential
  Spaces in values: "cost center", "my team"
```

---

## Mandatory vs Optional Tags

### Mandatory Tags

Mandatory tags are required on all resources. Missing mandatory tags should trigger:
1. An automated alert to the resource owner
2. A compliance violation in your governance dashboard
3. Eventually, automated remediation or resource termination

**Mandatory tag enforcement policy:**
```
Resource created without mandatory tags:
  Day 0:   Alert sent to owner
  Day 3:   Escalation to team lead
  Day 7:   Resource tagged with "owner=unknown, team=untagged"
  Day 14:  Resource scheduled for termination (with warning)
  Day 21:  Resource terminated (if still untagged)
```

### Optional Tags

Optional tags provide additional context but are not required for cost allocation. Examples:
- `version` — application version deployed
- `git-commit` — commit SHA that deployed this resource
- `ticket` — JIRA/GitHub issue that created this resource
- `created-by` — IAM user or role that created the resource

### Tag Value Standardization

Define allowed values for each mandatory tag to prevent inconsistency:

```yaml
# tag-taxonomy.yaml — source of truth for allowed tag values
tags:
  team:
    required: true
    allowed_values:
      - platform
      - data
      - frontend
      - backend
      - security
      - ml
    description: "Owning engineering team"
  
  environment:
    required: true
    allowed_values:
      - prod
      - staging
      - dev
      - sandbox
      - dr
    description: "Deployment environment"
  
  cost-center:
    required: true
    pattern: "^CC-[0-9]{4}$"
    description: "Finance cost center code (format: CC-XXXX)"
```

---

## Tag Governance and Enforcement

### AWS Tag Policies (Organizations)

AWS Organizations Tag Policies enforce tag compliance across all accounts in your organization:

```json
{
  "tags": {
    "team": {
      "tag_key": {
        "@@assign": "team"
      },
      "tag_value": {
        "@@assign": [
          "platform",
          "data",
          "frontend",
          "backend",
          "security",
          "ml"
        ]
      },
      "enforced_for": {
        "@@assign": [
          "ec2:instance",
          "ec2:volume",
          "rds:db",
          "s3:bucket",
          "lambda:function"
        ]
      }
    },
    "environment": {
      "tag_key": {
        "@@assign": "environment"
      },
      "tag_value": {
        "@@assign": ["prod", "staging", "dev", "sandbox"]
      }
    }
  }
}
```

### AWS Config Rules for Tag Compliance

```hcl
# Terraform: AWS Config rule for required tags
resource "aws_config_config_rule" "required_tags" {
  name = "required-tags-rule"

  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }

  input_parameters = jsonencode({
    tag1Key   = "team"
    tag2Key   = "environment"
    tag3Key   = "product"
    tag4Key   = "cost-center"
    tag5Key   = "owner"
  })

  scope {
    compliance_resource_types = [
      "AWS::EC2::Instance",
      "AWS::EC2::Volume",
      "AWS::RDS::DBInstance",
      "AWS::S3::Bucket",
      "AWS::Lambda::Function",
      "AWS::ECS::Service",
      "AWS::EKS::Cluster"
    ]
  }
}

# Auto-remediation: Lambda to tag non-compliant resources
resource "aws_config_remediation_configuration" "tag_remediation" {
  config_rule_name = aws_config_config_rule.required_tags.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-TagSSMDocument"
  automatic        = false  # Set to true for auto-remediation

  parameter {
    name           = "AutomationAssumeRole"
    static_value   = aws_iam_role.config_remediation.arn
  }
}
```

### Azure Policy for Tag Enforcement

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "in": [
            "Microsoft.Compute/virtualMachines",
            "Microsoft.Storage/storageAccounts",
            "Microsoft.Sql/servers/databases"
          ]
        },
        {
          "anyOf": [
            {"field": "tags['team']", "exists": "false"},
            {"field": "tags['environment']", "exists": "false"},
            {"field": "tags['cost-center']", "exists": "false"}
          ]
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

### GCP Label Constraints (Organization Policy)

```yaml
# GCP Organization Policy: Require labels on resources
name: projects/my-project/policies/compute.restrictCloudRunRegions
spec:
  rules:
    - condition:
        expression: >
          resource.type == "compute.googleapis.com/Instance" &&
          !has(resource.labels.team)
      enforce: true
```

---

## Automated Tagging with IaC

The most reliable way to ensure tag compliance is to enforce tagging at the IaC level, before resources are created.

### Terraform: Default Tags

```hcl
# provider.tf — apply default tags to all AWS resources
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      team        = var.team
      product     = var.product
      environment = var.environment
      cost-center = var.cost_center
      managed-by  = "terraform"
      owner       = var.owner_email
    }
  }
}

# Individual resources can add additional tags
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"

  tags = {
    Name    = "web-server-${var.environment}"
    project = var.project_name
    # Default tags from provider are automatically applied
  }
}
```

### Terraform: Tag Validation with Preconditions

```hcl
# Validate tag values at plan time
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["prod", "staging", "dev", "sandbox"], var.environment)
    error_message = "environment must be one of: prod, staging, dev, sandbox"
  }
}

variable "cost_center" {
  type        = string
  description = "Finance cost center code"

  validation {
    condition     = can(regex("^CC-[0-9]{4}$", var.cost_center))
    error_message = "cost_center must match format CC-XXXX (e.g., CC-1234)"
  }
}
```

### OPA/Conftest Policy for Tag Validation

```rego
# policies/tagging.rego
package tagging

required_tags := {"team", "environment", "product", "cost-center", "owner"}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  
  tags := resource.change.after.tags
  missing := required_tags - {tag | tags[tag]}
  count(missing) > 0
  
  msg := sprintf(
    "Resource %s is missing required tags: %v",
    [resource.address, missing]
  )
}

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_instance"
  
  env := resource.change.after.tags.environment
  not env in {"prod", "staging", "dev", "sandbox"}
  
  msg := sprintf(
    "Resource %s has invalid environment tag value: %s",
    [resource.address, env]
  )
}
```

---

## Tag Compliance Reporting

### Compliance Dashboard Metrics

Track these metrics weekly:

| Metric | Definition | Target |
|---|---|---|
| Overall tag compliance | % of resources with all mandatory tags | > 98% |
| Untagged spend | $ of spend from untagged resources | < 2% |
| Tag compliance by team | Per-team compliance rate | > 95% per team |
| Tag compliance by resource type | Compliance rate per resource type | > 95% per type |
| Time to tag new resources | Hours from creation to full tagging | < 24 hours |

### Compliance Report Query (AWS Athena + CUR)

```sql
-- Find untagged spend by service (last 30 days)
SELECT
  line_item_product_code AS service,
  SUM(line_item_unblended_cost) AS untagged_cost,
  COUNT(DISTINCT line_item_resource_id) AS untagged_resources
FROM
  cost_and_usage_report
WHERE
  line_item_usage_start_date >= DATE_ADD('day', -30, CURRENT_DATE)
  AND (
    resource_tags_user_team IS NULL OR resource_tags_user_team = ''
    OR resource_tags_user_environment IS NULL OR resource_tags_user_environment = ''
  )
  AND line_item_line_item_type = 'Usage'
GROUP BY
  line_item_product_code
ORDER BY
  untagged_cost DESC
LIMIT 20;
```

---

## Cross-Cloud Tagging Consistency

When operating across multiple clouds, maintain consistent tag/label semantics even though the implementation differs.

### Cross-Cloud Tag Mapping

| Concept | AWS Tag | Azure Tag | GCP Label |
|---|---|---|---|
| Team | `team` | `team` | `team` |
| Environment | `environment` | `environment` | `environment` |
| Product | `product` | `product` | `product` |
| Cost Center | `cost-center` | `cost-center` | `cost-center` |
| Owner | `owner` | `owner` | `owner` |
| Managed By | `managed-by` | `managed-by` | `managed-by` |

### Cloud-Specific Constraints

| Cloud | Key Constraints | Value Constraints | Max Tags |
|---|---|---|---|
| AWS | 128 chars, case-sensitive | 256 chars | 50 |
| Azure | 512 chars, case-insensitive | 256 chars | 50 |
| GCP | 63 chars, lowercase only | 63 chars, lowercase | 64 |

**GCP label normalization:**
```python
def normalize_for_gcp(tag_key: str, tag_value: str) -> tuple[str, str]:
    """
    Normalize tag key/value for GCP label requirements:
    - Lowercase only
    - Only letters, numbers, hyphens, underscores
    - Max 63 characters
    """
    import re
    
    def normalize(s: str) -> str:
        s = s.lower()
        s = re.sub(r'[^a-z0-9\-_]', '-', s)
        s = s[:63]
        return s
    
    return normalize(tag_key), normalize(tag_value)

# Example
key, value = normalize_for_gcp("Cost-Center", "CC-1234")
# Returns: ("cost-center", "cc-1234")
```

---

## Kubernetes Label Strategy for Cost Allocation

Kubernetes labels are the equivalent of cloud tags for workload cost allocation. Tools like Kubecost, OpenCost, and cloud provider cost allocation features use these labels to attribute pod costs to teams and products.

### Recommended Label Set

```yaml
# Standard labels for all Kubernetes workloads
apiVersion: apps/v1
kind: Deployment
metadata:
  name: checkout-api
  labels:
    # Cost allocation labels (must match cloud tag taxonomy)
    app.kubernetes.io/name: checkout-api
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: checkout
    team: backend
    product: checkout
    environment: prod
    cost-center: CC-1001
    
    # Operational labels
    app.kubernetes.io/version: "2.4.1"
    app.kubernetes.io/managed-by: helm
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: checkout-api
  template:
    metadata:
      labels:
        app.kubernetes.io/name: checkout-api
        team: backend
        product: checkout
        environment: prod
```

### Namespace-Level Cost Allocation

```yaml
# Namespace with cost allocation labels
apiVersion: v1
kind: Namespace
metadata:
  name: team-backend-prod
  labels:
    team: backend
    environment: prod
    cost-center: CC-1001
  annotations:
    # Kubecost annotations for cost allocation
    kubecost.com/team: backend
    kubecost.com/product: checkout
```

### OPA Policy for Kubernetes Label Enforcement

```rego
# policies/k8s-labels.rego
package kubernetes.labels

required_labels := {"team", "product", "environment", "cost-center"}

deny[msg] {
  input.request.kind.kind == "Deployment"
  
  labels := input.request.object.metadata.labels
  missing := required_labels - {label | labels[label]}
  count(missing) > 0
  
  msg := sprintf(
    "Deployment %s/%s is missing required labels: %v",
    [
      input.request.object.metadata.namespace,
      input.request.object.metadata.name,
      missing
    ]
  )
}
```

### Kubecost Allocation Configuration

```yaml
# kubecost values.yaml — configure cost allocation
kubecostProductConfigs:
  labelMappingConfigs:
    enabled: true
    owner_label: "team"
    team_label: "team"
    department_label: "cost-center"
    product_label: "product"
    environment_label: "environment"
  
  # Shared cost allocation
  sharedNamespaces: "kube-system,monitoring,ingress-nginx"
  sharedSplit: "weighted"  # Allocate shared costs proportionally
```

---

## Tag Governance Maturity Model

| Level | Characteristics |
|---|---|
| **Level 1: Ad-hoc** | No tagging standards, manual tagging, < 50% compliance |
| **Level 2: Defined** | Tag taxonomy documented, manual enforcement, 50–80% compliance |
| **Level 3: Managed** | IaC-enforced tagging, automated alerts, 80–95% compliance |
| **Level 4: Optimized** | Policy-as-code enforcement, auto-remediation, > 98% compliance |

Most organizations should target **Level 3** within 6 months of starting their FinOps journey, and **Level 4** within 12 months.
