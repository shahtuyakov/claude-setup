# Infrastructure as Code

## Terraform

### Project Structure

```
infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── versions.tf
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ecs/
│   └── rds/
└── environments/
    ├── staging/
    │   ├── main.tf
    │   ├── terraform.tfvars
    │   └── backend.tf
    └── production/
        ├── main.tf
        ├── terraform.tfvars
        └── backend.tf
```

### Provider Configuration

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}

# providers.tf
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "terraform"
    }
  }
}
```

### Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "myapp-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Variables

```hcl
# variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["staging", "production"], var.environment)
    error_message = "Environment must be staging or production."
  }
}

variable "project_name" {
  description = "Project name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "database_config" {
  description = "Database configuration"
  type = object({
    instance_class = string
    engine_version = string
    multi_az       = bool
  })
  default = {
    instance_class = "db.t3.micro"
    engine_version = "16"
    multi_az       = false
  }
}
```

```hcl
# terraform.tfvars (production)
environment  = "production"
project_name = "myapp"
aws_region   = "us-east-1"
vpc_cidr     = "10.0.0.0/16"

database_config = {
  instance_class = "db.r6g.large"
  engine_version = "16"
  multi_az       = true
}
```

---

## AWS Resources

### VPC Module

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  count                   = length(var.availability_zones)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "public"
  }
}

resource "aws_subnet" "private" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index + length(var.availability_zones))
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-private-${count.index + 1}"
    Type = "private"
  }
}

resource "aws_nat_gateway" "main" {
  count         = var.enable_nat_gateway ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0
  domain = "vpc"
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[0].id
    }
  }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = length(aws_subnet.private)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private.id
}
```

### ECS with Fargate

```hcl
# modules/ecs/main.tf
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = "${var.project_name}-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.cpu
  memory                   = var.memory
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name  = "app"
      image = "${var.ecr_repository_url}:${var.image_tag}"

      portMappings = [
        {
          containerPort = var.container_port
          protocol      = "tcp"
        }
      ]

      environment = [
        for key, value in var.environment_variables : {
          name  = key
          value = value
        }
      ]

      secrets = [
        for key, arn in var.secrets : {
          name      = key
          valueFrom = arn
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "wget -q --spider http://localhost:${var.container_port}/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])
}

resource "aws_ecs_service" "app" {
  name            = "${var.project_name}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 100
  }

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = var.container_port
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  lifecycle {
    ignore_changes = [desired_count]
  }
}

resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = var.max_capacity
  min_capacity       = var.min_capacity
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

### RDS PostgreSQL

```hcl
# modules/rds/main.tf
resource "aws_db_subnet_group" "main" {
  name       = "${var.project_name}-db-subnet"
  subnet_ids = var.private_subnet_ids
}

resource "aws_security_group" "rds" {
  name        = "${var.project_name}-rds-sg"
  description = "Security group for RDS"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = var.allowed_security_groups
  }
}

resource "random_password" "db_password" {
  length  = 32
  special = false
}

resource "aws_secretsmanager_secret" "db_credentials" {
  name = "${var.project_name}/database"
}

resource "aws_secretsmanager_secret_version" "db_credentials" {
  secret_id = aws_secretsmanager_secret.db_credentials.id
  secret_string = jsonencode({
    username = var.db_username
    password = random_password.db_password.result
    host     = aws_db_instance.main.address
    port     = 5432
    database = var.db_name
  })
}

resource "aws_db_instance" "main" {
  identifier     = "${var.project_name}-db"
  engine         = "postgres"
  engine_version = var.engine_version
  instance_class = var.instance_class

  allocated_storage     = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = var.db_name
  username = var.db_username
  password = random_password.db_password.result

  multi_az               = var.multi_az
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  backup_retention_period = 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "Mon:04:00-Mon:05:00"

  deletion_protection = var.environment == "production"
  skip_final_snapshot = var.environment != "production"
  final_snapshot_identifier = var.environment == "production" ? "${var.project_name}-final-snapshot" : null

  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_monitoring.arn

  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
}
```

---

## Pulumi

### Project Structure (TypeScript)

```
infrastructure/
├── Pulumi.yaml
├── Pulumi.staging.yaml
├── Pulumi.production.yaml
├── index.ts
├── vpc.ts
├── ecs.ts
├── rds.ts
└── package.json
```

### Pulumi Configuration

```yaml
# Pulumi.yaml
name: myapp-infrastructure
runtime: nodejs
description: Infrastructure for MyApp

