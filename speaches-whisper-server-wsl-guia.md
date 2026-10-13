# Guía de migración: contenedor Whisper → Speaches (GPU / WSL)

> Runbook probado y funcional. Resumen del estado final de esta implementación para replicarla
> de forma rápida en futuras sesiones.

---

## 1. Arquitectura final

| Elemento | Valor |
|---|---|
| Motor | Speaches (`ghcr.io/speaches-ai/speaches`) |
| Imagen | `0.9.0-rc.1-cuda-12.6.3` |
| Modelo STT | `h2oai/faster-whisper-large-v3-turbo` |
| Dispositivo / cuantización | `cuda` / `int8` |
| Puerto | `8000` (API compatible con OpenAI) |
| TTL del modelo | `-1` (nunca se descarga de VRAM) |
| Cache de modelos | bind mount `./hf_cache` → `/home/ubuntu/.cache/huggingface/hub` |
| Reinicio | `restart: unless-stopped` + Docker (systemd) + Task Scheduler de Windows |

Cadena de arranque automático:

```
Inicio de sesión Windows
  → Task Scheduler "WSL Speaches": wsl.exe -d Ubuntu -e sleep infinity
    → WSL arranca (systemd)
      → servicio Docker (enabled)
        → contenedor speaches-server (restart: unless-stopped)
```

---

## 2. Prerrequisitos (verificar una vez)

- WSL 2 con `systemd` habilitado: `/etc/wsl.conf` → `[boot] systemd=true`.
- Docker dentro de WSL con el servicio systemd `enabled`:
  ```bash
  systemctl is-enabled docker   # debe devolver "enabled"
  ```
- Docker Compose **≥ 2.30** (necesario para `gpus: all`):
  ```bash
  docker compose version
  ```
- `nvidia-container-toolkit` funcional (GPU visible):
  ```bash
  docker run --rm --gpus all nvidia/cuda:12.6.3-base-ubuntu24.04 nvidia-smi -L
  ```
- PowerShell con permisos de administrador (para la tarea programada).

---

## 3. Archivos de la implementación

### 3.1. `docker-compose.yml` (dentro de `~/speaches`)

```yaml
services:
  speaches:
    image: ghcr.io/speaches-ai/speaches:0.9.0-rc.1-cuda-12.6.3
    container_name: speaches-server
    restart: unless-stopped
    gpus: all
    ports:
      - "8000:8000"
    environment:
      - WHISPER__INFERENCE_DEVICE=cuda
      - WHISPER__COMPUTE_TYPE=int8
      - STT_MODEL_TTL=-1
    volumes:
      - ./hf_cache:/home/ubuntu/.cache/huggingface/hub
```

### 3.2. Script de descarga + parche del modelo (`patch_model.py`)

Necesario porque el modelo `h2oai/faster-whisper-large-v3-turbo` no declara en su
tarjeta de HuggingFace los metadatos que exige el *model registry* de Speaches
(`library_name: ctranslate2` y `pipeline_tag: automatic-speech-recognition`).

```python
import huggingface_hub
from pathlib import Path

repo_id = "h2oai/faster-whisper-large-v3-turbo"
snap = huggingface_hub.snapshot_download(
    repo_id=repo_id,
    repo_type="model",
    allow_patterns=[
        "config.json",
        "preprocessor_config.json",
        "model.bin",
        "tokenizer.json",
        "vocabulary.*",
        "README.md",
    ],
)
print("snapshot:", snap)

readme = Path(snap) / "README.md"
if readme.is_symlink():
    readme.unlink()
card = """---
library_name: ctranslate2
pipeline_tag: automatic-speech-recognition
license: mit
language:
- multilingual
---
# faster-whisper-large-v3-turbo

CTranslate2 model for automatic speech recognition.
"""
readme.write_text(card, encoding="utf-8")
print("patched:", readme)
```

### 3.3. Tarea programada de Windows (`register_task.ps1`)

```powershell
$ErrorActionPreference = 'Stop'

$action    = New-ScheduledTaskAction -Execute 'C:\Windows\System32\wsl.exe' -Argument '-d Ubuntu -e sleep infinity'
$trigger   = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$env:USERNAME"
$settings  = New-ScheduledTaskSettingsSet -Hidden -ExecutionTimeLimit ([TimeSpan]::Zero) `
                -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
$principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" `
                -LogonType Interactive -RunLevel Limited

Register-ScheduledTask -TaskName 'WSL Speaches' -Action $action -Trigger $trigger `
                -Settings $settings -Principal $principal -Force | Out-Null
```

---

## 4. Pasos de implementación

### Paso 1 — Estructura y compose

```bash
mkdir -p ~/speaches/hf_cache
cd ~/speaches
# crear docker-compose.yml con el contenido de 3.1
docker compose config > /dev/null && echo "config OK"
```

