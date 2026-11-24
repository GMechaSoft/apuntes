## Guía para configurar Rclone en Ubuntu 26.04 con soporte para nombres de archivo reales, rutas estándar de Linux y automatización total en segundo plano.

Requisito previo: Instalación y Vinculación

1. Instala Rclone desde la terminal:bash
sudo apt update && sudo apt install rclone -y
Usa el código con precaución.
2. Inicia la configuración interactiva:bash
rclone config
Usa el código con precaución.
3. Escribe n (New remote) y asígnale el nombre: gdrive
4. En la lista de servicios, busca Google Drive (escribe drive).
5. Deja en blanco (Enter) client_id y client_secret. En Scope, elige 1 (Acceso total).
6. En Use auto config? escribe y. Se abrirá tu navegador; inicia sesión con tu cuenta de Google y autoriza el acceso. Vuelve a la terminal y presiona q para salir.

OPCIÓN A: Montaje Virtual con Caché VFS (Para ahorrar espacio local)

Los archivos se quedan en la nube y solo se descargan temporalmente en tu SSD cuando los abres.
1. Crear el punto de acceso:bash
mkdir -p ~/GoogleDrive
Usa el código con precaución.
2. Automatizar el inicio con Ubuntu:
	• Abre la aplicación Aplicaciones al inicio (Startup Applications) desde el menú de Ubuntu.
	• Haz clic en Añadir y completa los campos:
		• Nombre: Google Drive (Rclone)
		• Comando: (Reemplaza tu_usuario por tu nombre real de Linux):bash
rclone mount gdrive: /home/tu_usuario/GoogleDrive --vfs-cache-mode writes --vfs-cache-max-size 10G --dir-cache-time 5m --bg
Usa el código con precaución.
		• Comentario: Montar Drive en la nube usando caché temporal.
	• Guarda los cambios. Se montará solo cada vez que enciendas la PC.

OPCIÓN B: Copia Física Local (Para trabajar 100% sin internet)

Descarga copias reales a tu disco duro. Usaremos un script automatizado para que se sincronice solo de forma bidireccional en segundo plano sin que tengas que ejecutar comandos manuales.

Paso 1: Crear la carpeta física local

bash
mkdir -p ~/MiDriveLocal
Usa el código con precaución.

Paso 2: Crear el script de automatización inteligente

1. Abre el editor desde la terminal:bash
nano ~/.sincroniza_drive.sh
Usa el código con precaución.
2. Pega el siguiente código (asegúrate de cambiar tu_usuario por tu nombre de usuario real en Ubuntu):bash
#!/bin/bash
# 1. Traer cambios nuevos de la nube a la PC sin borrar archivos locales
/usr/bin/rclone copy gdrive: /home/tu_usuario/MiDriveLocal

# 2. Subir cambios de la PC a la nube haciendo que la nube sea idéntica a tu PC
/usr/bin/rclone sync /home/tu_usuario/MiDriveLocal gdrive:
Usa el código con precaución.
3. Guarda el archivo presionando Ctrl + O, luego Enter y sal con Ctrl + X.
4. Dale permisos de ejecución al script:bash
chmod +x ~/.sincroniza_drive.sh
Usa el código con precaución.

Paso 3: Programar la sincronización automática (Cada 15 minutos)

1. Abre el planificador de tareas nativo del sistema:bash
crontab -e
Usa el código con precaución.
(Si te pide elegir un editor, presiona 1 para seleccionar nano).
2. Ve hasta el final del archivo y añade esta línea exacta:text
*/15 * * * * /home/tu_usuario/.sincroniza_drive.sh >/dev/null 2>&1
Usa el código con precaución.
(Recuerda cambiar tu_usuario por el tuyo).
3. Guarda y cierra (Ctrl + O, Enter, Ctrl + X).
A partir de este momento, tu Ubuntu ejecutará el script en silencio cada 15 minutos, manteniendo tus archivos locales y la nube perfectamente sincronizados de manera transparente.
¿Prefieres empezar aplicando la Opción A (Montaje Virtual) para ahorrar almacenamiento o la Opción B (Copia Local) con el script automatizado?
