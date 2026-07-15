---
tags: [homelab, ia, llm, ollama, linux, self-hosting]
fecha: 2026-07-16
servidor: "HL2-1 (192.168.1.35)"
hardware: "2x NVIDIA Tesla V100 16GB"
os: "Ubuntu Server 24.04"
---

# 🦙 Ollama — Instalación y Configuración Completa

> [!info] Resumen
> Instalación nativa de Ollama en Ubuntu Server 24.04 con systemd, modelos personalizados en `/home/escuela/modelos`, API expuesta en `0.0.0.0:11434`. Compatible con [[OPEN-WEBUI - INSTALACIÓN Y USO]] y [[LLAMA.CPP - INSTALACIÓN Y MODELOS]].

---

## 🎯 Objetivo

- Ollama como servicio nativo (sin Docker) gestionado por `systemd`
- Modelos almacenados en `/home/escuela/modelos`
- API accesible desde la red LAN (`0.0.0.0:11434`)
- Conexión con [[OPEN-WEBUI - INSTALACIÓN Y USO]] vía Docker

---

## ⚡ Comandos de uso diario

> [!tip] Referencia rápida
> Los comandos más útiles para monitorización y gestión diaria. Consulta la tabla completa más abajo para referencia total.

| Acción | Comando rápido |
|--------|----------------|
| **Monitor GPU en tiempo real** | `watch -n 1 nvidia-smi` |
| **Ver tokens generados / logs** | `journalctl -u ollama -f` |
| **Ejecutar modelo** | `ollama run qwen2.5-coder:14b` |
| **Listar modelos instalados** | `ollama list` |
| **Modelos cargados en memoria** | `ollama ps` |
| **Estado del servicio** | `systemctl status ollama` |

---

## 📋 Requisitos previos

> [!warning] GPU NVIDIA obligatoria para rendimiento real
> Sin drivers oficiales, Ollama ejecuta los modelos en CPU (lento e inusable).

| Requisito | Verificación |
|-----------|-------------|
| Ubuntu Server 24.04 | `lsb_release -a` |
| Drivers NVIDIA | `nvidia-smi` |
| GPU detectada | `lspci \| grep -i nvidia` |
| Acceso sudo | ✅ |

---

## 🛠️ Instalación paso a paso

### 1. Actualizar el sistema

```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Drivers NVIDIA (si aplica)

> **Nota:** Si el servidor dispone de una GPU NVIDIA y se desea que Ollama la utilice para acelerar la inferencia, es recomendable instalar primero los controladores oficiales.

```bash
# Detectar GPU
lspci | grep -i nvidia

# Instalar drivers recomendados
sudo ubuntu-drivers install

# Reiniciar para cargar módulos CUDA
sudo reboot

# Verificar
nvidia-smi
watch -n 1 nvidia-smi  # Monitor en tiempo real
```

> [!tip] Indicador de éxito
> Si `nvidia-smi` muestra Driver, CUDA y lista de GPUs → el stack gráfico está listo. Ollama lo detectará automáticamente.

Ejemplo de salida esperada:

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 570.xx.xx        Driver Version: 570.xx.xx       CUDA Version: xx |
|-------------------------------+----------------------+------------------------|
| GPU  Name                     | Bus-Id               | Memory-Usage          |
| 0    NVIDIA RTX xxxx          | 00000000:xx:00.0     | 0 MiB / xxxxx MiB     |
+-----------------------------------------------------------------------------+
```

### 3. Herramientas base

```bash
sudo apt install -y curl wget git nano htop ca-certificates acl
```

### 4. Instalar Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

El script configura automáticamente `systemd`.

```bash
# Comprobar la instalación
ollama --version

# Comprobar el servicio
sudo systemctl status ollama
```

> [!note] Detener el servicio antes de configurar
> ```bash
> sudo systemctl stop ollama
> ```

---

## ⚙️ Configuración avanzada

### A. Ruta personalizada para modelos

Por defecto Ollama usa `/usr/share/ollama/.ollama/models`. Lo cambiamos:

```bash
# Crear directorio de modelos
mkdir -p /home/escuela/modelos
sudo chown -R ollama:ollama /home/escuela/modelos
sudo chmod 755 /home/escuela/modelos

# Permitir al usuario 'ollama' acceder a /home/escuela (ACL)
sudo setfacl -m u:ollama:x /home/escuela
```

> [!note] Alternativa sin ACL
> ```bash
> sudo chmod 751 /home/escuela
> ```

### B. Override del servicio systemd

```bash
# Crear directorio de overrides
sudo mkdir -p /etc/systemd/system/ollama.service.d

# Crear archivo de configuración
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

**Contenido de `override.conf`:**

```ini
[Service]
Environment="OLLAMA_MODELS=/home/escuela/modelos"
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

> [!note] Guardar en nano
> `Ctrl + O` → Enter → `Ctrl + X`

### C. Aplicar cambios

