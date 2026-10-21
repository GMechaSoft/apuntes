# Bitácora: Reversión del desbloqueo automático LUKS + TPM 2.0 (Ubuntu 24.04)

**Fecha:** 2026-10-21
**Objetivo:** Revertir la implementación de desbloqueo automático por TPM (Dracut + `systemd-cryptenroll`) para dejar Ubuntu 24.04 en su estado nativo y que la actualización a Ubuntu 26.04 sea limpia (26 ya trae soporte TPM nativo con `initramfs-tools`).

---

## 1. Estado inicial detectado

| Elemento | Valor |
|---|---|
| SO | Ubuntu 24.04.5 LTS (noble) |
| Generador de initramfs | `dracut` (activo) |
| `initramfs-tools` | Purged (solo quedaban `-core` y `-bin`) |
| `/etc/crypttab` | `cryptlvm UUID=ed051848-… none luks,tpm2-device=auto,discard` |
| Config dracut | `/etc/dracut.conf.d/tpm2.conf` (`hostonly=yes`, `tpm2-tss`) |
| Hook de kernel | `/etc/kernel/postinst.d/dracut` |
| Enrollment TPM | token `systemd-tpm2` (PCR 7), keyslots 1 y 2 |

---

## 2. Comandos ejecutados (auditoría)

### A. Restaurar el generador de initramfs nativo

1. **Simulación** (sin aplicar): `apt-get -s install initramfs-tools` → confirma que se retira `dracut`.
2. `apt-get install -y initramfs-tools` → instala `initramfs-tools` y retira `dracut`.
3. `apt-get remove --purge -y dracut dracut-core dracut-install` → **incidente**: por el conflicto del paquete virtual `linux-initramfs-tool`, arrastró también `initramfs-tools`, `initramfs-tools-core` y `cryptsetup-initramfs`.
4. **Corrección:** `apt-get install -y initramfs-tools cryptsetup-initramfs` → restaura el generador y la integración LUKS (`busybox-initramfs`, `initramfs-tools-bin`, `initramfs-tools-core`, `klibc-utils`, `libklibc`, `dracut-install` como helper interno).

### B. Revertir configuración TPM

5. Respaldo y edición de `/etc/crypttab`:
   - `cp -a /etc/crypttab /etc/crypttab.bak`
   - `sed -i 's/,tpm2-device=auto,discard/,discard/' /etc/crypttab`
   - Resultado: `cryptlvm UUID=ed051848-… none luks,discard`
6. Eliminar config de Dracut: `rm -f /etc/dracut.conf.d/tpm2.conf` y `rmdir /etc/dracut.conf.d`.
7. Wipe del token TPM: `systemd-cryptenroll --wipe-slot=tpm2 /dev/nvme0n1p3`
   - Eliminó token `systemd-tpm2` y keyslot 1.

### C. Regenerar arranque

8. `update-initramfs -u -k all` → regenera `7.0.0-34` y `6.17.0-40`.
9. `update-initramfs -u -k 7.0.0-38-generic` → regenera el kernel en uso (sin salida, timestamp 14:39).
10. `update-grub` → regenera `grub.cfg` con los 3 kernels.

### D. Restaurar paquetes de escritorio purgados por el `--autoremove --purge` original

11. `apt-get install -y brltty tilix python3-nautilus`
    - Instaló: `brltty`, `tilix`, `tilix-common`, `libgtkd-3-0`, `libvted-3-0`, `libphobos2-ldc-shared106`.
    - `python3-nautilus` ya estaba presente.

---

## 3. Estado final verificado

| Elemento | Estado |
|---|---|
| `update-initramfs` | Presente (`/usr/sbin/update-initramfs`) |
| Generador `dracut` | Ausente |
| Hook de kernel | `initramfs-tools` |
| `/etc/crypttab` | `… none luks,discard` (sin TPM) |
| `/etc/dracut.conf.d/` | Eliminado |
| Tokens LUKS | Token `systemd-tpm2` eliminado; queda `systemd-recovery` (keyslot 2) |
| Keyslots LUKS | 0 = passphrase manual · 2 = recovery key |
| `cryptsetup-initramfs` | Instalado e initrds regenerados |
| Paquetes escritorio | Restaurados |

> **Nota:** el paquete `dracut-install` que queda instalado es un helper binario usado por `initramfs-tools` (NO es el generador). Es dependencia de `initramfs-tools` y no debe quitarse.

---

## 4. Pasos requeridos DESPUÉS del reinicio

### 4.1 Reiniciar (tener a mano la passphrase LUKS manual)
```bash
sudo reboot
```
El arranque volverá a pedir la passphrase (el desbloqueo automático por TPM quedó desactivado).

### 4.2 Verificar arranque y reversión
```bash
which update-initramfs     # debe devolver /usr/sbin/update-initramfs
which dracut               # no debe existir (sin salida)
cat /etc/crypttab          # debe ser: ... none luks,discard
sudo systemd-cryptenroll /dev/nvme0n1p3   # no debe listar token systemd-tpm2
```

### 4.3 (Opcional) Eliminar la recovery key si no se desea
La recovery key (`systemd-recovery`, keyslot 2) no es TPM ni afecta al upgrade; es respaldo. Para borrarla:
```bash
sudo systemd-cryptenroll --wipe-slot=recovery /dev/nvme0n1p3
```

### 4.4 Preparar la actualización a Ubuntu 26.04

1. Revisar/actualizar repos de terceros (los desactivará `do-release-upgrade`, pero conviene antes):
   - **CUDA** apunta a `ubuntu2004` → `cuda-ubuntu2004-x86_64.list` (desactualizado).
   - **OBS** es específico de `noble` → `obsproject-ubuntu-obs-studio-noble.sources`.
   - Actualizar a la suite de 26.04 o eliminar si no se usan.
2. Confirmar prerequisitos (ya verificados OK):
   - `Prompt=lts` en `/etc/update-manager/release-upgrades`.
   - Sin paquetes retenidos (`apt-mark showhold` vacío).
3. Actualizar el sistema actual antes del salto:
   ```bash
   sudo apt update && sudo apt full-upgrade
   ```
4. Lanzar la actualización (cuando 26.04 LTS esté disponible):
   ```bash
   sudo do-release-upgrade
   ```

> Tras el upgrade, si se desea, se puede re-activar el TPM de forma nativa con `systemd-cryptenroll --tpm2-device=auto` (sin Dracut).

---

## 5. Notas / lecciones

- `apt-get remove --purge dracut dracut-core` puede arrastrar `initramfs-tools` por el conflicto del paquete virtual `linux-initramfs-tool`. Si ocurre, reinstalar de inmediato: `apt-get install -y initramfs-tools cryptsetup-initramfs`.
- La passphrase manual (keyslot 0) nunca se vio afectada; el sistema siempre conserva una vía de arranque.
- El `--autoremove --purge` original eliminó paquetes de escritorio no relacionados (efecto colateral). Restaurados en esta sesión.
