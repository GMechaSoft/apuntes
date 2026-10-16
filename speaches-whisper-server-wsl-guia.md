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
| Declaración del modelo | Por petición (campo `model`), no en la config |
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

## 2. Modelo conceptual: declaración, descarga y memoria

Esta sección aclara cómo Speaches gestiona los modelos, para entender por qué la primera
puesta en marcha exige pasos manuales y por qué "descargar" (disco) no es lo mismo que
"cargar" (VRAM).

### 2.1. ¿Dónde se declara el modelo?

**No existe** un equivalente a `WHISPER_MODEL`. Speaches no mantiene una lista fija de
modelos en la configuración: el modelo se elige **en cada petición** con el campo `model`:

```bash
curl -s http://localhost:8000/v1/audio/transcriptions \
  -F file=@/tmp/test.wav \
  -F model="h2oai/faster-whisper-large-v3-turbo"
```

Las variables `WHISPER__*` (`inference_device`, `compute_type`) solo configuran la
**inferencia**, no la identidad del modelo. La única lista opcional es `PRELOAD_MODELS`
(ver sección 2.4), que sirve para descargar en el arranque, no para fijar el modelo de las
peticiones.

### 2.2. Los tres estados de un modelo

| Estado | Significado | Qué lo dispara |
|---|---|---|
| **Registro (registry)** | El modelo "existe" y es válido (`library_name: ctranslate2`, `pipeline_tag`) | Descubrimiento en HuggingFace (`GET /v1/registry`) |
| **Instalado / descargado** | Está en disco (`hf_cache`) | **Descarga explícita** (`model download` / `POST /v1/models/<id>`) o `PRELOAD_MODELS` al arranque |
| **Cargado en VRAM** | Ocupa memoria GPU | **Primera petición** que usa ese modelo (carga dinámica) |

### 2.3. La petición NO descarga el modelo

A diferencia de `faster-whisper-server` (que descarga automáticamente a partir de
`WHISPER_MODEL`), Speaches **no descarga** el modelo al recibir una transcripción. Si el
modelo no está instalado, responde `"not installed locally"`.

El modelo se deja disponible de forma **explícita y una sola vez** (queda persistente en
disco, ver sección 2.5):

```bash
# Vía oficial (funciona si la tarjeta HF ya declara library_name / pipeline_tag)
uvx speaches-cli model download Systran/faster-whisper-large-v3

# Equivalente por API
curl -X POST "http://localhost:8000/v1/models/Systran/faster-whisper-large-v3"
```

El modelo `h2oai/faster-whisper-large-v3-turbo` **no** declara esos metadatos, por lo que
la vía oficial falla (404 en `POST /v1/models` y `"not installed locally"`). De ahí el
*workaround* de la sección 4.2 (`patch_model.py`): descarga manual a `hf_cache` + parche del
`README.md`.

### 2.4. Precarga en el arranque (`PRELOAD_MODELS`)

```yaml
environment:
  - PRELOAD_MODELS=["Systran/faster-whisper-large-v3"]
```

- **Descarga** (a disco, secuencialmente) durante el arranque.
- Si un modelo falla o no está en el registro, **la aplicación no arranca**.
- **No lo carga en VRAM**; solo lo deja en disco. La carga en VRAM sigue ocurriendo en la
  primera petición.
- ⚠️ No está disponible en la imagen `0.9.0-rc.1` de esta guía; se añadió en `0.9.0-rc.3`
  (cuyas imágenes `0.9.0-rc.3-*` ya están publicadas en GHCR). Ver lección 2 de la sección 6.
- ⚠️ El modelo listado debe existir en el *registry* (metadatos `library_name`/
  `pipeline_tag`). `h2oai/faster-whisper-large-v3-turbo` no los declara, así que el
  preload **fallaría** con él (error `Model ... not found` al arrancar). El parche de la sección 4.2
  **no** lo evita: `PRELOAD_MODELS` valida contra el *registry remoto* de HuggingFace,
  mientras que la sección 4.2 solo corrige el README **local** (lo que habilita la transcripción,
  no la precarga). Precargar este modelo exigiría corregir sus metadatos en el propio repo
  de HuggingFace.

### 2.5. Persistencia en VRAM: `STT_MODEL_TTL`

| Valor | Efecto |
|---|---|
| `-1` | **Nunca se descarga de VRAM**. Permanece residente todo el tiempo que el contenedor esté vivo; las siguientes peticiones son inmediatas. |
| `300` (default) | Se libera de VRAM tras 5 min de inactividad; la siguiente petición lo recarga desde disco (~5 s). |
| `0` | Se libera inmediatamente tras cada uso. |

**Distinción disco vs. memoria:**

- **Disco (`hf_cache`)**: el modelo descargado persiste siempre (bind mount), incluso tras
  reiniciar el contenedor.
- **VRAM**: se libera **siempre al reiniciar el contenedor**, sin importar el TTL.
  `STT_MODEL_TTL=-1` solo evita la descarga *dentro del ciclo de vida* del contenedor;
  tras un reinicio, la primera petición lo recarga en VRAM (~5 s desde disco, sin
  re-descargar).

