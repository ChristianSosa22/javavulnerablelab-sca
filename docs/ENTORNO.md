# Entorno de desarrollo con Docker

Todo el laboratorio corre con Docker Desktop. No hace falta instalar Java 17,
Maven, Dependency-Track ni DefectDojo en la máquina: todo está definido en
[`docker-compose.sca.yml`](../docker-compose.sca.yml).

| Servicio            | Perfil       | Para qué sirve                                  | URL                     |
| ------------------- | ------------ | ----------------------------------------------- | ----------------------- |
| `maven`             | `tools`      | Shell con Java 17 + Maven 3.9                   | —                       |
| `dependency-check`  | `tools`      | Parte B: Dependency-Check 13.0.0 (HTML/XML/JSON/SARIF) | —                |
| `sbom`              | `tools`      | Parte C: SBOM CycloneDX JSON (`target/bom.json`) | —                      |
| `dtrack`            | `dtrack`     | Dependency-Track 4.14.6 (frontend + API)        | http://localhost:8080   |
| `nginx` + `uwsgi`…  | `defectdojo` | DefectDojo 3.4.300                              | http://localhost:8082   |

Los puertos solo se publican en `127.0.0.1`, así nada queda expuesto a la red.

## 1. Clave del NVD

Crea un archivo `.env` en la raíz del repositorio (ya está en `.gitignore`):

```
NVD_API_KEY=tu-clave
```

Compose lo pasa como variable de entorno al contenedor y Dependency-Check la
lee con `-DnvdApiKeyEnvironmentVariable=NVD_API_KEY`, así la clave nunca
aparece en la línea de comandos ni en el repositorio.

## 2. Verificar las herramientas

```powershell
docker --version
docker compose -f docker-compose.sca.yml run --rm maven mvn -version
docker compose -f docker-compose.sca.yml run --rm maven java -version
```

## 3. Comandos del laboratorio

```powershell
# Parte B: Dependency-Check (la primera vez descarga el NVD, 10-30 min)
docker compose -f docker-compose.sca.yml run --rm dependency-check

# Parte C: SBOM CycloneDX
docker compose -f docker-compose.sca.yml run --rm sbom

# Parte C: Dependency-Track (usuario admin / admin, pide cambiarla al entrar)
docker compose -f docker-compose.sca.yml --profile dtrack up -d

# Parte D: DefectDojo (usuario admin / SCA-lab-2026!)
docker compose -f docker-compose.sca.yml --profile defectdojo up -d

# Apagar todo (los datos quedan en volúmenes)
docker compose -f docker-compose.sca.yml --profile dtrack --profile defectdojo down
```

Los reportes quedan en `target/` (por ejemplo `target/dependency-check-report.html`
y `target/bom.json`), que también está en `.gitignore`.

La base del NVD se guarda en el volumen `sca-lab_m2`, así que las siguientes
ejecuciones solo descargan las actualizaciones. Para borrarla:
`docker volume rm sca-lab_m2`.

## Requisitos de memoria

Docker Desktop necesita unos 6 GB de RAM para Dependency-Track y DefectDojo
juntos. Si no alcanza, levanta uno, toma las capturas y apágalo antes de
levantar el otro.
