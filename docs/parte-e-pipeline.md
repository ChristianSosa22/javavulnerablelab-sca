# Parte E — Pipeline de GitHub Actions con Dependency-Check

Archivo: [`.github/workflows/sca-dependency-check.yml`](../.github/workflows/sca-dependency-check.yml)

## Diagrama

```
push a main ─┐
PR hacia main ├─► job dependency-check ─────────────────────────────────────────┐
manual ───────┘    checkout → Java 17 → restaurar caché NVD → Dependency-Check   │
                   → guardar caché NVD (always) → SARIF a Code scanning (always) │
                   → artefacto dependency-check-reports (always)                 │
               └─► job sbom (extra) ─────────────────────────────────────────────┤
                   checkout → Java 17 → CycloneDX makeAggregateBom               │
                   → artefacto sbom-cyclonedx                                    ┘
```

## Cumplimiento de los requisitos

| # | Requisito | Cómo se cumple |
| - | --------- | -------------- |
| 1 | push a main, PR hacia main y manual | `on: push/pull_request` con `branches: [main]` y `workflow_dispatch` |
| 2 | Permisos mínimos | `permissions: contents: read` y `security-events: write` a nivel de workflow; el job `sbom` baja a solo `contents: read` |
| 3 | Java 17 | `actions/setup-java@v6` con `distribution: temurin` y `java-version: "17"` |
| 4 | Clave del NVD como secreto | Secreto de repositorio `NVD_API_KEY`; el YAML solo usa `${{ secrets.NVD_API_KEY }}` como variable de entorno del paso y el plugin la lee con `-DnvdApiKeyEnvironmentVariable=NVD_API_KEY`. GitHub enmascara el valor en los logs |
| 5 | Caché de la base del NVD | `actions/cache/restore@v6` y `actions/cache/save@v6` sobre `~/.m2/repository/org/owasp/dependency-check-data` (ver abajo) |
| 6 | HTML, XML y SARIF | `-Dformats=HTML,XML,JSON,SARIF` |
| 7 | SARIF en Code scanning con categoría `dependency-check` | `github/codeql-action/upload-sarif@v4` con `category: dependency-check` e `if: always()` |
| 8 | HTML y XML como artefactos | `actions/upload-artifact@v7`, artefacto `dependency-check-reports` con `if: always()` |
| 9 | Cada línea comentada | Todas las líneas del YAML, incluidas las del script de shell, llevan comentario |
| Extra | Job de SBOM | Job `sbom` que ejecuta `makeAggregateBom` y sube `target/bom.json` como artefacto `sbom-cyclonedx` |

## Decisiones de diseño

**Caché del NVD (requisito 5).** La acción `actions/cache` "normal" solo guarda
la caché si el job termina bien. Como el análisis falla a propósito cuando hay
vulnerabilidades con CVSS ≥ 7, la caché nunca se guardaría. Por eso se separa
en dos pasos:

- `actions/cache/restore` con clave `nvd-<versión>-<fecha>` y
  `restore-keys: nvd-<versión>-`: si no hay caché de hoy, restaura la más
  reciente y Dependency-Check solo descarga los cambios desde esa fecha (unos
  segundos en lugar de 10 a 30 minutos).
- `actions/cache/save` con `if: always()`, que guarda la base actualizada
  aunque el análisis haya fallado, una vez por día.

La versión del plugin forma parte de la clave porque un cambio de versión
puede cambiar el esquema de la base H2.

**Carga inicial sin caché.** Al probar en local, la API del NVD respondía con
errores y reintentos, y la descarga completa por la API avanzaba al 12 % en 35
minutos. Por eso, solo cuando no se restauró ninguna caché
(`cache-matched-key` vacío), el job agrega `-DnvdDatafeedUrl` y hace la carga
inicial desde los *data feeds* JSON 2.0 oficiales del NVD. En las ejecuciones
siguientes la caché existe y Dependency-Check actualiza por la API con la clave
`NVD_API_KEY`, descargando solo los cambios.

**Puerta de seguridad (`failBuildOnCVSS=7`).** El job falla si alguna
dependencia tiene una vulnerabilidad *High* o *Critical*. Así un PR que
introduce una librería vulnerable queda marcado en rojo. Los pasos de
publicación usan `if: always()` (como en el pipeline de Semgrep), de modo que
el SARIF y los reportes se publican aunque el análisis falle. En este proyecto,
deliberadamente vulnerable, el job queda en rojo incluso después de la
remediación, porque siguen existiendo otras dependencias con CVSS ≥ 7
(`commons-collections`, `dom4j`, `mysql-connector-java`, etc.).

**Concurrencia.** `concurrency` con `cancel-in-progress` cancela la ejecución
anterior de la misma rama cuando llega un commit nuevo.

**OSS Index desactivado.** Desde 2025 Sonatype OSS Index exige credenciales;
sin ellas el analizador falla, así que se desactiva como en el comando de la
Parte B.

## Configurar el secreto

**Settings > Secrets and variables > Actions > New repository secret**, nombre
`NVD_API_KEY`. Desde la terminal:

```powershell
gh secret set NVD_API_KEY   # pide el valor sin mostrarlo
```
