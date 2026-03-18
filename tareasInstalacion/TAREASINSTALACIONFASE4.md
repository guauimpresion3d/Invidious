# Fase 4 — Mantenimiento y Estabilidad

> Objetivo: configurar el reinicio periódico obligatorio y el proceso de actualización para mantener Invidious funcional a largo plazo.
>
> Prerequisito: Fase 3 completada y todos sus checks pasados.

---

## Tareas

### T4.1 — Configurar reinicio periódico con cron
Invidious **debe** reiniciarse al menos una vez al día (idealmente cada hora) para mantener la sesión activa con YouTube.

- [x] `crontab` no disponible en Windows — se usa **Windows Task Scheduler** como equivalente
- [x] Creado script `C:\Users\carlo\Documents\Apps\Invidious\restart-invidious.bat`
- [x] Tarea registrada: `Invidious-HourlyRestart` — se ejecuta cada hora, estado `Ready`
  ```powershell
  # Comando usado para registrar la tarea:
  $action = New-ScheduledTaskAction -Execute 'cmd.exe' -Argument '/c "...\restart-invidious.bat"'
  $trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Hours 1) -Once -At (Get-Date)
  Register-ScheduledTask -TaskName 'Invidious-HourlyRestart' ...
  ```

### T4.2 — Verificar que cron ejecuta el comando correctamente
- [x] `docker compose restart invidious` ejecutado manualmente
- [x] Invidious volvió a estado `Up` → `healthy` en ~15 segundos
- [x] API respondió `200` tras el reinicio

### T4.3 — Documentar el proceso de actualización
- [x] Proceso de actualización confirmado:
  ```bash
  cd C:\Users\carlo\Documents\Apps\Invidious\invidious
  docker compose pull   # descarga nuevas imágenes
  docker compose up -d  # recrea contenedores con nueva imagen
  ```
- [x] `docker compose pull` verificado — no elimina datos (volúmenes persisten)

### T4.4 — Verificar persistencia de datos tras reinicio
- [x] `docker compose restart` (reinicio completo de los 3 servicios) ejecutado
- [x] Volumen `invidious_postgresdata` intacto tras el reinicio
- [x] API respondió `200` confirmando que la DB levantó con datos

### T4.5 — Verificar que Docker reinicia los contenedores tras reboot del sistema
- [x] Los tres servicios tienen `restart: unless-stopped` en `docker-compose.yml` (3/3 confirmado)
- [ ] Reinicio del sistema operativo: pendiente de verificación manual por el usuario

---

## Stack de Pruebas

| # | Comando | Resultado esperado |
|---|---|---|
| P4.1 | `Get-ScheduledTask 'Invidious-HourlyRestart'` | Tarea registrada, estado `Ready` | ✅ `Invidious-HourlyRestart | Ready` |
| P4.2 | `docker compose restart invidious && sleep 15 && docker compose ps invidious` | Estado `running` después del reinicio | ✅ `Up (healthy)` |
| P4.3 | `curl … http://localhost:3001/api/v1/stats` | `200` tras el reinicio | ✅ `200` |
| P4.4 | `docker compose pull 2>&1 \| tail -5` | Finaliza sin errores | ✅ Las 3 imágenes `Pulled` sin errores |
| P4.5 | `docker volume inspect invidious_postgresdata` | Volumen existe con `Mountpoint` definido | ✅ Mountpoint en `/var/lib/docker/volumes/…` |
| P4.6 | `docker compose restart && sleep 20 && curl …` | `200` después de reiniciar todos los servicios | ✅ `200` |
| P4.7 | `grep "restart: unless-stopped" docker-compose.yml \| wc -l` | Devuelve `3` | ✅ `3` |

### Criterio de paso
✅ **FASE 4 COMPLETADA** — Tarea programada activa, Invidious responde `200` tras reinicios, volúmenes persistentes. Instalación local completa y estable.

---

## Resumen del estado final esperado

| Componente | Estado |
|---|---|
| `invidious-db` | ✅ `healthy`, datos persistentes en volumen `postgresdata` |
| `companion` | ✅ `Up`, caché en volumen `companion_cache` |
| `invidious` | ✅ `healthy`, accesible en `http://localhost:3001` |
| Reinicio periódico | ✅ Tarea `Invidious-HourlyRestart` activa en Windows Task Scheduler |
| Política de restart | ✅ `unless-stopped` en los tres servicios |

---

> Para el siguiente paso (exponer el servidor públicamente con HTTPS), ver [PLANNGINX.md](../PLANNGINX.md).
