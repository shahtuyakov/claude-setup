# Cloud Deployment Patterns

## Vercel

### Project Configuration

```json
// vercel.json
{
  "framework": "nextjs",
  "regions": ["iad1", "sfo1"],
  "functions": {
    "app/api/**/*.ts": {
      "memory": 1024,
      "maxDuration": 30
    }
  },
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Cache-Control", "value": "no-store" }
      ]
    }
  ],
  "rewrites": [
    { "source": "/blog/:path*", "destination": "/posts/:path*" }
  ],
  "redirects": [
    { "source": "/old-page", "destination": "/new-page", "permanent": true }
  ]
}
```

### Environment Variables

```bash
# Set via CLI
vercel env add DATABASE_URL production
vercel env add DATABASE_URL preview
vercel env add DATABASE_URL development

# Pull to local
vercel env pull .env.local
```

### Edge Functions

```typescript
// app/api/edge/route.ts
export const runtime = 'edge';

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);

  return new Response(JSON.stringify({
    message: 'Hello from the edge!',
    region: process.env.VERCEL_REGION
  }), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

### Serverless Functions

```typescript
// app/api/data/route.ts
export const dynamic = 'force-dynamic';
export const maxDuration = 30; // seconds

export async function GET() {
  const data = await fetchFromDatabase();
  return Response.json(data);
}
```

### Deployment Commands

```bash
# Deploy preview
vercel

# Deploy production
vercel --prod

# Link to existing project
vercel link

# List deployments
vercel ls
```

---

## Railway

### Configuration

```json
// railway.json
{
  "build": {
    "builder": "nixpacks",
    "buildCommand": "npm run build"
  },
  "deploy": {
    "startCommand": "npm start",
    "healthcheckPath": "/health",
    "healthcheckTimeout": 300,
    "restartPolicyType": "ON_FAILURE",
    "restartPolicyMaxRetries": 3
  }
}
```

### Nixpacks Configuration

```toml
# nixpacks.toml
[phases.setup]
nixPkgs = ["nodejs-20_x", "postgresql"]

[phases.install]
cmds = ["npm ci"]

[phases.build]
cmds = ["npm run build"]

[start]
cmd = "npm start"
```

### Database Setup

```bash
# Add PostgreSQL plugin
railway add -p postgresql

# Connect to database
railway connect postgresql

# Run migrations
railway run npm run db:migrate
```

### Environment & Secrets

```bash
# Set environment variables
railway variables set DATABASE_URL=postgresql://...
railway variables set NODE_ENV=production

# View variables
railway variables

# Run command with variables
railway run npm run seed
```

### Deployment

```bash
# Deploy current directory
railway up

# Deploy with specific service
railway up --service api

# View logs
railway logs

# Open shell
railway shell
```

---

## Fly.io

### Configuration

```toml
# fly.toml
app = "myapp"
primary_region = "iad"

[build]
  dockerfile = "Dockerfile"

[env]
  NODE_ENV = "production"
  PORT = "8080"

