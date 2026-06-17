# AWS — Enumeración, scanners y checklist CIS

Comandos read-only para inventario y análisis. Todos en una sola línea para copy-paste.
Asume perfil de auditor con permisos de lectura (`SecurityAudit` / `ViewOnlyAccess`).

## 0. Setup y contexto
```bash
aws sts get-caller-identity
aws organizations list-accounts 2>/dev/null
aws ec2 describe-regions --query "Regions[].RegionName" --output text
```

## 1. Scanners automáticos (línea base)
```bash
prowler aws --profile auditor -M html json-ocsf
scout aws --profile auditor
steampipe plugin install aws && powerpipe benchmark run aws_compliance.benchmark.cis_v300
```

## 2. IAM (lo más crítico)
```bash
aws iam generate-credential-report && aws iam get-credential-report --query Content --output text | base64 -d
aws iam list-users --query "Users[].UserName" --output text
aws iam list-access-keys --user-name <user>
aws iam list-mfa-devices --user-name <user>
aws iam get-account-password-policy
aws accessanalyzer list-findings 2>/dev/null
aws iam list-attached-user-policies --user-name <user>
```
Exporta cada política y pásala al script:
```bash
aws iam get-policy-version --policy-arn <ARN> --version-id v1 | jq '.PolicyVersion.Document' | python3 scripts/cloud_posture_check.py - --check iam --json
```
Revisar: root con access keys, usuarios sin MFA, llaves > 90 días, políticas `*:*`, combos de escalada (`iam:PassRole`+`lambda:CreateFunction` / `ec2:RunInstances`), roles con `AssumeRole` permisivo. Para validar escaladas en lab autorizado: `pacu`.

## 3. S3 / Datos
```bash
aws s3api list-buckets --query "Buckets[].Name" --output text
aws s3api get-bucket-acl --bucket <b>
aws s3api get-bucket-policy --bucket <b> 2>/dev/null
aws s3api get-public-access-block --bucket <b>
aws s3api get-bucket-encryption --bucket <b> 2>/dev/null
aws s3api get-bucket-versioning --bucket <b>
```

## 4. Red
```bash
aws ec2 describe-security-groups --query "SecurityGroups[?IpPermissions[?IpRanges[?CidrIp=='0.0.0.0/0']]].GroupId"
aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,PublicIpAddress]" --output text
aws rds describe-db-instances --query "DBInstances[?PubliclyAccessible].DBInstanceIdentifier"
aws ec2 describe-flow-logs --query "FlowLogs[].FlowLogId"
```

## 5. Logging y detección
```bash
aws cloudtrail describe-trails
aws cloudtrail get-trail-status --name <trail>
aws guardduty list-detectors
aws configservice describe-configuration-recorders
aws securityhub get-enabled-standards 2>/dev/null
```

## Remediación — comandos de cambio (requieren aprobación)
```bash
aws s3api put-public-access-block --bucket <b> --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
aws s3api put-bucket-encryption --bucket <b> --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms"},"BucketKeyEnabled":true}]}'
aws ec2 revoke-security-group-ingress --group-id <sg> --protocol tcp --port 22 --cidr 0.0.0.0/0
aws iam delete-access-key --user-name <user> --access-key-id <key>
```

## Checklist CIS AWS Foundations (top hits)
- [ ] Root sin access keys y con MFA hardware
- [ ] MFA en todas las cuentas IAM
- [ ] Política de contraseñas robusta
- [ ] CloudTrail multi-region + validación de integridad + cifrado KMS
- [ ] S3 Block Public Access a nivel cuenta y bucket
- [ ] EBS/RDS/S3 cifrados
- [ ] GuardDuty + Security Hub + Config habilitados
- [ ] Sin SG a 0.0.0.0/0 en 22/3389/DB
- [ ] Access Analyzer sin hallazgos de acceso externo no intencional
- [ ] VPC Flow Logs habilitados
