# Plantilla de Informe — Hardening Cloud (antes/después)

Formato cloud-específico orientado a CIS Benchmark. El eje es el **delta del score de
postura**: demuestra que la nube quedó medible y verificablemente más segura. Rellena
todos los `[ ]`. Genera el informe en un `.md` propio dentro de la carpeta del proyecto.

---

```markdown
# Informe de Hardening Cloud — [Nombre del Proyecto]

**Nube:** [GCP / AWS / Azure]
**Identificador:** [PROJECT_ID / Cuenta / SUBSCRIPTION_ID]
**Ambiente:** [QA / Staging / Producción]
**Modo de trabajo:** [CLI directo / Copy-paste / Híbrido]
**Fecha:** [YYYY-MM-DD]
**Clasificación:** Confidencial

## Resumen ejecutivo

| Métrica | Antes | Después |
|---------|-------|---------|
| Score de postura (CIS % pass) | [X]% | [Y]% |
| Hallazgos Critical | [n] | [n] |
| Hallazgos High | [n] | [n] |
| Hallazgos Medium | [n] | [n] |
| Hallazgos Low | [n] | [n] |
| % remediado | — | [Z]% |

[Párrafo: estado general, principales riesgos cerrados, pendientes relevantes.]

## Postura por dominio CIS

| Dominio | Checks | Pass (antes) | Pass (después) |
|---------|--------|--------------|----------------|
| Identidad / IAM | [n] | [n] | [n] |
| Red | [n] | [n] | [n] |
| Almacenamiento | [n] | [n] | [n] |
| Cifrado | [n] | [n] | [n] |
| Logging / Monitoreo | [n] | [n] | [n] |

## Tabla de hallazgos

| # | Hallazgo | Clasificación CIS | Severidad | CVSS | Estado |
|---|----------|-------------------|-----------|------|--------|
| 1 | [nombre] | CIS [x.y] / MITRE [Txxxx] | Critical | 0.0 | Remediado / Pendiente |

---

## Hallazgo #N — [Nombre descriptivo]

#### Clasificación
**CIS [Benchmark] [control x.y]** — [nombre oficial] · (OWASP Cloud / MITRE ATT&CK si aplica)

#### CVSS 3.1
| Campo | Valor |
|-------|-------|
| **Score** | **X.X ([Severidad])** |
| **Vector** | `CVSS:3.1/AV:_/AC:_/PR:_/UI:_/S:_/C:_/I:_/A:_` |

#### Descripción
[Por qué existe y cómo se explota.]

#### Impacto
Un atacante podría [acción concreta] para [consecuencia de negocio].

#### Evidencia (ANTES)
```bash
[comando read-only ejecutado]
```
```json
[output que prueba la mala configuración]
```

#### Remediación aplicada
[Qué se cambió y por qué. Riesgo evaluado (lockout/caída) y vía de acceso confirmada.]
```bash
[comando de cambio ejecutado, en una línea]
```

#### Verificación (DESPUÉS)
```bash
[mismo comando read-only]
```
```json
[output que confirma la corrección]
```

**Estado:** Remediado [YYYY-MM-DD HH:MM] / Pendiente — [razón]

---

## Pendientes / no remediados

| # | Hallazgo | Razón | Acción recomendada |
|---|----------|-------|--------------------|
| | | | |

## Recomendaciones de hardening continuo

- IaC security gates (Terraform/CloudFormation/Bicep) en el pipeline.
- Monitoreo continuo: [GuardDuty+SecurityHub / SCC / Defender for Cloud].
- Rotación de credenciales y revisión periódica de IAM.
- Re-evaluación CIS trimestral.
```

---

## Archivo de evidencias

Genera además `evidencias[TICKET].md`: todos los comandos ejecutados (read-only y de
cambio) con fecha/hora y output exacto, sin texto explicativo. Solo comando y resultado.