```bash
sudo systemctl daemon-reload
sudo systemctl start ollama
sudo systemctl enable ollama --now
```

---

## 🔍 Verificación

### Confirmar configuración activa

```bash
systemctl show ollama --property=Environment
# ✅ Debe contener: OLLAMA_MODELS=/home/escuela/modelos OLLAMA_HOST=0.0.0.0:11434
```

### Descargar y ejecutar un modelo

```bash
ollama pull llama3.2
ollama run llama3.2
# Salir: /bye
```

### Estructura de modelos generada

```text
/home/escuela/modelos/
├── blobs/            # Datos binarios del modelo (sha256)
│   ├── sha256-xxxxx
│   └── ...
└── manifests/        # Metadatos y registry
    └── registry.ollama.ai
```

> Los archivos **sha256** son el propio modelo dividido en objetos internos. La carpeta `blobs` contiene los datos reales de los modelos, mientras que `manifests` tiene la información para que Ollama reconstruya cada modelo.

### Verificar puerto y red

```bash
# Puerto escuchando
sudo ss -tulpn | grep 11434
# ✅ Debe mostrar: 0.0.0.0:11434

# Probar API local
curl http://localhost:11434/api/tags

# Probar desde otro equipo de la red
curl http://192.168.1.126:11434/api/tags
```

### Firewall (si usas UFW)

```bash
sudo ufw allow 11434/tcp
```

---

## 📊 Referencia completa de comandos

| Acción | Comando | Descripción |
|--------|---------|-------------|
| **Ejecutar modelo** | `ollama run <nombre>` | Inicia sesión de chat (ej. `qwen2.5-coder:14b`) |
| **Descargar modelo** | `ollama pull <nombre>` | Descarga desde registry (ej. `llama3.2`, `mistral`) |
| **Listar modelos** | `ollama list` | Muestra modelos instalados y su tamaño en disco |
| **Modelos en memoria** | `ollama ps` | Muestra qué modelos están cargados actualmente en RAM/VRAM |
| **Detener modelo** | `ollama stop <nombre>` | Libera un modelo de la memoria sin borrarlo del disco |
| **Info del modelo** | `ollama show <nombre>` | Detalles técnicos (capas, sistema, parámetros) |
| **Eliminar modelo** | `ollama rm <nombre>` | Borra permanentemente un modelo del disco |
| **Estado servicio** | `systemctl status ollama` | Verifica si el daemon está activo y corriendo |
| **Reiniciar servicio** | `sudo systemctl restart ollama` | Reinicia el backend de Ollama tras cambios |
| **Logs en vivo** | `journalctl -u ollama -f` | Muestra logs del sistema (útil para ver tokens/error) |
| **Últimos errores** | `journalctl -u ollama -n 50 --no-pager` | Extrae las últimas 50 líneas de log rápidamente |
| **Tamaño ocupado** | `du -sh /home/escuela/modelos` | Calcula el espacio total usado por los modelos |
| **Monitor GPU** | `watch -n 1 nvidia-smi` | Uso de VRAM, temperatura y carga en tiempo real |
| **Ver proceso** | `ps aux \| grep ollama` | Muestra el PID y consumo del proceso Ollama |
| **Configuración systemd** | `systemctl cat ollama` | Muestra el archivo de servicio activo con overrides aplicados |

### Reset completo (si hay procesos zombis)

```bash
sudo systemctl stop ollama
sudo pkill -f "ollama"
sudo systemctl start ollama
```

---

## 🏗️ Estructura recomendada del servidor

```text
/home/escuela/
├── modelos/              # Ollama models (OLLAMA_MODELS)
│   ├── blobs/
│   └── manifests/
├── docker/               # Docker projects
│   └── open-webui/       # [[OPEN-WEBUI - INSTALACIÓN Y USO]]
└── backups/              # Backups manuales
```

---

## 📝 Notas

- Ollama está instalado de forma nativa (sin Docker).
- Los modelos se almacenan en `/home/escuela/modelos`.
- El servicio se gestiona mediante `systemd`.
- La API escucha en el puerto **11434**.
- Open WebUI podrá instalarse posteriormente mediante Docker y conectarse a esta instancia de Ollama.

---

## 🔗 Notas relacionadas

- [[LLAMA.CPP - INSTALACIÓN Y MODELOS]] — Inferencia directa con GPU (alternativa a Ollama)
- [[OPEN-WEBUI - INSTALACIÓN Y USO]] — Interfaz web para conectar con Ollama
- [[IA - ARQUITECTURA GENERAL]] — Vista global de la infraestructura IA
- [[RB-01 Despliegue LLM Local]] — Runbook completo de despliegue

---

> [!tip] llama-server vs Ollama en el mismo nodo
> Ambos pueden correr en HL2-1, pero **usa uno a la vez** para no saturar GPU y RAM. El `[[LLAMA.CPP - INSTALACIÓN Y MODELOS]]` da máximo control; Ollama es más sencillo de gestionar.
