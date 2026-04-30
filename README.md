# Kopia

Kopia es una herramienta de backup moderna con deduplicacion, cifrado y snapshots para proteger datos locales y remotos.

Referencia oficial de instalacion: https://kopia.io/docs/installation/

## Caracteristicas

- Backups incrementales con deduplicacion.
- Cifrado de extremo a extremo en el repositorio.
- Politicas de retencion por snapshots.
- Interfaz web y API mediante Kopia Server.
- Soporte para distintos backends de almacenamiento.

## Requisitos Previos

- Docker Engine instalado.
- Docker Compose instalado.
- Red Docker externa `proxy` creada (si vas a publicar mediante proxy inverso).
- Passwords seguras para `KOPIA_SERVER_PASSWORD` y `KOPIA_REPOSITORY_PASSWORD`.
- Ruta externa para `KOPIA_REPOSITORY_PATH`: disco secundario, NAS o almacenamiento de red — **nunca en el mismo disco que los datos a respaldar**.
- Permisos de escritura para el usuario del contenedor en `./config`, `./cache`, `./logs` y en la ruta de `KOPIA_REPOSITORY_PATH`.

## Archivos de este Repositorio

- `compose.yaml` - Servicio Kopia Server.
- `.env.example` - Variables de entorno recomendadas.
- `README.md` - Esta documentacion.
- `LICENSE` - Licencia del repositorio.

---

## Generar Claves Seguras

Antes del despliegue, genera passwords seguras:

```bash
openssl rand -base64 32
```

Usa valores distintos para `KOPIA_SERVER_PASSWORD` y `KOPIA_REPOSITORY_PASSWORD`.

---

## Despliegue con Docker Compose

### 1. Clonar el repositorio

```bash
git clone https://github.com/groales/kopia.git
cd kopia
```

### 2. Preparar `.env`

```bash
cp .env.example .env
```

Edita `.env` y ajusta passwords y rutas.

### 3. Revisar `compose.yaml`

```yaml
services:
  kopia:
    image: kopia/kopia:latest
    container_name: kopia
    restart: unless-stopped
    environment:
      - TZ=${TZ:-Europe/Madrid}
      - KOPIA_PASSWORD=${KOPIA_REPOSITORY_PASSWORD}
      - USER=${KOPIA_USER:-admin}
    volumes:
      - ./config:/app/config
      - ./cache:/app/cache
      - ./logs:/app/logs
      - ${KOPIA_REPOSITORY_PATH}:/repository
      - ${KOPIA_DATA_PATH}:/data:ro
      # Para browsing/mount de snapshots via FUSE
      - ./tmp:/tmp:shared
    cap_add:
      - SYS_ADMIN
    devices:
      - /dev/fuse
    security_opt:
      - apparmor:unconfined
      - seccomp:unconfined
    ports:
      - 51515:51515
    command:
      - server
      - start
      - --insecure
      - --address=0.0.0.0:51515
      - --server-username=${KOPIA_USER:-admin}
      - --server-password=${KOPIA_SERVER_PASSWORD}

networks:
  default:
    external: true
    name: proxy
```

### 4. Levantar el servicio

```bash
docker network create proxy
docker-compose up -d
```

### 5. Crear repositorio de backup (primer arranque)

```bash
docker-compose exec kopia sh -lc "kopia repository create filesystem --path=/repository --password=$KOPIA_REPOSITORY_PASSWORD"
```

---

## Metodo Alternativo: Crear Manualmente

Puedes copiar `compose.yaml` y `.env.example` a una carpeta nueva y ejecutar los mismos pasos.

---

## Acceso Inicial

- UI/API de Kopia: `http://IP_DEL_SERVIDOR:51515`
- Usuario: valor de `KOPIA_USER`
- Password: valor de `KOPIA_SERVER_PASSWORD`

Nota: el ejemplo usa `--insecure` para simplicidad. En produccion, publica Kopia detras de HTTPS con proxy inverso.

## Comandos Utiles

```bash
docker-compose logs -f kopia
docker-compose restart kopia
docker-compose pull
docker-compose up -d
docker-compose down
```

## Estructura de Volumenes

```text
Bind mounts:
├── ./config               -> /app/config
├── ./cache                -> /app/cache
├── ./logs                 -> /app/logs
├── ./tmp                  -> /tmp (shared, para mount de snapshots via FUSE)
├── KOPIA_REPOSITORY_PATH  -> /repository       (disco/NAS externo)
└── KOPIA_DATA_PATH        -> /data (solo lectura)
```

> **IMPORTANTE**: `KOPIA_REPOSITORY_PATH` debe apuntar a un almacenamiento fisicamente separado del host (NAS, disco externo, etc.). Guardar el repositorio en el mismo disco que los datos que se respaldan anula el proposito del backup.

## Configuracion Avanzada

- Para backend remoto (S3, B2, etc.), crea/conecta repositorio con los comandos oficiales de Kopia.
- Ajusta politicas de snapshots y retencion por host/path.
- Si usas proxy inverso, puedes retirar el puerto directo `51515`.

## Solucion de Problemas

- Si no inicia:
  - `docker-compose logs --tail 200 kopia`
- Si falla login:
  - verifica `KOPIA_USER` y `KOPIA_SERVER_PASSWORD` en `.env`.
- Si falla la creacion del repositorio:
  - valida permisos en `KOPIA_REPOSITORY_PATH`.
- Si `kopia snapshot mount` devuelve `/usr/bin/fusermount3: mount failed: Permission denied`:
  - confirma que el servicio mantiene `/dev/fuse`, `cap_add: SYS_ADMIN` y `security_opt` con `apparmor:unconfined` y `seccomp:unconfined`.
  - verifica en el host que existe `/dev/fuse` y que el modulo FUSE esta cargado.

## Seguridad

- Usa passwords fuertes y unicas.
- Publica por HTTPS si hay acceso externo.
- Mantiene `/data` en modo solo lectura para minimizar riesgo.
- Protege backups con cifrado y control de acceso.
- El contenedor requiere `SYS_ADMIN` y acceso a `/dev/fuse` para el mount de snapshots; limita el acceso al host Docker en consecuencia.

## Backup y Restauracion

- Backup: el repositorio ya reside en almacenamiento externo (`KOPIA_REPOSITORY_PATH`). Respalda adicionalmente `./config`.
- Restauracion: ajusta `KOPIA_REPOSITORY_PATH` en `.env` y levanta el servicio. Kopia conectara el repositorio automaticamente si `KOPIA_PASSWORD` es correcta.

## Actualizacion

```bash
docker-compose pull
docker-compose up -d
docker-compose logs -f kopia
```

## Recursos

- Documentacion oficial Kopia: https://kopia.io/docs/
- Instalacion: https://kopia.io/docs/installation/
- Comandos de repositorio: https://kopia.io/docs/reference/command-line/common/repository/
- Proyecto oficial: https://github.com/kopia/kopia

## Licencia

Este repositorio de configuracion es de uso libre. Revisa la licencia del proyecto original en su repositorio oficial.
