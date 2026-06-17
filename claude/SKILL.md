---
name: hardEningCLOUD_byDM20911
description: >-
  Auditoría y hardening integral de un proyecto en la nube (GCP, AWS o Azure) de
  punta a punta: reconocimiento total de la cuenta/proyecto/suscripción, detección
  de vulnerabilidades y malas configuraciones (CIS Benchmark, IAM, red, storage,
  cifrado, logging), remediación con aprobación humana previa, y un informe final
  detallado con antes/después y score de postura. Úsala SIEMPRE que el usuario diga
  "audita mi GCP/AWS/Azure", "hardening cloud", "revisa la seguridad de mi cuenta
  AWS", "endurece mi proyecto en Azure", "corre un CIS benchmark", "analiza la
  postura de mi nube", "qué está mal configurado en mi GCP", "asegura mis buckets/
  IAM/firewall", o entregue credenciales, IDs de proyecto/cuenta/suscripción, salidas
  de gcloud/aws/az, archivos de Terraform/CloudFormation/ARM/Bicep, o pida corregir
  hallazgos de seguridad cloud. Cubre las tres nubes y prioriza esta skill sobre
  invocaciones ad-hoc de herramientas para imponer una metodología reproducible
  (recon → análisis → remediación autorizada → informe).
---

# hardEningCLOUD_byDM20911 — Auditoría y Hardening Cloud (GCP · AWS · Azure)

Skill end-to-end para endurecer un proyecto cloud completo. No es solo un escáner: es
un flujo de trabajo que **descubre todo** lo que hay en la cuenta, **encuentra** lo que
está mal, **lo arregla con permiso del humano**, y **documenta** qué se halló y qué se
corrigió. Asume **autorización explícita** del cliente (el usuario opera sobre objetivos
propios o contratados); por eso no se pregunta por autorización legal, pero **sí** se pide
aprobación humana antes de cualquier cambio que escriba en la nube.

## Principio rector

El valor está en cerrar el ciclo: muchos escáneres listan hallazgos y se detienen. Aquí
el objetivo es dejar la nube **medible y verificablemente más segura** que como estaba, con
evidencia de antes y después. Por eso el informe gira en torno a un **score de postura**
(% de checks CIS que pasan) que sube tras la remediación.

---

## Reglas de seguridad operativa (no negociables)

Estas reglas existen porque tocar una cuenta cloud en producción puede causar caídas o
pérdida de acceso. El humano siempre manda sobre los cambios.

1. **Lectura libre, escritura con permiso.** Todos los comandos de *enumeración/análisis*
   (read-only: `describe`, `list`, `get`, `get-iam-policy`) se pueden ejecutar sin pedir
   confirmación. Cualquier comando que **modifique** la nube (crear/editar/borrar/aplicar
   política, cerrar puerto, activar logging, rotar llave) requiere **aprobación humana
   explícita por hallazgo** antes de ejecutarse.
2. **Nunca te auto-bloquees ni bloquees al cliente.** Antes de tocar IAM, firewall de
   gestión (SSH/RDP/Bastion) o reglas que afecten tu propia sesión, avisa el riesgo de
   lockout y confirma que hay una vía de acceso alternativa.
3. **Captura evidencia antes y después.** Para cada fix, guarda el estado previo (output
   read-only), el comando de cambio ejecutado, y el estado posterior que confirma la
   corrección. Esto alimenta el informe.
4. **Respeta el gate de MCP.** Si vas a usar cualquier herramienta `mcp__*`, primero
   anúnciala y espera confirmación (regla global del usuario).
5. **Sin cambios destructivos silenciosos.** Borrar recursos, rotar llaves en uso o cambiar
   políticas amplias se propone, se explica el impacto, y solo se ejecuta tras "sí".

---

## Fase 0 — Intake (SIEMPRE primero)

Antes de ejecutar nada, pregunta y confirma. El usuario decide cómo se trabaja. Hazlo
breve, en una sola tanda de preguntas:

