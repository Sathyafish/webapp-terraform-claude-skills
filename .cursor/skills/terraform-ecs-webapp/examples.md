# Terraform ECS Webapp - Examples

## Example 1: Basic Webapp Deployment

A minimal webapp deployment with all required components.

### `terraform.tfvars`

```hcl
project_name    = "mywebapp"
environment     = "production"
aws_region      = "us-east-1"
vpc_cidr        = "10.0.0.0/16"
container_image = "123456789012.dkr.ecr.us-east-1.amazonaws.com/mywebapp:latest"
container_port  = 8080
container_cpu   = 256
container_memory = 512
desired_count   = 2
health_check_path = "/health"
certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/abc-123-def"
availability_zone_count = 2
log_retention_days = 30
enable_waf_managed_rules = true
```

### Deployment Commands

```bash
cd terraform/
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

### Post-Deployment Verification

```bash
# Get the ALB DNS name
terraform output alb_dns_name

# Verify ECS service is running
aws ecs describe-services \
  --cluster $(terraform output -raw ecs_cluster_name) \
  --services $(terraform output -raw ecs_service_name) \
  --query 'services[0].runningCount'

# Verify WAF is attached
aws wafv2 get-web-acl-for-resource \
  --resource-arn $(terraform output -raw alb_arn)
```

---

## Example 2: Webapp with Secrets from SSM

Passing database credentials via SSM Parameter Store.

### Additional Terraform for SSM parameters

```hcl
resource "aws_ssm_parameter" "db_host" {
  name  = "/${var.project_name}/db-host"
  type  = "SecureString"
  value = "PLACEHOLDER"

  lifecycle {
    ignore_changes = [value]
  }
}

resource "aws_ssm_parameter" "db_password" {
  name  = "/${var.project_name}/db-password"
  type  = "SecureString"
  value = "PLACEHOLDER"

  lifecycle {
    ignore_changes = [value]
  }
}
```

### `terraform.tfvars` with secrets

```hcl
project_name    = "mywebapp"
environment     = "production"
aws_region      = "us-east-1"
container_image = "123456789012.dkr.ecr.us-east-1.amazonaws.com/mywebapp:latest"
container_port  = 3000
certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/abc-123-def"

container_environment = [
  { name = "NODE_ENV", value = "production" },
  { name = "PORT", value = "3000" }
]

container_secrets = [
  { name = "DB_HOST", valueFrom = "arn:aws:ssm:us-east-1:123456789012:parameter/mywebapp/db-host" },
  { name = "DB_PASSWORD", valueFrom = "arn:aws:ssm:us-east-1:123456789012:parameter/mywebapp/db-password" }
]
```

---

## Example 3: Multi-Webapp Setup

Deploying multiple webapps by reusing the Terraform as a module.

### Module structure

```
infrastructure/
├── modules/
│   └── ecs-webapp/        # Copy all .tf files here
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── vpc.tf
│       ├── security-groups.tf
│       ├── alb.tf
│       ├── ecs.tf
│       ├── waf.tf
│       ├── iam.tf
│       └── logs.tf
├── webapp-a/
│   └── main.tf
└── webapp-b/
    └── main.tf
```

### `webapp-a/main.tf`

```hcl
module "webapp" {
  source = "../modules/ecs-webapp"

  project_name    = "webapp-a"
  environment     = "production"
  aws_region      = "us-east-1"
  container_image = "123456789012.dkr.ecr.us-east-1.amazonaws.com/webapp-a:latest"
  container_port  = 8080
  certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/abc-123-def"
}

output "alb_dns_name" {
  value = module.webapp.alb_dns_name
}
```

---

## Example 4: Adding Auto Scaling

Extend the base ECS service with auto-scaling based on CPU utilization.

### `autoscaling.tf`

```hcl
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = var.desired_count
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.main.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "${var.project_name}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

---

## Example 5: Adding Route 53 DNS Record

Point a domain to the ALB.

### `dns.tf`

```hcl
variable "domain_name" {
  description = "Domain name for the webapp"
  type        = string
}

variable "route53_zone_id" {
  description = "Route 53 hosted zone ID"
  type        = string
}

resource "aws_route53_record" "webapp" {
  zone_id = var.route53_zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}
```

---

## WAF Testing

Verify Russian IP blocking is working:

```bash
# Check WAF rules
aws wafv2 get-web-acl \
  --name "$(terraform output -raw project_name)-waf-acl" \
  --scope REGIONAL \
  --id "$(terraform output -raw waf_web_acl_id)" \
  --query 'WebACL.Rules[?Name==`block-russia-traffic`]'

# Check WAF sampled requests (after some traffic)
aws wafv2 get-sampled-requests \
  --web-acl-arn "$(terraform output -raw waf_web_acl_arn)" \
  --rule-metric-name "$(terraform output -raw project_name)-russia-block" \
  --scope REGIONAL \
  --time-window '{"StartTime":"2025-01-01T00:00:00Z","EndTime":"2025-12-31T23:59:59Z"}' \
  --max-items 10
```

## Security Verification Checklist

After deployment, confirm:

```bash
# 1. Verify ALB is in public subnets
aws elbv2 describe-load-balancers \
  --names "$(terraform output -raw project_name)-alb" \
  --query 'LoadBalancers[0].AvailabilityZones'

# 2. Verify ECS tasks are in private subnets
aws ecs describe-tasks \
  --cluster "$(terraform output -raw ecs_cluster_name)" \
  --tasks $(aws ecs list-tasks --cluster "$(terraform output -raw ecs_cluster_name)" --query 'taskArns[0]' --output text) \
  --query 'tasks[0].attachments[0].details[?name==`subnetId`].value'

# 3. Verify ALB SG only allows 443
aws ec2 describe-security-groups \
  --group-ids "$(terraform output -raw alb_security_group_id)" \
  --query 'SecurityGroups[0].IpPermissions'

# 4. Verify WAF is associated with ALB
aws wafv2 get-web-acl-for-resource \
  --resource-arn "$(terraform output -raw alb_arn)"
```