[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = true
  auto_start_machines = true
  min_machines_running = 1

  [http_service.concurrency]
    type = "requests"
    hard_limit = 250
    soft_limit = 200

[[services]]
  protocol = "tcp"
  internal_port = 8080

  [[services.ports]]
    port = 80
    handlers = ["http"]

  [[services.ports]]
    port = 443
    handlers = ["tls", "http"]

  [[services.tcp_checks]]
    grace_period = "30s"
    interval = "15s"
    restart_limit = 0
    timeout = "2s"

  [[services.http_checks]]
    interval = "10s"
    grace_period = "5s"
    method = "get"
    path = "/health"
    protocol = "http"
    timeout = "2s"

[mounts]
  source = "data"
  destination = "/data"

[[vm]]
  cpu_kind = "shared"
  cpus = 1
  memory_mb = 512
```

### Multi-Region Deployment

```toml
# fly.toml
primary_region = "iad"

[env]
  PRIMARY_REGION = "iad"

# Scale to multiple regions
# flyctl regions add fra syd
```

```bash
# Add regions
fly regions add fra syd sin

# Scale machines
fly scale count 2 --region iad
fly scale count 1 --region fra
fly scale count 1 --region syd

# View status
fly status
```

### Database (Fly Postgres)

```bash
# Create database
fly postgres create --name myapp-db

# Attach to app
fly postgres attach myapp-db

# Connect directly
fly postgres connect -a myapp-db

# Proxy for local access
fly proxy 5432 -a myapp-db
```

### Secrets

```bash
# Set secrets
fly secrets set DATABASE_URL=postgresql://...
fly secrets set JWT_SECRET=super-secret-key

# List secrets
fly secrets list

# Unset
fly secrets unset OLD_SECRET
```

### Deployment

```bash
# Deploy
fly deploy

# Deploy with build args
fly deploy --build-arg NODE_ENV=production

# Rollback
fly releases list
fly deploy --image registry.fly.io/myapp:v5

# Scale
fly scale memory 1024
fly scale vm shared-cpu-2x
```

---

## Render

### Blueprint (Infrastructure as Code)

```yaml
# render.yaml
services:
  - type: web
    name: api
    runtime: node
    region: oregon
    buildCommand: npm ci && npm run build
    startCommand: npm start
    healthCheckPath: /health
    envVars:
      - key: NODE_ENV
        value: production
      - key: DATABASE_URL
        fromDatabase:
          name: mydb
          property: connectionString
    autoDeploy: true

  - type: worker
    name: worker
    runtime: node
    buildCommand: npm ci && npm run build
    startCommand: npm run worker
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: mydb
          property: connectionString

  - type: cron
    name: daily-job
    runtime: node
    buildCommand: npm ci && npm run build
    schedule: "0 0 * * *"
    startCommand: npm run daily-task

databases:
  - name: mydb
    databaseName: myapp
    plan: starter
    region: oregon

envVarGroups:
  - name: shared-settings
    envVars:
      - key: APP_NAME
        value: myapp
```

### Static Sites

```yaml
# render.yaml
services:
  - type: web
    name: frontend
    runtime: static
    buildCommand: npm run build
    staticPublishPath: ./dist
    headers:
      - path: /*
        name: Cache-Control
        value: public, max-age=31536000, immutable
    routes:
      - type: rewrite
        source: /*
        destination: /index.html
```

---

## AWS ECS with Fargate

### Task Definition

```json
{
  "family": "myapp",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/myapp:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "NODE_ENV", "value": "production" }
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:ACCOUNT:secret:myapp/database"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/myapp",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -q --spider http://localhost:3000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### Service Definition (CloudFormation)

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Resources:
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: myapp-cluster
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT
      DefaultCapacityProviderStrategy:
        - CapacityProvider: FARGATE
          Weight: 1
        - CapacityProvider: FARGATE_SPOT
          Weight: 4

  Service:
    Type: AWS::ECS::Service
    Properties:
      ServiceName: myapp-service
      Cluster: !Ref ECSCluster
      TaskDefinition: !Ref TaskDefinition
      DesiredCount: 2
      LaunchType: FARGATE
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: ENABLED
          Subnets:
            - !Ref PublicSubnet1
            - !Ref PublicSubnet2
          SecurityGroups:
            - !Ref ServiceSecurityGroup
      LoadBalancers:
        - ContainerName: app
          ContainerPort: 3000
          TargetGroupArn: !Ref TargetGroup
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100
        DeploymentCircuitBreaker:
          Enable: true
          Rollback: true
```

### Auto Scaling

```yaml
ScalableTarget:
  Type: AWS::ApplicationAutoScaling::ScalableTarget
  Properties:
    MaxCapacity: 10
    MinCapacity: 2
    ResourceId: !Sub service/${ECSCluster}/${Service.Name}
    RoleARN: !GetAtt AutoScalingRole.Arn
    ScalableDimension: ecs:service:DesiredCount
    ServiceNamespace: ecs

ScalingPolicy:
  Type: AWS::ApplicationAutoScaling::ScalingPolicy
  Properties:
    PolicyName: cpu-scaling
    PolicyType: TargetTrackingScaling
    ScalingTargetId: !Ref ScalableTarget
    TargetTrackingScalingPolicyConfiguration:
      PredefinedMetricSpecification:
        PredefinedMetricType: ECSServiceAverageCPUUtilization
      TargetValue: 70
      ScaleInCooldown: 300
      ScaleOutCooldown: 60
```

---

## AWS Lambda

### Serverless Framework

```yaml
# serverless.yml
service: myapp

provider:
  name: aws
  runtime: nodejs20.x
  region: us-east-1
  memorySize: 256
  timeout: 30
  environment:
    NODE_ENV: production
    DATABASE_URL: ${ssm:/myapp/database-url}
  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:GetItem
            - dynamodb:PutItem
          Resource: !GetAtt DynamoTable.Arn

functions:
  api:
    handler: dist/handler.main
    events:
      - http:
          path: /{proxy+}
          method: any
          cors: true
    vpc:
      securityGroupIds:
        - !Ref LambdaSecurityGroup
      subnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2

  worker:
    handler: dist/worker.handler
    events:
      - sqs:
          arn: !GetAtt WorkerQueue.Arn
          batchSize: 10

  scheduled:
    handler: dist/cron.handler
    events:
      - schedule: rate(1 hour)

resources:
  Resources:
    DynamoTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:service}-${sls:stage}
        BillingMode: PAY_PER_REQUEST
        AttributeDefinitions:
          - AttributeName: pk
            AttributeType: S
          - AttributeName: sk
            AttributeType: S
        KeySchema:
          - AttributeName: pk
            KeyType: HASH
          - AttributeName: sk
            KeyType: RANGE
```

---

## Deployment Strategies

### Blue-Green Deployment

```yaml
# AWS CodeDeploy appspec.yml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: app
          ContainerPort: 3000
        PlatformVersion: LATEST

Hooks:
  - BeforeInstall: "scripts/before_install.sh"
  - AfterInstall: "scripts/after_install.sh"
  - AfterAllowTestTraffic: "scripts/test_traffic.sh"
  - BeforeAllowTraffic: "scripts/before_traffic.sh"
  - AfterAllowTraffic: "scripts/after_traffic.sh"
```

### Canary Deployment

```yaml
# AWS CodeDeploy deployment config
DeploymentConfig:
  Type: AWS::CodeDeploy::DeploymentConfig
  Properties:
    DeploymentConfigName: Canary10Percent5Minutes
    ComputePlatform: ECS
    TrafficRoutingConfig:
      Type: TimeBasedCanary
      TimeBasedCanary:
        CanaryPercentage: 10
        CanaryInterval: 5  # minutes
```

### Rolling Update (Kubernetes style)

```yaml
# For ECS
DeploymentConfiguration:
  MaximumPercent: 150
  MinimumHealthyPercent: 100

# Deploys new tasks before removing old ones
# 100% healthy minimum = no downtime
```

---

## Environment Management

### Environment Variables Best Practices

```bash
# Development
.env.local          # Local overrides (gitignored)
.env.development    # Dev defaults

# Staging/Preview
.env.staging        # Staging config

# Production
.env.production     # Prod config (no secrets!)

# Secrets - use platform secret management
# Vercel: vercel env add
# Railway: railway variables set
# Fly.io: fly secrets set
# AWS: Secrets Manager / Parameter Store
```

### Multi-Environment Configuration

```typescript
// config/index.ts
const config = {
  development: {
    apiUrl: 'http://localhost:3000',
    debug: true,
  },
  staging: {
    apiUrl: 'https://staging.myapp.com',
    debug: true,
  },
  production: {
    apiUrl: 'https://api.myapp.com',
    debug: false,
  },
};

export default config[process.env.NODE_ENV || 'development'];
```

---

## Platform Comparison

| Feature | Vercel | Railway | Fly.io | Render | AWS ECS |
|---------|--------|---------|--------|--------|---------|
| Best For | Next.js | Rapid dev | Global edge | Full-stack | Enterprise |
| Pricing | $20/user | $5 hobby | Pay-per-use | $7+ | Usage-based |
| Auto-scale | ✅ | ✅ | ✅ | ✅ | ✅ |
| Database | ❌ | ✅ | ✅ | ✅ | ✅ |
| Docker | Limited | ✅ | ✅ | ✅ | ✅ |
| Edge | ✅ | ❌ | ✅ | ❌ | CloudFront |
| Setup time | Minutes | Minutes | Minutes | Minutes | Hours |
| Complexity | Low | Low | Medium | Low | High |
