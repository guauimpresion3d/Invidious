# Fase 3 — Despliegue y Verificación

> Objetivo: levantar los tres contenedores y confirmar que Invidious responde correctamente en `http://localhost:3000`.
>
> Prerequisito: Fase 2 completada y todos sus checks pasados.

---

## Tareas

### T3.1 — Descargar las imágenes Docker
- [x] `docker compose pull` ejecutado — las tres imágenes descargadas correctamente
  - `quay.io/invidious/invidious:latest` (109 MB)
  - `quay.io/invidious/invidious-companion:latest` (202 MB)
  - `docker.io/library/postgres:14` (628 MB)

### T3.2 — Levantar los servicios en modo background
- [x] `docker compose up -d` finaliza sin errores
- [x] **Problemas resueltos durante el despliegue:**
  - `init-invidious-db.sh` tenía CRLF (Windows) — convertido a LF con `sed`
  - Puerto 3000 ocupado por `open-webui` — Invidious reasignado al puerto `3001`
  - Claves de 20 chars regeneradas a exactamente 16 chars (requerimiento del companion)
  - `invidious_companion_key` estaba anidada incorrectamente — movida al nivel raíz del config
  - Healthcheck del companion usa `wget` que no existe en el contenedor mínimo — deshabilitado (`disable: true`); funcionalidad verificada desde red interna

### T3.3 — Verificar estado de los contenedores
- [x] Los tres servicios están `Up`
  - `invidious-db`: `healthy`
  - `invidious`: `healthy`
  - `companion`: `Up` (healthcheck deshabilitado, funcional verificado vía red interna)

### T3.4 — Revisar logs de arranque
- [x] `invidious-db`: sin errores SQL, inicialización completada correctamente
- [x] `invidious`: sin errores de conexión a DB ni de configuración

### T3.5 — Verificar respuesta HTTP de Invidious
- [x] `http://localhost:3001` → `302` (redirect a feed, comportamiento correcto)
- [x] `http://localhost:3001/api/v1/stats` → `200`

### T3.6 — Verificar respuesta del companion
- [x] `docker exec invidious-invidious-1 wget -qO- http://companion:8282/healthz` → `OK`
  - Puerto 8282 no expuesto al host (solo red interna Docker), verificado desde contenedor invidious

### T3.7 — Verificar acceso desde navegador
- [x] Invidious accesible en `http://localhost:3001`
- [ ] Verificación manual desde navegador pendiente (acción del usuario)

---

## Stack de Pruebas

| # | Comando | Resultado esperado |
|---|---|---|
| P3.1 | `docker compose ps` | Los tres servicios en estado `running` o `healthy` | ✅ Los tres `Up` |
| P3.2 | `docker compose ps invidious-db` | Estado `healthy` | ✅ `healthy` |
| P3.3 | `docker compose ps companion` | Estado `running` o `healthy` | ✅ `Up 54 seconds` |
| P3.4 | `docker compose ps invidious` | Estado `running` o `healthy` | ✅ `healthy` |
| P3.5 | `curl … http://localhost:3001` | `200` (o `302` redirect) | ✅ `302` → redirect a feed |
| P3.6 | `curl … http://localhost:3001/api/v1/stats` | `200` | ✅ `200` |
| P3.7 | `curl … /api/v1/comments/jNQXAC9IVRw` | JSON con datos | ✅ JSON con `commentCount: 10472490` |
| P3.8 | `docker compose logs invidious \| grep -i error` | Sin errores críticos | ✅ Sin salida |
| P3.9 | `docker volume ls \| grep …` | Ambos volúmenes listados | ✅ `invidious_postgresdata` e `invidious_companion_cache` |

### Criterio de paso
✅ **FASE 3 COMPLETADA** — Invidious responde en `http://localhost:3001`, API devuelve datos reales de YouTube. Listo para continuar a la Fase 4.

### Resolución de problemas comunes

| Síntoma | Probable causa | Acción |
|---|---|---|
| `invidious` sale con error de DB | `invidious-db` aún no está listo | Esperar 30s y ejecutar `docker compose restart invidious` |
| `curl` retorna `Connection refused` | Contenedor no levantó | `docker compose logs invidious` para ver el error |
| API retorna `500` | `hmac_key` vacío o inválido | Revisar T2.4, corregir y `docker compose up -d` |
