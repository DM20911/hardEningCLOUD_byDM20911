# Azure — Enumeración, scanners y checklist CIS

Comandos read-only para inventario y análisis. Una sola línea para copy-paste.
Asume rol de auditor con lectura (`Reader` + `Security Reader`).

## 0. Setup y contexto
```bash
az login
az account list --output table
az account set --subscription <SUB_ID>
az account management-group list 2>/dev/null
```

## 1. Scanners automáticos (línea base)
```bash
prowler azure --az-cli-auth -M html json-ocsf
scout azure --cli
steampipe plugin install azure && powerpipe benchmark run azure_compliance.benchmark.cis_v300
```

## 2. IAM / Entra ID
```bash
az role assignment list --all --output table
az role assignment list --role Owner --all --output table
az ad user list --query "[].{name:displayName,upn:userPrincipalName}" -o table
az ad sp list --all --query "[].displayName" -o table
az role assignment list --include-classic-administrators --all -o table
```
Revisar: exceso de Owners/Contributors, guest users, SP con credenciales de larga vida, ausencia de MFA / Conditional Access, PIM no usado para roles privilegiados, app registrations con secrets.

## 3. Storage / Datos
```bash
az storage account list --query "[].{name:name,https:enableHttpsTrafficOnly,public:allowBlobPublicAccess,tls:minimumTlsVersion}" -o table
az storage account show --name <acct> --query "networkRuleSet"
az storage container list --account-name <acct> --query "[?properties.publicAccess!=null].name" -o table
```

## 4. Red
```bash
az network nsg list --query "[].{name:name,rg:resourceGroup}" -o table
az network nsg rule list --nsg-name <nsg> -g <rg> --query "[?sourceAddressPrefix=='*' || sourceAddressPrefix=='0.0.0.0/0'].{name:name,port:destinationPortRange,access:access}" -o table
az network public-ip list -o table
```

## 5. Logging y detección
```bash
az monitor diagnostic-settings list --resource <RESOURCE_ID> 2>/dev/null
az monitor activity-log list --max-events 50 -o table
az security auto-provisioning-setting list 2>/dev/null
```

## Remediación — comandos de cambio (requieren aprobación)
```bash
az storage account update --name <acct> -g <rg> --allow-blob-public-access false
az storage account update --name <acct> -g <rg> --https-only true --min-tls-version TLS1_2
az network nsg rule update --nsg-name <nsg> -g <rg> --name <rule> --source-address-prefixes <CIDR_VPN>
az role assignment delete --assignee <upn> --role Owner --scope /subscriptions/<SUB_ID>
az keyvault update --name <kv> --enable-purge-protection true
```

## Checklist CIS Azure Foundations (top hits)
- [ ] MFA para todos los usuarios (especialmente admins) vía Conditional Access
- [ ] Sin guests innecesarios; PIM (Just-In-Time) para roles privilegiados
- [ ] Mínimo de Owners por suscripción
- [ ] `allowBlobPublicAccess=false`, HTTPS forzado, TLS 1.2+
- [ ] Storage con firewall / private endpoints
- [ ] Microsoft Defender for Cloud habilitado (planes relevantes)
- [ ] Diagnostic Settings → Log Analytics en recursos clave
- [ ] Activity Log con retención y alertas críticas
- [ ] NSG sin reglas Internet → 22/3389
- [ ] NSG Flow Logs habilitados
- [ ] Key Vault con soft-delete + purge protection
