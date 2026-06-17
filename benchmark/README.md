# Benchmark — hardEningCLOUD_byDM20911

Evaluación de la skill con subagentes independientes: **con skill** vs **baseline (sin skill)**.
Misma máquina, mismos prompts, mismas aserciones objetivas. 3 casos, 14 aserciones.

## Resultados (iteration-1)

| Caso | Aserciones | Con skill | Baseline | Δ |
|------|:----------:|:---------:|:--------:|:--:|
| `gcp-audit-and-fix` | 5 | **5/5 (100%)** | 3/5 (60%) | +40 pts |
| `aws-iam-policy-json` | 5 | **5/5 (100%)** | 4/5 (80%) | +20 pts |
| `azure-readonly-mode` | 4 | **4/4 (100%)** | 4/4 (100%) | 0 |
| **Total** | **14** | **14/14 (100%)** | **11/14 (79%)** | **+21 pts** |

## Costo (tokens / tiempo por caso)

| Caso | Con skill | Baseline |
|------|-----------|----------|
| gcp-audit-and-fix | 36.9k tok · 52s | 29.5k tok · 40s |
| aws-iam-policy-json | 45.1k tok · 80s | 32.2k tok · 71s |
| azure-readonly-mode | 40.1k tok · 88s | 31.6k tok · 70s |

## Qué mide cada caso

- **gcp-audit-and-fix** — ¿pide modo de trabajo? ¿hace preflight de CLIs? ¿pide alcance/remediación?
  ¿declara gate de aprobación? ¿plan CIS? El baseline omite intake y preflight.
- **aws-iam-policy-json** — ¿detecta el combo de escalada `iam:PassRole`+`lambda:CreateFunction`?
  ¿wildcard `ec2:*`? ¿exfiltración S3/Secrets? ¿remediación de mínimo privilegio? ¿usa el script?
- **azure-readonly-mode** — ¿respeta el "no toques nada"? ¿separa read-only de cambios? ¿cubre
  dominios CIS? Ambas configuraciones lo hacen bien (caso no discriminante; se mantiene como control).

## Observaciones del análisis

- La ventaja de la skill es **disciplina de proceso** (intake + preflight + plan), no solo detección.
- El caso Azure no discrimina (ambos 100%): un modelo fuerte ya respeta "no toques nada". Se conserva
  como control de regresión.
- Mejora detectada para iteration-2: en JSON de IAM puro usar `--check iam` (no `--check all`) para
  evitar 3 falsos positivos de S3 del analizador.