# Pulumi.production.yaml
config:
  aws:region: us-east-1
  myapp:environment: production
  myapp:vpcCidr: 10.0.0.0/16
  myapp:dbInstanceClass: db.r6g.large
  myapp:desiredCount: 3
```

### VPC Module

```typescript
// vpc.ts
import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

interface VpcArgs {
  projectName: string;
  vpcCidr: string;
  availabilityZones: string[];
  enableNatGateway?: boolean;
}

export class Vpc extends pulumi.ComponentResource {
  public readonly vpc: aws.ec2.Vpc;
  public readonly publicSubnets: aws.ec2.Subnet[];
  public readonly privateSubnets: aws.ec2.Subnet[];

  constructor(name: string, args: VpcArgs, opts?: pulumi.ComponentResourceOptions) {
    super("myapp:vpc:Vpc", name, {}, opts);

    const { projectName, vpcCidr, availabilityZones, enableNatGateway = false } = args;

    // VPC
    this.vpc = new aws.ec2.Vpc(`${name}-vpc`, {
      cidrBlock: vpcCidr,
      enableDnsHostnames: true,
      enableDnsSupport: true,
      tags: { Name: `${projectName}-vpc` },
    }, { parent: this });

    // Internet Gateway
    const igw = new aws.ec2.InternetGateway(`${name}-igw`, {
      vpcId: this.vpc.id,
    }, { parent: this });

    // Public Subnets
    this.publicSubnets = availabilityZones.map((az, i) => {
      return new aws.ec2.Subnet(`${name}-public-${i}`, {
        vpcId: this.vpc.id,
        cidrBlock: `10.0.${i}.0/24`,
        availabilityZone: az,
        mapPublicIpOnLaunch: true,
        tags: { Name: `${projectName}-public-${i}` },
      }, { parent: this });
    });

    // Private Subnets
    this.privateSubnets = availabilityZones.map((az, i) => {
      return new aws.ec2.Subnet(`${name}-private-${i}`, {
        vpcId: this.vpc.id,
        cidrBlock: `10.0.${i + 10}.0/24`,
        availabilityZone: az,
        tags: { Name: `${projectName}-private-${i}` },
      }, { parent: this });
    });

    // Route tables
    const publicRt = new aws.ec2.RouteTable(`${name}-public-rt`, {
      vpcId: this.vpc.id,
      routes: [{
        cidrBlock: "0.0.0.0/0",
        gatewayId: igw.id,
      }],
    }, { parent: this });

    this.publicSubnets.forEach((subnet, i) => {
      new aws.ec2.RouteTableAssociation(`${name}-public-rta-${i}`, {
        subnetId: subnet.id,
        routeTableId: publicRt.id,
      }, { parent: this });
    });

    this.registerOutputs({
      vpcId: this.vpc.id,
      publicSubnetIds: this.publicSubnets.map(s => s.id),
      privateSubnetIds: this.privateSubnets.map(s => s.id),
    });
  }
}
```

### ECS Service

```typescript
// ecs.ts
import * as aws from "@pulumi/aws";
import * as awsx from "@pulumi/awsx";
import * as pulumi from "@pulumi/pulumi";

interface EcsArgs {
  projectName: string;
  vpcId: pulumi.Input<string>;
  privateSubnetIds: pulumi.Input<string>[];
  publicSubnetIds: pulumi.Input<string>[];
  imageUri: pulumi.Input<string>;
  cpu?: number;
  memory?: number;
  desiredCount?: number;
  environmentVariables?: Record<string, pulumi.Input<string>>;
}

export class EcsService extends pulumi.ComponentResource {
  public readonly service: awsx.ecs.FargateService;
  public readonly loadBalancerDns: pulumi.Output<string>;

