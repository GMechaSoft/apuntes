# Voxtype: diagnóstico del hotkey y bucle de ydotoold

Diagnóstico realizado el 2026-11-24 sobre Voxtype 0.7.1 (Ubuntu/GNOME, Wayland, backend remoto `faster-whisper-server`).

**Síntoma:** de forma intermitente, al pulsar el atajo (`INSERT`, modo `toggle`) no se activa el micrófono y hay que reiniciar el daemon.

## 1. Resumen

| # | Hallazgo | Impacto | Estado |
|---|----------|---------|--------|
| 1 | El hub USB se reconecta y el daemon pierde el teclado (evdev) | **Causa probable del síntoma** | Mitigado (teclado en puerto directo); reemplazo del hub en camino |
| 2 | Cuelgues del daemon con CPU al 100 % y SIGKILL al detenerlo | Alto | Sin causa confirmada; ver sección 3 |
| 3 | `ydotoold` en bucle de fallo (unidad duplicada) | Bajo (ruido, CPU) | **Corregido** |
| 4 | Un `Connection reset by peer` del servidor remoto | Puntual | Transitorio, no reproducido |
| 5 | Errores silenciosos por el flag `-q` | Dificulta el diagnóstico | Pendiente |
| 6 | Sin respaldo local si el servidor remoto cae | Medio | Pendiente |

## 2. Hub USB y pérdida del hotkey (evdev)

**Evidencia**

- El teclado `BY Tech Gaming Keyboard` y el receptor Logitech cuelgan del hub Genesys (`usb 1-2`).
- El kernel registra desconexiones y reconexiones completas del hub: 2026-11-23 a las 20:40, 20:45 y 22:05, y 2026-11-24 a las 09:08.
- Cada reconexión crea nodos `/dev/input/eventN` nuevos. El daemon abre los dispositivos solo al iniciar, así que sus descriptores quedan apuntando a nodos muertos. El hotkey no llega y no se registra ningún error.
- Con el daemon sano, los descriptores abiertos coinciden con los dispositivos presentes (event3, 9, 10, 13, 22, 23).

**Mitigación aplicada por el usuario:** conectar el teclado directamente a un puerto USB sin pasar por el hub. La solución definitiva es reemplazar el hub.

**Alternativa robusta (no aplicada):** poner `[hotkey] enabled = false` y crear un atajo personalizado de GNOME que ejecute `voxtype record toggle`. Así el hotkey no depende de evdev.

**Cómo confirmarlo cuando vuelva a fallar** (antes de reiniciar):

```bash
# Check CPU and state of the daemon
ps -o pid,pcpu,stat -p $(pgrep -x voxtype)

# List the input devices the daemon holds open
ls -l /proc/$(pgrep -x voxtype)/fd | grep input

# Compare with the devices that currently exist
grep -E 'Name=|Handlers' /proc/bus/input/devices | paste - - | grep -i kbd
```

## 3. Cuelgues con CPU al 100 %

El tiempo de CPU consumido es casi igual al tiempo de reloj en varias ejecuciones, lo que indica un bucle de espera activa:

| Fecha | CPU | Reloj |
|-------|-----|-------|
| 2026-11-22 | 5 h 06 min | 5 h 05 min |
| 2026-11-20 | 1 h 43 min | 1 h 43 min |
| 2026-11-20 | 1 h 06 min | 1 h 06 min |
| 2026-11-18 | 8 min 39 s | 9 min 29 s |

En 6 ocasiones systemd tuvo que usar SIGKILL (`State 'stop-sigterm' timed out`) porque el daemon no respondió a SIGTERM. Una ejecución sana consume unos pocos segundos de CPU.

**Hipótesis:** descriptor evdev muerto tras la reconexión del hub. No está confirmada, porque `-q` silencia los logs. Verificar con los comandos de la sección 2 en el próximo fallo.

## 4. `ydotoold` en bucle de fallo (corregido)

**Causa raíz:** dos unidades de usuario arrancaban el mismo binario y competían por el socket `/run/user/1000/.ydotool_socket`.

| Unidad | Origen | Reinicio |
|--------|--------|----------|
| `ydotool.service` | Paquete `ydotool 1.0.4-3`, requerida por `voxtype.service` (`Wants=`) | `Restart=always` |
| `ydotoold.service` | Creada a mano el 2026-06-11 | `Restart=on-failure`, `RestartSec=2` |

