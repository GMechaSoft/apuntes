## 📋 Guía de Configuración: Servidor Whisper (WSL) para Cliente Voxtype
Esta guía explica cómo levantar un servidor de inteligencia artificial para transcripción de voz (faster-whisper-server) dentro de Docker en WSL (Windows), exponiendo únicamente el puerto 8000 a la red local de forma segura mediante un proxy de red nativo de Windows.
------------------------------
## REQUISITO PREVIO: Mantener la red de WSL aislada (Modo NAT)
Para evitar que todo el subsistema de Linux quede expuesto a la red local (lo que haría el modo mirrored), el archivo de configuración global de WSL en Windows debe mantener el aislamiento por red virtual.

* Ruta del archivo en Windows: %userprofile%\.wslconfig
* Contenido recomendado:

[wsl2]
processors=4
networkingMode=nat


------------------------------
## PASO 1: Desplegar el Contenedor Docker en WSL
Dentro de la terminal de Ubuntu en WSL, ejecuta el contenedor con soporte para GPU (NVIDIA CUDA) y mapea el puerto internamente:

sudo docker run \
  -d --restart unless-stopped \
  --name faster-whisper-server \
  --gpus all \
  -p 8000:8000 \
  -e WHISPER_MODEL=h2oai/faster-whisper-large-v3-turbo \
  -e WHISPER__INFERENCE_DEVICE=cuda \
  -e WHISPER__COMPUTE_TYPE=int8 \
  -e WHISPER__TTL=-1 \
  -e SPEACHES_VAD_FILTER=true \
  -v faster-whisper-models:/root/.cache/huggingface \
  fedirz/faster-whisper-server:latest-cuda

------------------------------
## PASO 2: Obtener la IP interna de la Máquina Virtual
Dado que WSL opera bajo una red NAT privada, necesitas conocer la IP que Windows le asignó internamente al adaptador de Linux. Ejecuta en la terminal de WSL:

ip addr show eth0 | grep -oP '(?<=inet )\d+(\.\d+){3}'

(Anota esta IP, por ejemplo: 172.25.4.12).
------------------------------
## PASO 3: Configurar el Puente de Red Seguro en Windows (Port Proxy)
Para que las peticiones externas de tu red local lleguen al contenedor, debes crear un túnel exclusivo en Windows que redirija el tráfico del puerto 8000 hacia la IP interna de WSL obtenida en el paso anterior.
Abre PowerShell como Administrador en Windows y ejecuta:

# 1. Habilitar el servicio de enrutamiento de red de Windows
Set-Service iphlpsvc -StartupType Automatic
Start-Service iphlpsvc
# 2. Limpiar cualquier regla proxy previa
netsh interface portproxy reset
# 3. Crear el puente exclusivo (Reemplaza TU_IP_WSL_172)
netsh interface portproxy add v4tov4 listenport=8000 listenaddress=0.0.0.0 connectport=8000 connectaddress=TU_IP_WSL_172

------------------------------
## PASO 4: Habilitar el Puerto en el Firewall de Windows
Asegúrate de que el Firewall de Windows Defender permita las conexiones entrantes en el puerto 8000 únicamente para redes seguras (Privadas). Ejecuta en PowerShell como Administrador:

New-NetFirewallRule -DisplayName "FasterWhisper Externo" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8000 -Profile Private

------------------------------
## 🔍 Comprobaciones de Estado (Troubleshooting)

* ¿El puente de Windows está activo? (Ejecutar en PowerShell de Windows):

Get-NetTCPConnection -LocalPort 8000 | Select-Object LocalAddress, LocalPort, State

Debe listar obligatoriamente una línea con 0.0.0.0 8000 Listen.
* ¿El servidor responde desde la Laptop Cliente? (Ejecutar en la laptop con Ubuntu):

nc -zv IP_DEL_SERVIDOR_WINDOWS 8000

Debe devolver un estado de conexión exitosa (Succeeded).