  constructor(name: string, args: EcsArgs, opts?: pulumi.ComponentResourceOptions) {
    super("myapp:ecs:Service", name, {}, opts);

    const {
      projectName,
      vpcId,
      privateSubnetIds,
      publicSubnetIds,
      imageUri,
      cpu = 256,
      memory = 512,
      desiredCount = 2,
      environmentVariables = {},
    } = args;

    // ALB
    const lb = new awsx.lb.ApplicationLoadBalancer(`${name}-alb`, {
      subnetIds: publicSubnetIds,
    }, { parent: this });

    // ECS Cluster
    const cluster = new aws.ecs.Cluster(`${name}-cluster`, {
      settings: [{
        name: "containerInsights",
        value: "enabled",
      }],
    }, { parent: this });

    // Fargate Service
    this.service = new awsx.ecs.FargateService(`${name}-service`, {
      cluster: cluster.arn,
      desiredCount,
      networkConfiguration: {
        subnets: privateSubnetIds,
        assignPublicIp: false,
      },
      taskDefinitionArgs: {
        container: {
          name: "app",
          image: imageUri,
          cpu,
          memory,
          essential: true,
          portMappings: [{
            containerPort: 3000,
            targetGroup: lb.defaultTargetGroup,
          }],
          environment: Object.entries(environmentVariables).map(([name, value]) => ({
            name,
            value,
          })),
        },
      },
    }, { parent: this });

    this.loadBalancerDns = lb.loadBalancer.dnsName;

    this.registerOutputs({
      serviceArn: this.service.service.arn,
      loadBalancerDns: this.loadBalancerDns,
    });
  }
}
```

### Main Entry Point

```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as aws from "@pulumi/aws";
import { Vpc } from "./vpc";
import { EcsService } from "./ecs";
import { RdsInstance } from "./rds";

const config = new pulumi.Config();
const projectName = config.require("projectName");
const environment = config.require("environment");
const vpcCidr = config.get("vpcCidr") || "10.0.0.0/16";

// Get availability zones
const azs = aws.getAvailabilityZones({
  state: "available",
});

// VPC
const vpc = new Vpc("main", {
  projectName,
  vpcCidr,
  availabilityZones: azs.then(a => a.names.slice(0, 2)),
  enableNatGateway: environment === "production",
});

// RDS
const database = new RdsInstance("main", {
  projectName,
  vpcId: vpc.vpc.id,
  subnetIds: vpc.privateSubnets.map(s => s.id),
  instanceClass: config.get("dbInstanceClass") || "db.t3.micro",
  multiAz: environment === "production",
});

// ECR Repository
const repo = new aws.ecr.Repository("app", {
  name: `${projectName}-app`,
  imageTagMutability: "MUTABLE",
});

// ECS Service
const service = new EcsService("app", {
  projectName,
  vpcId: vpc.vpc.id,
  privateSubnetIds: vpc.privateSubnets.map(s => s.id),
  publicSubnetIds: vpc.publicSubnets.map(s => s.id),
  imageUri: pulumi.interpolate`${repo.repositoryUrl}:latest`,
  desiredCount: config.getNumber("desiredCount") || 2,
  environmentVariables: {
    NODE_ENV: environment,
    DATABASE_URL: database.connectionString,
  },
});

// Exports
export const vpcId = vpc.vpc.id;
export const loadBalancerUrl = pulumi.interpolate`http://${service.loadBalancerDns}`;
export const ecrRepositoryUrl = repo.repositoryUrl;
```

---

## Best Practices

### State Management

```hcl
# Remote state with locking
terraform {
  backend "s3" {
    bucket         = "terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

### Secrets Handling

```hcl
# Never commit secrets - use variables or secret managers
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

# Use AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "myapp/database"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db.secret_string)
}
```

### Module Versioning

```hcl
# Pin module versions
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  # ...
}

# Or use Git tags
module "custom" {
  source = "git::https://github.com/org/modules.git//vpc?ref=v1.2.3"
}
```

---

## Commands Reference

### Terraform

```bash
# Initialize
terraform init

# Plan
terraform plan -var-file=production.tfvars

# Apply
terraform apply -var-file=production.tfvars

# Destroy
terraform destroy -var-file=production.tfvars

# Format
terraform fmt -recursive

# Validate
terraform validate

# State management
terraform state list
terraform state show aws_instance.main
terraform state mv aws_instance.old aws_instance.new
terraform import aws_instance.main i-1234567890

# Workspace management
terraform workspace list
terraform workspace new staging
terraform workspace select production
```

### Pulumi

```bash
# Initialize
pulumi new aws-typescript

# Preview
pulumi preview

# Deploy
pulumi up

# Destroy
pulumi destroy

# Stack management
pulumi stack ls
pulumi stack select production
pulumi stack export > backup.json
pulumi stack import < backup.json

# Config
pulumi config set aws:region us-east-1
pulumi config set --secret dbPassword supersecret

# Refresh state
pulumi refresh
```

---

## Comparison

| Feature | Terraform | Pulumi |
|---------|-----------|--------|
| Language | HCL | TypeScript, Python, Go, etc. |
| State | Remote backends | Pulumi Cloud or self-managed |
| Learning curve | Lower | Higher (need programming) |
| Testing | Limited | Full unit testing |
| IDE support | Good | Excellent |
| Ecosystem | Larger | Growing |
| Best for | Standard infra | Complex logic, loops |