---

## 3. Prerrequisitos (verificar una vez)

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

## 4. Archivos de la implementación

### 4.1. `docker-compose.yml` (dentro de `~/speaches`)

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

### 4.2. Script de descarga + parche del modelo (`patch_model.py`)

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

### 4.3. Tarea programada de Windows (`register_task.ps1`)

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

## 5. Pasos de implementación

### Paso 1 — Estructura y compose

```bash
mkdir -p ~/speaches/hf_cache
cd ~/speaches
# crear docker-compose.yml con el contenido de 4.1
docker compose config > /dev/null && echo "config OK"
```

### Paso 2 — Levantar el contenedor

```bash
cd ~/speaches && docker compose up -d
```

> La primera vez descarga la imagen (~5 GB). Sin `version:` (obsoleto en Compose v2).

### Paso 3 — Descargar y parchear el modelo

```bash
# copiar patch_model.py (4.2) al contenedor y ejecutarlo
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

## 6. Lecciones aprendidas (evitar errores ya cometidos)

1. **No usar `latest-cuda` ni `latest-cuda-12.6.3`**: son v0.8.x, con esquema viejo
   (`WHISPER__TTL`, sin `STT_MODEL_TTL`). Usar `0.9.0-rc.1-cuda-12.6.3`.
2. **`PRELOAD_MODELS` está disponible desde `v0.9.0-rc.3`**, cuyas imágenes
   (`0.9.0-rc.3-cuda-12.6.3`, etc.) ya están publicadas en GHCR. En `0.9.0-rc.1` (la
   versión de esta guía) **no existe**, por lo que **no hay precarga al arranque**: el
   modelo se carga en la primera petición (~5 s desde cache) y queda residente.
   > Nota de temporalidad: en la fecha de la sesión de configuración (12/10/2026) el
   > listado de GHCR **solo** contenía `0.9.0-rc.1-*`; las imágenes `0.9.0-rc.3-*` se
   > publicaron posteriormente. Al replicar, verifica el listado actual de tags antes de
   > asumir su existencia.
3. **Variable de TTL correcta en rc.1 es `STT_MODEL_TTL`** (no `WHISPER__TTL`).
4. **El *model registry* (v0.8+) rechaza modelos sin metadatos** `library_name`/
   `pipeline_tag`. Por eso el modelo `h2oai/...` da 404 en `POST /v1/models` y
   "not installed locally" en la transcripción. Solución: descargar manualmente y
   parchear el `README.md` (sección 4.2).
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

## 7. Rollback y limpieza

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

## 8. Disponibilidad inmediata tras el arranque (warmup)

Con esta configuración, tras reiniciar el contenedor el modelo **no** está en VRAM hasta
la primera petición (ver sección 2.5: la VRAM se libera siempre al reiniciar, aunque
`STT_MODEL_TTL=-1`). Desde disco se recarga en ~5 s, sin re-descargar.

Si se requiere disponibilidad inmediata sin tráfico real, añadir un "warmup": una petición
de transcripción automática tras el arranque (p. ej. un `curl` con un audio de prueba en
el `healthcheck` o en un contenedor auxiliar). Alternativa: migrar a una imagen que soporte
`PRELOAD_MODELS` (solo descarga en arranque, no carga en VRAM; ver sección 2.4) usando la imagen
`0.9.0-rc.3-*` ya publicada.

---

## 9. Exposición del puerto 8000 a la red local (WSL2 en modo NAT)

> Verificado en este equipo: WSL 2.7.3, red NAT por defecto (sin `.wslconfig`), `eth0` en
> `192.168.88.x` (la subred NAT **varía** entre equipos y reinicios), IP LAN de Windows
> `192.168.1.x`. El contenedor responde en `localhost:8000` pero **no** desde la LAN hasta
> aplicar esta sección.

### 9.1. Flujo del tráfico

```
Equipo LAN → http://<IP_Windows>:8000
  → [portproxy netsh, iphlpsvc] → <IP_eth0_WSL>:8000
    → [Docker publish 8000:8000] → contenedor:8000
```

Hacen falta dos piezas en Windows: un **portproxy** y una **regla de firewall** de entrada.
La IP de WSL cambia al reiniciar, por lo que el portproxy debe refrescarse en cada arranque.

### 9.2. Requisitos

- PowerShell **como Administrador** (`netsh portproxy` y el firewall lo exigen).
- Servicio IP Helper activo: `Get-Service iphlpsvc` → `Running` (`Set-Service iphlpsvc -StartupType Automatic`).
- Perfil de red **Private** (`Get-NetConnectionProfile`). Si es Public, la regla de firewall no aplicará.
- Contenedor publicado en `0.0.0.0:8000`: `docker port <contenedor>` → `8000/tcp -> 0.0.0.0:8000`.

### 9.3. Script `expose_lan.ps1`

Guardar, por ejemplo, en `C:\Users\<usuario>\expose_lan.ps1` (ejecutar como Administrador).
Idempotente: se puede lanzar las veces que haga falta.

```powershell
$ErrorActionPreference = 'Stop'
$port = 8000
$rule = "Whisper $port (WSL)"

