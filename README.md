# 📧 OpenArchiver - Plataforma moderna de archivado de emails autohospedada

[![GitHub Stars](https://img.shields.io/github/stars/LogicLabs-OU/OpenArchiver?style=for-the-badge&logo=github)](https://github.com/LogicLabs-OU/OpenArchiver)
[![GitHub Release](https://img.shields.io/github/v/release/LogicLabs-OU/OpenArchiver?style=for-the-badge&logo=github)](https://github.com/LogicLabs-OU/OpenArchiver/releases)
[![Docker Pulls](https://img.shields.io/docker/pulls/logiclabshq/open-archiver?style=for-the-badge&logo=docker)](https://hub.docker.com/r/logiclabshq/open-archiver)
[![License: AGPL-3.0](https://img.shields.io/badge/License-AGPL--3.0-blue.svg?style=for-the-badge)](https://www.gnu.org/licenses/agpl-3.0.html)
[![Discord](https://img.shields.io/badge/Discord-Community-5865F2?style=for-the-badge&logo=discord)](https://discord.gg/openarchiver)

## 📋 Descripción general

**OpenArchiver** es una plataforma moderna y profesional de archivado de emails con cumplimiento legal, diseñada para organizaciones que necesitan control total, privacidad y soberanía de datos. A diferencia de soluciones SaaS (que cobran por usuario/volumen), OpenArchiver es **open source, self-hosted**, y trata la retención de emails como una prioridad de cumplimiento.

Almacena emails en formato **.eml estándar** (no propietario), encriptados en reposo, con deduplicación y compresión que reducen el almacenamiento. Soporta backend **S3/MinIO** para escala, búsqueda full-text indexada vía **Meilisearch**, descubrimiento de threads (contexto de conversaciones), políticas de retención automáticas e integridad criptográfica verificable.

Stack moderno: **SvelteKit** frontend, **Node.js** backend, **PostgreSQL**, **Valkey** (Redis), **Meilisearch**. Despliegue sencillo con **Docker Compose**. Licencia **AGPL-3.0**.

## ✨ Características principales

- **Ingesta universal**: Gmail/Google Workspace, Microsoft 365, IMAP, archivos PST
- **Sincronización continua** en tiempo real (real-time sync)
- **Almacenamiento .eml estándar** con deduplicación y compresión
- **Encriptación en reposo** (AES-256)
- **Storage backends pluggables**: filesystem local o S3/MinIO
- **Búsqueda full-text blazingly fast** con Meilisearch (emails + adjuntos PDF, DOCX)
- **Thread discovery**: agrupa conversaciones y presenta contexto automáticamente
- **Faceted search**: filtros por fecha, sender, recipient, subject, tags
- **Retention policies** granulares con auto-delete configurable (GDPR, HIPAA, SOX)
- **Integridad criptográfica**: file hash, encriptación, reportes de integridad (PDF)
- **Auditoría completa**: access logs, quién vio qué y cuándo
- **Multi-usuario con RBAC**: roles y permisos granulares
- **API OpenAPI** autogenerada y documentada
- **Despliegue Docker Compose** production-ready

## 📋 Requisitos del sistema

- **Docker** y **Docker Compose** (v2+)
- **4 GB RAM** mínimo (2 GB si usas PostgreSQL/Valkey/Meilisearch externos)
- **20+ GB** espacio en disco (depende del volumen de emails + adjuntos)
- **Puerto 3000** (frontend, configurable via `PORT_FRONTEND`)
- **PostgreSQL 17+** (incluido en compose)
- **Valkey/Redis** (incluido en compose, para job queue)
- **Meilisearch** (incluido en compose, para búsqueda)
- Storage local o **S3/MinIO** (para emails)
- Navegador moderno

## 🐳 Instalación

### Paso 1: Clonar repositorio
```bash
git clone https://github.com/LogicLabs-OU/OpenArchiver.git
cd OpenArchiver
```

### Paso 2: Configurar variables de entorno
```bash
cp .env.example .env
nano .env
```

**Variables importantes a configurar:**
```env
# Base de datos
POSTGRES_USER=admin
POSTGRES_PASSWORD=tu_password_seguro
POSTGRES_DB=open_archive

# Frontend
PORT_FRONTEND=3000

# Almacenamiento emails
STORAGE_LOCAL_ROOT_PATH=/data/storage

# Claves secretas (se generan automáticamente si no existen)
# JWT_SECRET, ENCRYPTION_KEY, MEILI_MASTER_KEY, etc.
```

### Paso 3: Iniciar OpenArchiver
```bash
docker compose up -d

# Espera 1-2 minutos para que todos los servicios inicien
docker compose logs -f open-archiver
```

### Paso 4: Acceder a la interfaz
Abre **http://localhost:3000** — Verás la página de setup inicial.

---

### docker-compose.yml (referencia)
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:17-alpine
    container_name: openarchiver-postgres
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - openarchiver-network

  valkey:
    image: valkey/valkey:8-alpine
    container_name: openarchiver-valkey
    command: valkey-server --appendonly yes
    volumes:
      - valkey_data:/data
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - openarchiver-network

  meilisearch:
    image: getmeili/meilisearch:v1.11
    container_name: openarchiver-meilisearch
    environment:
      MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}
    volumes:
      - meilisearch_data:/meili_data
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:7700/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - openarchiver-network

  open-archiver:
    image: logiclabshq/open-archiver:latest
    container_name: open-archiver
    environment:
      - NODE_ENV=production
      - PORT=${PORT_FRONTEND}
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      - VALKEY_URL=redis://valkey:6379
      - MEILISEARCH_HOST=http://meilisearch:7700
      - MEILISEARCH_API_KEY=${MEILI_MASTER_KEY}
      - STORAGE_LOCAL_ROOT_PATH=${STORAGE_LOCAL_ROOT_PATH}
      - JWT_SECRET=${JWT_SECRET}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      # OAuth Google
      - GOOGLE_CLIENT_ID=${GOOGLE_CLIENT_ID}
      - GOOGLE_CLIENT_SECRET=${GOOGLE_CLIENT_SECRET}
      # OAuth Microsoft
      - MICROSOFT_CLIENT_ID=${MICROSOFT_CLIENT_ID}
      - MICROSOFT_CLIENT_SECRET=${MICROSOFT_CLIENT_SECRET}
    ports:
      - "${PORT_FRONTEND}:3000"
    volumes:
      - storage_data:${STORAGE_LOCAL_ROOT_PATH}
    depends_on:
      postgres:
        condition: service_healthy
      valkey:
        condition: service_healthy
      meilisearch:
        condition: service_healthy
    networks:
      - openarchiver-network
    restart: unless-stopped

volumes:
  postgres_data:
  valkey_data:
  meilisearch_data:
  storage_data:

networks:
  openarchiver-network:
    driver: bridge
```

## ⚙️ Configuración

1. **Credenciales de base de datos**: Define `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` en `.env`
2. **Puerto frontend**: Ajusta `PORT_FRONTEND` (default 3000)
3. **Ruta de almacenamiento**: Configura `STORAGE_LOCAL_ROOT_PATH` (ej. `/data/storage`)
4. **Claves secretas**: El script de entrypoint genera `JWT_SECRET`, `ENCRYPTION_KEY`, `MEILI_MASTER_KEY` automáticamente si no existen
5. **OAuth Google Workspace**: Crea credenciales en Google Cloud Console → `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`
6. **OAuth Microsoft 365**: Registra app en Azure AD → `MICROSOFT_CLIENT_ID`, `MICROSOFT_CLIENT_SECRET`
7. **S3/MinIO (opcional)**: Configura `STORAGE_S3_*` variables para usar backend objeto
8. **HTTPS en producción**: Usa reverse proxy (Caddy/Traefik/Nginx) terminando TLS

## 🚀 Primeros pasos

1. **Setup inicial**: Abre http://localhost:3000 → Ingresa usuario admin + password fuerte → Completa setup → Login
2. **Agregar fuente Gmail**: Dashboard → "Add Email Source" → Tipo: Google Workspace → "Connect to Google" → Autoriza OAuth → Nombre: "Mi Gmail" → Save → Sync automático inicia
3. **Ver progreso sincronización**: Dashboard → "Ingestion Status" → Emails descargados, último sync, estado
4. **Buscar emails (full-text)**: Dashboard → "Search" → Escribe palabra clave (subject, body, sender) → Resultados instantáneos
5. **Búsqueda avanzada**: Search → "Advanced" → Filtros: rango fecha, sender, recipient, tags → Combina criterios
6. **Thread discovery**: Click email en resultados → OpenArchiver detecta thread automáticamente → Muestra conversación completa
7. **Agregar Microsoft 365**: Dashboard → "Add Email Source" → Tipo: Microsoft 365 → "Connect to Microsoft" → Autoriza → Save
8. **Configurar retención**: Settings → "Retention Policies" → Crea policy: "Delete emails older than 7 years" → Aplica automático
9. **Ver integridad**: Settings → "Integrity Report" → Genera PDF con auditoría criptográfica (hashes, verificación encriptación)
10. **Agregar usuarios**: Settings → "Users" → "Add User" → Email, password, role (admin/user) → Permisos granulares por source

## 💡 Casos de uso

- **Equipos legales / eDiscovery**: Búsqueda potente de emails para litigio, integridad verificable
- **Cumplimiento regulatorio**: GDPR, HIPAA, SOX — Retention policies, auditoría, reportes de integridad
- **Gobierno / Administración pública**: Retención records, FOIA compliance, archivos permanentes
- **Empresas grandes**: Consolidación multi-cuenta, búsqueda centralizada, migración de datos
- **Organizaciones compliance-heavy**: Auditoría completa, integridad criptográfica, reportes PDF

## 🔒 Acceso remoto seguro

### Con Caddy (recomendado - HTTPS automático)
```caddyfile
archive.tudominio.com {
    reverse_proxy localhost:3000
}
```
Accede en **https://archive.tudominio.com** con HTTPS automático (Let's Encrypt).

> **Nota**: Ajusta `PORT_FRONTEND` en `.env` si usas puerto distinto.

## 🛠️ Gestión y mantenimiento

### Ver logs
```bash
docker compose logs -f open-archiver
docker compose logs -f postgres
docker compose logs -f meilisearch
```

### Backup de base de datos (CRÍTICO)
```bash
docker compose exec postgres pg_dump -U admin open_archive > openarchiver-$(date +%Y%m%d).sql
```

### Backup de storage (emails .eml)
```bash
rsync -av /path/to/storage /backups/openarchiver-storage-$(date +%Y%m%d)
```

### Reiniciar servicios
```bash
docker compose restart
```

### Actualizar a versión más reciente
```bash
docker compose pull
docker compose up -d
```

### Monitorear consumo de recursos
```bash
docker stats open-archiver postgres meilisearch valkey
```

### Mantenimiento Meilisearch (reindexado)
```bash
# Solo si la búsqueda parece lenta (raro)
docker compose exec open-archiver npm run reindex
```

## 📝 Licencia

**AGPL-3.0** — Código abierto, uso comercial permitido, modificaciones deben compartirse bajo misma licencia. Ver [LICENSE](https://github.com/LogicLabs-OU/OpenArchiver/blob/main/LICENSE).

---

> 📖 **Artículo original**: [Cómo instalar OpenArchiver en Docker - Plataforma moderna de archivado de emails autohospedado en Docker](https://genbyte.blogspot.com/2026/07/como-instalar-openarchiver-en-docker.html)  
> 🌐 **Web oficial**: [openarchiver.com](https://openarchiver.com) | 🐙 **GitHub**: [LogicLabs-OU/OpenArchiver](https://github.com/LogicLabs-OU/OpenArchiver) | 🐳 **Docker Hub**: [logiclabshq/open-archiver](https://hub.docker.com/r/logiclabshq/open-archiver)