1. **Modo de trabajo** — ¿cómo ejecutamos los comandos?
   - **A) Claude ejecuta CLI directo** — `gcloud`/`aws`/`az` ya autenticados en esta
     máquina. Claude corre los read-only solo, y los de cambio tras tu aprobación.
   - **B) Copy-paste** — Claude entrega comandos en **una sola línea** listos para pegar
     en tu zsh (acorde a entornos con VPN); tú ejecutas y pegas el output. Claude nunca
     toca la nube directo.
   - **C) Híbrido** — Claude ejecuta directo si detecta CLI autenticada; si no, cae a
     copy-paste.
2. **Nube objetivo** — GCP, AWS o Azure (o varias). Pide el identificador:
   `PROJECT_ID` (GCP) / cuenta o perfil (AWS) / `SUBSCRIPTION_ID` (Azure).
3. **Alcance** — ¿toda la cuenta/proyecto, o servicios/recursos específicos? ¿algo
   out-of-scope?
4. **Modo de remediación** — confirma el comportamiento por severidad (default abajo, el
   usuario puede ajustarlo):
   - **Críticos / Altos** → Claude **propone** el fix y lo aplica **solo tras aprobación
     explícita** (cambio sensible).
   - **Medios / Bajos seguros** (activar logging, cifrado por defecto, versioning) →
     Claude propone en lote y aplica tras un "sí" agrupado.
   - **Nada se aplica sin aprobación.** El usuario puede pasar a modo "solo entrega fixes"
     (no tocar la nube) en cualquier momento.

