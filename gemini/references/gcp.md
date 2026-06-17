# GCP — Enumeración, scanners y checklist CIS

Comandos read-only para inventario y análisis. Una sola línea para copy-paste.
Asume cuenta/SA de auditor con roles de lectura (`roles/viewer` + `roles/iam.securityReviewer`).

## 0. Setup y contexto
```bash
gcloud auth list
gcloud projects list
gcloud config set project <PROJECT_ID>
gcloud services list --enabled
```

## 1. Scanners automáticos (línea base)
```bash
prowler gcp --project-ids <PROJECT_ID> -M html json-ocsf
scout gcp --user-account
gcloud scc findings list <ORG_ID> 2>/dev/null
steampipe plugin install gcp && powerpipe benchmark run gcp_compliance.benchmark.cis_v300
```

## 2. IAM
```bash
gcloud projects get-iam-policy <PROJECT_ID> --format=json
gcloud iam service-accounts list
gcloud iam service-accounts keys list --iam-account=<SA_EMAIL>
gcloud projects get-iam-policy <PROJECT_ID> --flatten="bindings[].members" --filter="bindings.role:roles/owner" --format="table(bindings.members)"
gcloud projects get-iam-policy <PROJECT_ID> --format=json | python3 scripts/cloud_posture_check.py - --check iam --provider gcp --json
```
Revisar: roles primitivos (`owner`/`editor`/`viewer`) a usuarios, SA con roles primitivos, `allUsers`/`allAuthenticatedUsers` en bindings (público), llaves de SA user-managed (deben evitarse o rotar < 90 días), suplantación de SA (`iam.serviceAccountTokenCreator`).

## 3. Cloud Storage / Datos
```bash
gsutil ls
gsutil iam get gs://<bucket>
gcloud storage buckets describe gs://<bucket> --format="default(uniform_bucket_level_access,encryption)"
gsutil ls -L -b gs://<bucket> | grep -iE "public|allUsers"
```

## 4. Red
```bash
gcloud compute firewall-rules list --format="table(name,sourceRanges.list(),allowed[].map().firewall_rule().list())"
gcloud compute firewall-rules list --filter="sourceRanges:0.0.0.0/0" --format="table(name,allowed[].map().firewall_rule().list())"
gcloud compute instances list --format="table(name,networkInterfaces[].accessConfigs[].natIP.notnull().list())"
gcloud compute networks subnets list --format="table(name,region,enableFlowLogs)"
```

## 5. Logging y detección
```bash
gcloud logging sinks list
gcloud logging buckets list --location=global
gcloud projects get-iam-policy <PROJECT_ID> --format="json(auditConfigs)"
```

## Remediación — comandos de cambio (requieren aprobación)
```bash
gcloud storage buckets update gs://<bucket> --uniform-bucket-level-access
gsutil iam ch -d allUsers gs://<bucket>
gcloud compute firewall-rules update <rule> --source-ranges=<CIDR_VPN>
gcloud projects remove-iam-policy-binding <PROJECT_ID> --member="user:<email>" --role="roles/owner"
gcloud iam service-accounts keys delete <KEY_ID> --iam-account=<SA_EMAIL>
gcloud compute networks subnets update <subnet> --region=<region> --enable-flow-logs
```

## Checklist CIS GCP Foundations (top hits)
- [ ] Sin roles primitivos (owner/editor/viewer) — usar predefinidos/custom
- [ ] Sin llaves de SA user-managed (o rotación < 90 días)
- [ ] MFA/2SV obligatorio en toda la org
- [ ] Sin recursos con `allUsers`/`allAuthenticatedUsers`
- [ ] Uniform bucket-level access activado
- [ ] CMEK donde aplique; cifrado en reposo verificado
- [ ] Cloud Audit Logs (Admin + Data Access) habilitados, sin exenciones
- [ ] Security Command Center activo
- [ ] Sin firewall a 0.0.0.0/0 en 22/3389/DB
- [ ] VPC Flow Logs y Private Google Access habilitados
