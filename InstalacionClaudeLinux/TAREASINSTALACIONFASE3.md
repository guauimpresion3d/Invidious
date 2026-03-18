# Fase 3 — Despliegue y Verificación (Linux)

> Objetivo: levantar los tres contenedores y confirmar que Invidious responde correctamente en `http://localhost:3000` (o el puerto configurado).
>
> Prerequisito: Fase 2 completada y todos sus checks pasados.

---

## Notas importantes

- **Invidious en bucle de reinicio:** si aparece el mensaje `Config: Please configure a key if you are using invidious companion.` de forma repetida en los logs, `invidious_companion_key` está anidada incorrectamente — debe estar al nivel raíz del YAML (ver Fase 2, T2.5).
- **Companion sin herramientas HTTP:** el healthcheck ya debe estar deshabilitado en la Fase 2. Para verificar que el companion responde, hacerlo desde dentro del contenedor `invidious`, que sí tiene `wget`:
  ```bash
  docker exec invidious-invidious-1 wget -qO- http://companion:8282/healthz
  ```
- **Puerto 8282 del companion:** no se expone al host, solo está disponible en la red interna Docker (`invidious_default`). Los checks de este puerto siempre se hacen desde dentro de un contenedor.
- **`invidious-db` tarda en estar `healthy`:** el arranque inicial puede tardar 20-30 segundos mientras PostgreSQL inicializa el schema. El servicio `invidious` espera automáticamente gracias a `depends_on: condition: service_healthy`.

---

## Tareas

### T3.1 — Descargar las imágenes Docker
- [ ] Ejecutar desde el directorio `invidious/`:
  ```bash
  docker compose pull
  ```
- [ ] Confirmar que las tres imágenes se descargan sin errores (~900 MB en total):
  - `quay.io/invidious/invidious:latest`
  - `quay.io/invidious/invidious-companion:latest`
  - `docker.io/library/postgres:14`

### T3.2 — Levantar los servicios en modo background
- [ ] Ejecutar:
  ```bash
  docker compose up -d
  ```
- [ ] Confirmar que el comando finaliza sin errores

### T3.3 — Verificar estado de los contenedores
- [ ] Esperar ~30 segundos y ejecutar:
  ```bash
  docker compose ps
  ```
- [ ] Confirmar que `invidious-db` alcanza el estado `healthy`
- [ ] Confirmar que `invidious` alcanza el estado `healthy` (puede tardar hasta 60 segundos)
- [ ] Confirmar que `companion` está `Up`

### T3.4 — Revisar logs de arranque
- [ ] Revisar logs de la base de datos:
  ```bash
  docker compose logs invidious-db
  ```
  Confirmar que aparece `database system is ready to accept connections` y **no** aparece `cannot execute: required file not found`

- [ ] Revisar logs de Invidious:
  ```bash
  docker compose logs invidious
  ```
  Confirmar que **no** aparece `Config: Please configure a key if you are using invidious companion.` en bucle y **no** hay errores de conexión a la DB

### T3.5 — Verificar respuesta HTTP de Invidious
- [ ] Confirmar que la interfaz responde (ajustar puerto si se cambió en Fase 2):
  ```bash
  curl -s -o /dev/null -w "%{http_code}" http://localhost:3000
  ```
  Resultado esperado: `200` o `302` (redirect a feed)

- [ ] Confirmar que la API responde:
  ```bash
  curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats
  ```
  Resultado esperado: `200`

### T3.6 — Verificar respuesta del companion
- [ ] Verificar desde dentro del contenedor invidious:
  ```bash
  docker exec invidious-invidious-1 wget -qO- http://companion:8282/healthz
  ```
  Resultado esperado: `OK`

### T3.7 — Verificar acceso desde navegador
- [ ] Abrir `http://localhost:3000` en un navegador
- [ ] Confirmar que la interfaz de Invidious carga correctamente
- [ ] Realizar una búsqueda de prueba y confirmar que devuelve resultados

---

## Stack de Pruebas

Ejecutar desde el directorio `invidious/`:

| # | Comando | Resultado esperado |
|---|---|---|
| P3.1 | `docker compose ps` | Los tres servicios en estado `Up` |
| P3.2 | `docker compose ps invidious-db` | Estado `healthy` |
| P3.3 | `docker compose ps companion` | Estado `Up` |
| P3.4 | `docker compose ps invidious` | Estado `healthy` |
| P3.5 | `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000` | `200` o `302` |
| P3.6 | `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats` | `200` |
| P3.7 | `curl -s http://localhost:3000/api/v1/comments/jNQXAC9IVRw \| head -c 100` | JSON con datos reales de YouTube |
| P3.8 | `docker compose logs invidious 2>&1 \| grep -i "error\|fatal"` | Sin líneas de error crítico |
| P3.9 | `docker volume ls \| grep -E "postgresdata\|companion_cache"` | Ambos volúmenes listados |

### Criterio de paso
Los endpoints `/` y `/api/v1/stats` deben responder `200` o `302`. Si algún contenedor aparece como `exited` o `unhealthy`, revisar logs con `docker compose logs <servicio>` antes de continuar.

---

## Resolución de problemas comunes

| Síntoma | Probable causa | Acción |
|---|---|---|
| `invidious` en bucle con `Config: Please configure a key…` | `invidious_companion_key` anidada incorrectamente | Moverla al nivel raíz del YAML, luego `docker compose up -d` |
| `Bind for 0.0.0.0:3000 failed: port is already allocated` | Puerto 3000 ocupado por otro proceso | Cambiar el binding en `docker-compose.yml` a un puerto libre (`ss -tlnp \| grep :3000`) |
| `invidious` sale con error de conexión a DB | `invidious-db` aún no está `healthy` | Esperar 30s y ejecutar `docker compose restart invidious` |
| API retorna `500` | `hmac_key` vacío o inválido | Verificar valor en `docker-compose.yml`, corregir y `docker compose up -d` |
| Companion falla con `String must contain exactly 16 character(s)` | Clave no tiene exactamente 16 chars | Regenerar claves en Fase 1 con `pwgen 16 1` y actualizar `docker-compose.yml` |
| `permission denied` al ejecutar comandos Docker | Usuario no está en el grupo `docker` | Ejecutar `sudo usermod -aG docker $USER && newgrp docker` (ver Fase 1, T1.2) |
