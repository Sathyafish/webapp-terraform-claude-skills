---
name: terraform-ecs-webapp
description: Generate Terraform infrastructure for deploying a containerized webapp on AWS ECS Fargate with ALB, WAF, and VPC networking. Use when the user asks to create, deploy, or scaffold Terraform for a webapp, ECS service, container deployment, load balancer setup, WAF configuration, or Russian IP blocking.
---

# Terraform ECS Webapp Deployment

Generate production-ready Terraform code to deploy a containerized webapp on AWS ECS Fargate with ALB, WAF, and secure VPC networking.

## Architecture Overview

```
Internet → WAF → ALB (public subnets) → ECS Fargate Tasks (private subnets)
```

- **VPC**: Multi-AZ with public and private subnets
- **ALB**: Deployed in public subnets, HTTPS-only ingress
- **ECS Fargate**: Containers in private subnets, egress via NAT Gateway
- **WAF**: Attached to ALB, blocks Russian IP traffic via geo-match
- **Security Groups**: HTTPS (443) only from internet; ECS accepts traffic only from ALB

## Workflow

When the user requests a webapp Terraform deployment, follow these steps:

### Step 1: Gather Required Parameters

Collect from the user (or use sensible defaults):

| Parameter | Default | Description |
|-----------|---------|-------------|
| `project_name` | _required_ | Name prefix for all resources |
| `aws_region` | `us-east-1` | AWS region |
| `vpc_cidr` | `10.0.0.0/16` | VPC CIDR block |
| `container_image` | _required_ | Docker image URI (ECR or public) |
| `container_port` | `8080` | Port the container listens on |
| `container_cpu` | `256` | Fargate CPU units |
| `container_memory` | `512` | Fargate memory (MiB) |
| `desired_count` | `2` | Number of ECS tasks |
| `health_check_path` | `/health` | ALB health check endpoint |
| `certificate_arn` | _required_ | ACM certificate ARN for HTTPS |
| `availability_zones` | `2` | Number of AZs to use |

### Step 2: Generate Terraform File Structure

Create the following files:

```
terraform/
├── main.tf              # Provider, backend config
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── vpc.tf               # VPC, subnets, NAT, IGW, route tables
├── security-groups.tf   # ALB and ECS security groups
├── alb.tf               # ALB, target group, listeners
├── ecs.tf               # ECS cluster, task definition, service
├── waf.tf               # WAF Web ACL, geo-match rule
├── iam.tf               # ECS task execution and task roles
├── logs.tf              # CloudWatch log group
└── terraform.tfvars     # User-specific variable values
```

### Step 3: Generate Each Module

Follow the specifications below for each file. For detailed resource configurations, see [reference.md](reference.md). For complete examples, see [examples.md](examples.md).

---

## Module Specifications

### VPC (`vpc.tf`)

- VPC with DNS support and hostnames enabled
- Public subnets (one per AZ) with `map_public_ip_on_launch = true`
- Private subnets (one per AZ) with no public IP
- Internet Gateway attached to VPC
- NAT Gateway in first public subnet with Elastic IP
- Public route table: `0.0.0.0/0` → Internet Gateway
- Private route table: `0.0.0.0/0` → NAT Gateway
- Subnet CIDR allocation: use `cidrsubnet()` function

### Security Groups (`security-groups.tf`)

**ALB Security Group:**
- Ingress: HTTPS (443) from `0.0.0.0/0` and `::/0`
- Egress: All traffic to VPC CIDR

**ECS Security Group:**
- Ingress: Container port from ALB security group only
- Egress: All traffic (for pulling images, external API calls)

### ALB (`alb.tf`)

- Application Load Balancer in public subnets
- HTTPS listener (443) with ACM certificate, forwarding to target group
- HTTP listener (80) redirecting to HTTPS
- Target group: IP type, health check on configured path
- Enable `drop_invalid_header_fields`
- Enable access logging if S3 bucket provided

### ECS (`ecs.tf`)

- ECS Cluster with Container Insights enabled
- Fargate task definition with:
  - Container definition using provided image/port/cpu/memory
  - CloudWatch Logs log driver
  - `readonlyRootFilesystem = true` where possible
- ECS Service:
  - Fargate launch type, `LATEST` platform version
  - Network config: private subnets, ECS security group, no public IP
  - Load balancer block pointing to ALB target group
  - Deployment circuit breaker with rollback enabled

### WAF (`waf.tf`)

- WAF v2 Web ACL (regional scope, attached to ALB)
- **Geo-match rule** to block Russian traffic:
  - Statement: `geo_match_statement` with country code `RU`
  - Action: `block`
  - Priority: `1`
- **AWS Managed Rules** (recommended additions):
  - `AWSManagedRulesCommonRuleSet` (priority 2)
  - `AWSManagedRulesKnownBadInputsRuleSet` (priority 3)
- Default action: `allow`
- CloudWatch metrics enabled
- Associate Web ACL with ALB using `aws_wafv2_web_acl_association`

### IAM (`iam.tf`)

- **Task Execution Role**: allows ECS to pull images and write logs
  - Attach `AmazonECSTaskExecutionRolePolicy`
  - If using ECR, include ECR pull permissions
- **Task Role**: role assumed by the running container
  - Attach only permissions the app needs (least privilege)

### Logs (`logs.tf`)

- CloudWatch Log Group: `/ecs/{project_name}`
- Retention: 30 days (configurable)

### Outputs (`outputs.tf`)

Export: ALB DNS name, ALB ARN, ECS cluster name, ECS service name, VPC ID, WAF Web ACL ARN

---

## Security Requirements (Mandatory)

1. **No HTTP traffic**: ALB must redirect 80 → 443; no plain HTTP listener forwarding
2. **HTTPS only from internet**: ALB SG ingress limited to port 443
3. **ECS isolation**: ECS tasks in private subnets, accessible only from ALB
4. **WAF Russian IP block**: Geo-match rule blocking country code `RU`
5. **No hardcoded secrets**: Use `aws_ssm_parameter` or `aws_secretsmanager_secret` for secrets; reference via environment variables in task definition
6. **IAM least privilege**: Task role gets only required permissions
7. **Encryption**: ALB access logs encrypted, CloudWatch logs encrypted if KMS key provided

## Naming Convention

All resources use the pattern: `${var.project_name}-{resource-type}`

Examples: `myapp-vpc`, `myapp-alb`, `myapp-ecs-cluster`, `myapp-waf-acl`

## Tags

Apply consistent tags to all resources:

```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

## Validation Checklist

After generating all files, verify:

- [ ] `terraform fmt` passes on all files
- [ ] `terraform validate` passes
- [ ] ALB SG allows only 443 inbound
- [ ] ECS SG allows inbound only from ALB SG
- [ ] ECS tasks are in private subnets
- [ ] ALB is in public subnets
- [ ] WAF has geo-match block for `RU`
- [ ] WAF is associated with ALB
- [ ] No hardcoded secrets or credentials
- [ ] NAT Gateway exists for private subnet egress
- [ ] All resources have consistent tags

## Additional Resources

- For detailed Terraform resource configurations, see [reference.md](reference.md)
- For complete working examples, see [examples.md](examples.md)
