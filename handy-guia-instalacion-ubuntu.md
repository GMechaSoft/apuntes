# Handy - Guía de instalación en Ubuntu (Wayland/GNOME)

Guía práctica para instalar y configurar [**Handy**](https://handy.computer/) (speech-to-text offline con GPU) en **Ubuntu 24.04 con GNOME Wayland**, con soporte completo para caracteres acentuados (español, francés, portugués, etc.).

Basado en experiencia real con AMD Ryzen 7 3700U + Radeon Vega 10.

---

## Requisitos previos

```bash
echo $XDG_SESSION_TYPE              # Debe mostrar: wayland
ls /usr/share/vulkan/icd.d/*.json   # GPU Vulkan disponible
```

---

## 1. Instalar Handy (.deb)

```bash
sudo apt install -y libgtk-layer-shell0
wget -P /tmp https://github.com/cjpais/Handy/releases/download/v0.8.3/Handy_0.8.3_amd64.deb
sudo apt install -y /tmp/Handy_0.8.3_amd64.deb
```

---

## 2. Instalar dotool (tipiado de texto con soporte multilingüe)

**Por qué dotool y no ydotool:** `ydotool type` solo envía scancodes crudos — no maneja dead keys ni caracteres acentuados (`áéíóúñ¿¡`). Al encontrar uno, aborta silenciosamente. `dotool` integra `libxkbcommon` y respeta el layout real del teclado.

### 2.1 Compilar desde fuente

`dotool` no está en los repos de Ubuntu. Dependencias y compilación:

```bash
sudo apt install -y gcc golang libxkbcommon-dev scdoc git
git clone https://git.sr.ht/~geb/dotool /tmp/dotool
cd /tmp/dotool
./build.sh
sudo ./build.sh install
sudo udevadm control --reload && sudo udevadm trigger
rm -rf /tmp/dotool
```

### 2.2 Grupo `input`

```bash
sudo usermod -aG input $USER
```

Cerrar sesión y volver a entrar para que el grupo se active.

---

## 3. Script de pegado (External Script)

Crear `~/.local/bin/handy-paste.sh`:

```bash
#!/bin/bash
echo "type$1" | DOTOOL_XKB_LAYOUT=latam dotool
```

```bash
chmod +x ~/.local/bin/handy-paste.sh
```

**Importante:** `DOTOOL_XKB_LAYOUT=latam` le indica a dotool el layout del teclado. Ajustar según tu distribución:

| Distribución | `DOTOOL_XKB_LAYOUT` |
| --- | --- |
| Latinoamérica | `latam` |
| España | `es` |
| US Internacional | `us(intl)` |

---

## 4. Primer arranque de Handy

```bash
handy
```

### 4.1 Permisos

Conceder **Micrófono** y **Accesibilidad**.

### 4.2 Modelo GPU

Seleccionar **Whisper Small** (~487 MB) o **Medium** (~492 MB). Únicos modelos con aceleración GPU (Vulkan) en Linux.

---

## 5. Configurar Handy

**Settings > Advanced:**

| Ajuste | Valor | Motivo |
| --- | --- | --- |
| **Overlay Position** | **None** | El overlay roba el foco en Wayland |
| **Audio Feedback** | Activado | Compensa la ausencia del overlay |
| **Paste Method** | **External Script** | Evita condiciones de carrera del clipboard |
| **External Script Path** | `/home/tu_usuario/.local/bin/handy-paste` | Ruta absoluta, sin tilde |
| **Typing Tool** | `ydotool` o `dotool` | Solo afecta a Direct Input; irrelevante con External Script |

---

## 6. Atajo global (Wayland)

GNOME Settings > Keyboard > Keyboard Shortcuts > Custom Shortcuts:

| Campo | Valor |
| --- | --- |
| **Name** | `Toggle Handy` |
| **Command** | `handy --toggle-transcription` |
| **Shortcut** | `Super+O` |

---

## 7. Probar

1. Colocar cursor en un campo de texto
2. Pulsar `Super+O`
3. Hablar (incluir palabras con acentos: `canción`, `útil`, `pingüino`)
4. Pulsar de nuevo
5. El texto completo debe aparecer, con todos los acentos

---

## Troubleshooting

### Los caracteres acentuados no se escriben (texto truncado)

**Causa:** Se está usando `ydotool type` como herramienta de tipiado. `ydotool` no maneja dead keys — aborta al encontrar `á`, `é`, `í`, `ó`, `ú`, `ñ`.

**Solución:** Instalar `dotool` y configurar `DOTOOL_XKB_LAYOUT` (paso 2 y 3).

---

### El texto pegado es incorrecto (aparece otra cosa)

**Causa:** Handy restaura el contenido original del portapapeles solo **50 ms** después de enviar Ctrl+V/Ctrl+Shift+V. En GNOME Wayland esto es insuficiente y la app destino pega el contenido restaurado, no la transcripción. Es una condición de carrera en el código de Handy.

**Solución:** Usar **External Script** como paste method (paso 3 y 5). El script usa `dotool` directamente — sin portapapeles de por medio.

---

### El script externo no se ejecuta (“No existe el archivo”)

**Causa:** La ruta en **External Script Path** usa tilde (`~`). Handy no expande `~`.

**Solución:** Usar ruta absoluta: `/home/tu_usuario/.local/bin/handy-paste`.

---

### Error: “Compositor does not support the virtual keyboard protocol”

**Causa:** `wtype` está instalado y Handy lo intenta usar primero. GNOME Mutter no soporta `wlr-virtual-keyboard`.

**Solución:** Desinstalar `wtype`:

```bash
sudo apt remove -y wtype
```

---

### El overlay roba el foco

**Solución:** Settings > Advanced > **Overlay Position = None**.

---

### Mirrors de apt no responden

```bash
sudo sed -i 's|co\.archive\.ubuntu\.com|archive.ubuntu.com|g' /etc/apt/sources.list.d/ubuntu.sources
sudo apt update
```

---

### El modelo Whisper no descarga

```bash
mkdir -p ~/.config/com.pais.handy/models
wget -P ~/.config/com.pais.handy/models https://blob.handy.computer/ggml-small.bin
```

---

### No se detecta GPU Vulkan

```bash
sudo apt install -y vulkan-tools mesa-vulkan-drivers
vulkaninfo --summary
```

---

## Arquitectura del pipeline

```
Micrófono → VAD (Silero) → Whisper (GPU Vulkan) → Texto
                                                       ↓
                              ~/.local/bin/handy-paste (External Script)
                                                       ↓
                              DOTOOL_XKB_LAYOUT=latam dotool type "$1"
                                                       ↓
                              uinput (kernel) → App (caracteres acentuados OK)
```

- **Sin portapapeles** — `dotool` escribe directo al subsistema `uinput`
- **Sin condiciones de carrera** — el script controla el tiempo de ejecución
- **Layout consciente** — `libxkbcommon` traduce caracteres a scancodes según el layout real

---

## Referencias

- [Handy docs](https://handy.computer/docs)
- [Paste Methods](https://handy.computer/docs/paste-methods)
- [GitHub: cjpais/Handy](https://github.com/cjpais/Handy)
- [Discussion #718 — GNOME Wayland](https://github.com/cjpais/handy/discussions/718)
- [dotool — git.sr.ht/~geb/dotool](https://git.sr.ht/~geb/dotool)

---

## Handy frente a Voxtype

| Aspecto | Handy | Voxtype |
|---|---|---|
| Interfaz | Aplicación gráfica | Daemon con systemd y `config.toml` |
| Disparo | Atajo global de GNOME (`handy --toggle-transcription`) | Hotkey interno por evdev (grupo `input`) |
| Salida de texto | Script externo con `dotool` | Drivers en cascada (`dotool`, `ydotool`, `wtype`, portapapeles) |
| Transcripción | Local, Whisper con Vulkan | Local o remota (servidor Faster-Whisper) |
| Punto débil en GNOME | Condición de carrera del portapapeles | Hotkey evdev sensible a reconexiones USB |

Ambas dependen de `dotool` con `DOTOOL_XKB_LAYOUT` para escribir tildes en Wayland. Ver [voxtype-guia-configuracion-gnome-wayland.md](voxtype-guia-configuracion-gnome-wayland.md).
