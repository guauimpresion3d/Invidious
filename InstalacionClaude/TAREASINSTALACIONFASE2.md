# Fase 2 — Configuración

> Objetivo: crear y configurar el archivo `docker-compose.yml` con todos los valores correctos antes de levantar ningún contenedor.
>
> Prerequisito: Fase 1 completada y todos sus checks pasados.

---

## Notas importantes (aprendidas en ejecución previa)

- **`invidious_companion_key` es una clave de nivel raíz**, NO va anidada dentro del array `invidious_companion`. Error frecuente que hace que Invidious arranque en bucle con el mensaje `Config: Please configure a key if you are using invidious companion.`
- **El healthcheck del companion debe deshabilitarse** — el contenedor es un binario mínimo sin `wget`, `curl`, ni shell. Usar `healthcheck: disable: true`.
- **Verificar si el puerto 3000 está libre** antes de iniciar. Si está ocupado, cambiar el binding a `127.0.0.1:PUERTO_LIBRE:3000` y actualizar las URLs en los checks de la Fase 3.
- Las claves (`HMAC_KEY` y `COMPANION_KEY`) deben tener **exactamente 16 caracteres**.
- El `SERVER_SECRET_KEY` del companion y el `invidious_companion_key` de invidious deben tener el **mismo valor**.

---

## Tareas

### T2.1 — Verificar que el puerto no está en uso
- [ ] Comprobar que el puerto 3000 (o el elegido) está libre:
  ```bash
  # Linux:
  ss -tlnp | grep :3000
  # Windows:
  powershell -Command "Get-NetTCPConnection -LocalPort 3000 -ErrorAction SilentlyContinue"
  ```
- [ ] Si está ocupado, elegir un puerto alternativo (ej. `3001`) y usarlo en T2.4

### T2.2 — Crear el archivo `docker-compose.yml`
- [ ] Dentro del directorio `invidious/`, reemplazar el `docker-compose.yml` de desarrollo por el de producción con los tres servicios: `invidious`, `companion` e `invidious-db`

### T2.3 — Configurar el servicio `invidious-db`
- [ ] Asignar imagen `docker.io/library/postgres:14`
- [ ] Montar el volumen `postgresdata:/var/lib/postgresql/data`
- [ ] Montar `./config/sql:/config/sql`
- [ ] Montar `./docker/init-invidious-db.sh:/docker-entrypoint-initdb.d/init-invidious-db.sh`
- [ ] Establecer las variables de entorno:
  - `POSTGRES_DB: invidious`
  - `POSTGRES_USER: kemal`
  - `POSTGRES_PASSWORD: kemal`
- [ ] Agregar el healthcheck con `pg_isready` (interval 10s, retries 5)
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
- [ ] Establecer el puerto como `127.0.0.1:3000:3000` (ajustar si el puerto está ocupado)
- [ ] En `INVIDIOUS_CONFIG`, configurar el bloque `db` apuntando a `invidious-db`
- [ ] Establecer `check_tables: true`
- [ ] Establecer `hmac_key` con el valor de `HMAC_KEY` de la Fase 1
- [ ] Configurar `invidious_companion` (solo `private_url`) y `invidious_companion_key` como clave **raíz**:
  ```yaml
  INVIDIOUS_CONFIG: |
    ...
    invidious_companion:
      - private_url: "http://companion:8282/companion"
    invidious_companion_key: "TU_COMPANION_KEY"   # ← nivel raíz, NO dentro del array
  ```
- [ ] Agregar `depends_on: invidious-db: condition: service_healthy`
- [ ] Agregar el healthcheck apuntando a `/api/v1/comments/jNQXAC9IVRw`
- [ ] Agregar `restart: unless-stopped`

### T2.6 — Declarar los volúmenes nombrados
- [ ] Al final del `docker-compose.yml`, declarar:
  ```yaml
  volumes:
    postgresdata:
    companion_cache:
  ```

### T2.7 — Revisión final del archivo
- [ ] Confirmar que **no** aparece el atributo `version:` (obsoleto en Compose v2+)
- [ ] Confirmar que ningún campo contiene texto placeholder sin reemplazar
- [ ] Confirmar que `hmac_key` tiene el valor de `HMAC_KEY`
- [ ] Confirmar que `SERVER_SECRET_KEY` y `invidious_companion_key` tienen el **mismo valor** (`COMPANION_KEY`)
- [ ] Confirmar que `invidious_companion_key` está al **nivel raíz** del bloque YAML (no indentada dentro del array)

---

## Stack de Pruebas

Ejecutar estos checks sobre el archivo antes de levantar los contenedores.

| # | Comando | Resultado esperado |
|---|---|---|
| P2.1 | `ls docker-compose.yml` | El archivo existe en el directorio `invidious/` |
| P2.2 | `docker compose config` | Sin errores de YAML; imprime la configuración expandida |
| P2.3 | `grep "REEMPLAZAR\|CHANGE_ME\|placeholder" docker-compose.yml` | Sin salida |
| P2.4 | `grep "hmac_key" docker-compose.yml` | Muestra una línea con un valor real de 16 chars |
| P2.5 | `grep "SERVER_SECRET_KEY" docker-compose.yml` | Muestra una línea con un valor real de 16 chars |
| P2.6 | `grep "invidious_companion_key" docker-compose.yml` | Muestra una línea con un valor real de 16 chars |
| P2.7 | `docker compose config --services` | Lista exactamente: `invidious`, `companion`, `invidious-db` |
| P2.8 | `grep "127.0.0.1:" docker-compose.yml` | Confirma que el puerto está ligado solo a localhost |

### Criterio de paso
`docker compose config` debe ejecutarse sin errores y sin warnings. Solo entonces continuar a la Fase 3.
