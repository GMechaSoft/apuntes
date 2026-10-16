# Guía de configuración: servidor Whisper (WSL) para cliente Voxtype

Esta guía explica cómo levantar un servidor de transcripción de voz (`faster-whisper-server`) dentro de Docker en WSL (Windows) y exponer únicamente el puerto 8000 a la red local mediante un proxy de red nativo de Windows.

---

## Requisito previo: mantener la red de WSL aislada (modo NAT)

Para evitar que todo el subsistema de Linux quede expuesto a la red local (lo que haría el modo `mirrored`), el archivo de configuración global de WSL debe mantener el aislamiento por red virtual.

- Ruta del archivo en Windows: `%userprofile%\.wslconfig`
- Contenido recomendado:

```ini
[wsl2]
processors=4
networkingMode=nat
```

---

## Paso 1: desplegar el contenedor Docker en WSL

Dentro de la terminal de Ubuntu en WSL, ejecuta el contenedor con soporte para GPU (NVIDIA CUDA) y mapea el puerto:

```bash
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
```

---

## Paso 2: obtener la IP interna de la máquina virtual

WSL opera bajo una red NAT privada, así que hay que conocer la IP que Windows asignó al adaptador de Linux. Ejecuta en la terminal de WSL:

```bash
ip addr show eth0 | grep -oP '(?<=inet )\d+(\.\d+){3}'
```

Anota la IP (por ejemplo `172.25.4.12`). Cambia al reiniciar WSL o Windows.

---

## Paso 3: configurar el puente de red en Windows (portproxy)

Crea en Windows un túnel que redirija el tráfico del puerto 8000 hacia la IP interna de WSL. Abre PowerShell como Administrador y ejecuta:

```powershell
# 1. Habilitar el servicio de enrutamiento de red de Windows
Set-Service iphlpsvc -StartupType Automatic
Start-Service iphlpsvc

# 2. Eliminar la regla previa de este puerto (no usar "reset": borra todas las reglas)
netsh interface portproxy delete v4tov4 listenport=8000 listenaddress=0.0.0.0

# 3. Crear el puente (reemplaza TU_IP_WSL por la IP del paso 2)
netsh interface portproxy add v4tov4 listenport=8000 listenaddress=0.0.0.0 connectport=8000 connectaddress=TU_IP_WSL
```

---

## Paso 4: habilitar el puerto en el firewall de Windows

Permite las conexiones entrantes en el puerto 8000 únicamente para redes privadas. Ejecuta en PowerShell como Administrador:

```powershell
New-NetFirewallRule -DisplayName "FasterWhisper Externo" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 8000 -Profile Private
```

---

## Comprobaciones de estado

- ¿Está activo el puente de Windows? (PowerShell en Windows):

  ```powershell
  netsh interface portproxy show all
  ```

  Debe listar `0.0.0.0  8000  <IP_WSL>  8000`. `Get-NetTCPConnection` no muestra el portproxy.

- ¿Responde el servidor desde la laptop cliente? (terminal Ubuntu):

  ```bash
  nc -zv IP_DEL_SERVIDOR_WINDOWS 8000
  ```

  Debe devolver `succeeded`.

---

## Anexo: exposición del puerto 8000 a la red local (WSL2 en modo NAT)

> Verificado en este equipo: WSL 2.7.3, red NAT por defecto (sin `.wslconfig`), `eth0` en
> `192.168.88.x` (la subred NAT **varía** entre equipos y reinicios), IP LAN de Windows
> `192.168.1.x`. El contenedor responde en `localhost:8000` pero **no** desde la LAN hasta
> aplicar esta sección.

### C.1. Flujo del tráfico

```
Equipo LAN → http://<IP_Windows>:8000
  → [portproxy netsh, iphlpsvc] → <IP_eth0_WSL>:8000
    → [Docker publish 8000:8000] → contenedor:8000
```

Hacen falta dos piezas en Windows: un **portproxy** y una **regla de firewall** de entrada.
La IP de WSL cambia al reiniciar, por lo que el portproxy debe refrescarse en cada arranque.

### C.2. Requisitos

- PowerShell **como Administrador** (`netsh portproxy` y el firewall lo exigen).
- Servicio IP Helper activo: `Get-Service iphlpsvc` → `Running` (`Set-Service iphlpsvc -StartupType Automatic`).
- Perfil de red **Private** (`Get-NetConnectionProfile`). Si es Public, la regla de firewall no aplicará.
- Contenedor publicado en `0.0.0.0:8000`: `docker port <contenedor>` → `8000/tcp -> 0.0.0.0:8000`.

### C.3. Script `expose_lan.ps1`

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

### C.4. Persistencia: tarea programada al iniciar sesión

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

### C.5. Verificación

```powershell
netsh interface portproxy show all          # 0.0.0.0:8000 -> <IP eth0>:8000
Get-NetFirewallRule -DisplayName 'Whisper 8000*' | Select DisplayName,Enabled,Profile
curl.exe http://localhost:8000/health
curl.exe http://<IP_Windows>:8000/health    # IP de `ipconfig`
```

Desde otro equipo de la LAN: `nc -zv <IP_Windows> 8000` y `curl http://<IP_Windows>:8000/health`.

> `Get-NetTCPConnection -LocalPort 8000` **no** muestra el portproxy (lo atiende `iphlpsvc`,
> no es un socket TCP normal). Usar `netsh interface portproxy show all`.

### C.6. Alternativa: `networkingMode=mirrored`

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

### C.7. Seguridad

- El servidor **no tiene autenticación**: cualquiera con acceso al puerto puede usar la API.
  Mantener `-RemoteAddress LocalSubnet` o limitar a las IP necesarias.
- No abrir el 8000 en el router ni exponerlo a Internet. Para acceso amplio, poner un reverse
  proxy con autenticación delante.

### C.8. Rollback

```powershell
netsh interface portproxy delete v4tov4 listenport=8000 listenaddress=0.0.0.0
Get-NetFirewallRule -DisplayName 'Whisper 8000 (WSL)' | Remove-NetFirewallRule
Unregister-ScheduledTask -TaskName 'Whisper LAN expose' -Confirm:$false
```
