# AWS Penetration Testing — Pre-Flight Checklist

A condensed, copy-pasteable checklist for use during engagements. For full methodology, tooling, and exploitation notes see [README.md](README.md).

> Use only against systems you are authorized to test.

---

## Reconnaissance & Asset Discovery

- [ ] `aws sts get-caller-identity` — confirm identity and account ID
- [ ] Enumerate all regions: `aws ec2 describe-regions --query 'Regions[].RegionName'`
- [ ] Run CloudFox: `cloudfox aws --profile <p> all-checks`
- [ ] Enumerate all IAM actions available to this identity: `enumerate-iam`
- [ ] Scan GitHub/code repos for leaked credentials: `trufflehog`, `gitleaks`
- [ ] Enumerate public S3 buckets: `s3scanner`
- [ ] Search Shodan/Censys for exposed AWS assets tied to the org
- [ ] Check certificate transparency logs for subdomain AWS endpoints (S3, API GW, Elastic Beanstalk)
- [ ] Test for subdomain takeovers on unclaimed AWS endpoints

---

## IAM & Access Control

- [ ] List all IAM users, roles, groups, policies: `aws iam list-users`, `list-roles`, `list-groups`
- [ ] Identify `*:*` wildcard policies on users/roles
- [ ] Run PMapper: `pmapper graph create && pmapper analysis --privesc` — automated PrivEsc path analysis
- [ ] Check for `iam:PassRole` + Lambda/EC2/ECS/Glue/CloudFormation combinations (PrivEsc)
- [ ] Check for `iam:AttachUserPolicy`, `iam:CreatePolicyVersion`, `iam:AddUserToGroup` on low-priv principals
- [ ] Enumerate role trust policies — external accounts, broad conditions, missing ExternalId/MFA
- [ ] Verify root account has MFA, no active access keys
- [ ] Check permission boundaries on delegated admin roles
- [ ] Audit SCPs at org level — `cloudtrail:StopLogging` and `guardduty:DeleteDetector` should be denied

---

## Network & VPC

- [ ] Enumerate security groups — flag any `0.0.0.0/0` inbound on ports 22, 3389, 3306, 5432, 6379, 9200, 27017
- [ ] Review NACLs for overly permissive allow rules
- [ ] Check VPC flow logs are enabled for all VPCs
- [ ] Enumerate peering connections and Transit Gateway routes
- [ ] Verify no internet gateway routes in private subnets
- [ ] Test WAF rules for bypass (encoding, header manipulation, large payloads)

---

## S3 & Storage

- [ ] `aws s3 ls` — enumerate all buckets
- [ ] Test each bucket unauthenticated: `aws s3 ls s3://<bucket> --no-sign-request`
- [ ] Check Block Public Access at account level: `aws s3control get-public-access-block --account-id <id>`
- [ ] Review bucket policies for `Principal: "*"`
- [ ] Scan for sensitive files: `.env`, `*.pem`, `*.key`, backup files, `credentials`
- [ ] Check versioning for sensitive files accessible in older versions
- [ ] Enumerate public EBS snapshots: `aws ec2 describe-snapshots --owner-ids self --filters Name=public,Values=true`
- [ ] Scan S3 buckets for secrets in file contents: `trufflehog s3 --bucket <name>`

---

## Compute & EC2

- [ ] Check IMDSv2 enforcement: `aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,MetadataOptions.HttpTokens]'`
- [ ] Test SSRF to `http://169.254.169.254/latest/meta-data/iam/security-credentials/` on web apps
- [ ] Enumerate EC2 user data for credentials: `aws ec2 describe-instance-attribute --instance-id <id> --attribute userData`
- [ ] Check instance profiles — what IAM roles are attached to running instances?
- [ ] Enumerate public AMIs: `aws ec2 describe-images --owners self --filters Name=is-public,Values=true`
- [ ] Verify EBS volumes are encrypted

---

## Containers (EKS / ECR)

