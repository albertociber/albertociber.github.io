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
---
![Precious](/assets/images/CTFs/Precious-HTB/header.png)

# Enumeración 🔎

Como siempre realizamos una pequeña enumeración rápida con el siguiente comando buscando cualquier puerto abierto.

```bash
nmap -sS  -T5 --min-rate 5000 -p- 10.129.228.98 -vvv -oG allports
```
