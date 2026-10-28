# Parte C — SBOM y Dependency-Track

## 11. Generar el SBOM (CycloneDX JSON)

```powershell
docker compose -f docker-compose.sca.yml run --rm sbom
# equivale a:
# mvn -B org.cyclonedx:cyclonedx-maven-plugin:2.9.3:makeAggregateBom -DoutputFormat=json -DoutputName=bom
```

Resultado: `target/bom.json` (CycloneDX 1.6). Copia entregable en
[`entregables/bom.json`](../entregables/bom.json).

## 12. Componentes del SBOM

El SBOM tiene **18 componentes** (más el componente raíz
`org.cysecurity:JavaVulnerableLab`). `junit 3.8.1` no aparece porque es de
alcance `test` y el plugin lo excluye por defecto.

```powershell
docker compose -f docker-compose.sca.yml run --rm maven `
  sh -c "apt-get -qq update >/dev/null && apt-get -qq install -y jq >/dev/null && jq '.components | length' target/bom.json"
# 18
```

| # | Componente | Versión | Tipo |
| - | ---------- | ------- | ---- |
| 1 | mysql:mysql-connector-java | 5.1.26 | Directa |
| 2 | org.json:json | 20090211 | Directa |
| 3 | javax.servlet:jstl | 1.2 | Directa |
| 4 | org.hibernate:hibernate-core | 4.0.1.Final | Directa |
| 5 | javax.servlet:servlet-api | 2.3 | Directa (`provided`) |
| 6 | org.apache.logging.log4j:log4j-api | 2.12.1 | Directa |
| 7 | org.apache.logging.log4j:log4j-core | 2.12.1 | Directa |
| 8 | commons-collections:commons-collections | 3.1 | Directa |
| 9 | commons-fileupload:commons-fileupload | 1.3.3 | Directa |
| 10 | commons-io:commons-io | 2.6 | Directa |
| 11 | antlr:antlr | 2.7.7 | Transitiva (hibernate-core) |
| 12 | org.jboss.spec.javax.transaction:jboss-transaction-api_1.1_spec | 1.0.0.Final | Transitiva (hibernate-core) |
| 13 | dom4j:dom4j | 1.6.1 | Transitiva (hibernate-core) |
| 14 | xml-apis:xml-apis | 1.0.b2 | Transitiva (hibernate-core → dom4j) |
| 15 | org.hibernate.javax.persistence:hibernate-jpa-2.0-api | 1.0.1.Final | Transitiva (hibernate-core) |
| 16 | org.jboss.logging:jboss-logging | 3.1.0.CR2 | Transitiva (hibernate-core) |
| 17 | org.javassist:javassist | 3.15.0-GA | Transitiva (hibernate-core) |
| 18 | org.hibernate.common:hibernate-commons-annotations | 4.0.1.Final | Transitiva (hibernate-core) |

## 13. Proyecto en Dependency-Track

```powershell
docker compose -f docker-compose.sca.yml --profile dtrack up -d   # http://localhost:8080
```

1. Entrar con `admin` / `admin` (pide cambiar la contraseña).
2. **Administration > Vulnerability Sources > Google OSV Advisories**: activar
   el ecosistema **Maven** (sin esto el proyecto puede quedar en 0
   vulnerabilidades, porque los componentes CycloneDX se identifican por PURL).
3. **Projects > Create Project**: nombre `JavaVulnerableLab`, versión `1.0`,
   clasificador *Application*.
4. En el proyecto, **Components > Upload BOM** y elegir `target/bom.json`.
5. Si aún no hay hallazgos, usar **Reanalyze** (en *Audit Vulnerabilities*).

## 14. Resultados (Overview, Components, Audit Vulnerabilities)

**29 vulnerabilidades** en 8 de los 18 componentes:

| Severidad | Cantidad |
| --------- | -------- |
| Critical  | 4  |
| High      | 12 |
| Medium    | 11 |
| Low       | 2  |

| Componente | Versión | Vulnerabilidades (CVE) |
| ---------- | ------- | ---------------------- |
| log4j-core | 2.12.1 | **CVE-2021-44228 (Log4Shell, Critical)**, CVE-2021-45046 (Critical), CVE-2021-45105 (High), CVE-2021-44832, CVE-2025-68161, CVE-2026-34477, CVE-2026-34479, CVE-2026-34480 (Medium), CVE-2020-9488 (Low) |
| commons-collections | 3.1 | CVE-2015-7501 (Critical), CVE-2015-6420 (High) |
| dom4j *(transitiva)* | 1.6.1 | CVE-2020-10683 (Critical), CVE-2018-1000632 (High) |
| mysql-connector-java | 5.1.26 | CVE-2017-3523, CVE-2018-3258, CVE-2023-22102 (High), CVE-2015-2575, CVE-2017-3586, CVE-2019-2692, CVE-2022-21363 (Medium), CVE-2017-3589 (Low) |
| commons-fileupload | 1.3.3 | CVE-2023-24998, CVE-2025-48976 (High) |
| commons-io | 2.6 | CVE-2024-47554 (High), CVE-2021-29425 (Medium) |
| json | 20090211 | CVE-2022-45688, CVE-2023-5072 (High) |
| hibernate-core | 4.0.1.Final | CVE-2020-25638 (High), CVE-2019-14900 (Medium) |

Todas llegan por OSV (identificadores GHSA con su alias CVE).

## 15. Dependency Graph: origen de una dependencia transitiva vulnerable

En la pestaña **Dependency Graph** se ve:

```
JavaVulnerableLab 1.0
└── org.hibernate:hibernate-core 4.0.1.Final        (directa, declarada en pom.xml)
    ├── dom4j:dom4j 1.6.1                          (transitiva, CRITICAL)
    │   └── xml-apis:xml-apis 1.0.b2
    ├── commons-collections:commons-collections 3.1  (también directa, CRITICAL)
    ├── antlr:antlr 2.7.7
    ├── org.javassist:javassist 3.15.0-GA
    ├── org.jboss.logging:jboss-logging 3.1.0.CR2
    ├── org.hibernate.javax.persistence:hibernate-jpa-2.0-api 1.0.1.Final
    ├── org.jboss.spec.javax.transaction:jboss-transaction-api_1.1_spec 1.0.0.Final
    └── org.hibernate.common:hibernate-commons-annotations 4.0.1.Final
```

Otro caso interesante: `commons-collections 3.1` está declarada en el
`pom.xml`, pero Hibernate también la pide. Si se quitara la declaración
directa, la librería vulnerable seguiría en el WAR por el camino transitivo.
Lo mismo pasa con `commons-io 2.6`, que también llega por
`commons-fileupload 1.3.3`.

`dom4j 1.6.1` **no está en el `pom.xml`**: llega porque `hibernate-core
4.0.1.Final` la declara como dependencia de compilación. Tiene
**CVE-2020-10683** (CVSS 9.8, XXE: `SAXReader` procesa entidades externas por
defecto) y **CVE-2018-1000632** (inyección de elementos y atributos XML). Para
corregirla hay que actualizar Hibernate a una versión que traiga
`org.dom4j:dom4j 2.1.3` o posterior, o forzar esa versión con
`<dependencyManagement>` y verificar que Hibernate siga funcionando.