- [ ] List EKS clusters: `aws eks list-clusters`
- [ ] Check cluster endpoint public access and CIDR restrictions
- [ ] Review `aws-auth` ConfigMap or EKS Access Entries for `system:masters` bindings
- [ ] Enumerate service account IAM role associations (IRSA): `kubectl get serviceaccounts -A`
- [ ] Check for privileged pods: `kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].securityContext.privileged==true)'`
- [ ] Run kube-bench: `kube-bench run`
- [ ] Scan ECR images with Trivy: `trivy image <account>.dkr.ecr.<region>.amazonaws.com/<image>`
- [ ] Check for public ECR repositories: `aws ecr-public describe-repositories`

---

## Lambda & API Gateway

- [ ] List all Lambda functions: `aws lambda list-functions`
- [ ] Dump Lambda environment variables: `aws lambda get-function-configuration --function-name <name> | jq .Environment`
- [ ] Check Lambda execution roles for over-privilege
- [ ] Check Lambda resource-based policies for `Principal: "*"`
- [ ] Enumerate API Gateway APIs and stages: `aws apigateway get-rest-apis`
- [ ] Test API routes for missing auth (especially GET vs POST on same resource)
- [ ] Check access logs are enabled on API Gateway stages

---

## Databases

- [ ] List RDS instances: `aws rds describe-db-instances --query 'DBInstances[].[DBInstanceIdentifier,PubliclyAccessible,StorageEncrypted]'`
- [ ] Check for public RDS snapshots: `aws rds describe-db-snapshots --snapshot-type public`
- [ ] Verify ElastiCache requires auth (Redis AUTH token or RBAC)
- [ ] Test for SQL injection in apps backed by RDS
- [ ] Enumerate DynamoDB tables: `aws dynamodb list-tables`

---

## Secrets & Keys

- [ ] List Secrets Manager secrets: `aws secretsmanager list-secrets`
- [ ] Check who has `secretsmanager:GetSecretValue` — over-broad access?
- [ ] Enumerate SSM Parameter Store: `aws ssm describe-parameters`
- [ ] Retrieve SecureString parameters: `aws ssm get-parameter --name <n> --with-decryption`
- [ ] Audit KMS key policies: `aws kms list-keys` → `get-key-policy` for each
- [ ] Scan CloudFormation templates for hardcoded secrets

---

## Logging & Monitoring

- [ ] Verify CloudTrail enabled in all regions including global services trail
- [ ] Check log file validation enabled: `aws cloudtrail describe-trails --query 'trailList[].LogFileValidationEnabled'`
- [ ] Verify GuardDuty enabled in all regions: `aws guardduty list-detectors`
- [ ] Check Security Hub is enabled and aggregated
- [ ] Review GuardDuty suppression rules for silenced findings
- [ ] Verify VPC flow logs active for all VPCs
- [ ] Attempt `cloudtrail:StopLogging` — should be denied by SCP

---

## Post-Exploitation (Document & Clean Up)

- [ ] Log every IAM key created, role modified, or resource accessed
- [ ] Remove all backdoor users, access keys, policies, and Lambda functions at engagement end
- [ ] Confirm cleanup with client before closing

---

## Quick Reference Commands

```bash
# Identify current identity
aws sts get-caller-identity

# Enumerate all IAM permissions brute-force
python3 enumerate-iam.py --access-key <key> --secret-key <secret> --session-token <token>

# Run CloudFox full inventory
cloudfox aws --profile <profile> all-checks

# Run PMapper privilege escalation analysis
pmapper --profile <profile> graph create
pmapper --profile <profile> analysis --privesc

# Check IMDSv2 status across all EC2
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].[InstanceId,MetadataOptions.HttpTokens]' \
  --output table

# List Lambda env vars across all functions
aws lambda list-functions --query 'Functions[].FunctionName' --output text | \
  tr '\t' '\n' | xargs -I{} aws lambda get-function-configuration --function-name {} \
  --query '[FunctionName,Environment.Variables]'

# Find public EBS snapshots
aws ec2 describe-snapshots --owner-ids self \
  --filters Name=public,Values=true --query 'Snapshots[].[SnapshotId,Description]'
```
