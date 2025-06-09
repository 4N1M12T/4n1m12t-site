---
layout: post
title: ClamAV (Easy) - Proving Grounds Practice
date: 08.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, HTTP, SNMP, SMTP, ClamAV, PERL, black-hole.pl, Easy]
---

# Resolucion paso a paso de la maquina ClamAV     <br>



- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.229.42 -oG allPorts
```

![image.png](/assets/img/post-img/clamAV/image.png)   



- Enumeramos los servicios activos en los puertos que hemos descubierto:

```bash
nmap -sCV -p22,80 --min-rate 5000 192.168.229.232 -oN targeted
```

![image.png](/assets/img/post-img/clamAV/image%201.png)   



## WEB (HTTP)

- Accediendo a la web encontramos un código binario que intentamos descifrar y parece una contraseña  “**ifyoudontpwnmeuran00b**”. (En la pestaña vemos “**Ph33r**” que podría ser un nombre de usuario o algo que puede sernos útil más tarde): 


![image.png](/assets/img/post-img/clamAV/image%202.png) 


![image.png](/assets/img/post-img/clamAV/image%203.png) 



```bash
01101001 01100110 01111001 01101111 01110101 01100100 01101111 01101110 01110100 01110000 01110111 01101110 01101101 01100101 01110101 01110010 01100001 01101110 00110000 0011 0000 01100010 
```

- **ifyoudontpwnmeuran00b**  



## SMB

- Enumeramos SMB usando las credenciales que hemos conseguido pero no encontramos nada de interés.  



### SMBMAP

```bash
smbmap -H 192.168.229.42 -u 'Ph33r' -p 'ifyoudontpwnmeuran00b'
```

![image.png](/assets/img/post-img/clamAV/image%204.png)  



### SMBCLIENT

```bash
smbclient -L 192.168.229.42 -U Ph33r -N
```

![image.png](/assets/img/post-img/clamAV/image%205.png)  



### SNMP

- Enumeramos el servicio **SNMP** por el puerto **161** por **UDP** y vemos algo que llama la atención:

```bash
sudo nmap -sC -sV -sU -p161 192.168.229.42
```

![image.png](/assets/img/post-img/clamAV/image%206.png)  



- Analizando el primer escaneo de **NMAP** vemos la versión de **SENDMAIL 8.13.4** y buscando en **EXPLOITDB** encontramos un **EXPLOIT** (**black-hole.pl**) escrito en **PERL** que parece corresponder con la información que hemos recopilado:

[Sendmail with clamav-milter < 0.91.2 - Remote Command Execution](https://www.exploit-db.com/exploits/4761)  



- Lanzamos el **EXPLOIT** y vemos que nos pide un **HOST**.

![image.png](/assets/img/post-img/clamAV/image%207.png)  



- Le pasamos el **HOST** de la máquina víctima y recibimos el siguiente **OUTPUT** pero nada mas:

![image.png](/assets/img/post-img/clamAV/image%208.png)  



- Analizando el código del **EXPLOIT** parece que realiza un **ECHO** de una **STRING** con el puerto **31337** y envia la salida el archivo **/etc/inetd.conf**. Lanzamos un **NMAP** hacia el **31337** y vemos que tiene un estado **OPEN** y un servicio **ELITE**.

```bash
nmap -p 31337 -v -n 192.168.229.42
```

![image.png](/assets/img/post-img/clamAV/image%209.png)  



- Sabemos que en primer escaneo que hemos realizado con **NMAP** el puerto **31337** no estaba abierto. Intentamos conectarnos con **NC** apuntando directamente al puerto y vemos que hemos logrado la ejecución de comandos como **ROOT**:

```bash
nc -nv 192.168.229.42 31337
```

![image.png](/assets/img/post-img/clamAV/image%2010.png)  



- Nos movemos al directorio **ROOT** y podemos leer la **FLAG**:

![image.png](/assets/img/post-img/clamAV/image%2011.png)