La unidad manual fallaba con `Another ydotoold is running with the same socket` cada 2 s: unos **64 000 reinicios** desde el 2026-11-14. Su `ExecStartPre` borraba `/tmp/.ydotool_socket`, una ruta incorrecta para la versión 1.0.4, que usa `$XDG_RUNTIME_DIR`.

**Riesgo adicional:** si en algún arranque ganaba la unidad manual, el bucle pasaba a `ydotool.service` con `Restart=always` y reintentos cada 100 ms. No se verificó en arranques anteriores.

**Solución aplicada:** se eliminó la unidad manual y se conservó la del paquete.

```bash
systemctl --user disable --now ydotoold.service
rm ~/.config/systemd/user/ydotoold.service
systemctl --user daemon-reload
systemctl --user reset-failed
```

**Verificación:** `ydotool.service` activo (PID 4316 sin interrupción), `voxtype.service` activo y cero errores `Another ydotoold` tras el cambio. Pendiente confirmar en el próximo reinicio de sesión con `journalctl --user -u ydotool --since today`.

**Reversión:** restaurar el respaldo en `~/.config/systemd/user/ydotoold.service` y ejecutar `systemctl --user enable --now ydotoold`. No se recomienda.

## 5. Otros hallazgos

- **Error remoto puntual (2026-11-24 09:16):** `Connection reset by peer` en `http://192.168.1.7:8000/v1/audio/transcriptions`. El servidor respondió con normalidad minutos después (`/v1/models` 200, transcripción de prueba 200). Se considera transitorio.
- **Errores silenciosos:** el daemon corre con `-q`, por lo que los fallos no muestran ninguna notificación. Valorar las opciones de `[output.notification]`.
- **Sin respaldo local:** `model` apunta a un nombre de HuggingFace válido solo para el backend remoto. Existen modelos `ggml-*.bin` en `~/.local/share/voxtype/models` que podrían servir de respaldo.
- **Clave de backend:** la configuración usa `mode = "remote"` en `[whisper]`, mientras que la guía `server` del repositorio menciona `backend = "remote"`. Verificar contra la versión instalada (0.7.1).
- **Contenedores ajenos a Voxtype:** `personas-api` (Exited 137) y `personas-postgres` llevan detenidos desde hace 3 días.

## 6. Acciones pendientes

1. Reemplazar el hub USB (en camino).
2. Confirmar el arranque limpio de `ydotool.service` tras el próximo reinicio de sesión.
3. En el próximo fallo, capturar CPU y descriptores del daemon antes de reiniciarlo.
4. Decidir si se adopta el atajo de GNOME con `voxtype record toggle`.
5. Decidir si se configura un respaldo local y notificaciones de error.

---

## 7. Bitácora

Registro cronológico de acciones sobre el sistema y el repositorio. Las entradas más recientes van primero.

### 2026-11-24

#### Corrección del bucle de `ydotoold`

- **Qué:** se eliminó la unidad de usuario duplicada `~/.config/systemd/user/ydotoold.service`.
- **Por qué:** competía con `ydotool.service` (paquete) por el mismo socket y fallaba cada 2 s, con unos 64 000 reinicios desde 2026-11-14.
- **Cómo:**
  - Respaldo del archivo (fuera del repositorio).
  - `systemctl --user disable --now ydotoold.service`.
  - Borrado del archivo, `daemon-reload` y `reset-failed`.
- **Resultado:** `ydotool.service` activo sin interrupción (PID 4316), `voxtype.service` activo y cero errores `Another ydotoold` tras el cambio.
- **Reversión:** restaurar el respaldo y ejecutar `systemctl --user enable --now ydotoold`.
- **Pendiente:** confirmar el arranque limpio en el próximo reinicio de sesión.

#### Diagnóstico de Voxtype

- **Síntoma:** el atajo `INSERT` a veces no activa el micrófono.
- **Revisado:** estado del servicio, `journalctl` de todos los arranques desde 2026-10-12, configuración, permisos evdev (grupo `input`), conectividad con `192.168.1.7:8000`, descriptores abiertos por el daemon y eventos USB del kernel.
- **Conclusiones:** el daemon no cae. La causa probable es la reconexión del hub USB, que invalida los dispositivos evdev. El detalle está en [Voxtype - Hallazgos de diagnóstico.md](Voxtype%20-%20Hallazgos%20de%20diagn%C3%B3stico.md).
- **Acción del usuario:** teclado conectado a un puerto USB directo, sin el hub. Reemplazo del hub en camino.

#### Repositorio

- Se añadieron `Voxtype - Hallazgos de diagnóstico.md` y `Bitácora.md`.
- No se hizo ningún commit.
