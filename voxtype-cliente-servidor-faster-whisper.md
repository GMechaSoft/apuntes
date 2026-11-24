# Voxtype en arquitectura cliente-servidor (Faster-Whisper)

Delegar la transcripción a un servidor externo con Faster-Whisper, liberando al cliente de la carga del modelo. Complementa la nota [voxtype-guia-configuracion-gnome-wayland.md](voxtype-guia-configuracion-gnome-wayland.md).

## Requisitos previos

- **Servidor:** Docker instalado y puerto 8000 TCP accesible desde la red local.
- **Cliente:** Voxtype instalado. No requiere descargar ningún modelo local.
- **Red:** ambas máquinas deben verse entre sí (en este caso, servidor en `192.168.1.7`).

> Con backend remoto, la transcripción la hace el servidor. El modelo que se pide se indica en `remote_model` (ver la sección «Modelo del servidor»).

## 1. Servidor

Levantar el contenedor en el puerto 8000.

**Opción A: GPU NVIDIA (máxima velocidad)**

```bash
docker run -d \
  --name faster-whisper-server \
  --gpus all \
  -p 8000:8000 \
  -e WHISPER_MODEL=systran/faster-whisper-large-v3 \
  -v faster-whisper-models:/root/.cache/huggingface \
  fedirz/faster-whisper-server:latest-cuda
```

**Opción B: solo CPU**

```bash
docker run -d \
  --name faster-whisper-server \
  -p 8000:8000 \
  -e WHISPER_MODEL=systran/faster-whisper-large-v3 \
  -v faster-whisper-models:/root/.cache/huggingface \
  fedirz/faster-whisper-server:latest-cpu
```

## 2. Firewall del servidor

```bash
# Ubuntu / Debian (UFW)
sudo ufw allow 8000/tcp && sudo ufw reload

# RHEL / CentOS / Rocky (Firewalld)
sudo firewall-cmd --permanent --add-port=8000/tcp && sudo firewall-cmd --reload
```

## 3. Verificación desde el cliente

```bash
# Lists the models served (expects HTTP 200 and a JSON body)
curl http://192.168.1.7:8000/v1/models

# End-to-end check with a 2-second silent WAV (expects {"text":""})
curl http://192.168.1.7:8000/v1/audio/transcriptions \
  -F file=@silence.wav \
  -F model=<remote_model> \
  -F language=es
```

## 4. Cliente (Voxtype)

Editar `~/.config/voxtype/config.toml`, sección `[whisper]`:

```toml
[whisper]
mode = "remote"
remote_endpoint = "http://192.168.1.7:8000"
remote_model = "h2oai/faster-whisper-large-v3-turbo"  # must match an id from /v1/models
language = "es"
```

> **Clave del backend:** la guía original usaba `backend = "remote"`. La configuración que funciona en Voxtype 0.7.1 usa `mode = "remote"`. Si cambias de versión, comprueba el nombre de la clave con `voxtype configure`.
>
> **Modelo remoto:** el valor debe coincidir con un `id` que devuelva `/v1/models`. El ejemplo original `whisper-1` solo sirve si el servidor lo expone con ese nombre.

Alternativa interactiva: `voxtype configure` (backend `remote` y la URL del endpoint).

## Modelo del servidor: configurarlo explícitamente

El servidor no usa un modelo estándar de Voxtype (por ejemplo `medium`), sino uno de Hugging Face servido por Faster-Whisper: **`h2oai/faster-whisper-large-v3-turbo`**. Voxtype no lo deduce, así que hay que indicarlo a mano en `~/.config/voxtype/config.toml`.

1. Consultar el `id` exacto que expone el servidor:

   ```bash
   curl -s http://192.168.1.7:8000/v1/models
   # {"data":[{"id":"h2oai/faster-whisper-large-v3-turbo", ...}], ...}
   ```

2. Copiar ese `id` tal cual, en `[whisper]`:

   ```toml
   [whisper]
   mode = "remote"
   model = "h2oai/faster-whisper-large-v3-turbo"
   remote_model = "h2oai/faster-whisper-large-v3-turbo"
   remote_endpoint = "http://192.168.1.7:8000"
   language = "es"
   ```

3. Reiniciar el daemon: `systemctl --user restart voxtype`.

> **Configuración en uso:** en este equipo `model` y `remote_model` llevan el mismo valor. No se comprobó si `model` es imprescindible en modo remoto; se deja igual por si la versión lo valida al arrancar.

**Por qué este modelo:** entre los modelos probados en el servidor, este fue el que mejor resultado dio en dictado. Es una variante turbo de Whisper large-v3, pensada para ser más rápida que la versión completa. La elección se basa en pruebas propias, sin benchmark formal.

**Si falla:** un nombre que no coincida con el `id` del servidor impide transcribir. Comprobar con el `curl` anterior y con el segundo comando de la sección 3.

## 5. Reinicio

```bash
systemctl --user restart voxtype
```

El hotkey y los drivers de salida funcionan igual que en modo local.

## Errores corregidos respecto al borrador original

- La IP de ejemplo estaba truncada (`192.138.1` y `192.138.1.7`). Es `192.168.1.7`.
- Faltaba el puerto `:8000` en el `curl` y en `remote_endpoint`.

## Problemas conocidos

- `Connection reset by peer` puntual al transcribir: el servidor puede estar reiniciándose o cargando el modelo. Comprueba `/v1/models` y reintenta. Detalle en [voxtype-diagnostico-hotkey-ydotoold.md](voxtype-diagnostico-hotkey-ydotoold.md).
- No hay respaldo local si el servidor cae. Los modelos `ggml-*.bin` de `~/.local/share/voxtype/models` solo sirven con el backend local.
- El daemon se ejecuta con `-q`, así que los errores de red no muestran ninguna notificación. Se ven con `journalctl --user -u voxtype -f`.

## Notas relacionadas

- Montaje del servidor en WSL: [faster-whisper-server-wsl-guia.md](faster-whisper-server-wsl-guia.md) y [speaches-whisper-server-wsl-guia.md](speaches-whisper-server-wsl-guia.md).