Detecta el modo de comunicación: si trabajas en copy-paste, **una línea por comando**,
sin `\` de continuación, sin sangría inicial (los clientes markdown añaden espacios al
pegar; sugiere usar el botón de copiar del bloque).

### Preflight — verifica e instala las CLIs antes de empezar

No puedes auditar sin las herramientas. Justo después del intake y antes de Fase 1,
**verifica qué falta y ofrece instalarlo**. Instala solo lo necesario para la nube elegida
+ los scanners; instalar es un cambio en el sistema, así que **propón los comandos y pide
un "sí"** antes de ejecutarlos (o entrégalos para copy-paste en modo B).

Verifica presencia con `command -v <bin>`. Lo que se necesita por nube:

- **Común:** `python3`, `jq`, `pipx` (o `pip`).
- **AWS:** `aws` (AWS CLI v2).
- **GCP:** `gcloud` (Google Cloud SDK).
- **Azure:** `az` (Azure CLI).
- **Scanners (recomendados):** `prowler` (cubre las 3 nubes contra CIS), `scout` (ScoutSuite),
  opcional `steampipe` + `powerpipe`.

Comandos de instalación en macOS (Homebrew); en otras plataformas usa el gestor equivalente:
```bash
command -v jq >/dev/null || brew install jq
command -v aws >/dev/null || brew install awscli
command -v gcloud >/dev/null || brew install --cask google-cloud-sdk
command -v az >/dev/null || brew install azure-cli
command -v pipx >/dev/null || brew install pipx
command -v prowler >/dev/null || pipx install prowler
command -v scout >/dev/null || pipx install scoutsuite
command -v steampipe >/dev/null || brew install turbot/tap/steampipe
```
Tras instalar, confirma versiones (`aws --version`, `gcloud --version`, `az version`,
`prowler --version`) y que la CLI de la nube objetivo esté **autenticada** (`aws sts
get-caller-identity` / `gcloud auth list` / `az account show`). Si falta autenticación en
modo CLI directo, pídele al usuario que ejecute el login (`gcloud auth login`, `aws
configure`/SSO, `az login`) — sugiere el prefijo `! <comando>` para que corra en la sesión.

Tras el intake y el preflight, genera un **plan de auditoría** corto con los dominios CIS
que vas a cubrir y empieza por Fase 1.

---

## Fase 1 — Reconocimiento e inventario total

Objetivo: saber **todo** lo que existe antes de juzgarlo. Enumera identidades, red,
almacenamiento, cómputo, datos, logging y servicios habilitados.

Carga el archivo de la nube correspondiente y ejecuta su bloque de enumeración:

- AWS → `references/aws.md`
- GCP → `references/gcp.md`
- Azure → `references/azure.md`

Cada referencia trae: setup, scanners automáticos, y comandos read-only por dominio
(IAM, datos, red, logging) más su checklist CIS. Empieza siempre por correr el scanner
automático (Prowler/ScoutSuite) para tener una línea base amplia, y luego profundiza
manualmente en lo que el scanner marque o no cubra.

Guarda toda la salida cruda. Esa salida es la evidencia y el insumo del análisis.

---

## Fase 2 — Análisis y detección de vulnerabilidades

Convierte la salida cruda en hallazgos clasificados.

1. **Scanner automático como base.** Prowler emite resultados mapeados a CIS; úsalos como
   esqueleto. ScoutSuite da un HTML navegable. No te quedes solo con ellos: validan
   amplitud, no profundidad.
2. **Análisis dirigido de IAM, red y storage** con el script incluido. Para políticas IAM,
   ACLs de buckets o reglas de firewall en JSON, usa:
   ```bash
   python3 scripts/cloud_posture_check.py <config.json> --check <iam|s3|sg|all> --provider <aws|gcp|azure> --json
   ```
   **Elige el `--check` que corresponde al tipo de archivo**: para una política IAM usa `--check iam`,
   para configuración de bucket `--check s3`, para reglas de firewall `--check sg`. Reserva `--check all`
   para configs mixtas; sobre un JSON de IAM puro, `all` dispara falsos positivos de S3. S3/SG solo
   aplican con `--provider aws`; para GCP/Azure el analizador cubre IAM y los scanners cubren el resto.
   Detecta escaladas de privilegios (combos peligrosos tipo `iam:PassRole`+`lambda:CreateFunction`),
   exposición pública de storage y puertos de administración abiertos a 0.0.0.0/0. Usa
   `--severity-modifier internet-facing` para recursos expuestos y `regulated-data` para
   datos regulados (PCI/HIPAA/GDPR) — suben la severidad un nivel. Detalle de checks y
   mapeo MITRE en `references/cspm-checks.md`.
3. **Clasifica cada hallazgo** con: dominio CIS, severidad (Critical/High/Medium/Low/Info),
   CVSS 3.1, recurso afectado, evidencia (comando + output), y remediación propuesta
   (comando CLI y/o IaC corregido). Registra cada hallazgo en una lista de trabajo —
   considera usar TaskCreate para no perder ninguno.
4. **Calcula el score de postura inicial**: `% checks CIS que pasan`. Es el "antes".

---

## Fase 3 — Remediación con aprobación humana

Aquí se cierra el ciclo. Por cada hallazgo, sigue el patrón propose → approve → apply → verify.

1. **Propón** el fix mostrando: qué cambia, comando/IaC exacto, e **impacto y riesgo**
   (¿puede causar lockout o caída? ¿afecta producción?).
2. **Espera aprobación** según el modo de remediación pactado en el intake. Críticos/altos
   uno a uno; medios/bajos seguros se pueden aprobar en lote.
3. **Aplica** (modo A/C) o **entrega el comando en una línea** (modo B). Antes de aplicar,
   captura el estado previo read-only.
4. **Verifica** re-ejecutando el check read-only: el hallazgo debe desaparecer. Guarda el
   "después".
5. Si un fix falla o el usuario lo rechaza, márcalo como "no remediado — pendiente" con la
   razón. No lo escondas.

Orden recomendado: Críticos → Altos → Medios → Bajos. Dentro de críticos, prioriza lo
expuesto a internet y lo que afecta datos regulados.

Tras la remediación, **recalcula el score de postura**: ese es el "después".

---

## Fase 4 — Informe final

Genera un informe `.md` cloud-específico siguiendo `references/report-template.md`. Es un
formato nuevo orientado a CIS Benchmark, con énfasis en el **antes/después de remediación**
y el **delta del score de postura**. Incluye:

- Encabezado (proyecto, nube, ambiente, fecha, clasificación, modo de trabajo).
- Resumen ejecutivo: score antes → después, conteo de hallazgos por severidad, % remediado.
- Tabla de postura por dominio CIS (IAM, red, storage, cifrado, logging, …) con pass/fail.
- Hallazgos detallados: cada uno con clasificación CIS + (OWASP Cloud / MITRE ATT&CK Cloud
  si aplica), CVSS 3.1, descripción, impacto, evidencia (antes), remediación aplicada
  (comando), y verificación (después).
- Pendientes / no remediados con su razón.
- Recomendaciones de hardening continuo (IaC gates, monitoreo, rotación).

Genera también `evidencias[TICKET].md` con todos los comandos ejecutados (read-only y de
cambio), fecha/hora y output exacto, sin texto explicativo — solo comando y resultado.

Organiza los archivos en una carpeta por ticket/proyecto (ej. `audit_GCP_<fecha>/`), no
mezcles tickets distintos en la raíz.

---

## Matriz de cobertura por nube

| Dominio | AWS | GCP | Azure |
|---------|-----|-----|-------|
| Identidad / IAM | IAM users/roles/policies, MFA, access keys, escaladas | IAM bindings, SA keys, roles primitivos, allUsers | Entra ID, RBAC, Owners, PIM, MFA/CA |
| Almacenamiento | S3 (ACL, policy, public block, cifrado) | GCS (IAM, uniform access, allUsers) | Storage (public blob, HTTPS, SAS) |
| Red | Security Groups, NACL, puertos abiertos | Firewall rules, IPs públicas, Flow Logs | NSG, IPs públicas, Bastion |
| Cifrado | EBS/RDS/S3, KMS | CMEK, default encryption | Key Vault, TLS, cifrado en reposo |
| Logging | CloudTrail, GuardDuty, Config | Cloud Audit Logs, SCC, sinks | Diagnostic Settings, Activity Log, Defender |
| IaC | Terraform, CloudFormation | Deployment Manager, Terraform | ARM, Bicep, Terraform |

---

## Anti-patrones (evítalos)

1. **Aplicar fixes sin captura de "antes".** Sin estado previo no hay evidencia de mejora ni
   forma de revertir. Siempre fotografía el estado antes de cambiar.
2. **Tocar IAM o firewall de gestión sin advertir lockout.** Cerrar el puerto/permiso
   equivocado te deja a ti o al cliente fuera. Confirma vía de acceso alterna primero.
3. **Confiar solo en el scanner automático.** Prowler/ScoutSuite dan amplitud pero pierden
   lógica de negocio y combos de escalada. Profundiza manualmente lo que marquen.
4. **Analizar solo políticas de admin.** Las escaladas suelen nacer de políticas inocuas que
   combinan permisos. Revisa todas las identidades de producción, no solo las obviamente
   privilegiadas.
5. **"No hay datos en QA" ≠ "no hay vulnerabilidad".** Si la nube no rechaza un acceso que
   debería, la mala configuración existe aunque el recurso esté vacío.
6. **Remediar sin entender por qué se otorgó el permiso.** Documenta la justificación antes de
   quitar un permiso amplio, o volverá a aparecer.
7. **Informe sin antes/después.** El score de postura debe mostrar el delta; un informe que
   solo lista hallazgos no demuestra hardening.

---

## Archivos de la skill

- `references/aws.md` — enumeración, scanners y checklist CIS de AWS.
- `references/gcp.md` — enumeración, scanners y checklist CIS de GCP.
- `references/azure.md` — enumeración, scanners y checklist CIS de Azure.
- `references/cspm-checks.md` — matrices de checks (condición, severidad, MITRE, remediación).
- `references/report-template.md` — plantilla del informe cloud-específico con antes/después.
- `scripts/cloud_posture_check.py` — análisis de IAM/storage/red desde JSON (exit 0/1/2).
