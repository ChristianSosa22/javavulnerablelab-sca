# Parte B — Dependency-Check en local

## Comando

Con Maven instalado (comando del enunciado):

```powershell
$env:NVD_API_KEY = "tu-clave"
mvn -B org.owasp:dependency-check-maven:13.0.0:check `
  "-Dformat=ALL" "-DnvdApiKey=$env:NVD_API_KEY" "-DossindexAnalyzerEnabled=false"
```

Con el entorno Docker del repositorio (la clave se lee de `.env`):

```powershell
docker compose -f docker-compose.sca.yml run --rm dependency-check
```

`-Dformat=ALL` genera en `target/` los formatos HTML, XML, JSON y SARIF (además
de CSV, JUnit, Jenkins y GitLab). El reporte HTML se abre en el navegador desde
`target\dependency-check-report.html`. Copia entregable:
[`entregables/dependency-check-report.html`](../entregables/dependency-check-report.html).

> **Nota sobre la primera ejecución.** La API del NVD respondía con errores y
> reintentos (`Retrying request ... 7th time`), así que la carga inicial se hizo
> con los *data feeds* JSON 2.0 oficiales del NVD
> (`-DnvdDatafeedUrl=https://nvd.nist.gov/feeds/json/cve/2.0/nvdcve-2.0-{0}.json.gz`).
> Las ejecuciones siguientes usan la API con la clave y solo descargan cambios
> (11 segundos en total).

## Resultado

- **19** dependencias analizadas (18 JAR del WAR + `jquery.min.js` del webapp).
- **9** dependencias vulnerables.
- **31** vulnerabilidades.

| Severidad (CVSS v3) | Cantidad |
| ------------------- | -------- |
| Critical (9.0–10)   | 4  |
| High (7.0–8.9)      | 7  |
| Medium (4.0–6.9)    | 16 |
| Low (< 4.0)         | 4  |

## Tabla de vulnerabilidades

| # | Dependencia | Tipo | CVE | CVSS | Descripción |
| - | ----------- | ---- | --- | ---- | ----------- |
| 1 | log4j-core 2.12.1 | Directa | **CVE-2021-44228** | 10.0 | Log4Shell: RCE por *lookups* JNDI en los mensajes de log |
| 2 | commons-collections 3.1 | Directa y transitiva (hibernate) | CVE-2015-6420 | 9.8 | Deserialización insegura (`InvokerTransformer`) |
| 3 | dom4j 1.6.1 | **Transitiva** (hibernate-core) | CVE-2020-10683 | 9.8 | XXE: `SAXReader` procesa entidades externas |
| 4 | log4j-core 2.12.1 | Directa | CVE-2021-45046 | 9.0 | Corrección incompleta de Log4Shell (RCE en ciertas configuraciones) |
| 5 | mysql-connector-java 5.1.26 | Directa | CVE-2023-22102 | 8.3 | Toma de control de Connector/J |
| 6 | commons-fileupload 1.3.3 | Directa | CVE-2023-24998 | 7.5 | DoS por número ilimitado de partes en la petición |
| 7 | commons-fileupload 1.3.3 | Directa | CVE-2025-48976 | 7.5 | DoS por cabeceras multipart sin límite |
| 8 | log4j-core 2.12.1 | Directa | CVE-2026-34479 | 7.5 | `Log4j1XmlLayout` no escapa caracteres |
| 9 | log4j-api 2.12.1 | Directa | CVE-2026-34479 | 7.5 | Mismo CVE asociado al JAR de la API (posible falso positivo) |
| 10 | log4j-core 2.12.1 | Directa | CVE-2026-34480 | 7.5 | `XmlLayout` no escapa caracteres |
| 11 | hibernate-core 4.0.1.Final | Directa | CVE-2020-25638 | 7.4 | Inyección SQL con comentarios SQL habilitados |
| 12 | log4j-core 2.12.1 | Directa | CVE-2021-44832 | 6.6 | RCE vía JDBC Appender con configuración controlada |
| 13 | hibernate-core 4.0.1.Final | Directa | CVE-2019-14900 | 6.5 | Inyección SQL en la API Criteria |
| 14 | mysql-connector-java 5.1.26 | Directa | CVE-2019-2692 | 6.3 | Toma de control de Connector/J |
| 15–19 | jquery.min.js | Archivo JS en `src/main/webapp` | CVE-2012-6708, CVE-2015-9251, CVE-2019-11358, CVE-2020-11023, CVE-2020-7656 | 6.1 | XSS y *prototype pollution* en jQuery |
| 20 | log4j-core 2.12.1 | Directa | CVE-2026-34477 | 5.9 | Corrección incompleta de CVE-2025-68161 |
| 21 | log4j-api 2.12.1 | Directa | CVE-2026-34477 | 5.9 | Mismo CVE asociado al JAR de la API |
| 22 | log4j-core 2.12.1 | Directa | CVE-2021-45105 | 5.9 | DoS por recursión infinita en *lookups* |
| 23 | mysql-connector-java 5.1.26 | Directa | CVE-2020-2934 | 5.0 | Acceso no autorizado en Connector/J |
| 24 | commons-io 2.6 | Directa y transitiva (fileupload) | CVE-2021-29425 | 4.8 | *Path traversal* en `FileNameUtils.normalize` |
| 25 | log4j-core 2.12.1 | Directa | CVE-2025-68161 | 4.8 | El Socket Appender no verifica el nombre del host TLS |
| 26 | mysql-connector-java 5.1.26 | Directa | CVE-2020-2875 | 4.7 | Acceso no autorizado en Connector/J |
| 27 | commons-io 2.6 | Directa y transitiva (fileupload) | CVE-2024-47554 | 4.3 | DoS en `XmlStreamReader` |
| 28 | log4j-core 2.12.1 | Directa | CVE-2020-9488 | 3.7 | Validación incorrecta del certificado en el SMTP Appender |
| 29 | log4j-api 2.12.1 | Directa | CVE-2020-9488 | 3.7 | Mismo CVE asociado al JAR de la API |
| 30 | mysql-connector-java 5.1.26 | Directa | CVE-2020-2933 | 2.2 | DoS parcial en Connector/J |
| 31 | jquery.min.js | Archivo JS | jquery issue 162 | — | Problema conocido de jQuery 1.x (RetireJS) |

Dependencias **sin** hallazgos: antlr 2.7.7, hibernate-commons-annotations,
hibernate-jpa-2.0-api, javassist, jboss-logging, jboss-transaction-api,
**json 20090211**, jstl 1.2, servlet-api 2.3, xml-apis.

Observaciones:

- `jquery.min.js` no es una dependencia Maven: lo detecta el analizador de
  RetireJS al revisar los archivos del proyecto. Un SBOM de Maven no lo incluye.
- Dependency-Check asocia varios CVE de `log4j-core` también a `log4j-api`
  porque comparten el CPE `apache:log4j`; son falsos positivos probables.
- `json 20090211` aparece sin vulnerabilidades: su CPE no coincide con los de
  CVE-2022-45688 y CVE-2023-5072 (falso negativo; ver la comparación en la Parte D).
