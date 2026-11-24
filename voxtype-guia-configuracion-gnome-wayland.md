# Guía de configuración: Voxtype en GNOME (Wayland) con español

## Requisitos del sistema

- Ubuntu 24.04+ / Debian Trixie+ (glibc 2.39)
- GNOME Shell en Wayland
- Pertenencia al grupo `input` (para detección de hotkey vía evdev)

```bash
sudo usermod -aG input $USER
# Cerrar sesión y volver a entrar
```

---

## 1. Instalación

```bash
sudo apt install ./voxtype_0.7.1-1_amd64.deb

# Dependencias recomendadas
sudo apt install wtype wl-clipboard libnotify-bin pipewire-alsa playerctl

# Dependencias adicionales para salida de texto
sudo apt install ydotool
```

> **Nota**: `dotool` debe instalarse desde fuente si no está en los repositorios. Binario esperado en `/usr/local/bin/dotool`.

---

## 2. Descarga del modelo

Para español se recomienda el modelo **medium** (multilingüe, 1.5 GB), que ofrece mejor gramática, puntuación y manejo de tildes que `base` o `small`.

```bash
voxtype setup --download --model medium
```

Otros modelos útiles para español:

| Modelo | Tamaño | Calidad | Recomendado para |
|--------|--------|---------|------------------|
| `small` | 466 MB | Buena | Hardware limitado |
| `medium` | 1.5 GB | Muy buena | Uso general |
| `large-v3-turbo` | 1.6 GB | Excelente | GPU potente |

---

## 3. Configuración de idioma

```bash
voxtype configure
# O editar manualmente:
```

```toml
# ~/.config/voxtype/config.toml
[whisper]
model = "medium"
language = "es"
```

---

## 4. Configuración del hotkey

En GNOME/Wayland se usa el hotkey interno (evdev). Las combinaciones vía compositor
no están disponibles porque GNOME no expone `wlr_virtual_keyboard`.

```toml
[hotkey]
enabled = true
key = "RIGHTALT"
```

Teclas alternativas: `SCROLLLOCK`, `PAUSE`, `F13`–`F24`.

> **Precaución**: `Ctrl+Espacio` choca con IBus (cambio de método de entrada) en GNOME.

---

## 5. Orden de drivers de salida

En GNOME, `wtype` no funciona porque el compositor (Mutter) no soporta
`wlr_virtual_keyboard`. `dotool` requiere la variable de entorno
`DOTOOL_XKB_LAYOUT=latam` para escribir caracteres acentuados (á, é, í, ó, ú, ü, ñ).

```toml
[output]
mode = "type"
fallback_to_clipboard = true
driver_order = ["dotool", "ydotool", "wtype", "clipboard"]
```

### Inyectar variable de entorno en el servicio

Crear override de systemd:

```bash
mkdir -p ~/.config/systemd/user/voxtype.service.d

cat > ~/.config/systemd/user/voxtype.service.d/env.conf << 'EOF'
[Service]
Environment="DOTOOL_XKB_LAYOUT=latam"
EOF

systemctl --user daemon-reload
```

---

## 6. Servicio systemd

```bash
voxtype setup systemd
systemctl --user enable --now voxtype
```

Verificar:

```bash
systemctl --user status voxtype
voxtype status
```

---

## 7. Verificación

### Probar transcripción directa

```bash
voxtype transcribe /ruta/al/audio.wav
```

### Probar push-to-talk

Mantener presionado **Right Alt**, hablar, soltar. El texto aparece
en el cursor.

### Ver logs

```bash
journalctl --user -u voxtype -f
```

### Simular sin micrófono

```bash
voxtype record start    # inicia grabación
voxtype record stop     # detiene y transcribe
voxtype record cancel   # cancela
```

---

## 8. Resumen de archivos clave

| Archivo | Propósito |
|---------|-----------|
| `~/.config/voxtype/config.toml` | Configuración principal |
| `~/.local/share/voxtype/models/ggml-medium.bin` | Modelo Whisper descargado |
| `~/.config/systemd/user/voxtype.service.d/env.conf` | Variable `DOTOOL_XKB_LAYOUT` |
| `/run/user/1000/voxtype/state` | Estado actual del daemon |

---

## 9. Problemas comunes

### No se escriben tildes (á, é, í, ó, ú, ü, ñ)

- Verificar que `driver_order` prioriza `dotool` o `ydotool`. `wtype` no funciona en GNOME.
- Verificar que `DOTOOL_XKB_LAYOUT=latam` está en el entorno del servicio:
  ```bash
  systemctl --user show voxtype | grep DOTOOL
  ```
- Verificar que `ydotoold` está corriendo: `pgrep ydotoold`

### El servicio falla al iniciar

- Verificar que el modelo está descargado: `ls ~/.local/share/voxtype/models/`
- Verificar permisos del grupo `input`: `groups $USER | grep input`

### El hotkey no responde

- Confirmar que `[hotkey] enabled = true`
- Probar con `voxtype record start` desde terminal para aislar si es problema del daemon o del hotkey
- En GNOME, ciertas teclas pueden estar capturadas por atajos del sistema


### Reinicios constantes o el micrófono no se activa

- Ver [voxtype-diagnostico-hotkey-ydotoold.md](voxtype-diagnostico-hotkey-ydotoold.md): un hub USB que se reconecta invalida los dispositivos evdev del daemon sin dejar errores en el log.
- Conectar el teclado directamente a un puerto USB, sin hub, evita el problema.

### `ydotoold` falla en bucle ("Another ydotoold is running")

- El paquete `ydotool` ya trae `ydotool.service`, y Voxtype lo requiere con `Wants=`.
- No crear una unidad manual `ydotoold.service`: compite por el mismo socket.

---

## 10. Backend remoto

Para delegar la transcripción en un servidor con GPU, ver [voxtype-cliente-servidor-faster-whisper.md](voxtype-cliente-servidor-faster-whisper.md).

---

## 11. Notas técnicas

- **GPU**: Whisper usa aceleración Vulkan. Este sistema tiene AMD Radeon Vega 10 (RADV).
- **x86-64-v3**: Los binarios precompilados requieren AVX2 + FMA + BMI1/2 (Haswell 2013+).
- **Idiomas**: El modelo `medium` soporta 99 idiomas. La detección automática (`language = "auto"`) está disponible para entornos multilingües.

---

*Guía generada el 2026-06-22 para GNOME 46 en Ubuntu 24.04.*
