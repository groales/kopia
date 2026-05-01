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
- Definir valores seguros en `.env` para `USERNAME`, `SECRET_PASSWORD` y `KOPIA_PASSWORD`.
- Definir rutas reales en `.env` para `KOPIA_DATA_DIR` y `KOPIA_REPOSITORY_DIR`.
- Permisos de escritura sobre `./config`, `./cache`, `./logs` y `KOPIA_REPOSITORY_DIR`.

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

Usa valores distintos para `SECRET_PASSWORD` y `KOPIA_PASSWORD`.

En el despliegue:
- `SECRET_PASSWORD`: password de acceso al servidor web/API.
- `KOPIA_PASSWORD`: password del repositorio de backups.

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

El `compose.yaml` usa estas variables automaticamente desde `.env`.

### 3. Revisar `compose.yaml`

```yaml
services:
  kopia:
    image: kopia/kopia:latest
    hostname: ${HOSTNAME:-kopia}
    container_name: ${CONTAINER_NAME:-kopia}
    restart: unless-stopped
    ports:
      - 51515:51515
    # Setup the server that provides the web gui
    command:
      - server
      - start
      - --disable-csrf-token-checks
      - --insecure
      - --address=0.0.0.0:51515
      - --server-username=${USERNAME:-admin}
      - --server-password=${SECRET_PASSWORD}
    environment:
      # Set repository password
      KOPIA_PASSWORD: ${KOPIA_PASSWORD}
      USER: ${USERNAME:-admin}
    volumes:
      # Mount local folders needed by kopia
      - ./config:/app/config
      - ./cache:/app/cache
      - ./logs:/app/logs
      # Mount local folders to snapshot
      - ${KOPIA_DATA_DIR}:/data:ro
      # Mount repository location
      - ${KOPIA_REPOSITORY_DIR}:/repository

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
docker-compose exec kopia sh -lc "kopia repository create filesystem --path=/repository --password=$KOPIA_PASSWORD"
```

---

## Metodo Alternativo: Crear Manualmente

Puedes copiar `compose.yaml` y `.env.example` a una carpeta nueva y ejecutar los mismos pasos.

---

## Acceso Inicial

- UI/API de Kopia: `http://IP_DEL_SERVIDOR:51515`
- Usuario: valor de `USERNAME` en `.env`
- Password: valor de `SECRET_PASSWORD` en `.env`

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
├── KOPIA_DATA_DIR         -> /data (solo lectura)
└── KOPIA_REPOSITORY_DIR   -> /repository
```

> **IMPORTANTE**: `KOPIA_REPOSITORY_DIR` debe apuntar a almacenamiento separado (NAS/disco externo) para no perder backups junto con el host principal.

## Configuracion Avanzada

- Para backend remoto (S3, B2, etc.), crea/conecta repositorio con los comandos oficiales de Kopia.
- Ajusta politicas de snapshots y retencion por host/path.
- Si usas proxy inverso, puedes retirar el puerto directo `51515`.

## Solucion de Problemas

- Si no inicia:
  - `docker-compose logs --tail 200 kopia`
- Si falla login:
  - verifica `USERNAME` y `SECRET_PASSWORD` en `.env`.
- Si falla la creacion del repositorio:
  - valida permisos en `KOPIA_REPOSITORY_DIR`.

## Seguridad

- Usa passwords fuertes y unicas.
- Publica por HTTPS si hay acceso externo.
- Mantiene `/data` en modo solo lectura para minimizar riesgo.
- Protege backups con cifrado y control de acceso.

## Backup y Restauracion

- Backup: respalda periodicamente `KOPIA_REPOSITORY_DIR` y la configuracion (`./config`, `./cache`, `./logs`).
- Restauracion: vuelve a montar esas rutas y levanta el servicio.

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
