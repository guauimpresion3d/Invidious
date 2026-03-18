# Plan de Instalación de Invidious con Docker

> Basado en la documentación oficial: https://docs.invidious.io/installation/

---

## Requisitos Previos

### Hardware mínimo
- **Uso personal/privado:** 2 GB RAM libres, 20 GB de espacio en disco

### Software requerido
- Docker Engine instalado
- Docker Compose V2 (comando `docker compose`, con espacio — no `docker-compose`)
- `pwgen` para generar claves secretas (`apt install pwgen` o `brew install pwgen`)

---

## Paso 1 — Clonar el repositorio

El repositorio **debe** clonarse porque PostgreSQL necesita montar los scripts de inicialización SQL desde él.

```bash
git clone https://github.com/iv-org/invidious.git
cd invidious
```

---

## Paso 2 — Generar claves secretas

Se necesitan **dos claves distintas**: una para Invidious y otra para el companion.

```bash
pwgen 16 1   # ejecutar dos veces, guardar ambas salidas
```

- `HMAC_KEY` → clave para Invidious (mínimo 20 caracteres recomendado)
- `COMPANION_KEY` → clave para el servicio companion

---

## Paso 3 — Crear el archivo `docker-compose.yml`

En la raíz del repositorio clonado, crear/editar `docker-compose.yml`:

```yaml
version: "3"

services:
  invidious:
    image: quay.io/invidious/invidious:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"   # Cambiar a "3000:3000" si no hay reverse proxy
    environment:
      INVIDIOUS_CONFIG: |
        db:
          dbname: invidious
          user: kemal
          password: kemal          # Cambiar en producción
          host: invidious-db
          port: 5432
        check_tables: true
        hmac_key: "REEMPLAZAR_CON_HMAC_KEY"
        invidious_companion:
          - private_url: "http://companion:8282/companion"
            invidious_companion_key: "REEMPLAZAR_CON_COMPANION_KEY"
        # Descomentar si se usa reverse proxy con HTTPS:
        # https_only: true
        # domain: "tu-dominio.com"
        # external_port: 443
        # use_pubsub_feeds: true
        # use_innertube_for_captions: true
    depends_on:
      invidious-db:
        condition: service_healthy
    healthcheck:
      test: wget -nv --tries=1 --spider http://127.0.0.1:3000/api/v1/comments/jNQXAC9IVRw || exit 1
      interval: 30s
      timeout: 5s
      retries: 2

  companion:
    image: quay.io/invidious/invidious-companion:latest
    restart: unless-stopped
    environment:
      SERVER_SECRET_KEY: "REEMPLAZAR_CON_COMPANION_KEY"
    volumes:
      - companion_cache:/cache
    healthcheck:
      test: wget -nv --tries=1 --spider http://127.0.0.1:8282/healthz || exit 1
      interval: 30s
      timeout: 5s
      retries: 2

  invidious-db:
    image: docker.io/library/postgres:14
    restart: unless-stopped
    volumes:
      - postgresdata:/var/lib/postgresql/data
      - ./config/sql:/config/sql
      - ./docker/init-invidious-db.sh:/docker-entrypoint-initdb.d/init-invidious-db.sh
    environment:
      POSTGRES_DB: invidious
      POSTGRES_USER: kemal
      POSTGRES_PASSWORD: kemal   # Cambiar en producción
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgresdata:
  companion_cache:
```

### Valores que DEBEN reemplazarse antes de iniciar:
| Placeholder | Descripción |
|---|---|
| `REEMPLAZAR_CON_HMAC_KEY` | Salida del primer `pwgen 16 1` |
| `REEMPLAZAR_CON_COMPANION_KEY` | Salida del segundo `pwgen 16 1` (mismo valor en ambos servicios) |
| `kemal` (contraseña) | Cambiar por una contraseña segura en producción |

---

## Paso 4 — Iniciar los servicios

```bash
docker compose up -d
```

Verificar que los contenedores estén corriendo:

```bash
docker compose ps
docker compose logs -f invidious
```

Invidious estará disponible en: `http://localhost:3000`

---

## Paso 5 — Mantenimiento y actualizaciones

### Reinicio periódico (requerido)
Invidious debe reiniciarse **al menos una vez al día** (idealmente cada hora) para mantener la sesión con YouTube.

Agregar al crontab del sistema:
```bash
crontab -e
# Reiniciar cada hora:
0 * * * * docker compose -f /ruta/a/invidious/docker-compose.yml restart invidious
```

### Actualizar a la última versión
```bash
docker compose pull
docker compose up -d
```

### Ver logs
```bash
docker compose logs -f invidious
docker compose logs -f companion
```

---

## Referencias

- Documentación de instalación: https://docs.invidious.io/installation/
- Configuración completa: https://github.com/iv-org/invidious/blob/master/config/config.example.yml
- Wiki de Invidious companion: https://github.com/iv-org/invidious-companion/wiki
- Mantenimiento de base de datos: https://docs.invidious.io/database-maintenance/
- Para exponer el servidor públicamente, ver [PLANNGINX.md](./PLANNGINX.md)
