
# Tutorial: Despliegue de n8n con Docker Rootless, Traefik y SSL en Debian

Este documento detalla paso a paso cómo configurar un servidor seguro utilizando Docker en modo **Rootless** (sin privilegios de root) para desplegar n8n con HTTPS automático.

## 📋 Requisitos Previos
* Servidor VPS con Debian 12/13 limpio.
* Acceso inicial como `root`.
* Dominio apuntando a la IP del servidor (ej: `elvismacas.com`).

---

## 🚀 Fase 1: Preparación del Sistema (Como Root)

Primero, preparamos el sistema operativo para permitir que un usuario normal ejecute contenedores y exponga puertos web (80/443).

```bash
# 1. Instalar dependencias necesarias para Rootless y gestión de usuarios
apt-get update && apt-get install -y uidmap dbus-user-session fuse-overlayfs curl

# 2. Cargar el módulo del kernel nf_tables (Crítico para la red en Debian)
modprobe nf_tables
echo "nf_tables" >> /etc/modules

# 3. Crear el usuario que administrará Docker (ej: 'docker-admin')
useradd -m -s /bin/bash docker-admin
passwd docker-admin

# 4. Habilitar persistencia del servicio (Linger)
# Esto permite que los contenedores sigan corriendo cuando cierras sesión SSH.
loginctl enable-linger docker-admin

# 5. Permitir puertos bajos (80 y 443) sin ser root
echo 'net.ipv4.ip_unprivileged_port_start=0' > /etc/sysctl.d/90-docker-rootless.conf
sysctl --system

```

---

## 🐳 Fase 2: Instalación de Docker Rootless (Como Usuario)

Ahora cambiamos al usuario `docker-admin` para realizar la instalación segura.

```bash
# Cambiar de usuario
su - docker-admin

# Instalar Docker Rootless (Script oficial)
curl -fsSL [https://get.docker.com/rootless](https://get.docker.com/rootless) | sh

```

### Configuración de Variables de Entorno

Al finalizar la instalación, debemos decirle a la terminal dónde está Docker. Agrega esto al final de tu `.bashrc`:

```bash
# Abrir archivo de configuración
nano ~/.bashrc

# --- PEGAR AL FINAL ---
export PATH=/home/docker-admin/bin:$PATH
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
# ----------------------

# Aplicar cambios
source ~/.bashrc

```

### Iniciar el servicio

```bash
systemctl --user start docker
systemctl --user enable docker

```

---

## 🛠 Fase 3: Instalar Docker Compose

El instalador rootless a veces no incluye el plugin de Compose. Lo instalamos manualmente:

```bash
mkdir -p ~/.docker/cli-plugins/
curl -SL [https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64](https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64) -o ~/.docker/cli-plugins/docker-compose
chmod +x ~/.docker/cli-plugins/docker-compose

# Verificar versión
docker compose version

```

---

## 📦 Fase 4: Despliegue de n8n + Traefik

No usaremos el repositorio por defecto de n8n (que es para desarrollo), crearemos una estructura limpia para producción.

### 1. Estructura de Directorios

```bash
cd ~
mkdir -p n8n-stack/letsencrypt
mkdir -p n8n-stack/postgres_data
mkdir -p n8n-stack/n8n_data
cd n8n-stack

```

### 2. Archivo `.env`

Crea el archivo `.env` con tus secretos.
**Nota:** No definimos `DOCKER_SOCKET` aquí porque lo pondremos directo en el YAML para evitar errores de lectura.

```bash
nano .env

```

*(Contenido)*:

```env
DOMAIN_NAME=tu-dominio.com
SSL_EMAIL=admin@tu-dominio.com

# Base de Datos
POSTGRES_USER=n8n_user
POSTGRES_PASSWORD=TuPasswordSeguroDB
POSTGRES_DB=n8n

# n8n
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=TuPasswordPanel
N8N_ENCRYPTION_KEY=GeneraUnaClaveLargaAqui
GENERIC_TIMEZONE=America/Guayaquil

```

### 3. Archivo `compose.yml` (Configuración Definitiva)

Este archivo tiene la corrección crítica para que Traefik encuentre el socket en modo Rootless.

```bash
nano compose.yml

```

```yaml
services:
  traefik:
    image: traefik:v3
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      # RUTA HARDCODED: Apunta directo al socket del usuario 1000
      - /run/user/1000/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt
    command:
      - "--api.dashboard=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=${SSL_EMAIL}"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    networks:
      - n8n-net

  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}
      - N8N_HOST=${DOMAIN_NAME}
      - WEBHOOK_URL=https://${DOMAIN_NAME}/
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
    ports:
      - "127.0.0.1:5678:5678"
    volumes:
      - ./n8n_data:/home/node/.n8n
    networks:
      - n8n-net
    depends_on:
      postgres:
        condition: service_healthy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${DOMAIN_NAME}`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=myresolver"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"

networks:
  n8n-net:
    driver: bridge

```

### 4. Iniciar Servicios

```bash
docker compose up -d

```

---

## 🚨 Solución de Problemas (Troubleshooting)

Durante la instalación pueden surgir estos errores clave. Aquí explicamos por qué pasan y cómo se solucionan.

### Error 1: "Docker Daemon not found / Missing system requirements"

* **Síntoma:** Al instalar docker, se queja de `iptables` o `nf_tables`.
* **Causa:** Debian a veces no carga el módulo de filtrado de red por defecto.
* **Solución:** Ejecutar `modprobe nf_tables` como **root**.

### Error 2: "Traefik error: Permission denied /var/run/docker.sock"

* **Síntoma:** Traefik no arranca o dice que no puede conectar al daemon. Si revisas el contenedor (`ls -ld /var/run/docker.sock`), ves que es un directorio (`drwxr...`) en lugar de un archivo.
* **Causa:** Docker intentó montar el socket, pero como la ruta estaba mal o la variable vacía, creó una carpeta vacía para "llenar el hueco".
* **Solución:**
1. Apagar todo: `docker compose down` (para borrar la carpeta falsa).
2. Poner la ruta explícita en el `compose.yml`: `/run/user/1000/docker.sock`.
3. Levantar de nuevo.



### Error 3: "n8n EACCES: permission denied, open config"

* **Síntoma:** El contenedor de n8n se reinicia constantemente.
* **Causa:** En modo Rootless, el usuario interno (ID 1000) no tiene permiso para escribir en las carpetas creadas en el host.
* **Solución Rápida (Insegura):** `chmod -R 777 n8n_data`
* **Solución Correcta:** Asignar el dueño correcto usando un contenedor temporal:
```bash
docker run --rm -v ./n8n_data:/data alpine chown -R 1000:1000 /data

```



### Error 4: "Permission denied" en puertos 80/443

* **Síntoma:** Traefik falla al intentar bindear los puertos web.
* **Causa:** Linux prohíbe a usuarios normales usar puertos < 1024.
* **Solución:** Ejecutar como **root**: `sysctl net.ipv4.ip_unprivileged_port_start=0`.
