# Fase 1 — Preparación del Entorno

> Objetivo: tener el sistema listo con todas las dependencias y el código fuente antes de tocar cualquier configuración.

---

## Notas importantes (aprendidas en ejecución previa)

- **Windows:** `pwgen` requiere permisos de administrador para instalar vía Chocolatey. Usar `openssl rand` como alternativa (incluido en Git for Windows).
- **Las claves deben tener exactamente 16 caracteres** — el companion falla si recibe una clave de longitud diferente.
- Generar las claves **antes** de crear el `docker-compose.yml` para tenerlas a mano en la Fase 2.

---

## Tareas

### T1.1 — Verificar recursos de hardware
- [ ] Confirmar que hay al menos **2 GB de RAM libre** disponible
- [ ] Confirmar que hay al menos **20 GB de espacio en disco** libre

### T1.2 — Instalar Docker Engine
- [ ] Instalar Docker Engine siguiendo la guía oficial para el SO en uso
- [ ] Confirmar que el servicio Docker está activo y corriendo

### T1.3 — Verificar Docker Compose V2
- [ ] Ejecutar `docker compose version` (con espacio, sin guión)
- [ ] Confirmar que la versión reportada es V2 o superior

### T1.4 — Instalar generador de claves
- [ ] **Linux/macOS:** instalar `pwgen`:
  ```bash
  apt install pwgen      # Debian/Ubuntu
  brew install pwgen     # macOS
  ```
- [ ] **Windows (sin admin):** verificar que `openssl` esté disponible (incluido en Git for Windows):
  ```bash
  openssl version
  ```

### T1.5 — Clonar el repositorio de Invidious
- [ ] Ejecutar desde el directorio donde se quiere instalar:
  ```bash
  git clone https://github.com/iv-org/invidious.git
  cd invidious
  ```

### T1.6 — Generar las dos claves secretas (exactamente 16 caracteres)
- [ ] Generar `HMAC_KEY` (exactamente 16 chars) y guardarla:
  ```bash
  # Linux/macOS con pwgen:
  pwgen 16 1

  # Windows / alternativa con openssl:
  openssl rand -base64 16 | tr -d '=+/\n' | head -c 16
  ```
- [ ] Generar `COMPANION_KEY` (exactamente 16 chars) ejecutando el mismo comando nuevamente
- [ ] Guardar ambos valores — se necesitarán en la Fase 2
- [ ] Confirmar que ambas claves son **distintas entre sí**

### T1.7 — Corregir line endings del script de inicialización (Windows)
- [ ] En Windows el script puede tener CRLF, lo que impide que el contenedor Linux lo ejecute:
  ```bash
  sed -i 's/\r//' invidious/docker/init-invidious-db.sh
  file invidious/docker/init-invidious-db.sh  # debe decir "ASCII text executable", NO "CRLF"
  ```

---

## Stack de Pruebas

Ejecutar todos los comandos a continuación y verificar que cada uno produce el resultado esperado antes de pasar a la Fase 2.

| # | Comando | Resultado esperado |
|---|---|---|
| P1.1 | `free -h` (Linux) o `powershell -Command "(Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory/1MB"` (Windows) | ≥ 2 GB libres |
| P1.2 | `df -h .` | Columna `Avail` muestra ≥ 20G |
| P1.3 | `docker --version` | Imprime versión de Docker |
| P1.4 | `docker compose version` | Imprime `Docker Compose version v2.x.x` o superior |
| P1.5 | `systemctl is-active docker` (Linux) o `docker info` (Windows) | `active` / sin error |
| P1.6 | `pwgen 16 1` o `openssl rand -base64 16 \| tr -d '=+/\n' \| head -c 16` | Cadena de 16 caracteres |
| P1.7 | `ls invidious/docker/init-invidious-db.sh` | El archivo existe |
| P1.8 | `ls invidious/config/sql/` | Lista archivos `.sql` |
| P1.9 | `file invidious/docker/init-invidious-db.sh` | `ASCII text executable` (sin `CRLF`) |
| P1.10 | Comparar ambas claves generadas | Distintas entre sí, exactamente 16 caracteres cada una |

### Criterio de paso
Todos los checks deben pasar sin errores. Si alguno falla, resolver el problema en esta fase antes de continuar.
