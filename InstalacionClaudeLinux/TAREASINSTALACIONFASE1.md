# Fase 1 — Preparación del Entorno (Linux)

> Objetivo: tener el sistema listo con todas las dependencias y el código fuente antes de tocar cualquier configuración.
>
> Sistema operativo: Linux (Debian/Ubuntu o derivados). Ajustar gestores de paquetes para otras distros (`dnf`, `pacman`, etc.).

---

## Notas importantes

- **Las claves deben tener exactamente 16 caracteres** — el companion falla con cualquier otra longitud.
- Generar las claves **antes** de crear el `docker-compose.yml` para tenerlas a mano en la Fase 2.
- En Linux, `git clone` descarga los archivos con line endings LF nativos. No se requiere conversión de CRLF.
- Si Docker se instala como paquete del sistema (`apt install docker.io`), puede venir con una versión antigua sin Compose V2. Instalar desde los repositorios oficiales de Docker para garantizar la versión correcta.
- Si el usuario actual no pertenece al grupo `docker`, todos los comandos `docker` requerirán `sudo`. Agregar el usuario al grupo evita esto (ver T1.2).

---

## Tareas

### T1.1 — Verificar recursos de hardware
- [ ] Confirmar que hay al menos **2 GB de RAM libre** disponible:
  ```bash
  free -h
  ```
- [ ] Confirmar que hay al menos **20 GB de espacio en disco** libre en la partición de destino:
  ```bash
  df -h .
  ```

### T1.2 — Instalar Docker Engine
- [ ] Instalar Docker Engine desde los repositorios oficiales de Docker:
  ```bash
  # Debian/Ubuntu
  curl -fsSL https://get.docker.com | sh
  ```
- [ ] Agregar el usuario actual al grupo `docker` para ejecutar sin `sudo`:
  ```bash
  sudo usermod -aG docker $USER
  newgrp docker   # aplica el cambio sin cerrar sesión
  ```
- [ ] Habilitar e iniciar el servicio:
  ```bash
  sudo systemctl enable docker
  sudo systemctl start docker
  ```

### T1.3 — Verificar Docker Compose V2
- [ ] Ejecutar (con espacio, sin guión):
  ```bash
  docker compose version
  ```
- [ ] Confirmar que la versión reportada es **v2.x.x o superior**
- [ ] Si aparece `docker: 'compose' is not a docker command`, instalar el plugin:
  ```bash
  sudo apt install docker-compose-plugin
  ```

### T1.4 — Instalar pwgen
- [ ] Instalar `pwgen`:
  ```bash
  sudo apt install pwgen        # Debian/Ubuntu
  sudo dnf install pwgen        # Fedora/RHEL
  sudo pacman -S pwgen          # Arch
  ```
- [ ] Verificar que está disponible:
  ```bash
  pwgen --version
  ```

### T1.5 — Clonar el repositorio de Invidious
- [ ] Ejecutar desde el directorio donde se quiere instalar:
  ```bash
  git clone https://github.com/iv-org/invidious.git
  cd invidious
  ```
- [ ] Verificar que los archivos críticos están presentes:
  ```bash
  ls docker/init-invidious-db.sh
  ls config/sql/
  ```

### T1.6 — Generar las dos claves secretas (exactamente 16 caracteres)
- [ ] Generar `HMAC_KEY`:
  ```bash
  pwgen 16 1
  ```
- [ ] Generar `COMPANION_KEY` ejecutando el mismo comando nuevamente
- [ ] Anotar ambas claves — se necesitarán en la Fase 2
- [ ] Confirmar que son **distintas entre sí** y tienen exactamente **16 caracteres** cada una:
  ```bash
  echo -n "TU_CLAVE" | wc -c   # debe imprimir 16
  ```

---

## Stack de Pruebas

Ejecutar todos los comandos y verificar el resultado esperado antes de pasar a la Fase 2.

| # | Comando | Resultado esperado |
|---|---|---|
| P1.1 | `free -h` | Columna `available` muestra ≥ 2G |
| P1.2 | `df -h .` | Columna `Avail` muestra ≥ 20G |
| P1.3 | `docker --version` | `Docker version 24.x.x` o superior |
| P1.4 | `docker compose version` | `Docker Compose version v2.x.x` o superior |
| P1.5 | `systemctl is-active docker` | `active` |
| P1.6 | `docker ps` (sin sudo) | Lista vacía o contenedores — sin error de permisos |
| P1.7 | `pwgen 16 1` | Cadena de exactamente 16 caracteres |
| P1.8 | `ls docker/init-invidious-db.sh` | El archivo existe |
| P1.9 | `ls config/sql/` | Lista archivos `.sql` |
| P1.10 | `echo -n "TU_HMAC_KEY" \| wc -c && echo -n "TU_COMPANION_KEY" \| wc -c` | `16` en ambas líneas |
| P1.11 | `[ "$HMAC_KEY" != "$COMPANION_KEY" ] && echo "OK" \|\| echo "FAIL"` | `OK` |

### Criterio de paso
Todos los checks deben pasar sin errores. Si alguno falla, resolver en esta fase antes de continuar.
