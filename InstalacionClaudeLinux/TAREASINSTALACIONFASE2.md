# Fase 2 — Configuración (Linux)

> Objetivo: crear y configurar el archivo `docker-compose.yml` con todos los valores correctos antes de levantar ningún contenedor.
>
> Prerequisito: Fase 1 completada y todos sus checks pasados.

---

## Notas importantes

- **`invidious_companion_key` es una clave de nivel raíz** — NO va anidada dentro del array `invidious_companion`. Si se anida, Invidious arranca en bucle con el mensaje `Config: Please configure a key if you are using invidious companion.`
- **El healthcheck del companion debe deshabilitarse** — el contenedor es un binario mínimo sin `wget`, `curl` ni shell. Usar `healthcheck: disable: true`.
- **Verificar que el puerto 3000 esté libre** antes de configurar. Si está ocupado, cambiar el binding del host a otro puerto libre (ej. `127.0.0.1:3001:3000`).
- Las claves `HMAC_KEY` y `COMPANION_KEY` deben tener **exactamente 16 caracteres**.
- `SERVER_SECRET_KEY` (companion) e `invidious_companion_key` (invidious) deben tener el **mismo valor**.
- El atributo `version:` al inicio del `docker-compose.yml` es obsoleto en Compose v2+ y genera warnings — no incluirlo.

---

## Tareas

### T2.1 — Verificar que el puerto 3000 está libre
- [ ] Comprobar que el puerto no está en uso:
  ```bash
  ss -tlnp | grep :3000
  ```
- [ ] Si está ocupado, identificar el proceso y elegir un puerto alternativo:
  ```bash
  ss -tlnp | grep :3000   # ver qué proceso lo usa
  ```
- [ ] Anotar el puerto a usar (por defecto `3000`, alternativa `3001`, etc.)

### T2.2 — Crear el archivo `docker-compose.yml` de producción
- [ ] Dentro del directorio `invidious/`, reemplazar el `docker-compose.yml` de desarrollo (que hace build local) por el de producción con las imágenes pre-construidas y los tres servicios: `invidious`, `companion` e `invidious-db`

### T2.3 — Configurar el servicio `invidious-db`
- [ ] Asignar imagen `docker.io/library/postgres:14`
- [ ] Montar el volumen `postgresdata:/var/lib/postgresql/data`
- [ ] Montar `./config/sql:/config/sql`
- [ ] Montar `./docker/init-invidious-db.sh:/docker-entrypoint-initdb.d/init-invidious-db.sh`
- [ ] Establecer las variables de entorno:
  - `POSTGRES_DB: invidious`
  - `POSTGRES_USER: kemal`
  - `POSTGRES_PASSWORD: kemal`
- [ ] Agregar healthcheck con `pg_isready` (interval 10s, retries 5):
  ```yaml
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
    interval: 10s
    timeout: 5s
    retries: 5
  ```
- [ ] Agregar `restart: unless-stopped`

### T2.4 — Configurar el servicio `companion`
- [ ] Asignar imagen `quay.io/invidious/invidious-companion:latest`
- [ ] Establecer `SERVER_SECRET_KEY` con el valor de `COMPANION_KEY` de la Fase 1
- [ ] Montar volumen `companion_cache:/cache`
- [ ] **Deshabilitar el healthcheck** (el contenedor no tiene utilidades HTTP):
  ```yaml
  healthcheck:
    disable: true
  ```
- [ ] Agregar `restart: unless-stopped`

### T2.5 — Configurar el servicio `invidious`
- [ ] Asignar imagen `quay.io/invidious/invidious:latest`
- [ ] Establecer el puerto (ajustar si está ocupado):
  ```yaml
  ports:
    - "127.0.0.1:3000:3000"
  ```
- [ ] En `INVIDIOUS_CONFIG`, configurar el bloque `db` apuntando a `invidious-db`
- [ ] Establecer `check_tables: true`
- [ ] Establecer `hmac_key` con el valor de `HMAC_KEY` de la Fase 1
- [ ] Configurar `invidious_companion` e `invidious_companion_key` — **`invidious_companion_key` va al nivel raíz**, no dentro del array:
  ```yaml
  INVIDIOUS_CONFIG: |
    db:
      dbname: invidious
      user: kemal
      password: kemal
      host: invidious-db
      port: 5432
    check_tables: true
    hmac_key: "TU_HMAC_KEY_16_CHARS"
    invidious_companion:
      - private_url: "http://companion:8282/companion"
    invidious_companion_key: "TU_COMPANION_KEY_16_CHARS"   # ← nivel raíz
  ```
- [ ] Agregar `depends_on` con condición `service_healthy` para `invidious-db`
- [ ] Agregar healthcheck:
  ```yaml
  healthcheck:
    test: wget -nv --tries=1 --spider http://127.0.0.1:3000/api/v1/comments/jNQXAC9IVRw || exit 1
    interval: 30s
    timeout: 5s
    retries: 2
  ```
- [ ] Agregar `restart: unless-stopped`

### T2.6 — Declarar los volúmenes nombrados
- [ ] Al final del `docker-compose.yml`, agregar:
  ```yaml
  volumes:
    postgresdata:
    companion_cache:
  ```

### T2.7 — Revisión final del archivo
- [ ] Confirmar que **no** aparece el atributo `version:` en el archivo
- [ ] Confirmar que no quedan placeholders sin reemplazar (`CHANGE_ME`, `TU_`, etc.)
- [ ] Confirmar que `hmac_key` tiene el valor de `HMAC_KEY` (16 chars)
- [ ] Confirmar que `SERVER_SECRET_KEY` e `invidious_companion_key` tienen el **mismo valor** (`COMPANION_KEY`, 16 chars)
- [ ] Confirmar que `invidious_companion_key` está al **nivel raíz** del bloque YAML

---

## Stack de Pruebas

Ejecutar desde el directorio `invidious/`:

| # | Comando | Resultado esperado |
|---|---|---|
| P2.1 | `ls docker-compose.yml` | El archivo existe |
| P2.2 | `docker compose config` | Sin errores ni warnings de YAML |
| P2.3 | `grep -E "CHANGE_ME\|TU_\|placeholder" docker-compose.yml` | Sin salida |
| P2.4 | `grep "hmac_key" docker-compose.yml` | Línea con valor real de 16 chars |
| P2.5 | `grep "SERVER_SECRET_KEY" docker-compose.yml` | Línea con valor real de 16 chars |
| P2.6 | `grep "invidious_companion_key" docker-compose.yml` | Línea con valor real de 16 chars |
| P2.7 | `docker compose config --services` | Lista: `invidious`, `companion`, `invidious-db` |
| P2.8 | `grep "127.0.0.1:" docker-compose.yml` | Puerto ligado solo a localhost |
| P2.9 | `grep "^version:" docker-compose.yml` | Sin salida (atributo obsoleto no presente) |

### Criterio de paso
`docker compose config` debe ejecutarse sin errores ni warnings. Solo entonces continuar a la Fase 3.
