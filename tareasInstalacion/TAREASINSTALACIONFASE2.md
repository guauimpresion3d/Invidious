# Fase 2 — Configuración

> Objetivo: crear y configurar el archivo `docker-compose.yml` con todos los valores correctos antes de levantar ningún contenedor.
>
> Prerequisito: Fase 1 completada y todos sus checks pasados.

---

## Tareas

### T2.1 — Crear el archivo `docker-compose.yml`
- [x] Reemplazado el `docker-compose.yml` de desarrollo del repo por configuración de producción con los tres servicios: `invidious`, `companion` e `invidious-db`

### T2.2 — Configurar el servicio `invidious-db`
- [x] Imagen `docker.io/library/postgres:14`
- [x] Volumen `postgresdata:/var/lib/postgresql/data`
- [x] Bind mount `./config/sql:/config/sql`
- [x] Bind mount `./docker/init-invidious-db.sh:/docker-entrypoint-initdb.d/init-invidious-db.sh`
- [x] Variables: `POSTGRES_DB: invidious`, `POSTGRES_USER: kemal`, `POSTGRES_PASSWORD: kemal`
- [x] Healthcheck con `pg_isready` (interval 10s, retries 5)
- [x] `restart: unless-stopped`

### T2.3 — Configurar el servicio `companion`
- [x] Imagen `quay.io/invidious/invidious-companion:latest`
- [x] `SERVER_SECRET_KEY: "88T54O60MTOsTakfJ1Ga"` (COMPANION_KEY de Fase 1)
- [x] Volumen `companion_cache:/cache`
- [x] Healthcheck apuntando a `http://127.0.0.1:8282/healthz`
- [x] `restart: unless-stopped`

### T2.4 — Configurar el servicio `invidious`
- [x] Imagen `quay.io/invidious/invidious:latest`
- [x] Puerto `127.0.0.1:3000:3000` (solo local)
- [x] Bloque `db` apuntando a `invidious-db:5432`
- [x] `check_tables: true`
- [x] `hmac_key: "dSZmixGBc3YDHSVwJeIO"` (HMAC_KEY de Fase 1)
- [x] `private_url: "http://companion:8282/companion"` y `invidious_companion_key: "88T54O60MTOsTakfJ1Ga"`
- [x] `depends_on: invidious-db: condition: service_healthy`
- [x] Healthcheck apuntando a `/api/v1/comments/jNQXAC9IVRw`
- [x] `restart: unless-stopped`

### T2.5 — Declarar los volúmenes nombrados
- [x] `postgresdata` y `companion_cache` declarados en sección `volumes:`

### T2.6 — Revisión final del archivo
- [x] Sin placeholders `REEMPLAZAR_CON_` en el archivo
- [x] `hmac_key` y `invidious_companion_key` con valores reales y distintos entre sí
- [x] `SERVER_SECRET_KEY` (companion) e `invidious_companion_key` (invidious) tienen el **mismo valor**: `88T54O60MTOsTakfJ1Ga`
- [x] Atributo obsoleto `version: "3"` eliminado (generaba warning en Docker Compose v5)

---

## Stack de Pruebas

Ejecutar estos checks sobre el archivo antes de levantar los contenedores.

| # | Comando | Resultado esperado |
|---|---|---|
| P2.1 | `ls docker-compose.yml` | El archivo existe en el directorio `invidious/` | ✅ Existe |
| P2.2 | `docker compose config` | Sin errores de YAML; imprime la configuración expandida | ✅ Sin errores ni warnings |
| P2.3 | `grep "REEMPLAZAR_CON_" docker-compose.yml` | Sin salida (ningún placeholder quedó sin reemplazar) | ✅ Sin salida |
| P2.4 | `grep "hmac_key" docker-compose.yml` | Muestra una línea con un valor real (no vacío, no placeholder) | ✅ `hmac_key: "dSZmixGBc3YDHSVwJeIO"` |
| P2.5 | `grep "SERVER_SECRET_KEY" docker-compose.yml` | Muestra una línea con un valor real | ✅ `SERVER_SECRET_KEY: "88T54O60MTOsTakfJ1Ga"` |
| P2.6 | `grep "invidious_companion_key" docker-compose.yml` | Muestra una línea con un valor real | ✅ `invidious_companion_key: "88T54O60MTOsTakfJ1Ga"` |
| P2.7 | `docker compose config --services` | Lista exactamente: `invidious`, `companion`, `invidious-db` | ✅ Los tres servicios presentes |
| P2.8 | `grep "127.0.0.1:3000" docker-compose.yml` | Confirma que el puerto está ligado solo a localhost | ✅ `"127.0.0.1:3000:3000"` |

### Criterio de paso
✅ **FASE 2 COMPLETADA** — `docker compose config` sin errores, sin placeholders, los tres servicios configurados correctamente. Listo para continuar a la Fase 3.
