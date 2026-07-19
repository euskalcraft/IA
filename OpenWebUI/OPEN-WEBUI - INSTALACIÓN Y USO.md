---
tags:
  - homelab
  - ia
  - llm
  - docker
  - open-webui
fecha: 2026-07-16
servidor: HL2-1 (192.168.1.35)
puerto: 3000
---

# 🌐 Open WebUI — Instalación y Uso

> [!info] Resumen
> Interfaz web moderna para conectar con backends LLM locales ([[OLLAMA - INSTALACIÓN Y CONFIGURACIÓN COMPLETA]] o [[LLAMA.CPP - INSTALACIÓN Y MODELOS]]). Se despliega vía Docker Compose en Ubuntu Server 24.04.

---

## 🏗️ Arquitectura

```text
Ubuntu Server 24.04 (HL2-1: 192.168.1.35)
│
├── Ollama (nativo, systemd)          ← Puerto 11434
│   └── Modelos en /home/escuela/modelos
│
├── llama.cpp (proceso directo)        ← Puerto 8080 (alternativa a Ollama)
│   └── Modelos GGUF en ~/mi_modelo_*/
│
├── Docker Engine
│   └── Open WebUI (contenedor)       ← Puerto 3000 → UI web
│       ├── Conecta con Ollama: http://host.docker.internal:11434
│       └── Datos persistentes: /opt/open-webui/data/
```

> [!tip] Backend activo
> Usa **Ollama** o **llama-server**, no ambos a la vez — compiten por GPU y RAM.

---

## 📋 Requisitos previos

| Requisito | Estado |
|-----------|--------|
| Ubuntu Server 24.04 | ✅ |
| Docker Engine instalado | Ver [[CS-01 Docker]] |
| Backend LLM corriendo (Ollama o llama.cpp) | Verificar puerto |

---

## 🛠️ Instalación paso a paso

### 1. Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y
```

### 2. Instalar Docker Engine (si no está ya)

```bash
# Dependencias para repositorio oficial
sudo apt install -y ca-certificates curl gnupg lsb-release

# Clave GPG de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Repositorio oficial
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker + Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verificar
docker --version
docker compose version
sudo docker run hello-world  # ✅ "Hello from Docker!"

# Permitir usar Docker sin sudo
sudo usermod -aG docker $USER
newgrp docker
docker ps  # Debe funcionar sin sudo
```

### 3. Crear directorio de Open WebUI

```bash
sudo mkdir -p /opt/open-webui
sudo chown -R $USER:$USER /opt/open-webui
cd /opt/open-webui
```

### 4. Crear `docker-compose.yml`

```bash
nano docker-compose.yml
```

**Contenido:**

```yaml
services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://host.docker.internal:11434
    extra_hosts:
      - host.docker.internal:host-gateway
    volumes:
      - ./data:/app/backend/data
```

### 5. Descargar imagen y levantar contenedor

```bash
docker compose pull
docker compose up -d
```

### 6. Verificar estado

```bash
docker ps
# ✅ Debe aparecer 'open-webui' en estado Up
```
```bash
#Consultar registros del docker
docker compose logs -f
```
---

## 🌐 Acceso y configuración inicial

### Abrir la interfaz web

```
http://IPDELSERVIDOR:3000
```

En el primer acceso se crea el usuario administrador.

### Conectar con Ollama (nativo)

Ir a **Settings → Connections** y verificar que la URL sea:

```
http://host.docker.internal:11434
```

Si Ollama funciona correctamente, los modelos aparecen automáticamente.

### Conectar con llama.cpp server

En **Settings → Connections → OpenAI API**:

- **Base URL:** `http://192.168.1.35:8080/v1`
- **API Key:** (vacío — llama-server no requiere key por defecto)

---

## 📊 Administración rápida

| Acción | Comando |
|--------|---------|
| Iniciar | `cd /opt/open-webui && docker compose up -d` |
| Detener | `cd /opt/open-webui && docker compose down` |
| Reiniciar | `cd /opt/open-webui && docker compose restart` |
| Estado | `cd /opt/open-webui && docker compose ps` |
| Logs en vivo | `cd /opt/open-webui && docker compose logs -f` |
| Actualizar | `cd /opt/open-webui && docker compose pull && docker compose up -d` |
| Consumo recursos | `docker stats` |

---

## 💾 Datos persistentes

Todos los datos se almacenan en:

```text
/opt/open-webui/data/
```

Contiene: usuarios, conversaciones, configuración, historial y base de datos.

### Copia de seguridad

```bash
tar -czvf open-webui-backup.tar.gz /opt/open-webui/data
```

### Restauración

```bash
cd /opt/open-webui && docker compose down
tar -xzvf open-webui-backup.tar.gz -C /
docker compose up -d
```

> [!tip] Los datos permanecen intactos entre actualizaciones gracias al volumen persistente.

---

## 🔗 Notas relacionadas

- [[OLLAMA - INSTALACIÓN Y CONFIGURACIÓN COMPLETA]] — Backend LLM nativo (recomendado para uso general)
- [[LLAMA.CPP - INSTALACIÓN Y MODELOS]] — Backend LLM avanzado (control total, modelos grandes)
- [[IA - ARQUITECTURA GENERAL]] — Vista global de la infraestructura IA