### Paso 2 — Levantar el contenedor

```bash
cd ~/speaches && docker compose up -d
```

> La primera vez descarga la imagen (~5 GB). Sin `version:` (obsoleto en Compose v2).

### Paso 3 — Descargar y parchear el modelo

```bash
# copiar patch_model.py (3.2) al contenedor y ejecutarlo
docker cp patch_model.py speaches-server:/tmp/patch_model.py
docker exec speaches-server /home/ubuntu/speaches/.venv/bin/python /tmp/patch_model.py
```

> Alternativa: descargar a `hf_cache` desde el host y parchear el `README.md` de la
> carpeta `models--h2oai--faster-whisper-large-v3-turbo/snapshots/*/README.md`.

### Paso 4 — Verificar

```bash
curl -s http://localhost:8000/v1/models
# debe listar: h2oai/faster-whisper-large-v3-turbo

curl -s http://localhost:8000/v1/audio/transcriptions \
  -H "Content-Type: multipart/form-data" \
  -F file=@/tmp/test.wav \
  -F model="h2oai/faster-whisper-large-v3-turbo" \
  -F response_format=json
# debe responder {"text":"..."}

docker exec speaches-server nvidia-smi --query-gpu=memory.used --format=csv,noheader
# memoria VRAM ocupada ⇒ modelo residente

docker logs speaches-server 2>&1 | grep -i "not unloading"
# "is idle, not unloading" ⇒ TTL=-1 funcionando
```

### Paso 5 — Tarea programada (PowerShell como Administrador)

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File register_task.ps1
schtasks /Query /TN "WSL Speaches"   # estado "Ready"
schtasks /Run   /TN "WSL Speaches"   # prueba sin reiniciar
```

### Paso 6 — Cutover (al migrar desde el contenedor viejo)

```bash
docker stop faster-whisper-server && docker rm faster-whisper-server
```

---

## 5. Lecciones aprendidas (evitar errores ya cometidos)

1. **No usar `latest-cuda` ni `latest-cuda-12.6.3`**: son v0.8.x, con esquema viejo
   (`WHISPER__TTL`, sin `STT_MODEL_TTL`). Usar `0.9.0-rc.1-cuda-12.6.3`.
2. **`PRELOAD_MODELS` no está en ningún tag Docker publicado**: existe solo en
   `v0.9.0-rc.3`, que no publicó imagen. Por tanto **no hay precarga al arranque**;
   el modelo se carga en la primera petición (~5 s desde cache) y queda residente.
3. **Variable de TTL correcta en rc.1 es `STT_MODEL_TTL`** (no `WHISPER__TTL`).
4. **El *model registry* (v0.8+) rechaza modelos sin metadatos** `library_name`/
   `pipeline_tag`. Por eso el modelo `h2oai/...` da 404 en `POST /v1/models` y
   "not installed locally" en la transcripción. Solución: descargar manualmente y
   parchear el `README.md` (sección 3.2).
5. **`gpus: all` requiere Compose ≥ 2.30.** El bloque `deploy.resources.reservations.devices`
   solo funciona en Swarm, no con `docker compose up`.
6. **El contenedor corre como `ubuntu` (UID 1000)**, no root; el cache está en
   `/home/ubuntu/.cache/huggingface/hub`. Al reutilizar un volumen viejo (`root`),
   vigilar permisos.
7. **`compute_type=int8`** para el modelo "turbo" (ya está cuantizado en int8), no `float16`.
8. **El modelo "turbo" de Systran no existe**: el equivalente correcto es
   `h2oai/faster-whisper-large-v3-turbo`.
9. **Puerto 8000 en conflicto**: si el contenedor viejo lo ocupa, detenerlo antes de
   levantar el nuevo.
10. **Task Scheduler**: usar `wsl.exe -d Ubuntu` (no la ruta `C:\Program Files\WindowsApps\...`
    que es versionada y protegida); `-e sleep infinity` mantiene la VM viva.

---

## 6. Rollback y limpieza

```bash
# Rollback del contenedor
cd ~/speaches && docker compose down

# Limpieza opcional (libera disco)
docker volume rm faster-whisper-models      # volumen viejo del contenedor anterior
docker image prune -a                        # imágenes sin usar (cuidado: borra todas)

# Quitar la tarea programada
schtasks /Delete /TN "WSL Speaches" /F
```

---

## 7. Nota sobre la precarga en el arranque

Con esta configuración, tras un reinicio el modelo **no** está en VRAM hasta la
primera petición (luego queda residente por `STT_MODEL_TTL=-1`). Si se requiere
disponibilidad inmediata sin peticiones, añadir un "warmup" (p. ej. una petición de
transcripción automática tras el arranque) o migrar a una imagen que soporte
`PRELOAD_MODELS` (requiere compilar `v0.9.0-rc.3` desde el `Dockerfile`).
