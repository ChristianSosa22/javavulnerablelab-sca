# Parte D — DefectDojo y comparación con Dependency-Track

## 16–17. Producto e importación

```powershell
docker compose -f docker-compose.sca.yml --profile defectdojo up -d   # http://localhost:8082
```

1. **Products > Add Product**: nombre `JavaVulnerableLab`, tipo *Research and Development*.
2. En el producto: **Engagements > Add New Interactive Engagement**
   (`SCA local - Dependency-Check 13.0.0`).
3. **Import Scan Results**: tipo **Dependency Check Scan**, archivo
   `target\dependency-check-report.xml`.

Resultado de la prueba importada: **31 hallazgos activos**.

| Severidad | DefectDojo |
| --------- | ---------- |
| Critical  | 4  |
| High      | 7  |
| Medium    | 16 |
| Low       | 4  |

Coincide 1 a 1 con el reporte HTML de Dependency-Check (Parte B).

## 18. Comparación Dependency-Check (DefectDojo) vs Dependency-Track

| | Dependency-Check / DefectDojo | Dependency-Track |
| --- | --- | --- |
| Entrada | JAR resueltos + archivos del proyecto | SBOM CycloneDX (`bom.json`) |
| Identificación | CPE inferido por evidencia | PURL exacto (`pkg:maven/...`) |
| Fuentes | NVD, CISA KEV, RetireJS | NVD + Google OSV (GitHub Advisories) |
| Componentes vulnerables | 9 | 8 |
| Hallazgos | 31 | 29 |

### ¿Coinciden las dependencias vulnerables?

En su mayoría sí. **7 componentes** aparecen como vulnerables en ambas:
`log4j-core 2.12.1`, `commons-collections 3.1`, `dom4j 1.6.1`,
`mysql-connector-java 5.1.26`, `commons-fileupload 1.3.3`, `commons-io 2.6` y
`hibernate-core 4.0.1.Final`. Las dos herramientas detectan Log4Shell
(CVE-2021-44228) y las otras críticas de log4j, `dom4j` y `commons-collections`.
A nivel de pares *componente + CVE* hay **19 coincidencias**.

| Componente | En ambas | Solo Dependency-Check | Solo Dependency-Track |
| ---------- | -------- | --------------------- | --------------------- |
| log4j-core 2.12.1 | 9 (los mismos 9 CVE) | — | — |
| log4j-api 2.12.1 | — | CVE-2026-34479, CVE-2026-34477, CVE-2020-9488 | — |
| commons-collections 3.1 | CVE-2015-6420 | — | **CVE-2015-7501** (Critical) |
| dom4j 1.6.1 | CVE-2020-10683 | — | CVE-2018-1000632 |
| json 20090211 | — | — | CVE-2022-45688, CVE-2023-5072 |
| mysql-connector-java 5.1.26 | CVE-2023-22102, CVE-2019-2692 | CVE-2020-2934, CVE-2020-2875, CVE-2020-2933 | CVE-2017-3523, CVE-2018-3258, CVE-2017-3586, CVE-2017-3589, CVE-2015-2575, CVE-2022-21363 |
| commons-fileupload 1.3.3 | 2 | — | — |
| commons-io 2.6 | 2 | — | — |
| hibernate-core 4.0.1.Final | 2 | — | — |
| jquery.min.js | — | 6 (RetireJS) | — |
| **Total** | **19** | **12** | **10** |

### ¿Alguna herramienta encontró vulnerabilidades que la otra no?

Sí, en los dos sentidos:

- **Solo Dependency-Check (12):**
  - `jquery.min.js` (6): está copiado en `src/main/webapp`, no es una
    dependencia Maven, así que no aparece en el SBOM. Dependency-Check lo
    encuentra porque también analiza archivos JavaScript con RetireJS.
  - `log4j-api` (3): Dependency-Check asigna al JAR de la API el mismo CPE
    `cpe:/a:apache:log4j` que al de core, así que hereda CVE que en realidad
    afectan a `log4j-core`. Son falsos positivos probables. OSV usa el PURL
    exacto y no los reporta.
  - `mysql-connector-java` (3): CVE de los boletines de Oracle de 2020 cuyo CPE
    cubre la versión, pero que el aviso de GitHub/OSV no lista para 5.1.26.
- **Solo Dependency-Track (10):**
  - `json 20090211` (2): el CPE del NVD para estos CVE (`json-java`) no coincide
    con la evidencia que Dependency-Check extrae del JAR, así que lo da por
    limpio (falso negativo). OSV lo identifica por `pkg:maven/org.json/json`.
  - `commons-collections` CVE-2015-7501 y `dom4j` CVE-2018-1000632: en el NVD
    tienen configuraciones de CPE que no incluyen estas versiones o
    productos, mientras que el aviso GHSA sí define el rango Maven afectado.
  - `mysql-connector-java` (6): avisos GHSA con rangos de versión más amplios
    que los CPE del NVD.

### ¿Por qué ocurre?

1. **Identificación distinta**: CPE inferido por evidencia (Dependency-Check)
   frente a PURL exacto del SBOM (Dependency-Track). El CPE produce falsos
   positivos (`log4j-api`) y falsos negativos (`json`).
2. **Fuentes distintas**: NVD frente a NVD + OSV/GitHub Advisories. Cada base
   tiene su propia cobertura y sus rangos de versiones.
3. **Alcance distinto**: Dependency-Check analiza todo el proyecto (incluidos
   archivos JS); el SBOM de Maven solo describe dependencias Maven.
4. **Momento de actualización**: cada herramienta tiene su copia local de las
   bases y las actualiza en momentos distintos.

Por eso conviene usar ambas: una sola herramienta habría dejado sin detectar
vulnerabilidades críticas (CVE-2015-7501 en Dependency-Check o jQuery en
Dependency-Track).
