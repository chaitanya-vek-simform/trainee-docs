[🏠 Home](../README.md) · [Edge Cases](.)

# ☁️ Cloud & Data Centers — Edge Cases (110)

> **Audience:** Cloud engineers, DevOps, SREs, and architects working with AWS, GCP, Azure, or multi-cloud environments.  
> **Coverage:** IAM surprises, compute quirks, storage pitfalls, network oddities, IaC traps, cost blowouts, and DR failures.

---

## Table of Contents

1. [IAM & Security (EC-001–015)](#iam--security)
2. [Compute — EC2/VMs/Autoscaling (EC-016–025)](#compute--ec2vmsautoscaling)
3. [Storage — S3/Object Storage (EC-026–035)](#storage--s3object-storage)
4. [Databases — RDS/Cloud SQL (EC-036–045)](#databases--rdscloud-sql)
5. [VPC & Cloud Networking (EC-046–055)](#vpc--cloud-networking)
6. [Serverless & FaaS (EC-056–065)](#serverless--faas)
7. [Containers — ECS/EKS/GKE (EC-066–075)](#containers--ecseksgke)
8. [Infrastructure as Code — Terraform (EC-076–085)](#infrastructure-as-code--terraform)
9. [Monitoring & Observability (EC-086–095)](#monitoring--observability)
10. [Cost & FinOps (EC-096–105)](#cost--finops)
11. [Disaster Recovery & Resilience (EC-106–110)](#disaster-recovery--resilience)
12. [Production FAQ](#production-faq)

---

## IAM & Security

### EC-001 — IAM Role Assumed Credentials Expire Mid-Operation
**Category:** IAM | **Severity:** Critical | **Env:** Prod

**Scenario:** Long-running Lambda, ECS task, or EC2 instance assumes an IAM role. The STS credentials expire (1 hour default) mid-operation. API calls start failing with `ExpiredTokenException`.

```
  Time 0:00  → EC2 starts → assumes role → gets credentials (expire at 1:00)
  Time 0:30  → Operation starts (e.g., S3 copy of 10GB)
  Time 1:00  → Credentials expire mid-copy → AccessDenied on next S3 API call!
```

**Detection:**
```bash
aws sts get-caller-identity   # check expiry field
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>
# Check "Expiration" field in the JSON response
```

**Fix:**
```bash
# Use AWS SDK — it auto-refreshes role credentials
# For long operations, use session tokens with longer duration:
aws sts assume-role --role-arn arn:aws:iam::123:role/MyRole \
  --role-session-name MySession --duration-seconds 43200  # 12 hours max

# For EC2: instance profile credentials auto-refresh via IMDS — use SDK!
```

> ⚠️ **DevOps Gotcha:** AWS SDKs handle credential rotation automatically. Shell scripts using `aws sts assume-role` do NOT auto-refresh. Long shell scripts will hit expiry. Wrap long operations in Python/Go using the SDK with built-in credential rotation.

---

### EC-002 — Overly Permissive `*` Action in IAM Policy Grants Unexpected Access
**Category:** IAM | **Severity:** Critical | **Env:** Prod

**Scenario:** Policy grants `s3:*` intending to allow all S3 operations on one bucket. Includes `s3:CreateBucket`, `s3:DeleteBucket`, `s3:PutBucketPolicy` — allowing full S3 control.

```json
// DANGEROUS — allows creating/deleting ANY bucket, modifying ANY policy:
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}
// SAFE — restrict to specific bucket and operations:
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

> ⚠️ **DevOps Gotcha:** Use AWS IAM Access Analyzer to simulate and validate policies. Apply principle of least privilege — start with deny-all and explicitly allow only what's needed.

---

### EC-003 — Resource-Based Policy Allows Cross-Account Access Unintentionally
**Category:** IAM | **Severity:** Critical | **Env:** Prod

**Scenario:** S3 bucket policy has `"Principal": "*"` intended to allow public read. In a cross-account context, `*` means ANYONE in ANY AWS account can access the bucket.

**Detection:**
```bash
aws s3api get-bucket-policy --bucket mybucket | python3 -m json.tool
# Look for "Principal": "*" or "Principal": {"AWS": "*"}
```

> ⚠️ **DevOps Gotcha:** AWS added "Block Public Access" settings as an extra guardrail. Ensure all of these are enabled at account and bucket level in production. `"Principal": "*"` should only ever be used for intentionally public content.

---

### EC-004 — IAM Permission Boundary Not Propagated to Child Roles
**Category:** IAM | **Severity:** High | **Env:** Prod

**Scenario:** Administrator creates a role with a permission boundary limiting actions. An application using that role creates sub-roles WITHOUT the permission boundary. Sub-roles have full permissions, bypassing the boundary intent.

**Detection:**
```bash
aws iam get-role --role-name MyRole | jq '.Role.PermissionsBoundary'
# Ensure sub-roles also have PermissionsBoundary set
```

> ⚠️ **DevOps Gotcha:** Use SCPs (Service Control Policies) in AWS Organizations to enforce boundaries at the account level — they cannot be bypassed by IAM policies within the account.

---

### EC-005 — Secrets in Environment Variables Visible in Process List
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** Application started with `APP_DB_PASSWORD=secret ./app`. Other users on the same system can see the password via `ps aux` or `/proc/<PID>/environ`.

```
  $ ps aux | grep app
  user 1234 ./app   (APP_DB_PASSWORD=secret visible to all users!)
```

**Fix:**
```bash
# Use secrets manager instead:
DB_PASS=$(aws secretsmanager get-secret-value --secret-id prod/db/password \
  --query SecretString --output text | jq -r .password)

# Or use files with restrictive permissions:
chmod 600 /run/secrets/db-password
```

> ⚠️ **DevOps Gotcha:** In Kubernetes, `env` in pod spec leaks to `kubectl describe pod`. Use `envFrom` with Secret references or mount as files. Even Kubernetes Secrets are base64-encoded (not encrypted) at rest unless you enable KMS envelope encryption.

---

### EC-006 — CloudTrail Log Tampering via Log File Validation Disabled
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** CloudTrail enabled but log file validation is not. Attacker with S3 access deletes or modifies audit logs covering their tracks.

**Fix:**
```bash
aws cloudtrail create-trail --name my-trail \
  --s3-bucket-name my-audit-bucket \
  --enable-log-file-validation
aws cloudtrail update-trail --name my-trail --enable-log-file-validation
```

---

### EC-007 — MFA Delete Bypassed via Presigned URLs
**Category:** Security | **Severity:** High | **Env:** Prod

**Scenario:** S3 bucket has MFA delete enabled on versioned bucket. Attacker uses a presigned URL (generated by a user's credentials pre-MFA) to delete objects.

> ⚠️ **DevOps Gotcha:** Presigned URLs inherit the permissions of the signing principal. They can bypass MFA Delete because they use permanent credentials. Set presigned URL expiry to the minimum necessary.

---

### EC-008 — EC2 Instance Metadata Service (IMDS) v1 Exploitable via SSRF
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** Web application with SSRF vulnerability. Attacker tricks app into making request to `http://169.254.169.254/latest/meta-data/iam/security-credentials/`. Gets IAM credentials of the EC2 instance.

```
  SSRF → http://169.254.169.254/latest/meta-data/iam/security-credentials/role-name
  → Returns: AccessKeyId, SecretAccessKey, Token
  → Attacker can now call AWS APIs as the EC2 instance!
```

**Fix:**
```bash
# Enforce IMDSv2 (requires token in request):
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxx \
  --http-tokens required \
  --http-endpoint enabled

# Or at launch time:
aws ec2 run-instances ... --metadata-options "HttpTokens=required"
```

---

### EC-009 — Service Control Policy Silent Denial — No Error Detail
**Category:** IAM | **Severity:** High | **Env:** Prod

**Scenario:** SCP in AWS Organizations denies an action. The caller receives `AccessDenied` with no indication it was an SCP vs an IAM policy denial. Debugging is difficult.

**Detection:**
```bash
# Check what SCPs apply to the account:
aws organizations list-policies-for-target \
  --target-id <account-id> \
  --filter SERVICE_CONTROL_POLICY

# Use IAM Policy Simulator to test permissions including SCPs
```

---

### EC-010 — Cross-Region STS Endpoint Not Enabled
**Category:** IAM | **Severity:** Medium | **Env:** Prod

**Scenario:** Application in `ap-south-1` calls global STS endpoint (`sts.amazonaws.com`) which routes to `us-east-1`. If `us-east-1` has issues, IAM authentication fails globally.

**Fix:**
```bash
# Use regional STS endpoints:
aws sts get-caller-identity --endpoint-url https://sts.ap-south-1.amazonaws.com
# In SDK config: sts_regional_endpoints=regional
```

---

### EC-011 — IAM Policy Condition Key Case-Sensitive Match Fails
**Category:** IAM | **Severity:** Medium | **Env:** Prod

**Scenario:** IAM policy condition: `"StringEquals": {"aws:RequestedRegion": "us-east-1"}`. User requests from `US-EAST-1` (uppercase). StringEquals is case-sensitive — condition fails, falls through to allow.

**Fix:**
```json
// Use StringEqualsIgnoreCase for case-insensitive comparison:
"Condition": {
  "StringEqualsIgnoreCase": {
    "aws:RequestedRegion": "us-east-1"
  }
}
```

---

### EC-012 — AWS Organizations Root Not Protected by SCP
**Category:** Security | **Severity:** Critical | **Env:** Prod

**Scenario:** SCP applied to member accounts. Root account (management account) is exempt from SCPs. Attacker or operator uses root account to bypass all restrictions.

> ⚠️ **DevOps Gotcha:** AWS management account is exempt from SCPs. Never use management account for workloads. Always use member accounts. Enable AWS Config, GuardDuty, and Security Hub in the management account with delegated admin.

---

### EC-013 — Wildcard Resource in Assume Role Policy Allows Any Principal
**Category:** IAM | **Severity:** Critical | **Env:** Prod

**Scenario:** Role trust policy has `"Principal": "*"`. Any AWS principal — including external accounts — can assume this role if they have no condition restrictions.

**Fix:**
```json
// DANGEROUS:
{ "Principal": "*" }
// SAFE — restrict to specific account and service:
{
  "Principal": {
    "Service": "ec2.amazonaws.com",
    "AWS": "arn:aws:iam::123456789:role/specific-role"
  }
}
```

---

### EC-014 — IAM Policy Simulator Doesn't Test Resource Policies
**Category:** IAM | **Severity:** Medium | **Env:** Prod

**Scenario:** IAM Policy Simulator shows access allowed. Actual request denied because an S3 bucket policy or KMS key policy explicitly denies the principal. Simulator doesn't combine both.

> ⚠️ **DevOps Gotcha:** Use `aws iam simulate-custom-policy` with `--resource-policy` parameter to test resource-based policies alongside identity policies. Or use Access Analyzer for comprehensive policy validation.

---

### EC-015 — GuardDuty Findings Suppressed Without Review
**Category:** Security | **Severity:** High | **Env:** Prod

**Scenario:** GuardDuty generates alert for legitimate CI/CD process (crypto mining alarm false positive). Team suppresses the finding type. Later, a real cryptominer goes undetected.

> ⚠️ **DevOps Gotcha:** Never suppress GuardDuty finding types globally. Create suppression rules scoped to specific resources/accounts that generate known false positives. Review suppressed findings quarterly.

---

## Compute — EC2/VMs/Autoscaling

### EC-016 — Instance Store Data Lost on Stop (Not Terminate)
**Category:** Compute | **Severity:** Critical | **Env:** Prod

**Scenario:** EC2 instance with instance store volumes. Engineer stops the instance (not terminates). Instance store data is LOST — stopping an EC2 moves it to different physical hardware.

```
  Instance Store:
  STOP action → data LOST (moved to new host)
  REBOOT action → data preserved (same host)
  TERMINATE action → data LOST (expected)
```

> ⚠️ **DevOps Gotcha:** Instance store is ephemeral. Use EBS for persistent data. Instance store is appropriate for caches, temp data, and shuffle space that can be regenerated.

---

### EC-017 — Auto Scaling Group Launch Template vs Launch Configuration Drift
**Category:** Compute | **Severity:** High | **Env:** Prod

**Scenario:** ASG uses Launch Configuration (deprecated). Team updates the Launch Template. ASG continues launching instances with old Launch Configuration. New instances have wrong configuration.

**Detection:**
```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names my-asg \
  | jq '.AutoScalingGroups[].LaunchTemplate // .AutoScalingGroups[].LaunchConfigurationName'
```

> ⚠️ **DevOps Gotcha:** AWS deprecated Launch Configurations. Migrate all ASGs to Launch Templates. When updating templates, verify the ASG is referencing the new template version (`$Latest` or specific version).

---

### EC-018 — EC2 Spot Instance Termination Not Handled Gracefully
**Category:** Compute | **Severity:** High | **Env:** Prod

**Scenario:** Application running on Spot instances. AWS reclaims instance with 2-minute notice. Application doesn't handle the interruption notice and loses in-flight work.

**Detection:**
```bash
# Poll instance metadata for interruption notice:
curl -s http://169.254.169.254/latest/meta-data/spot/termination-time
# Returns 404 normally, returns time if interruption pending
```

**Fix:**
```bash
# Run a polling loop in your startup script:
while true; do
  RESULT=$(curl -s -o /dev/null -w "%{http_code}" \
    http://169.254.169.254/latest/meta-data/spot/termination-time)
  if [ "$RESULT" == "200" ]; then
    # Drain, checkpoint, graceful shutdown
    shutdown_gracefully.sh
    break
  fi
  sleep 5
done &
```

---

### EC-019 — EC2 CPU Credit Exhaustion Throttles T-Series Instances
**Category:** Compute | **Severity:** High | **Env:** Prod

**Scenario:** `t3.medium` instance runs smoothly until CPU credits run out. Then CPU is throttled to baseline (20%). Application response time degrades 5–10x with no obvious cause.

**Detection:**
```bash
# CloudWatch metrics:
# CPUSurplusCreditBalance → going to 0 means throttling is imminent
# CPUCreditBalance → credits remaining
# CPUCreditUsage → rate of credit consumption
```

**Fix:**
```bash
# Enable unlimited mode (charges for surplus credits):
aws ec2 modify-instance-credit-specification \
  --instance-credit-specifications InstanceId=i-xxx,CpuCredits=unlimited
# Or move to a non-burstable instance type (m5, c5, r5)
```

> ⚠️ **DevOps Gotcha:** `t3.micro` and `t3.small` instances are perfect for low-traffic services but will surprise you under sustained load. CPU credit exhaustion can cause a production incident that looks like an application bug.

---

### EC-020 — AMI Snapshot Includes Secrets From /etc Environment Files
**Category:** Security/Compute | **Severity:** Critical | **Env:** Prod

**Scenario:** Team bakes an AMI from a running instance. `/etc/app/config.env` contains database passwords. AMI (and all instances launched from it) contains the secrets.

> ⚠️ **DevOps Gotcha:** Always clean up sensitive files before creating an AMI. Use cloud-init or SSM Parameter Store to inject secrets at launch time, not bake-time. Rotate credentials if an AMI with secrets was shared or made public.

---

### EC-021 — Scale-In Terminates Wrong Instance (Not the Oldest/Newest)
**Category:** Compute | **Severity:** Medium | **Env:** Prod

**Scenario:** Custom scale-in policy terminates an instance in the middle of serving long-running requests. Default ASG termination policy terminates the oldest instance, which may be the most loaded.

**Fix:**
```bash
# Set instance protection on critical instances:
aws autoscaling set-instance-protection \
  --instance-ids i-xxx \
  --auto-scaling-group-name my-asg \
  --protected-from-scale-in
# Use lifecycle hooks for scale-in to drain gracefully
```

---

### EC-022 — EC2 Placement Group Node Failure Takes Down All Group Members
**Category:** Compute | **Severity:** High | **Env:** Prod

**Scenario:** Cluster Placement Group used for low-latency HPC. Single rack failure takes down ALL instances in the group (they're on the same physical rack/switch).

```
  Cluster Placement Group = same rack = low latency
                          = single point of failure (rack failure)
  
  Partition Placement Group = spread across partitions (racks)
                            = better fault tolerance, some latency increase
```

---

### EC-023 — EBS Snapshot Restore Performance Lower Than Expected
**Category:** Compute | **Severity:** High | **Env:** Prod

**Scenario:** EBS volume restored from snapshot. Application reads are slow (400ms latency vs normal 1ms). Snapshot restore is lazy — blocks are pulled from S3 on first access.

**Fix:**
```bash
# Force eager initialization (pre-warm the volume):
dd if=/dev/xvdf of=/dev/null bs=1M status=progress
# Or use fio:
fio --filename=/dev/xvdf --rw=read --bs=1M --direct=1 --numjobs=1 \
  --ioengine=libaio --iodepth=32 --name=prefill
```

> ⚠️ **DevOps Gotcha:** After restoring RDS from snapshot or launching EC2 from AMI backed by large snapshot, always pre-warm the volume before production traffic. AWS calls this "volume initialization".

---

### EC-024 — User Data Script Runs Only at First Boot
**Category:** Compute | **Severity:** Medium | **Env:** Prod

**Scenario:** EC2 user data script configures the instance. Instance stopped and restarted. Expecting user data to run again to re-apply config. It doesn't run on restart.

**Fix:**
```bash
# Force user data to run on every boot:
# In user data, add at the top:
# Content-Type: text/cloud-config
#cloud-config
bootcmd:
  - /path/to/startup-script.sh

# Or add to /etc/rc.local for every boot
# Or use cloud-init with "always" run frequency
```

---

### EC-025 — Auto Scaling Cooldown Period Prevents Rapid Scale-Out
**Category:** Compute | **Severity:** High | **Env:** Prod

**Scenario:** Traffic spikes rapidly. ASG adds one instance, waits 5 minutes (cooldown). During cooldown, no more instances are added. Original instances are overwhelmed.

**Fix:**
```bash
# Reduce cooldown period and use target tracking:
# Target tracking auto-adjusts and can scale continuously
# Set cooldown to 60-120 seconds for fast-response services
# Use predictive scaling for known traffic patterns
```

---

## Storage — S3/Object Storage

### EC-026 — S3 Eventual Consistency Causes Read-After-Write Anomaly
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** Object uploaded to S3. Immediately listed with `s3:ListObjects`. Object not in listing. S3 achieved strong consistency in 2020 for new objects but some client-side caches or old SDKs still have eventual consistency behavior.

> ⚠️ **DevOps Gotcha:** AWS S3 now provides strong read-after-write consistency (since December 2020). However, S3 Versioning with concurrent PUT operations on the same key can still have race conditions. Design applications to not depend on S3 as a coordination mechanism.

---

### EC-027 — S3 Object ACL Overrides Bucket Policy
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** Bucket policy denies all public access. Object is uploaded with `ACL: public-read`. The object is publicly accessible, bypassing the bucket-level deny.

```
  Evaluation order:
  1. Block Public Access settings (trump everything)
  2. Bucket ACL
  3. Object ACL  ← can override bucket policy if BPA not set!
  4. Bucket Policy
  5. IAM Identity Policy
```

**Fix:**
```bash
# Enable all Block Public Access settings:
aws s3api put-public-access-block --bucket mybucket \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true
```

---

### EC-028 — S3 Multipart Upload Leaves Parts After Failure
**Category:** Storage | **Severity:** Medium | **Env:** Prod (Cost)

**Scenario:** Large file upload to S3 fails mid-way. Multipart upload parts are left behind. They are not visible via `s3:ListObjects` but DO consume storage and incur charges.

**Detection:**
```bash
aws s3api list-multipart-uploads --bucket mybucket
```

**Fix:**
```bash
# Abort incomplete multipart uploads:
aws s3api abort-multipart-upload --bucket mybucket \
  --key myfile --upload-id <upload-id>

# Set lifecycle rule to auto-abort:
aws s3api put-bucket-lifecycle-configuration --bucket mybucket \
  --lifecycle-configuration '{"Rules":[{"ID":"abort-incomplete-mpu",
  "Status":"Enabled","AbortIncompleteMultipartUpload":{"DaysAfterInitiation":7}}]}'
```

> ⚠️ **DevOps Gotcha:** Always set a lifecycle rule to abort incomplete multipart uploads (7 days is reasonable). Companies have been surprised by significant S3 costs from accumulated multipart upload parts.

---

### EC-029 — S3 Cross-Region Replication Doesn't Copy Existing Objects
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** Enable S3 Cross-Region Replication. Expect all existing objects to be replicated. Only NEW objects written after enabling replication are replicated. Existing objects are ignored.

**Fix:**
```bash
# For existing objects, use S3 Batch Operations:
aws s3control create-job --account-id 123456789 \
  --operation '{"S3ReplicateObject":{}}' \
  --manifest file://manifest.json ...
# Or use sync:
aws s3 sync s3://source-bucket s3://destination-bucket
```

---

### EC-030 — S3 Object Tags Not Copied With `s3 cp`
**Category:** Storage | **Severity:** Medium | **Env:** Prod

**Scenario:** Objects with tags used for lifecycle policies and cost allocation. `aws s3 cp` doesn't copy tags. Destination objects have no tags — lifecycle rules don't apply.

**Fix:**
```bash
# Use --metadata-directive COPY to include metadata:
aws s3 cp s3://src/obj s3://dst/obj --metadata-directive COPY
# Tags still require separate copy:
# Use s3api copy-object with --tag-directive COPY (if within same bucket/region)
aws s3api copy-object --copy-source src/obj --bucket dst --key obj \
  --tag-directive COPY
```

---

### EC-031 — S3 Requester Pays Causes Unexpected Bills for Data Transfer
**Category:** Storage | **Severity:** Medium | **Env:** Prod (Cost)

**Scenario:** S3 bucket has Requester Pays enabled (for public dataset). Internal application downloads from this bucket using its own IAM credentials — the ACCOUNT is billed for data transfer.

---

### EC-032 — S3 Versioning Can't Be Disabled — Only Suspended
**Category:** Storage | **Severity:** Medium | **Env:** Prod

**Scenario:** Enable S3 versioning by mistake. Try to disable it. S3 versioning cannot be disabled — only suspended. All previous versions remain and continue to incur storage costs.

**Fix:**
```bash
# Suspend versioning:
aws s3api put-bucket-versioning --bucket mybucket \
  --versioning-configuration Status=Suspended

# Delete old versions with lifecycle rule:
aws s3api put-bucket-lifecycle-configuration --bucket mybucket \
  --lifecycle-configuration '{"Rules":[{"ID":"delete-old-versions",
  "Status":"Enabled","NoncurrentVersionExpiration":{"NoncurrentDays":30}}]}'
```

---

### EC-033 — S3 Select Performance Worse Than Full Object Download for Small Files
**Category:** Storage | **Severity:** Low | **Env:** Prod

**Scenario:** S3 Select used to query small JSON files (< 256KB). Per-request overhead of S3 Select exceeds savings from filtering. End-to-end latency is higher than just downloading and parsing locally.

> ⚠️ **DevOps Gotcha:** S3 Select has per-query overhead. It's cost-effective and faster only for large files (> 256KB) where you're selecting a small fraction of the data. Use it for large CSVs, Parquet files in data lakes.

---

### EC-034 — S3 Bucket Policy vs IAM Policy Conflicts — Explicit Deny Wins
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** IAM policy allows `s3:DeleteObject`. Bucket policy has explicit `Deny` for `s3:DeleteObject`. IAM policy has no effect — explicit deny always overrides allow.

```
  Evaluation logic:
  1. Explicit DENY anywhere → DENY (final)
  2. Explicit ALLOW in any policy → ALLOW
  3. Implicit DENY (no ALLOW found) → DENY
```

---

### EC-035 — Object Lock Prevents Deletion During Incident Response
**Category:** Storage | **Severity:** High | **Env:** Prod

**Scenario:** S3 Object Lock with COMPLIANCE mode set for compliance reasons. Security incident requires deleting an object with malicious content. Object cannot be deleted — even by root/admin.

> ⚠️ **DevOps Gotcha:** GOVERNANCE mode allows deletion with `s3:BypassGovernanceRetention` permission. COMPLIANCE mode allows ZERO exceptions — not even AWS support can delete the object before retention period. Choose mode carefully.

---

## Databases — RDS/Cloud SQL

### EC-036 — RDS Multi-AZ Failover Takes 60–120 Seconds
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** RDS Multi-AZ configured for HA. Primary fails. Application expects instant failover. DNS propagation for new endpoint takes 60–120 seconds. Applications with short DNS TTL (< 60s) see more than expected downtime.

```
  RDS Multi-AZ Failover:
  1. Primary fails        → 0s
  2. Standby promoted     → ~20s
  3. DNS updated          → +20s
  4. DNS propagation      → +60s ← this is the slow part
  5. App reconnects       → depends on app's reconnect timeout
```

**Fix:**
```bash
# Use RDS Proxy (auto-reconnects without full DNS TTL wait)
# Or set application connection pool to reconnect aggressively:
# JDBC: autoReconnect=true, connectTimeout=5000, socketTimeout=60000
```

---

### EC-037 — Aurora Read Replica Lag Causes Stale Reads
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** Application writes to Aurora primary, then immediately reads from read replica. Read replica is 100ms–5s behind. Read returns stale data. Critical for payment confirmations, inventory checks.

**Detection:**
```bash
# Monitor AuroraReplicaLag CloudWatch metric
# Or query:
SHOW REPLICA STATUS\G   # Seconds_Behind_Source field
```

**Fix:**
```bash
# Route time-sensitive reads to primary:
# In application: separate read/write connection strings
# For critical reads: always use writer endpoint
# Set read endpoint only for analytics/reporting queries
```

---

### EC-038 — RDS Parameter Group Changes Require Reboot
**Category:** Database | **Severity:** Medium | **Env:** Prod

**Scenario:** Change RDS parameter group for performance tuning. Expect immediate effect. Static parameters (like `max_connections`) require reboot. Dynamic parameters take effect immediately.

**Detection:**
```bash
# Check parameter type:
aws rds describe-db-parameters --db-parameter-group-name mygroup \
  | jq '.Parameters[] | select(.ParameterName=="max_connections") | .ApplyType'
# "static" = reboot required, "dynamic" = immediate
```

---

### EC-039 — Automated Backup Window Overlaps With Peak Traffic
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** RDS automated backup window defaults to 3:00–4:00 AM UTC. For Asia-Pacific customers, this is peak business hours. I/O performance degrades during backup.

**Fix:**
```bash
aws rds modify-db-instance --db-instance-identifier mydb \
  --preferred-backup-window "19:00-20:00"  # 12AM IST = off-peak
--apply-immediately
```

---

### EC-040 — RDS Storage Autoscaling Triggers at 10% Free Space — Too Late
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** Storage autoscaling kicks in when only 10% free AND 5GB below threshold. For a 100GB DB, autoscaling triggers at 10GB free. Write operations can fail in the time between threshold hit and storage expansion completion.

> ⚠️ **DevOps Gotcha:** Monitor `FreeStorageSpace` CloudWatch metric and alert at 25% free (not 10%). Storage expansion takes several minutes during which write performance may degrade.

---

### EC-041 — Database Connection Pool Exhaustion Under Load
**Category:** Database | **Severity:** Critical | **Env:** Prod

**Scenario:** Application scales horizontally (more instances). Each instance has its own connection pool of 100 connections. With 20 instances × 100 = 2000 connections to RDS MySQL. RDS `max_connections` is 500. New connections fail.

```
  RDS max_connections ≈ DBInstanceClassMemory / 12582880
  db.t3.medium (4GB RAM): 4GB / 12MB ≈ 320 connections
  
  Scale to 10 app servers × 50 connections/server = 500 → EXCEEDED!
```

**Fix:**
```bash
# Use RDS Proxy to pool and multiplex connections:
# RDS Proxy maintains persistent connection pool to DB
# App instances connect to Proxy (thousands of connections allowed)
# Proxy multiplexes to limited DB connection pool

# Or use PgBouncer/ProxySQL for self-managed databases
```

---

### EC-042 — Long-Running Transactions Block Schema Migrations
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** Running `ALTER TABLE` on a busy table (MySQL). Operation requires table lock. Waiting for existing long-running queries to complete. Table is effectively locked for all new reads/writes during wait.

**Fix:**
```bash
# Use pt-online-schema-change (Percona) for zero-downtime migrations:
pt-online-schema-change --alter "ADD INDEX idx_user_id (user_id)" \
  D=mydb,t=mytable --execute

# Or gh-ost (GitHub's online schema migration tool)
# For PostgreSQL: pg_repack for table rewrites without locking
```

---

### EC-043 — Aurora Serverless v1 Cold Start Pauses 25+ Seconds
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** Aurora Serverless v1 configured to pause after 5 minutes of inactivity. First API request after pause waits 25–60 seconds for DB to resume. User sees timeout.

**Fix:**
```bash
# Aurora Serverless v2 doesn't pause (scales to 0 ACUs takes seconds)
# For v1: disable auto-pause or set minimum capacity to 1 ACU
# Or keep alive with scheduled Lambda pinging the DB
```

---

### EC-044 — Read Replica Promoted to Primary — Old Primary Can't Rejoin
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** Manual failover — promote read replica to primary. Old primary comes back online. Old primary cannot be demoted back to replica automatically. Requires manual data resync.

---

### EC-045 — RDS Snapshot Restore Creates New Endpoint — DNS Update Required
**Category:** Database | **Severity:** High | **Env:** Prod

**Scenario:** Restore RDS from snapshot to a new instance. New instance has a different endpoint. All application configs pointing to old endpoint must be updated. Often missed, causing persistent "DB not found" errors.

> ⚠️ **DevOps Gotcha:** Use CNAME aliases for database endpoints in Route53. `db.prod.internal` → `rds-endpoint.amazonaws.com`. During restore, update the CNAME, not every application config file.

---

## VPC & Cloud Networking

### EC-046 — Security Group Stateful — Can Block Return Traffic If Rules Misconfigured
**Category:** VPC | **Severity:** High | **Env:** Prod

**Scenario:** Inbound rule allows TCP 443 from client. No outbound rule configured. AWS Security Groups ARE stateful — return traffic is automatically allowed. But NACL (Network ACL) is stateless and WILL block return traffic without explicit egress rules.

```
  Security Group (STATEFUL):
  Allow inbound 443 → return traffic automatically allowed ✅
  
  NACL (STATELESS):
  Allow inbound 443 → must ALSO allow outbound ephemeral ports (1024-65535)
  Otherwise return traffic is blocked!
```

---

### EC-047 — VPC Flow Logs Don't Capture Rejected Traffic Reason
**Category:** VPC | **Severity:** Medium | **Env:** Prod

**Scenario:** VPC Flow Logs show `REJECT` for certain traffic. Can't tell if rejection was from Security Group or Network ACL.

**Detection:**
```bash
# Enable both VPC Flow Logs AND Security Group flow logs
# Security Group flow logs (newer feature) show which SG rule matched
# Or use AWS Network Firewall with detailed logging
```

---

### EC-048 — Private Subnet Without NAT Gateway Can't Reach AWS Services
**Category:** VPC | **Severity:** High | **Env:** Prod

**Scenario:** EC2 in private subnet needs to reach S3 for startup scripts. No NAT Gateway configured. Instance has no internet access. User data script fails — instance never fully initializes.

**Fix:**
```bash
# Options (in order of cost/complexity):
# 1. NAT Gateway (recommended, $0.045/hour + data processing)
# 2. VPC Endpoint for S3 (free, works only for AWS services)
# 3. NAT Instance (cheaper but you manage it)
```

---

### EC-049 — Elastic IP Not Released — Incurring Charges When Not Attached
**Category:** VPC/Cost | **Severity:** Medium | **Env:** Prod

**Scenario:** EC2 terminated but Elastic IP not released. AWS charges for unattached EIPs ($0.005/hour ≈ $3.60/month per IP). Over months and many terminated instances, this adds up.

**Detection:**
```bash
aws ec2 describe-addresses | jq '.Addresses[] | select(.InstanceId == null)'
```

> ⚠️ **DevOps Gotcha:** Add EIP cleanup to your instance termination runbook or Lambda. AWS Trusted Advisor and Cost Anomaly Detection can flag idle EIPs.

---

### EC-050 — Route Table Missing Blackhole Route After VPN Peer Disconnects
**Category:** VPC | **Severity:** High | **Env:** Prod

**Scenario:** Site-to-Site VPN route added to route table. VPN tunnel goes down. Route remains in table pointing to a "dead" VPN target. Traffic destined for on-prem is silently dropped (blackholed).

**Detection:**
```bash
aws ec2 describe-route-tables | jq '.RouteTables[].Routes[] | select(.State == "blackhole")'
```

---

### EC-051 — VPC Peering Route Not Bidirectional
**Category:** VPC | **Severity:** High | **Env:** Prod

**Scenario:** VPC-A peered with VPC-B. Route added in VPC-A pointing to VPC-B. Forgot to add route in VPC-B pointing to VPC-A. Connection is one-way — VPC-A can initiate but responses are dropped.

---

### EC-052 — AWS PrivateLink Endpoint Policy Blocks Specific Actions
**Category:** VPC | **Severity:** Medium | **Env:** Prod

**Scenario:** VPC Endpoint (PrivateLink) for S3 has a restrictive endpoint policy. Application can reach S3 but certain operations (e.g., `s3:DeleteObject`) are denied. Confusing because the IAM policy allows it.

> ⚠️ **DevOps Gotcha:** VPC Endpoint policies are evaluated in addition to IAM policies. An explicit Deny in endpoint policy blocks the action even with IAM Allow. Start with the AWS-managed default endpoint policy and tighten from there.

---

### EC-053 — Inter-AZ Data Transfer Costs Add Up Silently
**Category:** VPC/Cost | **Severity:** Medium | **Env:** Prod

**Scenario:** Application in AZ-a talks to ElastiCache in AZ-b. $0.01/GB inter-AZ transfer. At 1TB/day inter-AZ traffic: $10/day = $300/month surprise on the bill.

```
  Same AZ traffic:   FREE
  Cross-AZ traffic:  $0.01/GB each direction = $0.02/GB round-trip
  Cross-region:      $0.02–$0.09/GB
```

> ⚠️ **DevOps Gotcha:** Always place ElastiCache, RDS read replicas, and other frequently-accessed resources in the same AZ as primary compute. Use AZ-aware routing in your application.

---

### EC-054 — Security Group Rule Limit (60 rules) Reached
**Category:** VPC | **Severity:** High | **Env:** Prod

**Scenario:** Complex application with many security groups. AWS default limit: 60 inbound + 60 outbound rules per security group. At scale, teams hit this limit and can't add new rules.

**Fix:**
```bash
# Request limit increase via AWS Support
# Or restructure: use prefix lists instead of individual CIDRs
# Or use AWS Network Firewall for complex rule sets
```

---

### EC-055 — DNS Hostnames Not Enabled in Custom VPC
**Category:** VPC | **Severity:** Medium | **Env:** Prod

**Scenario:** Create custom VPC. EC2 instances don't get DNS hostnames. `curl http://ec2-ip-address.compute-1.amazonaws.com` fails. RDS endpoint DNS resolution fails.

**Fix:**
```bash
aws ec2 modify-vpc-attribute --vpc-id vpc-xxx --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id vpc-xxx --enable-dns-support
# Both must be enabled!
```

---

## Serverless & FaaS

### EC-056 — Lambda Cold Start Causes SLA Violation Under Infrequent Traffic
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** Lambda function used for API. Between 2–4 AM (low traffic), functions expire from warm pool. First request after hours of inactivity has 2–8 second cold start. P99 latency SLA of 1 second violated.

```
  Lambda cold start breakdown:
  ┌────────────────────────────────────────────────────────┐
  │ Infrastructure init (50–500ms) → Runtime init (50ms)  │
  │ → Handler init code (your code, could be 1000ms+)     │
  └────────────────────────────────────────────────────────┘
  Total: 100ms (Go) to 3000ms+ (Java/large Node.js packages)
```

**Fix:**
```bash
# Provisioned Concurrency — keep N instances warm:
aws lambda put-provisioned-concurrency-config \
  --function-name myfunction \
  --qualifier $LATEST \
  --provisioned-concurrent-executions 5

# Or use Scheduled CloudWatch Events to ping function every 5 minutes
```

> ⚠️ **DevOps Gotcha:** Java Lambdas with Spring Boot have notorious cold starts (3–5s). Use Quarkus native compilation or switch to Go/Python for latency-sensitive functions.

---

### EC-057 — Lambda Execution Role Has No Logs Permission — Errors Silent
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** Lambda execution role missing `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`. Lambda function runs but logs are silently dropped. Debugging is impossible.

**Fix:**
```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "arn:aws:logs:*:*:*"
}
```

---

### EC-058 — Lambda Timeout Set Too Low — Causes Partial Processing
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** Lambda processing SQS messages. Timeout set to 30 seconds. Some messages take 45 seconds. Lambda times out, message returns to queue, processed again → duplicate processing.

> ⚠️ **DevOps Gotcha:** Set Lambda timeout to at least 6× your P99 processing time. For SQS: set `visibilityTimeout = Lambda timeout × 6`. This prevents the message from becoming visible again while Lambda is still processing.

---

### EC-059 — Lambda Concurrency Throttle Causes SQS Message Pile-Up
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** Lambda reserved concurrency set to 10. SQS queue receives 1000 messages/second. 10 Lambdas can't keep up. Queue depth grows. Messages expire (DLQ) or are processed hours later.

**Detection:**
```bash
# CloudWatch metrics: Throttles, ConcurrentExecutions
# SQS: ApproximateNumberOfMessagesVisible
```

> ⚠️ **DevOps Gotcha:** Reserved concurrency LIMITS Lambda — it doesn't guarantee it. To GUARANTEE minimum concurrency, use Provisioned Concurrency. Set SQS alarm on queue depth to detect processing lag.

---

### EC-060 — Lambda VPC Config Increases Cold Start and Limits Scale
**Category:** Serverless | **Severity:** Medium | **Env:** Prod

**Scenario:** Lambda placed inside VPC to access RDS. VPC-attached Lambdas have longer cold starts (ENI creation) and are limited by available IP addresses in subnet.

> ⚠️ **DevOps Gotcha:** Lambda in VPC now uses Hyperplane ENIs (shared across invocations) — cold start penalty is eliminated in 2019+. BUT Lambda still consumes IPs from your subnet. Use /24 subnets dedicated to Lambda.

---

### EC-061 — Step Functions Express Workflow vs Standard — Different Semantics
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** Standard Workflow used for high-volume (10,000+ executions/second) workload. Hits state transitions limit. Switching to Express Workflow for cost — but Express doesn't guarantee exactly-once execution.

```
  Standard: exactly-once execution, long duration (up to 1 year), audit trail
  Express: at-least-once execution, short duration (< 5 min), no audit trail
  Choose wrong type → duplicate processing or hit limits!
```

---

### EC-062 — API Gateway 29-Second Timeout — Can't Be Changed
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** API Gateway has a hard 29-second integration timeout. Long-running Lambda (60+ seconds) always times out at the gateway. API returns 504 even though Lambda completes.

**Fix:**
```bash
# For long operations, use async patterns:
# 1. Return 202 Accepted immediately
# 2. Process asynchronously (SQS + Lambda)
# 3. Client polls for result or uses WebSocket API
```

---

### EC-063 — Lambda Dead Letter Queue Not Configured — Failed Events Lost
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** Lambda triggered by S3 event. Lambda fails with exception. Without DLQ or SQS as trigger, the event is retried twice then silently dropped.

**Fix:**
```bash
# Configure DLQ on the Lambda function:
aws lambda update-function-configuration \
  --function-name myfunction \
  --dead-letter-config TargetArn=arn:aws:sqs:region:account:my-dlq
```

---

### EC-064 — Lambda Layers Not Available in All Regions
**Category:** Serverless | **Severity:** Medium | **Env:** Prod

**Scenario:** Lambda Layer (public ARN) referenced from Lambda in `ap-south-2`. Layer doesn't exist in that region. Function fails to deploy or execute.

> ⚠️ **DevOps Gotcha:** Lambda Layers are region-specific. Public layers published by AWS (e.g., for Insights, Extensions) are not always available in newer/less common regions. Always test in your target region.

---

### EC-065 — EventBridge Rule Throttled — Events Lost Without DLQ
**Category:** Serverless | **Severity:** High | **Env:** Prod

**Scenario:** EventBridge rule targets Lambda. Lambda throttled (429). EventBridge retries for up to 24 hours (configurable). Without DLQ on the EventBridge target, events are dropped after retry exhaustion.

---

## Containers — ECS/EKS/GKE

### EC-066 — ECS Task Stops Because of Memory Soft Limit — No Visible Error
**Category:** Containers | **Severity:** High | **Env:** Prod

**Scenario:** ECS task definition sets `memoryReservation` (soft limit) but not `memory` (hard limit). If the container uses more than `memoryReservation`, ECS may stop the task to reclaim resources for other tasks. Error is opaque.

```
  memoryReservation: soft limit — used for placement, can exceed
  memory: hard limit — exceeded → container killed by OOM
  
  Best practice: set both:
  memoryReservation: 256  (for placement)
  memory: 512             (hard cap to prevent OOM cascade)
```

---

### EC-067 — Kubernetes Resource Requests Cause Node Overcommit
**Category:** Containers | **Severity:** High | **Env:** Prod (K8s)

**Scenario:** Pods deployed with no resource requests. Kubernetes schedules many pods on each node. Actual memory usage exceeds node capacity. OOM killer starts evicting pods.

```
  No requests = Kubernetes sees node as "infinite capacity"
  → over-schedules pods
  → physical memory exhausted
  → OOM killer starts killing processes
  → pod evictions cascade
```

> ⚠️ **DevOps Gotcha:** Always set resource requests AND limits. Use LimitRange objects in namespaces to enforce minimum requests for all pods. Monitor node memory with `kubectl top nodes`.

---

### EC-068 — Kubernetes Liveness Probe Kills Healthy App Under Load
**Category:** Containers | **Severity:** High | **Env:** Prod

**Scenario:** Liveness probe HTTP check with 1-second timeout. App under heavy load takes 2 seconds to respond. Kubernetes kills the container (liveness probe failed). Container restarts. More load → more restarts → cascading failure.

**Fix:**
```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5     # give it 5 seconds
  failureThreshold: 3   # 3 consecutive failures before restart
```

---

### EC-069 — Docker Image Uses `latest` Tag — Silent Regressions
**Category:** Containers | **Severity:** High | **Env:** Prod

**Scenario:** `FROM node:latest` in Dockerfile. Node.js 18 `latest`. Three months later, `latest` is Node 20. Application breaks with deprecation errors. No change to Dockerfile, but behavior changed.

**Fix:**
```dockerfile
# Pin to specific versions:
FROM node:18.17.1-alpine3.18
# Or use digest:
FROM node@sha256:abc123...   # immutable
```

---

### EC-070 — EKS Node Group Not Draining Before Upgrade Causes Pod Disruption
**Category:** Containers | **Severity:** High | **Env:** Prod

**Scenario:** EKS managed node group upgraded. Old nodes terminated before pods gracefully migrated. Pods are hard-killed. Connections dropped.

**Fix:**
```bash
# Use PodDisruptionBudget to protect critical apps:
kubectl apply -f - <<EOF
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: myapp
EOF
```

---

### EC-071 — Container Timezone Different From Host
**Category:** Containers | **Severity:** Medium | **Env:** Prod

**Scenario:** Container logs show UTC timestamps. Application on host uses IST. Log correlation fails. Scheduled tasks inside container fire at wrong times.

**Fix:**
```dockerfile
# Mount host timezone:
# In docker-compose.yml:
volumes:
  - /etc/localtime:/etc/localtime:ro
  - /etc/timezone:/etc/timezone:ro
# Or set in Dockerfile:
ENV TZ=Asia/Kolkata
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
```

---

### EC-072 — Kubernetes Horizontal Pod Autoscaler Can't Scale Down to Zero
**Category:** Containers | **Severity:** Medium | **Env:** Prod

**Scenario:** HPA configured with `minReplicas: 1`. During nights/weekends, traffic drops to zero but 1 pod always runs. For batch/event-driven workloads this wastes resources.

**Fix:**
```bash
# Use KEDA (Kubernetes Event Driven Autoscaling) for scale-to-zero:
# KEDA watches SQS queue depth, Kafka consumer lag, etc.
# Scales to 0 when no events, scales from 0 on first event
```

---

### EC-073 — Init Container Failure Keeps Pod in Init State — Looks Like Hang
**Category:** Containers | **Severity:** Medium | **Env:** Prod

**Scenario:** Kubernetes pod has init container that fails silently (exit 0 but wrong state). Main container never starts. Pod shows `Init:0/1` indefinitely. Looks like a hang.

**Detection:**
```bash
kubectl describe pod mypod | grep -A10 "Init Containers"
kubectl logs mypod -c init-container-name
```

---

### EC-074 — ECS Service Discovery Returns Stale IPs After Task Replacement
**Category:** Containers | **Severity:** High | **Env:** Prod

**Scenario:** ECS service uses Cloud Map service discovery. Task replaced (new deployment). Cloud Map DNS still returns old task IP for up to 30 seconds (TTL). Traffic to old IP fails.

> ⚠️ **DevOps Gotcha:** Use very low TTL (5–10s) for Cloud Map DNS during active deployments. Or use ECS service connect which uses Envoy proxy and doesn't rely on DNS TTL for routing.

---

### EC-075 — Kubernetes Secret Rotation Doesn't Restart Pods
**Category:** Containers | **Severity:** High | **Env:** Prod

**Scenario:** Kubernetes Secret updated with new database password. Pods using `envFrom: secretRef` don't automatically pick up the new value — pods must be restarted.

**Fix:**
```bash
# Trigger rolling restart:
kubectl rollout restart deployment myapp
# Or use Reloader operator which automatically restarts
# pods when referenced ConfigMaps/Secrets change
```

---

## Infrastructure as Code — Terraform

### EC-076 — `terraform apply` Destroys and Recreates Resources With `forces new resource`
**Category:** IaC | **Severity:** Critical | **Env:** Prod

**Scenario:** Changing an immutable field (like EC2 `ami`, RDS `engine`, subnet) forces Terraform to destroy and recreate the resource. Data is lost. Traffic to ELB fails during recreation.

```
  terraform plan output to watch for:
  # aws_db_instance.main must be replaced
  -/+ resource "aws_db_instance" "main" {
      ~ engine_version = "13.x" -> "14.x"  # forces replacement!
```

**Fix:**
```bash
# Use lifecycle meta-argument to prevent accidental deletion:
lifecycle {
  prevent_destroy = true
}
# Or create new resource first, then migrate, then delete old:
# blue/green with Terraform workspaces
```

> ⚠️ **DevOps Gotcha:** ALWAYS run `terraform plan` and review output before `apply`. Use `terraform plan -out=plan.tfplan` + `terraform apply plan.tfplan` to ensure what you reviewed is what gets applied. Never auto-apply without human review in production.

---

### EC-077 — Terraform State Drift — Manual Changes Overwritten
**Category:** IaC | **Severity:** Critical | **Env:** Prod

**Scenario:** Manual AWS Console change made during incident (e.g., increase instance type). Next `terraform apply` reverts the change because Terraform state doesn't know about it.

**Detection:**
```bash
terraform plan   # will show "change" back to original
terraform refresh  # updates state with current reality
```

**Fix:**
```bash
# Import manually-created resources:
terraform import aws_instance.web i-1234567890

# Or use terraform refresh before any apply:
terraform refresh && terraform plan
```

---

### EC-078 — Terraform Remote State Lock Not Released After Crash
**Category:** IaC | **Severity:** High | **Env:** Prod

**Scenario:** `terraform apply` running. Operator's laptop crashes (or CTRL+C at wrong moment). DynamoDB lock not released. Next `terraform apply` fails with "Error acquiring the state lock".

**Fix:**
```bash
# List and remove stale locks:
terraform force-unlock <LOCK_ID>
# Lock ID is shown in the error message
# Or manually delete the DynamoDB item:
aws dynamodb delete-item --table-name tf-lock-table \
  --key '{"LockID": {"S": "my-bucket/my.tfstate-md5"}}'
```

---

### EC-079 — Terraform `count` vs `for_each` — Index-Based Deletion Issue
**Category:** IaC | **Severity:** High | **Env:** Prod

**Scenario:** Using `count` to create 3 subnets. Remove the middle one. Terraform deletes and recreates subnets 2 and 3 (index shifts). Use `for_each` instead to avoid index-based reshuffling.

```hcl
# FRAGILE — using count:
resource "aws_subnet" "subnets" {
  count = 3   # remove index 1 → indices 1,2 shift → destroy+create!
}

# STABLE — using for_each:
resource "aws_subnet" "subnets" {
  for_each = toset(["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"])
  # Remove one CIDR → only that subnet is destroyed, others untouched
}
```

---

### EC-080 — Sensitive Values in Terraform State Stored in Plaintext
**Category:** IaC/Security | **Severity:** Critical | **Env:** Prod

**Scenario:** Terraform generates RDS password and stores it in state file. State file in S3 has incorrect permissions — readable by all IAM users. Database password exposed.

**Fix:**
```bash
# Encrypt state with SSE-KMS:
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:..."
    dynamodb_table = "terraform-lock"
  }
}
# Restrict bucket access with bucket policy
# Enable S3 Block Public Access
```

---

### EC-081 — Terraform Module Upgrade Breaks Existing Resources
**Category:** IaC | **Severity:** High | **Env:** Prod

**Scenario:** Update Terraform AWS provider from 3.x to 5.x. Provider upgrade changes resource attribute names, default behaviors, or deprecates resources. `terraform plan` shows unexpected destroy+create operations.

**Fix:**
```bash
# Pin provider versions:
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.67"  # allow patch updates only
    }
  }
}
# Test upgrades in non-prod first with terraform plan output review
```

---

### EC-082 — `terraform destroy` Can't Find Resources Due to Missing State
**Category:** IaC | **Severity:** High | **Env:** Prod

**Scenario:** Accidentally delete Terraform state file. Now `terraform destroy` doesn't know what was created. Resources remain running, costing money, and possibly a security risk.

**Fix:**
```bash
# Reconstruct state by importing:
terraform import resource_type.resource_name resource_id
# Or use terraformer to scan and import existing infrastructure:
terraformer import aws --resources=ec2,s3,rds --regions=us-east-1
```

---

### EC-083 — Terraform Workspace `default` Used in Production
**Category:** IaC | **Severity:** High | **Env:** Prod

**Scenario:** Team uses Terraform workspaces for environments. New team member runs `terraform apply` without switching from `default` workspace. Resources created in wrong environment.

**Fix:**
```bash
# Enforce workspace check in code:
locals {
  allowed_workspaces = ["dev", "staging", "prod"]
}
resource "null_resource" "workspace_check" {
  triggers = {
    check = contains(local.allowed_workspaces, terraform.workspace) ? "" : \
    file("ERROR: Must use a named workspace, not '${terraform.workspace}'")
  }
}
```

---

### EC-084 — Circular Dependency Between Terraform Modules
**Category:** IaC | **Severity:** Medium | **Env:** Prod

**Scenario:** Module A outputs a value that Module B needs. Module B outputs a value that Module A needs. Terraform can't resolve the dependency graph — errors with "cycle detected".

> ⚠️ **DevOps Gotcha:** Break circular dependencies by extracting the shared resource into a third "base" module, then having A and B depend on base. Or use data sources to look up the resource instead of using output references.

---

### EC-085 — `taint` Replaced With `-replace` Flag — Behavior Difference
**Category:** IaC | **Severity:** Low | **Env:** Prod

**Scenario:** Old tutorials use `terraform taint resource`. In Terraform 0.15.2+, `taint` is deprecated. New flag is `terraform apply -replace=resource`. Behavior is same but documentation misleads.

---

## Monitoring & Observability

### EC-086 — CloudWatch Alarm State Is INSUFFICIENT_DATA — Alerts Never Fire
**Category:** Monitoring | **Severity:** High | **Env:** Prod

**Scenario:** CloudWatch alarm configured on Lambda error rate. Lambda never invoked → no datapoints → alarm stays in `INSUFFICIENT_DATA` state. Alarm action (SNS notification) never fires.

**Detection:**
```bash
aws cloudwatch describe-alarms --alarm-names my-alarm \
  | jq '.MetricAlarms[].StateValue'
# INSUFFICIENT_DATA = no data to evaluate
```

**Fix:**
```bash
# Set alarm to treat missing data as breaching:
aws cloudwatch put-metric-alarm ... --treat-missing-data breaching
# Or set to: missing, notBreaching, ignore, breaching
```

---

### EC-087 — CloudWatch Log Group Retention Not Set — Logs Accumulate Forever
**Category:** Monitoring | **Severity:** Medium | **Env:** Prod (Cost)

**Scenario:** Lambda creates CloudWatch Log Group with retention `Never Expire`. After 6 months, log storage cost exceeds Lambda compute cost.

**Fix:**
```bash
# Set retention policy on existing log groups:
aws logs put-retention-policy \
  --log-group-name /aws/lambda/myfunction \
  --retention-in-days 30

# For all Lambda log groups:
aws logs describe-log-groups --query 'logGroups[?retentionInDays==`null`].logGroupName' \
  --output text | while read lg; do
    aws logs put-retention-policy --log-group-name "$lg" --retention-in-days 30
done
```

---

### EC-088 — High-Cardinality Dimensions Break CloudWatch Cost
**Category:** Monitoring | **Severity:** High | **Env:** Prod (Cost)

**Scenario:** Custom metric uses user ID as a dimension. With 1 million users, 1 million metric dimensions created. CloudWatch charges $0.30/metric/month → $300,000/month just for metrics.

> ⚠️ **DevOps Gotcha:** Never use high-cardinality values (user IDs, request IDs, session tokens) as CloudWatch metric dimensions. Use aggregated dimensions (region, env, service). Use Logs Insights for per-request analysis.

---

### EC-089 — Distributed Tracing Headers Stripped by Load Balancer
**Category:** Observability | **Severity:** Medium | **Env:** Prod

**Scenario:** Application emits X-Ray traces. ALB strips `X-Amzn-Trace-Id` header. Downstream services start new traces. No end-to-end trace visibility across service boundaries.

**Detection:**
```bash
# Check if ALB preserves trace header:
curl -I https://api.example.com   # look for X-Amzn-Trace-Id in response
```

---

### EC-090 — Prometheus Retention Period Too Short — Missed Alerting Window
**Category:** Monitoring | **Severity:** High | **Env:** Prod

**Scenario:** Prometheus with 15-day retention. Alert rule evaluates over 30-day window (`rate[30d]`). Prometheus can't compute the range — alert never fires.

**Fix:**
```bash
# Match retention period to your longest alert window:
# prometheus.yml:
storage:
  tsdb:
    retention.time: 45d   # > 30 day alert windows
    retention.size: 50GB
# Or use Thanos/Cortex for long-term storage
```

---

### EC-091 — Grafana Dashboard Shows "No Data" for Recent Metrics
**Category:** Monitoring | **Severity:** Medium | **Env:** Prod

**Scenario:** Grafana dashboard shows "No Data" for the last 5 minutes even though metrics are being generated. Prometheus scrape interval is 60 seconds, and time range ends at "now".

**Fix:**
```bash
# In Grafana query: set Min step to match scrape interval
# Or add $__rate_interval variable instead of hardcoded intervals
# Add "Min time interval" = scrape interval in datasource config
```

---

### EC-092 — Alert Fatigue From Noisy Alert Rules
**Category:** Monitoring | **Severity:** Medium | **Env:** Prod

**Scenario:** Prometheus alert fires for every single pod restart, even single transient crash. Ops team gets 200 alerts/day. Real alerts get ignored.

**Fix:**
```yaml
# Add "for" duration and smarter thresholds:
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
  for: 5m    # must be sustained for 5 minutes
  labels:
    severity: critical
```

---

### EC-093 — CloudTrail Not Enabled in All Regions
**Category:** Monitoring/Security | **Severity:** Critical | **Env:** Prod

**Scenario:** CloudTrail enabled in `us-east-1`. Attacker creates resources in `ap-southeast-1`. No audit trail for those API calls.

**Fix:**
```bash
# Create multi-region trail:
aws cloudtrail create-trail --name global-trail \
  --s3-bucket-name my-audit-bucket \
  --is-multi-region-trail \
  --include-global-service-events
```

---

### EC-094 — Log Aggregation Bottleneck — Logs Dropped Under High Load
**Category:** Monitoring | **Severity:** High | **Env:** Prod

**Scenario:** Fluentd/Logstash shipping logs to Elasticsearch. Under high load, buffer fills. Logs are dropped (or the log shipper crashes). Critical error logs during incidents are lost.

**Fix:**
```bash
# Use persistent queues (Kafka as buffer):
# App → FluentBit → Kafka → Logstash → Elasticsearch
# Kafka provides durability + backpressure handling
# Or configure fluentd with file-based buffers:
<buffer>
  @type file
  path /var/log/fluentd-buffer
  overflow_action block   # block instead of drop
</buffer>
```

---

### EC-095 — Synthetic Monitoring Misses Real User Performance
**Category:** Monitoring | **Severity:** Medium | **Env:** Prod

**Scenario:** Synthetic monitor (CloudWatch Canary) runs every 5 minutes from a single region with a pristine network. Real users in India experience 8-second load times. Monitor shows 200ms.

> ⚠️ **DevOps Gotcha:** Supplement synthetic monitoring with Real User Monitoring (RUM). Use AWS CloudFront real-time logs or frontend timing APIs (Navigation Timing, Resource Timing) to capture actual user experience metrics.

---

## Cost & FinOps

### EC-096 — Data Transfer Out Cost Exceeds Compute Cost
**Category:** FinOps | **Severity:** High | **Env:** Prod

**Scenario:** Application serves large media files from EC2. At 100TB/month outbound transfer: 100TB × $0.09/GB = $9,000/month just for data transfer. More than the EC2 compute cost.

```
  Cost breakdown trap:
  EC2 m5.xlarge (4 vCPU, 16GB) = $140/month
  100TB data transfer out from EC2 = $9,000/month
  
  Fix: CloudFront CDN
  CloudFront data transfer = $0.012–$0.02/GB
  100TB via CloudFront = $1,200–$2,000/month (87% savings!)
```

---

### EC-097 — Unused Reserved Instance Wasted After Service Redesign
**Category:** FinOps | **Severity:** High | **Env:** Prod

**Scenario:** 3-year Reserved Instance purchased for `r5.4xlarge`. 8 months later, service migrated to Fargate. Reserved Instance continues incurring charges until expiry.

**Fix:**
```bash
# Sell unused Reserved Instances on AWS Reserved Instance Marketplace
# Or: convert to Savings Plans (more flexible, covers EC2/Lambda/Fargate)
# Always prefer Compute Savings Plans over Reserved Instances for flexibility
```

---

### EC-098 — S3 Glacier Restore Cost More Expensive Than Original Data
**Category:** FinOps | **Severity:** Medium | **Env:** Prod

**Scenario:** 10TB of data moved to S3 Glacier Deep Archive ($1/TB/month). Need to restore for audit. Expedited restore of 10TB: $250/TB × 10TB = $2,500 one-time fee! More than a year of storage.

> ⚠️ **DevOps Gotcha:** Glacier is cheap to STORE but expensive to RETRIEVE. Always calculate retrieval costs before archiving. Standard retrieval (3–5 hours) costs $0.02/GB. Expedited ($0.03/GB + $10/request). Budget retrieval (5–12 hours) costs $0.0025/GB.

---

### EC-099 — NAT Gateway Cost Surprise for Dev Environments
**Category:** FinOps | **Severity:** Medium | **Env:** Dev

**Scenario:** Development environment uses NAT Gateway for internet access from private subnets. NAT Gateway: $0.045/hour = $32/month per NAT + $0.045/GB data processing. Dev environment generating 1TB/month: $45 data + $32 hourly = $77/month per dev environment.

**Fix:**
```bash
# For dev environments, use NAT Instance instead of NAT Gateway:
# NAT Instance t3.nano = $3.40/month vs NAT Gateway $32+/month
# Or use VPC Endpoint for AWS services (no NAT needed)
```

---

### EC-100 — CloudWatch Logs Insights Query Cost Underestimated
**Category:** FinOps | **Severity:** Medium | **Env:** Prod

**Scenario:** Team runs frequent Logs Insights queries across large log groups. $0.005/GB scanned × 100 queries/day × 50GB/query = $25/day = $750/month just for log queries.

> ⚠️ **DevOps Gotcha:** CloudWatch Logs Insights charges per GB scanned. Scope queries to specific log groups and time windows. Export frequently-queried logs to S3 and use Athena ($5/TB scanned — much cheaper for large analytical queries).

---

### EC-101 — EBS Snapshot Costs Accumulate Without Cleanup Policy
**Category:** FinOps | **Severity:** Medium | **Env:** Prod

**Scenario:** 365 daily EBS snapshots of 500GB volume. Incremental cost: first snapshot = 500GB, subsequent snapshots = incremental changes (~5GB/day). After 365 days: 500 + 364×5 = 2,320GB of snapshots at $0.05/GB/month = $116/month.

**Fix:**
```bash
# Use Data Lifecycle Manager:
aws dlm create-lifecycle-policy \
  --description "EBS Daily Backup" \
  --state ENABLED \
  --execution-role-arn arn:aws:iam::123:role/dlm-role \
  --policy-details file://policy.json
# policy.json: keep last 7 snapshots, delete older
```

---

### EC-102 — Idle RDS Instance Consumes Full Cost
**Category:** FinOps | **Severity:** Medium | **Env:** Dev

**Scenario:** Development RDS `db.r5.2xlarge` created for load testing. Testing completed. Instance idle for 3 months. $700/month wasted.

**Fix:**
```bash
# Create automated stop/start schedule:
aws rds stop-db-instance --db-instance-identifier dev-db
# Note: RDS auto-starts after 7 days — needs recurring Lambda to re-stop
# Or use Instance Scheduler solution from AWS
```

---

### EC-103 — Lambda Provisioned Concurrency Charged Even When Idle
**Category:** FinOps | **Severity:** Medium | **Env:** Prod

**Scenario:** Lambda provisioned concurrency set to 50 for low-latency. Charges: $0.000064/GB-second × 50 instances × 730 hours/month × 1GB = $2,336/month even with zero invocations.

> ⚠️ **DevOps Gotcha:** Provisioned concurrency has a flat hourly cost regardless of invocations. Use Application Auto Scaling to scale provisioned concurrency based on schedule or utilization, reducing idle cost.

---

### EC-104 — Orphaned Load Balancer After Kubernetes Service Delete
**Category:** FinOps | **Severity:** Medium | **Env:** Prod

**Scenario:** Kubernetes Service of type `LoadBalancer` creates AWS ELB. `kubectl delete service` in Kubernetes — but under some conditions (e.g., cluster deleted before service), the ELB remains provisioned and billed.

**Detection:**
```bash
aws elbv2 describe-load-balancers | jq '.LoadBalancers[] | \
  select(.State.Code == "active") | .LoadBalancerArn'
# Cross-reference with kubectl get svc --all-namespaces
```

---

### EC-105 — Free Tier Exhausted by Auto-Created Resources
**Category:** FinOps | **Severity:** Medium | **Env:** Dev

**Scenario:** New AWS account. CloudTrail, Config, GuardDuty, Security Hub all auto-enabled (recommended). These have costs beyond the free tier. Monthly bill surprise for new users.

> ⚠️ **DevOps Gotcha:** Set up AWS Budgets and Cost Anomaly Detection on day 1 in every new account. Set a $10 budget alert to catch unexpected charges immediately.

---

## Disaster Recovery & Resilience

### EC-106 — RPO vs RTO Misaligned With Business Requirements
**Category:** DR | **Severity:** Critical | **Env:** Prod

**Scenario:** Technical team designs for 1-hour RPO (1 hour of data loss OK). Business team assumes zero data loss is possible. Discovered only after an actual DR event.

```
  RPO (Recovery Point Objective): How much data loss is acceptable?
  RTO (Recovery Time Objective): How long can the service be down?
  
  RPO=0 requires synchronous replication (expensive, performance impact)
  RPO=15min requires async replication every 15 min
  RPO=24h requires daily backups
  
  Cost increases dramatically as RPO approaches 0!
```

> ⚠️ **DevOps Gotcha:** Document RPO and RTO in writing with business sign-off. Validate with regular DR drills. Many teams discover their actual RTO (hours) differs vastly from assumed RTO (minutes) only during an actual incident.

---

### EC-107 — Backup Validated by File Existence — Not by Restore Test
**Category:** DR | **Severity:** Critical | **Env:** Prod

**Scenario:** Daily backups confirmed by checking S3 for new .tar.gz files. Six months later, restore test reveals the backup process was writing corrupt archives (out of disk space during compression, partial writes). All backups are useless.

> ⚠️ **DevOps Gotcha:** A backup that hasn't been tested is not a backup — it's a hope. Run automated restore tests monthly: restore to a separate instance, run integrity checks, validate application health.

---

### EC-108 — Cross-Region Failover DNS Change Delayed by High TTL
**Category:** DR | **Severity:** Critical | **Env:** Prod

**Scenario:** Primary region fails. Team updates Route53 record to point to DR region. DNS TTL is 300 seconds (5 minutes). All clients cached the old IP continue to hit the dead primary for 5 minutes.

**Fix:**
```bash
# Pre-configure low TTL (30–60 seconds) on DR-critical records
# Use Route53 health check based failover:
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXX \
  --change-batch '{"Changes":[{"Action":"UPSERT","ResourceRecordSet":{
    "Name":"api.example.com","Type":"A",
    "SetIdentifier":"primary","Failover":"PRIMARY",
    "HealthCheckId":"hc-xxx","TTL":30,
    "ResourceRecords":[{"Value":"1.2.3.4"}]}}]}'
```

---

### EC-109 — DR Environment Not Tested Under Production Load
**Category:** DR | **Severity:** High | **Env:** Prod

**Scenario:** DR environment provisioned with smaller instance sizes ("we'll upgrade if needed"). During actual DR failover: smaller instances can't handle production traffic. Manual upgrade takes 20 minutes while users see errors.

> ⚠️ **DevOps Gotcha:** DR environment must be tested with production-equivalent load. Use Infrastructure as Code so DR environment is identical to production. Run quarterly DR drills with simulated production traffic.

---

### EC-110 — Kubernetes Cluster Backup Doesn't Include PersistentVolumes
**Category:** DR | **Severity:** Critical | **Env:** Prod

**Scenario:** Velero backup configured to backup Kubernetes objects (Deployments, Services, ConfigMaps). PersistentVolume snapshots not included. Cluster restored — apps running but databases empty.

**Fix:**
```bash
# Enable Velero volume snapshots:
velero install \
  --provider aws \
  --bucket my-backup-bucket \
  --use-volume-snapshots=true \
  --snapshot-location-config region=us-east-1

# Create backup with volume snapshots:
velero backup create prod-backup --include-namespaces=production \
  --default-volumes-to-fs-backup=true
```

---

## Production FAQ

**Q1: EC2 instances are stopping randomly. Where do I look first?**  
A: Check `aws ec2 describe-instance-status --instance-ids i-xxx`. Look for scheduled events (host maintenance, retirement). Check CloudTrail for `StopInstances` API calls. Check ASG scale-in events and spot interruption notices in instance metadata.

**Q2: AWS bill is 3× higher than expected this month. How to diagnose?**  
A: Use AWS Cost Explorer → "Cost by Service" + "Cost by Usage Type". Look for data transfer costs (often 10× estimated), unused EBS volumes, forgotten Reserved Instances on wrong instance types. Enable Cost Anomaly Detection for automated alerts.

**Q3: IAM `AccessDenied` error but the policy looks correct.**  
A: Check in order: (1) SCPs in Organizations, (2) Permission boundaries on the role, (3) Resource-based policies (S3 bucket policy, KMS key policy), (4) VPC endpoint policies, (5) AWS Organizations delegated admin. Use IAM Access Analyzer to simulate.

**Q4: Lambda function timing out but logs show it completes in 2 seconds.**  
A: Timeout set too low? Check environment — if Lambda is in a VPC, cold starts take longer. Check if Lambda is waiting for network calls that aren't completing (security group blocks outbound to external service). Use X-Ray tracing to find the slow subsegment.

**Q5: Terraform plan shows no changes but resources are different from code.**  
A: Run `terraform refresh` to sync state with reality. Some resources have attributes that Terraform doesn't track (e.g., tags added via Console). Use `terraform import` to bring untracked resources under management.

**Q6: RDS automated backup stopped working.**  
A: Check: (1) backup retention period is > 0, (2) no ongoing maintenance window conflict, (3) disk space on instance (needs space for binary logs), (4) RDS event log for backup errors. Reboot can sometimes unstick a broken backup.

**Q7: Kubernetes pods evicted unexpectedly.**  
A: Check node conditions: `kubectl describe node | grep -A5 Conditions`. `MemoryPressure` or `DiskPressure` triggers pod eviction. Check pod `priorityClassName` — lower priority pods evicted first. Review Eviction API events: `kubectl get events --field-selector reason=Evicted`.

**Q8: CloudFront serving stale content despite origin change.**  
A: CloudFront default TTL is 24 hours. Create an invalidation: `aws cloudfront create-invalidation --distribution-id EXXX --paths "/*"`. Or configure proper `Cache-Control` headers at origin: `Cache-Control: max-age=300, s-maxage=3600` (different for browsers vs CDN).

**Q9: SQS messages being processed but still visible in queue.**  
A: Check `visibilityTimeout` — if Lambda takes longer than `visibilityTimeout`, message becomes visible again while Lambda is still processing → double processing. Set `visibilityTimeout ≥ 6 × Lambda timeout`.

**Q10: New VPC resources can't reach internet despite NAT Gateway configured.**  
A: Check in order: (1) route table associated with subnet, (2) route `0.0.0.0/0` → NAT Gateway, (3) NAT Gateway is in a PUBLIC subnet with its own route to Internet Gateway, (4) NAT Gateway status is "available", (5) security group outbound rules allow traffic.

---

> **Next:** [Git Edge Cases](git_edge_cases.md) · [Nginx Edge Cases](nginx_edge_cases.md)  
> **Back:** [Networking Edge Cases](networking_edge_cases.md) · [Batch State](batch_state.md) · [🏠 Home](../README.md)
