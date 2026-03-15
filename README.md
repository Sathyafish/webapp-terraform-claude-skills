# Terraform ECS Webapp - Claude Skill

A Claude agent skill that generates production-ready Terraform infrastructure for deploying containerized webapps on AWS ECS Fargate with ALB, WAF, and secure VPC networking.

## Architecture

```
Internet → WAF → ALB (public subnets) → ECS Fargate Tasks (private subnets)
```

| Component | Placement | Purpose |
|-----------|-----------|---------|
| VPC | Multi-AZ | Public and private subnets with NAT Gateway |
| ALB | Public subnets | HTTPS-only ingress, HTTP redirects to HTTPS |
| ECS Fargate | Private subnets | Containerized webapp, no public IP |
| WAF | Attached to ALB | Geo-match rule blocks Russian IP traffic (`RU`) |
| Security Groups | VPC | ALB accepts only port 443; ECS accepts only from ALB |

## Claude Skill Structure

```
.cursor/skills/terraform-ecs-webapp/
├── SKILL.md         # Main instructions - workflow, module specs, security rules
├── reference.md     # Complete Terraform resource definitions for all .tf files
└── examples.md      # Working examples, deployment commands, verification steps
```

### SKILL.md

The main skill file (196 lines) provides Claude with:

- **Architecture overview** of the target infrastructure
- **Step-by-step workflow** for gathering parameters and generating Terraform
- **Module specifications** for each `.tf` file (VPC, ALB, ECS, WAF, IAM, etc.)
- **Mandatory security requirements** the generated code must satisfy
- **Naming conventions and tagging standards** for consistency
- **Validation checklist** to verify the output before deployment

### reference.md

Detailed, copy-ready Terraform resource definitions for all infrastructure components. Claude references this file when generating the actual `.tf` files.

### examples.md

Complete working examples including:

- Basic webapp deployment with `terraform.tfvars`
- Webapp with secrets from SSM Parameter Store
- Multi-webapp setup using Terraform modules
- Auto-scaling configuration
- Route 53 DNS integration
- Post-deployment verification commands

## Generated Terraform Files

When triggered, the skill instructs Claude to produce:

| File | Content |
|------|---------|
| `main.tf` | Provider config, backend, common tags |
| `variables.tf` | All input variables with validation |
| `outputs.tf` | ALB DNS, ECS cluster/service, VPC ID, WAF ARN |
| `vpc.tf` | VPC, public/private subnets, IGW, NAT Gateway, route tables |
| `security-groups.tf` | ALB SG (HTTPS only) and ECS SG (ALB only) |
| `alb.tf` | ALB, HTTPS listener, HTTP-to-HTTPS redirect, target group |
| `ecs.tf` | ECS cluster, Fargate task definition, service |
| `waf.tf` | WAF v2 Web ACL, geo-match block for RU, managed rule groups |
| `iam.tf` | Task execution role, task role (least privilege) |
| `logs.tf` | CloudWatch log group |

## Security Highlights

- **HTTPS-only** access from the internet (port 443)
- **HTTP redirect** - port 80 automatically redirects to HTTPS
- **Private ECS tasks** - containers isolated in private subnets, reachable only from ALB
- **WAF geo-blocking** - blocks all traffic originating from Russia via geo-match on country code `RU`
- **Managed WAF rules** - AWS managed rule groups for common threats and known bad inputs
- **No hardcoded secrets** - uses SSM Parameter Store or Secrets Manager
- **Least-privilege IAM** - task roles scoped to only required permissions

## Usage

Ask Claude to create Terraform for a webapp deployment. The skill activates on prompts like:

- "Create Terraform for my webapp on ECS"
- "Deploy a container to ECS with ALB and WAF"
- "Scaffold Terraform infrastructure for a web application"
- "Set up WAF to block Russian IP traffic for my webapp"

Claude will collect the required parameters and generate the full set of Terraform files.

## Required Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `project_name` | Yes | — | Name prefix for all AWS resources |
| `container_image` | Yes | — | Docker image URI (ECR or public registry) |
| `certificate_arn` | Yes | — | ACM certificate ARN for HTTPS |
| `aws_region` | No | `us-east-1` | AWS region for deployment |
| `container_port` | No | `8080` | Port the container listens on |
| `container_cpu` | No | `256` | Fargate CPU units |
| `container_memory` | No | `512` | Fargate memory in MiB |
| `desired_count` | No | `2` | Number of ECS task instances |
| `health_check_path` | No | `/health` | ALB health check endpoint |
| `vpc_cidr` | No | `10.0.0.0/16` | VPC CIDR block |