# Mantener WSL vivo (si se apaga, el contenedor y el portproxy dejan de servir)
Start-Process wsl.exe -ArgumentList '-d Ubuntu -e sleep infinity' -WindowStyle Hidden

# IP de eth0 (hostname -I devuelve también los bridges de Docker; no fiarse del primer valor)
$wslIp = $null
for ($i = 0; $i -lt 30 -and -not $wslIp; $i++) {
    try {
        $wslIp = (wsl -d Ubuntu -e bash -c "ip -4 addr show eth0 | grep -oP '(?<=inet )\d+(\.\d+){3}'").Trim()
    } catch { }
    if (-not $wslIp) { Start-Sleep -Seconds 2 }
}
if (-not $wslIp) { throw 'No se pudo obtener la IP de eth0 en WSL' }

# Portproxy (delete solo de esta regla; "reset" borraría todas)
netsh interface portproxy delete v4tov4 listenport=$port listenaddress=0.0.0.0 2>$null | Out-Null
netsh interface portproxy add v4tov4 listenport=$port listenaddress=0.0.0.0 connectport=$port connectaddress=$wslIp

# Firewall: recrear para garantizar puerto/perfil correctos
Get-NetFirewallRule -DisplayName $rule -ErrorAction SilentlyContinue | Remove-NetFirewallRule
New-NetFirewallRule -DisplayName $rule -Direction Inbound -Action Allow -Protocol TCP `
    -LocalPort $port -Profile Private -RemoteAddress LocalSubnet | Out-Null

netsh interface portproxy show all
Write-Host "portproxy $port -> $wslIp listo"
```

`-RemoteAddress LocalSubnet` limita el acceso a la subred local (el portproxy conserva la IP
de origen). Quitarlo solo si se necesita más alcance.

### 9.4. Persistencia: tarea programada al iniciar sesión

```powershell
$action    = New-ScheduledTaskAction -Execute 'powershell.exe' `
               -Argument '-NoProfile -ExecutionPolicy Bypass -File C:\Users\<usuario>\expose_lan.ps1'
$trigger   = New-ScheduledTaskTrigger -AtLogOn -User "$env:USERDOMAIN\$env:USERNAME"
$principal = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" `
               -RunLevel Highest -LogonType Interactive
$settings  = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries
Register-ScheduledTask -TaskName 'Whisper LAN expose' -Action $action -Trigger $trigger `
  -Principal $principal -Settings $settings -Force
```

- `RunLevel Highest` es obligatorio (elevación para `netsh`).
- Con `AtLogOn`, sin sesión iniciada no hay acceso LAN. Es lo habitual en un equipo de escritorio.
- El script ya lanza `sleep infinity`, por lo que no depende de otras tareas que mantengan WSL vivo.

### 9.5. Verificación

```powershell
netsh interface portproxy show all          # 0.0.0.0:8000 -> <IP eth0>:8000
Get-NetFirewallRule -DisplayName 'Whisper 8000*' | Select DisplayName,Enabled,Profile
curl.exe http://localhost:8000/health
curl.exe http://<IP_Windows>:8000/health    # IP de `ipconfig`
```

Desde otro equipo de la LAN: `nc -zv <IP_Windows> 8000` y `curl http://<IP_Windows>:8000/health`.

> `Get-NetTCPConnection -LocalPort 8000` **no** muestra el portproxy (lo atiende `iphlpsvc`,
> no es un socket TCP normal). Usar `netsh interface portproxy show all`.

### 9.6. Alternativa: `networkingMode=mirrored`

`%USERPROFILE%\.wslconfig`:

```ini
[wsl2]
networkingMode=mirrored
```

Después `wsl --shutdown`. WSL comparte las interfaces de Windows y no hace falta portproxy,
pero:

- Se expone **toda** la VM, no solo el 8000.
- El **firewall de Hyper-V** bloquea la entrada por defecto (`DefaultInboundAction=Block`).
  Además de la regla de Windows Defender hace falta (como Administrador):
  ```powershell
  New-NetFirewallHyperVRule -Name 'Whisper8000' -DisplayName 'Whisper 8000' -Direction Inbound `
    -VMCreatorId '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -Protocol TCP -LocalPorts 8000
  ```
- Puede interferir con VPN y Docker Desktop.

### 9.7. Seguridad

- El servidor **no tiene autenticación**: cualquiera con acceso al puerto puede usar la API.
  Mantener `-RemoteAddress LocalSubnet` o limitar a las IP necesarias.
- No abrir el 8000 en el router ni exponerlo a Internet. Para acceso amplio, poner un reverse
  proxy con autenticación delante.

### 9.8. Rollback

```powershell
netsh interface portproxy delete v4tov4 listenport=8000 listenaddress=0.0.0.0
Get-NetFirewallRule -DisplayName 'Whisper 8000 (WSL)' | Remove-NetFirewallRule
Unregister-ScheduledTask -TaskName 'Whisper LAN expose' -Confirm:$false
```
