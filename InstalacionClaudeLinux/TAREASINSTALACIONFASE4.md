# Fase 4 — Mantenimiento y Estabilidad (Linux)

> Objetivo: configurar el reinicio periódico obligatorio y el proceso de actualización para mantener Invidious funcional a largo plazo.
>
> Prerequisito: Fase 3 completada y todos sus checks pasados.

---

## Notas importantes

- **El reinicio periódico es obligatorio.** Sin él, YouTube bloquea las sesiones y Invidious deja de funcionar. Reiniciar al menos una vez al día, idealmente cada hora.
- El crontab debe usar la **ruta absoluta** tanto al script como al directorio de trabajo — cron no hereda el `PATH` del usuario.
- `docker compose pull` + `docker compose up -d` actualiza las imágenes **sin borrar los volúmenes de datos**.
- Con `restart: unless-stopped`, Docker reinicia los contenedores automáticamente tras un reboot del sistema, siempre que el servicio `docker` esté habilitado con `systemctl enable docker`.

---

## Tareas

### T4.1 — Crear el script de reinicio
- [ ] Crear el archivo `restart-invidious.sh` en el directorio padre de `invidious/` (ajustar la ruta):
  ```bash
  cat > /ruta/a/restart-invidious.sh << 'EOF'
  #!/bin/bash
  cd /ruta/absoluta/a/invidious
  /usr/bin/docker compose restart invidious
  EOF
  ```
- [ ] Dar permisos de ejecución:
  ```bash
  chmod +x /ruta/a/restart-invidious.sh
  ```
- [ ] Probar que el script funciona ejecutándolo manualmente:
  ```bash
  /ruta/a/restart-invidious.sh
  ```

### T4.2 — Configurar reinicio periódico con cron
- [ ] Abrir el crontab del usuario actual:
  ```bash
  crontab -e
  ```
- [ ] Agregar la siguiente línea al final (ajustar la ruta):
  ```cron
  0 * * * * /ruta/absoluta/a/restart-invidious.sh >> /var/log/invidious-restart.log 2>&1
  ```
- [ ] Guardar y salir del editor
- [ ] Verificar que la entrada quedó registrada:
  ```bash
  crontab -l | grep invidious
  ```

### T4.3 — Verificar que el reinicio periódico funciona
- [ ] Probar el reinicio manualmente desde el directorio `invidious/`:
  ```bash
  docker compose restart invidious
  ```
- [ ] Esperar ~15 segundos y confirmar que Invidious vuelve a estado `Up`:
  ```bash
  sleep 15 && docker compose ps invidious
  ```
- [ ] Confirmar que la API responde `200` tras el reinicio:
  ```bash
  curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats
  ```

### T4.4 — Verificar persistencia de datos tras reinicio completo
- [ ] Reiniciar todos los servicios:
  ```bash
  docker compose restart
  ```
- [ ] Esperar ~20 segundos y confirmar que la API responde `200`:
  ```bash
  sleep 20 && curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats
  ```
- [ ] Confirmar que el volumen de la base de datos persiste intacto:
  ```bash
  docker volume inspect invidious_postgresdata
  ```

### T4.5 — Verificar que Docker reinicia los contenedores tras reboot del sistema
- [ ] Confirmar que `docker` está habilitado para arrancar con el sistema:
  ```bash
  systemctl is-enabled docker
  ```
  Si responde `disabled`, habilitarlo:
  ```bash
  sudo systemctl enable docker
  ```
- [ ] Confirmar que los tres servicios tienen `restart: unless-stopped` en el `docker-compose.yml`:
  ```bash
  grep "restart: unless-stopped" docker-compose.yml | wc -l
  ```
  Resultado esperado: `3`
- [ ] Opcional: reiniciar el sistema y verificar que los contenedores levantan automáticamente

---

## Stack de Pruebas

Ejecutar desde el directorio `invidious/`:

| # | Comando | Resultado esperado |
|---|---|---|
| P4.1 | `crontab -l \| grep invidious` | Muestra la línea del cron configurada en T4.2 |
| P4.2 | `docker compose restart invidious && sleep 15 && docker compose ps invidious` | Estado `Up` tras el reinicio |
| P4.3 | `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats` | `200` tras el reinicio de invidious |
| P4.4 | `docker compose pull 2>&1 \| tail -5` | Las 3 imágenes `Pulled` sin errores |
| P4.5 | `docker volume inspect invidious_postgresdata` | Volumen existe con `Mountpoint` definido |
| P4.6 | `docker compose restart && sleep 20 && curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats` | `200` tras reinicio completo |
| P4.7 | `grep "restart: unless-stopped" docker-compose.yml \| wc -l` | `3` |
| P4.8 | `systemctl is-enabled docker` | `enabled` |

### Criterio de paso
El cron debe estar registrado, Invidious debe responder `200` después de un reinicio manual, y los volúmenes de datos deben persistir. Con todos estos checks en verde, la instalación local está completa y estable.

---

## Resumen del estado final esperado

| Componente | Estado esperado |
|---|---|
| `invidious-db` | `healthy`, datos persistentes en volumen `postgresdata` |
| `companion` | `Up`, caché en volumen `companion_cache` |
| `invidious` | `healthy`, accesible en `http://localhost:3000` (o el puerto configurado) |
| Reinicio periódico | Cron activo, ejecuta cada hora via `restart-invidious.sh` |
| Política de restart | `unless-stopped` en los tres servicios |
| Docker en arranque | `systemctl enable docker` activo |

---

> Para el siguiente paso (exponer el servidor públicamente con HTTPS), ver `PLANNGINX.md`.
