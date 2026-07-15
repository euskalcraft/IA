
# Instalación de Ollama en Ubuntu Server 24.04 (Sin Docker)

## Objetivo

Instalar Ollama de forma nativa en Ubuntu Server 24.04, utilizando el servicio `systemd` y almacenando todos los modelos en una carpeta personalizada:

```
/home/escuela/modelos
```

Posteriormente se instalará Open WebUI mediante Docker, reutilizando esta instalación.

---

# 1. Actualizar el sistema 

```bash
sudo apt update
```

```bash
sudo apt upgrade -y
```

---

# 2. Instalar drivers y dependencias

---

# Instalación de los drivers NVIDIA

> **Nota:** Si el servidor dispone de una GPU NVIDIA y se desea que Ollama la utilice para acelerar la inferencia, es recomendable instalar primero los controladores oficiales.

## Comprobar el hardware detectado

Antes de instalar los drivers, se puede comprobar qué tarjetas gráficas detecta el sistema:

```bash
lspci | grep -i nvidia
```

## Instalar los drivers recomendados

Ubuntu incluye una utilidad que detecta automáticamente el controlador más adecuado para la GPU instalada:

```bash
sudo ubuntu-drivers install
```

Una vez finalizada la instalación, es recomendable reiniciar el servidor:

```bash
sudo reboot
```

## Comprobar que los drivers funcionan

Tras el reinicio, ejecutar:

```bash
nvidia-smi
```

Si todo ha ido correctamente, aparecerá una pantalla similar a esta:

- Versión del driver instalado.
- Versión de CUDA soportada.
- Modelo de la(s) GPU(s) detectada(s).
- Memoria disponible.
- Procesos que estén utilizando la GPU.

Por ejemplo:

```text
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 570.xx.xx        Driver Version: 570.xx.xx       CUDA Version: xx |
|-------------------------------+----------------------+------------------------|
| GPU  Name                     | Bus-Id               | Memory-Usage          |
| 0    NVIDIA RTX xxxx          | 00000000:xx:00.0     | 0 MiB / xxxxx MiB     |
+-----------------------------------------------------------------------------+
```

Si `nvidia-smi` muestra correctamente la información de la GPU, los drivers están instalados y funcionando correctamente, y Ollama podrá utilizarlos automáticamente para ejecutar los modelos compatibles.

##### **Instalar dependecias:**

```bash
sudo apt install -y curl wget git nano htop ca-certificates acl
```

---

# 3. Instalar Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Comprobar la instalación:

```bash
ollama --version
```

---

# 4. Comprobar el servicio

```bash
sudo systemctl status ollama
```

---

# 5. Detener el servicio

```bash
sudo systemctl stop ollama
```

---

# 6. Crear la carpeta donde se almacenarán los modelos

```bash
mkdir -p /home/escuela/modelos
```

---

# 7. Asignar permisos

```bash
sudo chown -R ollama:ollama /home/escuela/modelos
```

```bash
sudo chmod 755 /home/escuela/modelos
```

Como el servicio se ejecuta con el usuario **ollama**, hay que permitirle atravesar el directorio personal:

```bash
sudo chmod 751 /home/escuela
```

También se puede hacer mediante ACL (más recomendable si no se quieren modificar permisos):

```bash
sudo setfacl -m u:ollama:x /home/escuela
```

---

# 8. Crear el override del servicio

Crear el directorio:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
```

Crear el archivo:

```bash
sudo nano /etc/systemd/system/ollama.service.d/override.conf
```

Contenido:

```ini
[Service]
Environment="OLLAMA_MODELS=/home/escuela/modelos"
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Guardar con:

```
CTRL + O
ENTER
CTRL + X
```

---

# 9. Recargar systemd

```bash
sudo systemctl daemon-reload
```

---

# 10. Arrancar Ollama

```bash
sudo systemctl start ollama
```

Activarlo al iniciar el servidor:

```bash
sudo systemctl enable ollama
```

---

# 11. Verificar que la configuración se ha aplicado

```bash
systemctl show ollama --property=Environment
```

Debe aparecer algo parecido a:

```
Environment=PATH=... OLLAMA_MODELS=/home/escuela/modelos OLLAMA_HOST=0.0.0.0:11434
```

---

# 12. Descargar un modelo

Ejemplo:

```bash
ollama pull llama3.2
```

---

# 13. Ver modelos instalados

```bash
ollama list
```

---

# 14. Ejecutar un modelo

```bash
ollama run llama3.2
```

Salir del chat:

```
/bye
```

---

# 15. Comprobar que el modelo está en la carpeta correcta

```bash
ls /home/escuela/modelos
```

Debe aparecer:

```
blobs
manifests
```

Comprobar el contenido:

```bash
ls /home/escuela/modelos/blobs
```

```bash
ls /home/escuela/modelos/manifests
```

La estructura correcta será similar a:

```
/home/escuela/modelos

├── blobs
│   ├── sha256-xxxxxxxx
│   ├── sha256-xxxxxxxx
│   └── ...
│
└── manifests
    └── registry.ollama.ai
```

Los archivos **sha256** son el propio modelo dividido en objetos internos.

---

# 16. Ver tamaño ocupado

```bash
du -sh /home/escuela/modelos
```

---

# 17. Mostrar información de un modelo

```bash
ollama show llama3.2
```

---

# 18. Eliminar un modelo

```bash
ollama rm llama3.2
```

---

# 19. Ver los logs del servicio

```bash
sudo journalctl -u ollama -f
```

Ver los últimos errores:

```bash
sudo journalctl -u ollama -n 50 --no-pager
```

##### **VER TOKENS QUE GENERA:**

sudo journalctl -u ollama -f

watch -n 1 nvidia-smi

---

# 20. Verificar que el servicio está funcionando

```bash
sudo systemctl status ollama
```

---

# 21. Ver el contenido del servicio

```bash
sudo systemctl cat ollama
```

---

# 22. Ver el proceso en ejecución

```bash
ps aux | grep ollama
```

---

# 23. Comprobar el puerto

```bash
sudo ss -tulpn | grep 11434
```

Debe escuchar en:

```
0.0.0.0:11434
```

---

# 24. Probar la API

Desde el propio servidor:

```bash
curl http://localhost:11434/api/tags
```

Desde otro equipo:

```bash
curl http://IP_DEL_SERVIDOR:11434/api/tags
```

---

# 25. Detener el servicio

```bash
sudo systemctl stop ollama
```

---

# 26. Reiniciar el servicio

```bash
sudo systemctl restart ollama
```

---

# 27. Reiniciar completamente si hubiera un proceso manual

Parar el servicio:

```bash
sudo systemctl stop ollama
```

Eliminar procesos manuales:

```bash
sudo pkill -f "ollama"
```

Volver a iniciar:

```bash
sudo systemctl start ollama
```

---

# Estructura recomendada del servidor

```
/home/escuela

├── modelos
│   ├── blobs
│   └── manifests
│
├── docker
│   └── open-webui
│
└── backups
```

---

# Notas

- Ollama está instalado de forma nativa (sin Docker).
- Los modelos se almacenan en `/home/escuela/modelos`.
- El servicio se gestiona mediante `systemd`.
- La API escucha en el puerto **11434**.
- Open WebUI podrá instalarse posteriormente mediante Docker y conectarse a esta instancia de Ollama.
- La carpeta `blobs` contiene los datos reales de los modelos.
- La carpeta `manifests` contiene la información necesaria para que Ollama reconstruya cada modelo.
- Ver TOKENS que genrea