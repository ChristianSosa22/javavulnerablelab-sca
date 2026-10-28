# Remediación: Log4Shell en log4j 2.12.1

## Vulnerabilidad elegida

`org.apache.logging.log4j:log4j-core:2.12.1` (dependencia directa) tiene
**CVE-2021-44228 (Log4Shell, CVSS 10.0)**: un mensaje de log con
`${jndi:ldap://atacante/x}` hace que log4j descargue y ejecute código remoto.
La aplicación la usa de verdad: `vulnerability/log4shell/Search.jsp` registra
la búsqueda del usuario con log4j.

## Cambio

```diff
-  <version>2.12.1</version>   <!-- log4j-api -->
-  <version>2.12.1</version>   <!-- log4j-core -->
+  <version>2.12.4</version>
+  <version>2.12.4</version>
```

### ¿Por qué 2.12.4 y no 2.25.x?

El proyecto compila para Java 7 (`maven.compiler.target 1.7`) y se despliega
en `tomcat:7.0.82`. Desde la versión 2.13, log4j exige Java 8. **2.12.4** es
la última versión de la rama para Java 7 y corrige:

| CVE | CVSS | Estado con 2.12.4 |
| --- | ---- | ----------------- |
| CVE-2021-44228 (Log4Shell) | 10.0 | Corregido |
| CVE-2021-45046 | 9.0 | Corregido |
| CVE-2021-45105 | 5.9 | Corregido |
| CVE-2021-44832 | 6.6 | Corregido |
| CVE-2020-9488 | 3.7 | Corregido |
| CVE-2025-68161, CVE-2026-34477, CVE-2026-34479, CVE-2026-34480 | 4.8–7.5 | **Sin corrección para Java 7** (requieren 2.25.4+, que exige Java 8) |

Las cuatro restantes son un ejemplo real de la pregunta 23: la rama 2.12 ya no
recibe parches, así que el riesgo se gestiona con mitigaciones (no usar
`XmlLayout`, `Log4j1XmlLayout` ni el Socket Appender con TLS) hasta migrar la
aplicación a Java 8 y log4j 2.25.4 o superior.

## Cómo lo refleja el pipeline

- El PR dispara el workflow (`pull_request` hacia `main`).
- Después del merge, el workflow corre en `main` y sube un SARIF nuevo con la
  categoría `dependency-check`.
- En **Security > Code scanning**, las alertas de log4j que ya no aparecen en
  el SARIF (CVE-2021-44228, CVE-2021-45046, CVE-2021-45105, CVE-2021-44832 y
  CVE-2020-9488) pasan a **Closed / Fixed**.
- El job sigue en rojo porque quedan otras dependencias con CVSS ≥ 7
  (`commons-collections`, `dom4j`, `mysql-connector-java`, etc.); remediarlas
  no es parte de este cambio.

## Verificación local

`docker compose -f docker-compose.sca.yml run --rm dependency-check` con el
cambio aplicado:

| | Antes (2.12.1) | Después (2.12.4) |
| --- | --- | --- |
| Vulnerabilidades totales | 31 | **25** |
| log4j-core | 9 | 4 (CVE-2025-68161, CVE-2026-34477, CVE-2026-34479, CVE-2026-34480) |
| log4j-api | 3 | 2 (CVE-2026-34477, CVE-2026-34479) |
| CVE-2021-44228 (Log4Shell) | Sí | **No** |

## Resultado en Code scanning

| | Antes | Después |
| --- | --- | --- |
| Alertas abiertas | 31 | **25** |
| Alertas cerradas (*Fixed*) | 0 | **12** |

GitHub identifica cada alerta por CVE **y ubicación** (el JAR). Se cierran las
12 alertas de `log4j-*-2.12.1.jar`, incluida **CVE-2021-44228**, y se abren 6
en `log4j-*-2.12.4.jar` para los 4 CVE sin corrección en Java 7:
31 − 12 + 6 = 25.
