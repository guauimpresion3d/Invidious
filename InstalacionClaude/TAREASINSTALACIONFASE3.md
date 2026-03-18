# Fase 3 — Despliegue y Verificación

> Objetivo: levantar los tres contenedores y confirmar que Invidious responde correctamente en `http://localhost:3000` (o el puerto configurado).
>
> Prerequisito: Fase 2 completada y todos sus checks pasados.

---

## Notas importantes (aprendidas en ejecución previa)

- **Puerto ocupado:** si el puerto elegido ya está en uso, Docker fallará al crear el contenedor `invidious`. Verificarlo antes en T2.1 (Fase 2) y ajustar el binding.
- **Companion sin herramientas HTTP:** el healthcheck por defecto usa `wget`, que no existe en el contenedor. Ya debe estar deshabilitado desde la Fase 2 (`healthcheck: disable: true`). Verificar la salud del companion desde el contenedor invidious: `docker exec invidious-invidious-1 wget -qO- http://companion:8282/healthz`.
- **Invidious en bucle de reinicio:** si aparece el mensaje `Config: Please configure a key if you are using invidious companion.` y el contenedor no levanta, revisar que `invidious_companion_key` está al nivel raíz en el `docker-compose.yml` (no anidada en el array `invidious_companion`).
- **Puerto 8282 del companion:** no se expone al host, solo está disponible en la red interna Docker. Los checks de este puerto deben ejecutarse desde dentro de un contenedor.

---

## Tareas

### T3.1 — Descargar las imágenes Docker
- [ ] Ejecutar desde el directorio `invidious/`:
  ```bash
  docker compose pull
  ```
- [ ] Esperar a que las tres imágenes se descarguen completamente sin errores (~900 MB en total)

### T3.2 — Levantar los servicios en modo background
- [ ] Ejecutar:
  ```bash
  docker compose up -d
  ```
- [ ] Confirmar que el comando finaliza sin errores

### T3.3 — Verificar estado de los contenedores
- [ ] Ejecutar `docker compose ps` y confirmar que los tres servicios están `Up`
- [ ] Confirmar que `invidious-db` alcanza el estado `healthy`
- [ ] Confirmar que `invidious` alcanza el estado `healthy` (puede tardar 30-60 segundos)

### T3.4 — Revisar logs de arranque
- [ ] Revisar logs de la base de datos:
  ```bash
  docker compose logs invidious-db
  ```
  Confirmar que termina con `database system is ready to accept connections` y **sin** el error `cannot execute: required file not found`
- [ ] Revisar logs de Invidious:
  ```bash
  docker compose logs invidious
  ```
  Confirmar que **no** aparece el mensaje `Config: Please configure a key if you are using invidious companion.` en bucle

### T3.5 — Verificar respuesta HTTP de Invidious
- [ ] Ejecutar (ajustar el puerto si se cambió en Fase 2):
  ```bash
  curl -s -o /dev/null -w "%{http_code}" http://localhost:3000
  ```
  Confirmar que retorna `200` o `302` (redirect a feed)

### T3.6 — Verificar respuesta del companion
- [ ] Verificar desde dentro del contenedor invidious (puerto 8282 es solo interno):
  ```bash
  docker exec invidious-invidious-1 wget -qO- http://companion:8282/healthz
  ```
  Confirmar que retorna `OK`

### T3.7 — Verificar acceso desde navegador
- [ ] Abrir `http://localhost:3000` (o el puerto configurado) en un navegador
- [ ] Confirmar que la interfaz de Invidious carga correctamente
- [ ] Realizar una búsqueda de prueba y confirmar que devuelve resultados

---

## Stack de Pruebas

| # | Comando | Resultado esperado |
|---|---|---|
| P3.1 | `docker compose ps` | Los tres servicios en estado `Up` |
| P3.2 | `docker compose ps invidious-db` | Estado `healthy` |
| P3.3 | `docker compose ps companion` | Estado `Up` |
| P3.4 | `docker compose ps invidious` | Estado `healthy` |
| P3.5 | `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000` | `200` o `302` |
| P3.6 | `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats` | `200` |
| P3.7 | `curl -s http://localhost:3000/api/v1/comments/jNQXAC9IVRw \| head -c 100` | JSON con datos reales |
| P3.8 | `docker compose logs invidious 2>&1 \| grep -i "error"` | Sin líneas de error crítico |
| P3.9 | `docker volume ls \| grep -E "postgresdata\|companion_cache"` | Ambos volúmenes listados |

### Criterio de paso
Los endpoints `/` y `/api/v1/stats` deben responder `200` o `302`. Si algún contenedor aparece como `exited` o `unhealthy`, revisar logs con `docker compose logs <servicio>` antes de continuar.

### Resolución de problemas comunes

| Síntoma | Probable causa | Acción |
|---|---|---|
| `invidious` en bucle con `Config: Please configure a key…` | `invidious_companion_key` anidada incorrectamente | Moverla al nivel raíz del YAML y `docker compose up -d` |
| `cannot execute: required file not found` en logs de DB | `init-invidious-db.sh` tiene CRLF | Ejecutar `sed -i 's/\r//' docker/init-invidious-db.sh`, borrar el volumen y relanzar |
| `Bind for 0.0.0.0:3000 failed: port is already allocated` | Puerto ocupado por otro proceso | Cambiar el puerto en `docker-compose.yml` a uno libre |
| `invidious` sale con error de DB | `invidious-db` aún no está listo | Esperar 30s y ejecutar `docker compose restart invidious` |
| API retorna `500` | `hmac_key` vacío o inválido | Revisar T2.5, corregir y `docker compose up -d` |
| Companion falla al arrancar con `String must contain exactly 16 character(s)` | Clave no tiene exactamente 16 chars | Regenerar claves en Fase 1 con longitud exacta de 16 |
