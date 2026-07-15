---
tags: [homelab, ia, llm, cuda, linux, self-hosting]
fecha: 2026-07-13
hardware: "2x NVIDIA Tesla V100 16GB"
os: "Ubuntu 24.04"
---

# 🦙 Instalación de llama.cpp con soporte CUDA (2x Tesla V100 16GB)

> [!info] Resumen
> Guía completa y **verificada** para compilar e instalar `llama.cpp` con aceleración CUDA en un servidor Ubuntu 24.04 con 2x Tesla V100 (16GB c/u, Volta, `sm_70`). Incluye la solución al error de descarga por HTTPS y el flujo final que funcionó.

---

## ✅ Requisitos previos

- Ubuntu 24.04 instalado y actualizado
- Drivers NVIDIA correctos y funcionales (verificado con `nvidia-smi`)
- Acceso `sudo`

```bash
nvidia-smi
```

```bash
lsb_release -a
```

> [!tip] Dato clave
> Las Tesla V100 son arquitectura **Volta**, *compute capability* **7.0** → usaremos `-DCMAKE_CUDA_ARCHITECTURES=70` en la compilación.

---

## 🧰 1. Dependencias de compilación

```bash
sudo apt update
```

```bash
sudo apt install -y build-essential cmake git pkg-config libcurl4-openssl-dev libssl-dev
```

| Paquete | Función |
|---|---|
| `build-essential` | gcc/g++ |
| `cmake` | sistema de build oficial de llama.cpp |
| `libcurl4-openssl-dev` | soporte de descarga de modelos |
| `libssl-dev` | soporte TLS/HTTPS necesario para `-hf` |

---

## ⚡ 2. Instalar CUDA Toolkit (nvcc)

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
```

```bash
sudo dpkg -i cuda-keyring_1.1-1_all.deb
```

```bash
sudo apt update
```

```bash
sudo apt install -y cuda-toolkit-12-6
```

> [!warning] No instales el meta-paquete `cuda` completo
> Instala solo `cuda-toolkit-12-6` para no tocar el driver NVIDIA que ya tienes funcionando correctamente.

Configurar PATH:

```bash
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
```

```bash
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
```

```bash
source ~/.bashrc
```

Verificar:

```bash
nvcc --version
```

---

## 📥 3. Clonar el repositorio oficial

```bash
cd ~
git clone https://github.com/ggml-org/llama.cpp.git
```

```bash
cd llama.cpp
```

---

## 🛠️ 4. Compilar con CUDA + OpenSSL

> [!bug] Error encontrado
> Al intentar descargar un modelo con `-hf` apareció:
> ```
> E get_repo_commit: error: HTTPS is not supported. Please rebuild with one of:
>   -DLLAMA_OPENSSL=ON (default, requires OpenSSL dev files installed)
> ```
> **Solución:** instalar `libssl-dev` (paso 1) y añadir el flag `-DLLAMA_OPENSSL=ON` al configurar CMake.

```bash
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=70 -DCMAKE_BUILD_TYPE=Release -DLLAMA_OPENSSL=ON
```

```bash
cmake --build build --config Release -j$(nproc)
```

Verificar binarios generados:

```bash
ls build/bin/
```

```bash
./build/bin/llama-cli --version
```

---

## 📂 5. Descarga manual del modelo a `~/modelos`

Crear la carpeta:

```bash
mkdir -p ~/modelos
```

Descargar el modelo GGUF directamente con `wget`:

```bash
wget -O ~/modelos/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf "https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF/resolve/main/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf"
```

Verificar la descarga:

```bash
ls -lh ~/modelos/
```

> [!tip] Buscar modelos GGUF existentes en el sistema
> ```bash
> find ~ -iname "*.gguf" 2>/dev/null
> ```

---

## 🚀 6. Ejecución con `llama-cli`

Prueba básica (100% offload a GPU):

```bash
cd ~/llama.cpp
```

```bash
./build/bin/llama-cli -m ~/modelos/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -p "Hola, ¿funciona CUDA?" -n 64 -ngl 99
```

Con reparto entre las **2 V100**:

```bash
./build/bin/llama-cli -m ~/modelos/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -p "Hola, ¿funciona CUDA?" -n 64 -ngl 99 --split-mode layer --tensor-split 1,1
```

| Flag | Descripción |
|---|---|
| `-m` | ruta al modelo `.gguf` |
| `-ngl 99` | número de capas a offloadear a GPU (99 = todas) |
| `--split-mode layer` | reparte capas entre GPUs (por defecto) |
| `--split-mode row` | reparte por filas de tensores (mejor con NVLink) |
| `--tensor-split 1,1` | reparto proporcional a partes iguales entre las 2 GPUs |

Comprobar uso real de ambas GPUs mientras corre:

```bash
watch -n 1 nvidia-smi
```

---

## 🌐 7. Servidor API compatible con OpenAI

```bash
cd ~/llama.cpp
```

```bash
./build/bin/llama-server -m ~/modelos/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -ngl 99 --split-mode layer --tensor-split 1,1 --host 0.0.0.0 --port 8080
```

Probar con `curl`:

```bash
curl http://localhost:8080/v1/chat/completions -H "Content-Type: application/json" -d '{"messages":[{"role":"user","content":"Hola"}]}'
```

### Ejecutar en segundo plano (background)

```bash
nohup ./build/bin/llama-server -m ~/modelos/Meta-Llama-3.1-8B-Instruct-Q4_K_M.gguf -ngl 99 --split-mode layer --tensor-split 1,1 --host 0.0.0.0 --port 8080 > ~/llama-server.log 2>&1 &
```

Ver logs en vivo:

```bash
tail -f ~/llama-server.log
```

---

## 🔄 8. Mantenimiento — Actualizar llama.cpp

```bash
cd ~/llama.cpp && git pull
```

```bash
cmake --build build --config Release -j$(nproc)
```

---

## 📌 Notas técnicas

- **Repositorio oficial:** [`ggml-org/llama.cpp`](https://github.com/ggml-org/llama.cpp) (renombrado desde `ggerganov/llama.cpp`)
- **Compute capability V100:** `7.0` → `sm_70`
- **Sistema de build:** CMake exclusivamente (el Makefile legacy está descontinuado)
- Si ves `offloaded 0/N` capas al ejecutar → el binario no se enlazó con CUDA. Borra `build/` y repite la configuración de CMake.
- Si vuelve a fallar la descarga con `-hf` tras compilar con `-DLLAMA_OPENSSL=ON`, confirma que `libssl-dev` está instalado y borra `build/` para forzar una reconfiguración limpia.

---

## 🗂️ Checklist final

- [x] Drivers NVIDIA verificados con `nvidia-smi`
- [x] CUDA Toolkit 12.6 instalado (`nvcc` funcional)
- [x] Compilado con `GGML_CUDA=ON`, `CMAKE_CUDA_ARCHITECTURES=70`, `LLAMA_OPENSSL=ON`
- [x] Modelo GGUF descargado en `~/modelos/`
- [x] Inferencia CLI funcionando con offload a GPU
- [x] Servidor API OpenAI-compatible corriendo en `:8080`
