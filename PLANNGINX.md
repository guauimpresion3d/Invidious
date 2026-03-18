# Plan de Configuración NGINX para Invidious (Servidor Público)

> Basado en la documentación oficial: https://docs.invidious.io/nginx/
>
> Este documento asume que Invidious ya está instalado y corriendo localmente.
> Ver [PLAN.md](./PLAN.md) para la instalación base.

---

## Requisitos Previos

- Invidious corriendo en `127.0.0.1:3000` (instalación local completada)
- NGINX instalado (`apt install nginx`)
- Un dominio apuntando a la IP del servidor
- Certbot instalado (`apt install certbot python3-certbot-nginx`)

---

## Paso 1 — Obtener certificado SSL

```bash
certbot certonly --nginx -d invidious.tu-dominio.com
```

Los certificados se generarán en:
- `/etc/letsencrypt/live/invidious.tu-dominio.com/fullchain.pem`
- `/etc/letsencrypt/live/invidious.tu-dominio.com/privkey.pem`

---

## Paso 2 — Crear la configuración de NGINX

Crear el archivo `/etc/nginx/sites-available/invidious`:

```nginx
server {
    listen 80;
    listen [::]:80;
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;

    server_name invidious.tu-dominio.com;

    access_log off;
    error_log /var/log/nginx/error.log crit;

    ssl_certificate /etc/letsencrypt/live/invidious.tu-dominio.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/invidious.tu-dominio.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    if ($https = '') { return 301 https://$host$request_uri; }
}
```

Activar el sitio y recargar NGINX:

```bash
ln -s /etc/nginx/sites-available/invidious /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

---

## Paso 3 — Actualizar la configuración de Invidious

En `docker-compose.yml`, descomentar y ajustar las siguientes líneas dentro de `INVIDIOUS_CONFIG`:

```yaml
https_only: true
domain: "invidious.tu-dominio.com"
external_port: 443
use_pubsub_feeds: true
use_innertube_for_captions: true  # Recomendado para servidores en datacenter
```

Reiniciar Invidious para aplicar los cambios:

```bash
docker compose up -d
```

---

## Paso 4 — Verificar

- Acceder a `https://invidious.tu-dominio.com` y confirmar que carga con HTTPS.
- Confirmar que `http://` redirige automáticamente a `https://`.

---

## Referencias

- Documentación NGINX oficial de Invidious: https://docs.invidious.io/nginx/
- Configuración completa de Invidious: https://github.com/iv-org/invidious/blob/master/config/config.example.yml
