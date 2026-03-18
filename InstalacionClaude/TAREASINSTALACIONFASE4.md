# Fase 4 — Mantenimiento y Estabilidad

> Objetivo: configurar el reinicio periódico obligatorio y el proceso de actualización para mantener Invidious funcional a largo plazo.
>
> Prerequisito: Fase 3 completada y todos sus checks pasados.

---

## Notas importantes (aprendidas en ejecución previa)

- **El reinicio periódico es obligatorio.** Sin él, YouTube bloquea las sesiones y Invidious deja de funcionar. Reiniciar al menos una vez al día, idealmente cada hora.
- En **Windows**, `crontab` no está disponible — usar Windows Task Scheduler con `schtasks` o PowerShell.
- En **Linux**, usar `crontab -e` con la ruta absoluta al `docker-compose.yml`.
- `docker compose pull` + `docker compose up -d` actualiza las imágenes **sin borrar los volúmenes de datos**.

---

## Tareas

### T4.1 — Crear script de reinicio
- [ ] Crear un script que ejecute el reinicio (ajustar la ruta al directorio real):

  **Linux/macOS** — crear `restart-invidious.sh`:
  ```bash
  #!/bin/bash
  cd /ruta/absoluta/a/invidious
  docker compose restart invidious
  ```
  ```bash
  chmod +x restart-invidious.sh
  ```

  **Windows** — crear `restart-invidious.bat`:
  ```bat
  @echo off
  cd /d "C:\ruta\a\invidious"
  docker compose restart invidious
  ```

### T4.2 — Configurar reinicio periódico automático
- [ ] **Linux/macOS** — agregar entrada en crontab:
  ```bash
  crontab -e
  # Agregar la siguiente línea:
  0 * * * * /ruta/absoluta/a/restart-invidious.sh
  ```

- [ ] **Windows** — registrar tarea en Task Scheduler vía PowerShell:
  ```powershell
  $action = New-ScheduledTaskAction -Execute 'cmd.exe' -Argument '/c "C:\ruta\a\restart-invidious.bat"'
  $trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Hours 1) -Once -At (Get-Date)
  $settings = New-ScheduledTaskSettingsSet -ExecutionTimeLimit (New-TimeSpan -Minutes 5)
  Register-ScheduledTask -TaskName 'Invidious-HourlyRestart' -Action $action -Trigger $trigger -Settings $settings -Description 'Reinicia Invidious cada hora' -Force
  ```

### T4.3 — Verificar que el reinicio periódico funciona
- [ ] Probar el reinicio manualmente:
  ```bash
  docker compose restart invidious
  ```
- [ ] Esperar ~15 segundos y confirmar que Invidious vuelve a estado `Up`
- [ ] Confirmar que la API responde `200` tras el reinicio:
  ```bash
  curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats
  ```

### T4.4 — Verificar persistencia de datos tras reinicio completo
- [ ] Reiniciar todos los servicios:
  ```bash
  docker compose restart
  ```
- [ ] Esperar ~20 segundos y confirmar que la API responde `200`
- [ ] Confirmar que el volumen `postgresdata` persiste:
  ```bash
  docker volume inspect invidious_postgresdata
  ```

### T4.5 — Verificar política de restart automático tras reboot del sistema
- [ ] Confirmar que los tres servicios tienen `restart: unless-stopped` en el `docker-compose.yml`
- [ ] Opcional: reiniciar el sistema y verificar que los contenedores levantan automáticamente sin intervención manual

---

## Stack de Pruebas

| # | Comando | Resultado esperado |
|---|---|---|
| P4.1 (Linux) | `crontab -l \| grep invidious` | Muestra la línea del cron configurada |
| P4.1 (Windows) | `powershell -Command "Get-ScheduledTask 'Invidious-HourlyRestart'"` | Tarea en estado `Ready` |
| P4.2 | `docker compose restart invidious && sleep 15 && docker compose ps invidious` | Estado `Up` después del reinicio |
| P4.3 | `curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats` | `200` tras el reinicio |
| P4.4 | `docker compose pull 2>&1 \| tail -5` | Finaliza sin errores |
| P4.5 | `docker volume inspect invidious_postgresdata` | Volumen existe con `Mountpoint` definido |
| P4.6 | `docker compose restart && sleep 20 && curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/api/v1/stats` | `200` después de reiniciar todos los servicios |
| P4.7 | `grep "restart: unless-stopped" docker-compose.yml \| wc -l` | Devuelve `3` |

### Criterio de paso
El reinicio periódico debe estar configurado, Invidious debe responder `200` después de un reinicio manual, y los volúmenes de datos deben persistir. Con todos estos checks en verde, la instalación local está completa y estable.

---

## Resumen del estado final esperado

| Componente | Estado esperado |
|---|---|
| `invidious-db` | `healthy`, datos persistentes en volumen `postgresdata` |
| `companion` | `Up`, caché en volumen `companion_cache` |
| `invidious` | `healthy`, accesible en `http://localhost:3000` (o el puerto configurado) |
| Reinicio periódico | Activo, ejecuta cada hora |
| Política de restart | `unless-stopped` en los tres servicios |

---

> Para el siguiente paso (exponer el servidor públicamente con HTTPS), ver `PLANNGINX.md`.
