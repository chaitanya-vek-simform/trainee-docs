# ☁️ DevOps Automation Notes — Part 2: AWS Automation Deep Dive

> **Series:** Part 2 of 4 | ← [Part 1](./part1_intro_and_what_to_automate.md) | [Part 3 →](./part3_azure_automation.md)

---

## Table of Contents

1. [AWS IAM Automation](#1-aws-iam-automation)
2. [EC2 & Auto Scaling Automation](#2-ec2--auto-scaling-automation)
3. [S3 Automation](#3-s3-automation)
4. [RDS & Database Automation](#4-rds--database-automation)
5. [Lambda & Serverless Automation](#5-lambda--serverless-automation)
6. [EKS (Kubernetes) Automation](#6-eks-kubernetes-automation)
7. [CI/CD: CodePipeline + CodeBuild](#7-cicd-codepipeline--codebuild)
8. [CloudWatch: Monitoring & Alerting Automation](#8-cloudwatch-monitoring--alerting-automation)
9. [Security Automation on AWS](#9-security-automation-on-aws)
10. [Cost Optimization Automation on AWS](#10-cost-optimization-automation-on-aws)

---

## 1. AWS IAM Automation

### What is IAM and why automate it?

IAM = Identity and Access Management. It controls **who** can do **what** on **which** AWS resource.

```
┌────────────────────────────────────────────────────────────────┐
│  IAM — CORE MENTAL MODEL                                       │
│                                                                │
│  Principal (WHO)   +  Action (WHAT)  +  Resource (WHERE)      │
│       │                   │                   │               │
│  ┌────▼──────┐      ┌─────▼──────┐    ┌──────▼──────┐       │
│  │ IAM User  │      │ s3:GetObject│    │ arn:aws:s3: │       │
│  │ IAM Role  │      │ ec2:Start   │    │ :::my-bucket│       │
│  │ IAM Group │      │ rds:Describe│    │ /*          │       │
│  │ AWS Svc   │      └────────────┘    └─────────────┘       │
│  └───────────┘                                                │
│                                                                │
│  LEAST PRIVILEGE RULE: Grant ONLY what is needed, nothing more│
└────────────────────────────────────────────────────────────────┘
```

### What to Automate with IAM

| Task | How to Automate | Tool |
|------|----------------|------|
| Create roles for EC2/Lambda | Terraform `aws_iam_role` resource | Terraform |
| Attach policies to roles | `aws_iam_role_policy_attachment` | Terraform |
| Rotate access keys | Lambda + EventBridge schedule | AWS Lambda |
| Detect unused IAM users | AWS Config rule + Lambda remediation | AWS Config |
| OIDC for GitHub Actions | `aws_iam_openid_connect_provider` | Terraform |
| Enforce MFA policy | SCP (Service Control Policy) in AWS Org | AWS Organizations |

### Key Automation: OIDC for CI/CD (No Static Keys)

```
┌────────────────────────────────────────────────────────────────┐
│  GITHUB ACTIONS → AWS (no hardcoded keys)                      │
│                                                                │
│  GitHub Actions                                                │
│       │  1. Request short-lived token from GitHub OIDC        │
│       ▼                                                        │
│  GitHub OIDC Provider                                          │
│       │  2. Send JWT token to AWS STS                         │
│       ▼                                                        │
│  AWS STS (AssumeRoleWithWebIdentity)                           │
│       │  3. Return temp credentials (15min - 1hr)             │
│       ▼                                                        │
│  AWS Resources (S3, ECR, EKS, etc.)                            │
│                                                                │
│  Result: No stored secrets. Credentials expire automatically.  │
└────────────────────────────────────────────────────────────────┘
```

```yaml
# GitHub Actions workflow — AWS OIDC (no access keys needed)
jobs:
  deploy:
    permissions:
      id-token: write   # Required for OIDC
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
```

> **Gotcha:** Never store `AWS_ACCESS_KEY_ID` as a GitHub secret for production pipelines. Use OIDC instead. Static keys that leak = instant security incident.

---

## 2. EC2 & Auto Scaling Automation

### Architecture Flow

```
┌────────────────────────────────────────────────────────────────┐
│  EC2 AUTO SCALING — HOW IT WORKS                               │
│                                                                │
│  CloudWatch Alarm                                              │
│  "CPU > 70% for 5 min"                                         │
│       │                                                        │
│       ▼                                                        │
│  Auto Scaling Group (ASG)                                      │
│  ├── Launch Template (defines EC2 config)                     │
│  ├── Min: 2  Desired: 3  Max: 10                              │
│  └── Scaling Policy: add 2 instances                          │
│       │                                                        │
│       ▼                                                        │
│  New EC2 instances launch                                      │
│  ├── cloud-init runs (install app, configure)                  │
│  └── Register with ALB Target Group                           │
│       │                                                        │
│       ▼                                                        │
│  ALB routes traffic to healthy instances                       │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate with EC2

| Task | Automation Method |
|------|------------------|
| Provision EC2 instances | Terraform `aws_instance` |
| Bootstrap software on launch | `user_data` script (cloud-init) |
| Auto-scale on CPU/memory | ASG + CloudWatch alarms |
| Patch OS automatically | AWS Systems Manager Patch Manager |
| Run commands without SSH | SSM Session Manager / Run Command |
| Schedule start/stop of dev instances | EventBridge + Lambda |
| Take EBS snapshots | AWS Data Lifecycle Manager (DLM) |

### Auto-Shutdown Dev Instances (Cost Saver)

```bash
# Lambda function triggered by EventBridge schedule
# Stop all EC2 instances tagged Environment=dev at 8 PM weekdays

import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2', region_name='us-east-1')
    
    instances = ec2.describe_instances(
        Filters=[
            {'Name': 'tag:Environment', 'Values': ['dev']},
            {'Name': 'instance-state-name', 'Values': ['running']}
        ]
    )
    
    ids = [i['InstanceId'] 
           for r in instances['Reservations'] 
           for i in r['Instances']]
    
    if ids:
        ec2.stop_instances(InstanceIds=ids)
        print(f"Stopped {len(ids)} dev instances: {ids}")
```

> **Gotcha:** `stop` vs `terminate` — `stop` keeps the instance (restartable), `terminate` permanently destroys it. Never terminate production instances via automation without a confirmation step. Also: stopped EC2 still charges for EBS storage.

---

## 3. S3 Automation

### S3 Storage Lifecycle — Cost Automation

```
┌────────────────────────────────────────────────────────────────┐
│  S3 LIFECYCLE POLICY — AUTOMATIC COST OPTIMIZATION            │
│                                                                │
│  Day 0   Object uploaded to S3 Standard                       │
│           Cost: $0.023/GB/month                               │
│                │                                               │
│  Day 30  Auto-move to S3 Standard-IA (Infrequent Access)     │
│           Cost: $0.0125/GB/month  (45% cheaper)               │
│                │                                               │
│  Day 90  Auto-move to S3 Glacier Instant Retrieval            │
│           Cost: $0.004/GB/month   (83% cheaper)               │
│                │                                               │
│  Day 365 Auto-move to S3 Glacier Deep Archive                 │
│           Cost: $0.00099/GB/month (96% cheaper)               │
│                │                                               │
│  Day 730 Auto-delete (if compliance allows)                   │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate with S3

| Task | How |
|------|-----|
| Create buckets with versioning/encryption | Terraform `aws_s3_bucket` |
| Lifecycle policies (tiering, expiry) | `aws_s3_bucket_lifecycle_configuration` |
| Block public access (default should be on) | `aws_s3_bucket_public_access_block` |
| Enable access logging | `aws_s3_bucket_logging` |
| Cross-region replication for DR | `aws_s3_bucket_replication_configuration` |
| Pre-signed URLs (temp access) | AWS SDK / Lambda |
| Trigger Lambda on object upload | S3 Event Notification |

### S3 Event-Driven Automation

```
┌────────────────────────────────────────────────────────────────┐
│  S3 EVENT → LAMBDA → PROCESSING PIPELINE                      │
│                                                                │
│  User uploads image to S3                                     │
│       │                                                        │
│       ▼                                                        │
│  S3 Event Notification (ObjectCreated)                        │
│       │                                                        │
│       ▼                                                        │
│  Lambda Function triggered                                     │
│  ├── Resize image                                              │
│  ├── Generate thumbnails                                       │
│  ├── Extract metadata                                         │
│  └── Store result to another S3 prefix                        │
│                                                                │
│  Common patterns:                                              │
│  ├── CSV upload → Lambda → parse → insert to RDS             │
│  ├── Log file → Lambda → parse → push to CloudWatch          │
│  └── PDF upload → Lambda → OCR → store to DynamoDB           │
└────────────────────────────────────────────────────────────────┘
```

> **Gotcha:** S3 bucket names are globally unique across ALL AWS accounts. If you try to create `my-app-bucket`, it may already be taken by someone else. Use a suffix like account ID or random string: `my-app-bucket-123456789`.

---

## 4. RDS & Database Automation

### What to Automate with RDS

| Task | Automation Method |
|------|-----------------|
| Provision RDS instance | Terraform `aws_db_instance` |
| Enable automated backups | `backup_retention_period = 7` in Terraform |
| Multi-AZ for HA | `multi_az = true` in Terraform |
| Read replicas for scaling | `aws_db_instance` with `replicate_source_db` |
| Parameter group tuning | `aws_db_parameter_group` |
| Secret rotation (password) | AWS Secrets Manager rotation Lambda |
| DB snapshots cross-region | Lambda + EventBridge |
| Restore from snapshot | Terraform `snapshot_identifier` |

### Secret Rotation Flow

```
┌────────────────────────────────────────────────────────────────┐
│  AUTOMATED DB PASSWORD ROTATION                                │
│                                                                │
│  EventBridge rule: every 30 days                              │
│       │                                                        │
│       ▼                                                        │
│  Secrets Manager triggers rotation Lambda                     │
│       │                                                        │
│       ▼                                                        │
│  Lambda:                                                       │
│  1. Generate new password                                      │
│  2. Update RDS password (ALTER USER ...)                      │
│  3. Update secret in Secrets Manager                          │
│  4. Test connection with new password                         │
│  5. Mark rotation complete                                     │
│                                                                │
│  App reads password from Secrets Manager at runtime           │
│  → App never has hardcoded passwords                          │
└────────────────────────────────────────────────────────────────┘
```

> **Gotcha:** If your app caches the DB password and Secrets Manager rotates it, your app will start getting auth failures until it re-reads the secret. Use Secrets Manager SDK with caching (TTL < rotation interval), or use IAM auth for RDS (no password at all).

---

## 5. Lambda & Serverless Automation

### Lambda Mental Model

```
┌────────────────────────────────────────────────────────────────┐
│  LAMBDA = "Run code without managing servers"                  │
│                                                                │
│  Traditional server:                Lambda:                   │
│  ┌──────────────────────┐           ┌──────────────────────┐  │
│  │ Server always running│           │ Code sleeps (no cost) │  │
│  │ Pay 24/7             │           │ Event arrives         │  │
│  │ You manage patches   │           │ Lambda wakes up       │  │
│  │ You scale manually   │           │ Runs 100ms-15min      │  │
│  └──────────────────────┘           │ Scales to 1000s auto  │  │
│                                     │ Pay only for duration │  │
│                                     └──────────────────────┘  │
│                                                                │
│  Lambda Triggers (Events):                                     │
│  ├── S3 event (file upload)                                   │
│  ├── API Gateway (HTTP request)                               │
│  ├── EventBridge (scheduled / custom events)                  │
│  ├── SQS / SNS message                                        │
│  ├── DynamoDB stream                                          │
│  ├── CloudWatch alarm                                         │
│  └── Cognito (user signup hook)                               │
└────────────────────────────────────────────────────────────────┘
```

### Common DevOps Lambda Automation Patterns

| Pattern | What it Does |
|---------|-------------|
| **Cron Job** | EventBridge schedule → Lambda (replace old cron servers) |
| **Auto-remediation** | CloudWatch Alarm → Lambda → fix the issue |
| **EC2 start/stop** | EventBridge schedule → Lambda → start/stop EC2 |
| **Cleanup orphaned resources** | Daily Lambda → find unused EBS/IPs → delete |
| **Slack alerting** | SNS → Lambda → format message → Slack webhook |
| **Cert expiry alert** | EventBridge weekly → Lambda → check ACM certs → alert |
| **Cost anomaly action** | AWS Cost Anomaly alert → Lambda → notify/throttle |

> **Gotcha — Cold Starts:** Lambda has a "cold start" delay (100ms–3s) when the function hasn't been called recently. For latency-sensitive APIs, use Provisioned Concurrency to keep Lambda warm. For background automation tasks, cold starts don't matter.

---

## 6. EKS (Kubernetes) Automation

### EKS Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  EKS — MANAGED KUBERNETES ON AWS                               │
│                                                                │
│  AWS manages:                    You manage:                   │
│  ├── Control plane (etcd,        ├── Worker nodes (EC2/Fargate)│
│  │   API server, scheduler)      ├── Node groups              │
│  ├── Control plane HA           ├── Kubernetes workloads      │
│  └── Control plane patches      ├── Cluster add-ons           │
│                                  └── IAM roles for pods (IRSA) │
│                                                                │
│  EKS Automation Flow:                                          │
│                                                                │
│  Terraform → EKS cluster + node groups                        │
│       │                                                        │
│       ▼                                                        │
│  Helm → Install cluster add-ons                               │
│  ├── AWS Load Balancer Controller                             │
│  ├── Cluster Autoscaler                                        │
│  ├── External DNS                                             │
│  └── Metrics Server                                           │
│       │                                                        │
│       ▼                                                        │
│  Argo CD → Watch Git repo → deploy app manifests to cluster   │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate with EKS

| Task | Tool |
|------|------|
| Create EKS cluster | Terraform `aws_eks_cluster` |
| Create managed node groups | Terraform `aws_eks_node_group` |
| Install add-ons (LB Controller, Autoscaler) | Helm via Terraform `helm_release` |
| IAM for pods (IRSA) | Terraform OIDC + IAM role |
| Auto-scale nodes | Cluster Autoscaler or Karpenter |
| Auto-scale pods | HPA (CPU/memory) or KEDA (events) |
| Rolling deployments | Argo CD (GitOps) |
| Upgrade K8s version | Terraform + managed node group rolling update |
| Collect logs | Fluent Bit DaemonSet → CloudWatch Logs |

> **Gotcha — Cluster Autoscaler vs Karpenter:** Cluster Autoscaler adds/removes entire nodes slowly (2-3 min). Karpenter (newer, AWS-built) provisions the right-sized node in ~60 seconds for pending pods. For new clusters, prefer Karpenter.

---

## 7. CI/CD: CodePipeline + CodeBuild

### AWS-Native CI/CD Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  AWS NATIVE CI/CD PIPELINE                                     │
│                                                                │
│  GitHub / CodeCommit                                           │
│       │  push to main branch                                   │
│       ▼                                                        │
│  CodePipeline (orchestrator)                                   │
│  ├── Stage 1: Source (pull from GitHub)                       │
│  ├── Stage 2: Build (CodeBuild)                               │
│  │   ├── Install dependencies                                  │
│  │   ├── Run tests                                            │
│  │   ├── docker build -t my-app:$COMMIT_SHA .                │
│  │   ├── docker push to ECR                                   │
│  │   └── Output: imagedefinitions.json                        │
│  ├── Stage 3: Deploy to ECS / EKS                             │
│  │   ├── ECS: CodeDeploy blue-green                           │
│  │   └── EKS: kubectl set image or Argo CD sync              │
│  └── Stage 4: Approval gate (manual for prod)                 │
│       │                                                        │
│       ▼                                                        │
│  SNS notification → Slack / Email on success/failure          │
└────────────────────────────────────────────────────────────────┘
```

### buildspec.yml (CodeBuild configuration)

```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
      - IMAGE_TAG=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c1-7)
  build:
    commands:
      - docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG .
      - docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
  post_build:
    commands:
      - printf '[{"name":"app","imageUri":"%s"}]' "$ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG" > imagedefinitions.json
artifacts:
  files: imagedefinitions.json
```

> **Gotcha:** CodeBuild runs in a fresh container every time — no persistent state. Don't rely on local files between builds. Use S3 artifacts or caching buckets to share build outputs between pipeline stages.

---

## 8. CloudWatch: Monitoring & Alerting Automation

### Observability Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  AWS OBSERVABILITY — FULL PICTURE                              │
│                                                                │
│  App/Infra                                                     │
│  ├── EC2: CPU, disk, memory (CloudWatch Agent)                │
│  ├── RDS: connections, IOPS, CPU (built-in)                   │
│  ├── ALB: request count, latency, errors (built-in)           │
│  └── Lambda: invocations, errors, duration (built-in)         │
│       │                                                        │
│       ▼                                                        │
│  CloudWatch Metrics                                            │
│       │                                                        │
│       ▼                                                        │
│  CloudWatch Alarms                                             │
│  ├── CPU > 80% for 10 min → SNS topic                        │
│  ├── Error rate > 1% → SNS topic                              │
│  └── Disk > 90% → SNS topic                                   │
│       │                                                        │
│       ▼                                                        │
│  SNS Topic                                                     │
│  ├── → PagerDuty (on-call rotation)                           │
│  ├── → Slack (team channel)                                   │
│  └── → Lambda (auto-remediation)                              │
└────────────────────────────────────────────────────────────────┘
```

### What to Automate in CloudWatch

| Task | Tool |
|------|------|
| Create alarms for EC2/RDS/Lambda | Terraform `aws_cloudwatch_metric_alarm` |
| Send alerts to Slack | SNS → Lambda → Slack webhook |
| Log retention policies | Terraform `aws_cloudwatch_log_group` with `retention_in_days` |
| Custom metrics from app | CloudWatch Embedded Metric Format (EMF) |
| Dashboard creation | Terraform `aws_cloudwatch_dashboard` |
| Anomaly detection | CloudWatch Anomaly Detection (ML-based) |
| Composite alarms | Combine multiple alarms (e.g., high CPU AND high memory) |

> **Gotcha:** EC2 does NOT send memory metrics to CloudWatch by default. You must install the **CloudWatch Agent** on the instance. Use SSM to install it automatically across all instances.

---

## 9. Security Automation on AWS

### Security Automation Stack

```
┌────────────────────────────────────────────────────────────────┐
│  AWS SECURITY AUTOMATION LAYERS                                │
│                                                                │
│  Layer 1: Prevention (stop before it happens)                 │
│  ├── SCPs: block entire actions at org level                  │
│  ├── AWS Config rules: enforce tag/encryption requirements    │
│  ├── IAM permission boundaries: cap what devs can do          │
│  └── Terraform + tfsec/checkov: scan IaC before deploy        │
│                                                                │
│  Layer 2: Detection (know when it happens)                    │
│  ├── CloudTrail: log every API call                           │
│  ├── GuardDuty: ML-based threat detection                     │
│  ├── AWS Config: detect config drift                          │
│  └── Security Hub: aggregate all findings                     │
│                                                                │
│  Layer 3: Response (fix it automatically)                     │
│  ├── Config Rule → Lambda → auto-remediate (tag, fix ACL)    │
│  ├── GuardDuty finding → Lambda → block IP in WAF            │
│  └── Root login → CloudTrail → EventBridge → alert           │
└────────────────────────────────────────────────────────────────┘
```

### Auto-remediation Example: Unencrypted S3 Bucket

```
AWS Config detects: S3 bucket without encryption
       │
       ▼
Config rule: NON_COMPLIANT
       │
       ▼
EventBridge triggers Lambda
       │
       ▼
Lambda enables S3 encryption:
  s3.put_bucket_encryption(
    Bucket=bucket_name,
    ServerSideEncryptionConfiguration={...AES256...}
  )
       │
       ▼
SNS notification: "Auto-remediated: encrypted bucket X"
```

> **Gotcha — GuardDuty Cost:** GuardDuty charges per GB of log data analyzed. In large accounts, this can add up. Enable it (it's critical for security) but set up cost alerts. Disable it for unused regions — it still charges if left on.

---

## 10. Cost Optimization Automation on AWS

### Cost Automation Toolkit

| Automation | Savings | How |
|------------|---------|-----|
| Dev EC2 auto-stop at night | 60-70% on dev | EventBridge + Lambda |
| Spot instances for CI/CD | 60-90% on compute | ASG with spot fleet |
| S3 lifecycle tiering | 50-90% on old data | S3 lifecycle rules |
| Delete unused EBS volumes | Stop paying for orphans | Lambda weekly scan |
| Delete unattached Elastic IPs | $0.005/hr per IP | Lambda weekly scan |
| Savings Plans purchase | 30-72% on EC2 | AWS Cost Explorer recommendation |
| Rightsizing recommendations | 10-40% | AWS Compute Optimizer → Lambda apply |

### Weekly Cleanup Lambda Pattern

```python
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Find unattached EBS volumes
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'status', 'Values': ['available']}]
    )
    
    for v in volumes['Volumes']:
        # Only delete if tagged for cleanup or older than 7 days
        vol_id = v['VolumeId']
        print(f"Orphaned volume found: {vol_id}, Size: {v['Size']}GB")
        # Add your deletion logic here (with approval if needed)
        # ec2.delete_volume(VolumeId=vol_id)
```

> **Gotcha:** Never auto-delete volumes without confirming they're truly unused. Always tag your volumes (`terraform apply` adds tags). Only delete volumes with `status=available` AND no recent snapshot activity AND older than N days.

---

*← [Part 1](./part1_intro_and_what_to_automate.md) | Next → [Part 3: Azure Automation](./part3_azure_automation.md)*
