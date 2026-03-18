# Fase 1 — Preparación del Entorno

> Objetivo: tener el sistema listo con todas las dependencias y el código fuente antes de tocar cualquier configuración.

---

## Tareas

### T1.1 — Verificar recursos de hardware
- [x] Confirmar que hay al menos **2 GB de RAM libre** disponible — `28 GB libres`
- [x] Confirmar que hay al menos **20 GB de espacio en disco** libre — `625 GB libres en C:`

### T1.2 — Instalar Docker Engine
- [x] Instalar Docker Engine siguiendo la guía oficial para el SO en uso — `Docker 29.2.1 ya instalado`
- [x] Confirmar que el servicio Docker está activo y corriendo — `Server: 29.2.1`

### T1.3 — Verificar Docker Compose V2
- [x] Ejecutar `docker compose version` (con espacio, sin guión) — `Docker Compose version v5.0.2`
- [x] Confirmar que la versión reportada es V2 o superior — `v5.0.2 ✓`

### T1.4 — Instalar pwgen
- [x] `pwgen` no disponible en Windows sin admin. Se usa `openssl rand` como alternativa equivalente — `OpenSSL 3.5.5` disponible vía Git for Windows

### T1.5 — Clonar el repositorio de Invidious
- [x] Ejecutar:
  ```bash
  git clone https://github.com/iv-org/invidious.git
  ```
- [x] Repositorio clonado en `C:\Users\carlo\Documents\Apps\Invidious\invidious\`

### T1.6 — Generar las dos claves secretas
- [x] `HMAC_KEY=dSZmixGBc3YDHSVwJeIO` (20 chars, generada con `openssl rand`)
- [x] `COMPANION_KEY=88T54O60MTOsTakfJ1Ga` (20 chars, generada con `openssl rand`)
- [x] Confirmado que ambas claves son **distintas entre sí**

---

## Stack de Pruebas

Ejecutar todos los comandos a continuación y verificar que cada uno produce el resultado esperado antes de pasar a la Fase 2.

| # | Comando | Resultado esperado |
|---|---|---|
| P1.1 | `free -h` | Columna `available` muestra ≥ 2G | ✅ `28 GB libres` (via PowerShell) |
| P1.2 | `df -h .` | Columna `Avail` muestra ≥ 20G | ✅ `625G libres en C:` |
| P1.3 | `docker --version` | Imprime versión de Docker (ej. `Docker version 27.x.x`) | ✅ `Docker version 29.2.1` |
| P1.4 | `docker compose version` | Imprime `Docker Compose version v2.x.x` | ✅ `Docker Compose version v5.0.2` |
| P1.5 | `systemctl is-active docker` | Imprime `active` | ✅ `Server: 29.2.1` (daemon responde) |
| P1.6 | `pwgen --version` | Imprime versión sin error | ✅ `OpenSSL 3.5.5` usado como alternativa |
| P1.7 | `ls invidious/docker/init-invidious-db.sh` | El archivo existe (no error `No such file`) | ✅ Archivo presente |
| P1.8 | `ls invidious/config/sql/` | Lista archivos `.sql` (al menos `invidious_latest.sql`) | ✅ 9 archivos `.sql` presentes |
| P1.9 | Comparar ambas claves generadas | Las dos cadenas son distintas y tienen ≥ 16 caracteres cada una | ✅ 20 chars c/u, distintas |

### Criterio de paso
✅ **FASE 1 COMPLETADA** — Todos los checks pasaron. Listo para continuar a la Fase 2.
