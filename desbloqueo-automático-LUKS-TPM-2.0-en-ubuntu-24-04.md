# Manual: Desbloqueo Automático LUKS + TPM 2.0 en Ubuntu 24.04 (Ryzen 7)

Este manual describe cómo configurar el **desbloqueo automático** de una partición cifrada con **LUKS2** utilizando el chip de seguridad **AMD fTPM (TPM 2.0)** en **Ubuntu 24.04** (Ryzen 7).

## 1. Justificación técnica: ¿por qué Dracut?

En Ubuntu 24.04 Desktop, el generador de initramfs predeterminado es `initramfs-tools`. Para este caso de uso, presenta limitaciones relevantes:

- **Compatibilidad limitada con systemd:** `initramfs-tools` se apoya en scripts legacy de shell que no interpretan correctamente opciones modernas de `systemd-cryptsetup` en `/etc/crypttab` (por ejemplo, `tpm2-device=auto`).
- **Detección incompleta de hardware en el arranque:** al incorporar TPM, estos scripts pueden omitir módulos/controladores necesarios (por ejemplo, para NVMe), lo que puede derivar en fallos de arranque.
- **Alineación con Ubuntu Core:** Ubuntu Core utiliza **Dracut**, que integra `systemd` desde las primeras fases del arranque. Por eso, en Core el soporte TPM funciona de forma más nativa.
- **Dirección futura de Ubuntu:** Canonical ha indicado que Ubuntu Desktop migrará a Dracut en 25.10. Adoptarlo en 24.04 adelanta el sistema a una arquitectura más moderna.

## 2. Implementación paso a paso

### Paso 1: Migración del generador de initramfs

Instala Dracut (y herramientas TPM) y elimina `initramfs-tools` para evitar conflictos.

```bash
sudo apt install dracut tpm2-tools --autoremove --purge
```

**Explicación:** instala Dracut y utilidades para TPM. La opción `--purge` elimina la configuración de `initramfs-tools` para minimizar colisiones.

### Paso 2: Habilitar soporte TPM en Dracut

Indica a Dracut que incluya los módulos/librerías requeridos para comunicarse con el fTPM.

```bash
echo -e 'hostonly="yes"\nadd_dracutmodules+=" tpm2-tss "' | sudo tee /etc/dracut.conf.d/tpm2.conf
```

**Explicación:**

- `hostonly="yes"` genera una imagen optimizada para tu hardware.
- `add_dracutmodules+=" tpm2-tss "` agrega el módulo `tpm2-tss` para habilitar el acceso al TPM.

### Paso 3: Configurar `/etc/crypttab`

Primero, identifica el UUID de la partición LUKS.

```bash
lsblk -f
sudo blkid | grep crypto_LUKS
```

Luego, edita `/etc/crypttab` para que el sistema sepa cómo abrir el volumen. Sustituye `aqui-UUID-de-unidad` por el UUID real.

> **Nota:** el archivo `/etc/crypttab` se edita con un editor (por ejemplo, `sudo nano /etc/crypttab`). El comando siguiente muestra un ejemplo de línea completa.
> 

```bash
# Ejemplo de línea en /etc/crypttab:
# cryptlvm UUID=aqui-UUID-de-unidad none luks,tpm2-device=auto,discard
```

**Explicación de campos/opciones:**

- **`cryptlvm`**: nombre lógico del mapeo una vez se desbloquee el volumen.
- **`UUID=...`**: identificador único de la partición.
- **`none`**: no se usa un archivo de clave (keyfile); el desbloqueo se realizará con otro método (TPM).
- **`luks`**: indica que el volumen usa LUKS.
- **`tpm2-device=auto`**: habilita el uso automático del TPM 2.0 para liberar la clave.
- **`discard`**: habilita TRIM en SSD (recomendado si tu escenario lo permite).

### Paso 4: Vincular el volumen al TPM (enrollment)

**4.1. Limpiar el slot TPM (opcional, pero recomendable si ya existía configuración previa):**

```bash
sudo systemd-cryptenroll --wipe-slot=tpm2 /dev/nvme0n1p3
```

**Explicación:** elimina el slot TPM existente para evitar conflictos y preparar un enrolamiento limpio.

**4.2. Enrolar el volumen con PCR 7:**

```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7 /dev/nvme0n1p3
```

**Explicación:** asocia el volumen LUKS con el TPM 2.0, de forma que el desbloqueo automático solo ocurra si el estado medido coincide con el esperado. **PCR 7** suele reflejar el estado de Secure Boot y componentes relacionados.

### Paso 5: Regenerar initramfs y actualizar GRUB

```bash
sudo dracut -f
sudo update-grub
```

**Explicación:**

- `dracut -f` regenera la imagen de arranque incorporando los cambios (TPM, módulos, etc.).
- `update-grub` asegura que GRUB apunte a la imagen de arranque actualizada.

## 3. Gestión de la frase de paso (passphrase)

### Cambiar la clave manual

El desbloqueo por TPM es una mejora de conveniencia. Mantén una clave manual para contingencias. Para cambiarla:

```bash
sudo cryptsetup luksChangeKey /dev/nvme0n1p3
```

**Explicación:** solicitará la clave actual y luego la nueva. Este cambio afecta la clave manual (slot asociado) y no elimina el token TPM.

### ¿Es necesario reconfigurar tras cambiar la clave?

**No.** El token TPM se almacena como metadatos (LUKS2 JSON) de forma independiente. Cambiar la passphrase manual no invalida el enrolamiento TPM. No necesitas regenerar Dracut ni actualizar GRUB por este motivo.

## 4. Plan de rescate (rollback)

Si el sistema no arranca, puedes recuperar el acceso desde GRUB:

1. En el menú de GRUB, selecciona la entrada de Ubuntu y presiona `e`.
2. Añade `init=/bin/sh` al final de la línea que comienza por `linux`.
3. Presiona `Ctrl+X` para iniciar y ejecuta:

```bash
# Reparar crypttab eliminando las opciones de TPM
mount -o remount,rw /
sed -i 's/,tpm2-device=auto,discard//' /etc/crypttab
reboot -f
```

---

**Nota de seguridad:** se recomienda generar una **clave de recuperación** (Recovery Key) y guardarla fuera del equipo:

```bash
sudo systemd-cryptenroll --recovery-key /dev/nvme0n1p3
```