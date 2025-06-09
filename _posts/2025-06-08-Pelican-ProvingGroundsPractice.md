---
layout: post
title: Pelican (Intermediate) - Proving Grounds Practice
date: 08.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, Privilege Escalation, PKEXEC, PWNKIT, Zookeeper 3.4.6-1569965, Zookeeper, Exhibitor, SUID, Intermediate]
---

 
# Resolucion paso a paso de la maquina PELICAN:  

  

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.229.98 -oG allPorts
```

![image.png](/assets/img/post-img/Pelican/image.png)
  


- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p22,139,445,631,2181,2222,8080,8081,46295 --min-rate 5000 192.168.229.98 -oN targeted
```

![image.png](/assets/img/post-img/Pelican/image%201.png)

![image.png](/assets/img/post-img/Pelican/image%202.png)
  


### EXPLOITDB

- Analizando el **OUTPUT** de **NMAP** podemos ver varios puertos y servicios diferentes. Después de analizar algunas opciones realizamos una búsqueda en **GOOGLE** de “***Zookeeper 3.4.6-1569965***” ya que vemos que parece ejecutarse en el puerto **2181** y encontramos una entrada que hace referencia a **EXHIBITOR** que también los vemos en el puerto **8081** seguido de una **URL** que por la ruta parece ser un **ENPOINT** de alguna **API**.
  


[Exhibitor Web UI 1.7.1 - Remote Code Execution](https://www.exploit-db.com/exploits/48654)
  


### EXPLOTACIÓN

- Leyendo la información parece que se trata de un **RCE** que se realiza desde la **INTERFAZ GRÁFICA** de esta aplicación. Además, también vemos una opción para realizar el mismo proceso desde **CURL** donde vemos una **RUTA** con una estructura muy parecida a la que vimos en el **OUTPUT** de **NMAP**:

![image.png](/assets/img/post-img/Pelican/image%203.png)
  


- Seguimos los pasos que nos indica desde la INTERFAZ GRÁFICA y nos ponemos en escucha con **NC** en nuestra **MÁQUINA ATACANTE**:

![image.png](/assets/img/post-img/Pelican/image%204.png)
  


- Una vez hacemos el **COMMIT** recibimos la conexión:

```bash
nc -lvnp 4444
```

![image.png](/assets/img/post-img/Pelican/image%205.png)
  


### TRATAMIENTO DE LA TTY

- Realizamos el tratamiento de la **TTY** para poder trabajar con comodidad desde una **SHELL INTERACTIVA**:

```bash
script /dev/null -c bash

control+z

stty raw -echo; fg

reset xterm

export TERM=xterm-256color

export SHELL=bash

stty rows 52 columns 214 
```
  


- En el directorio principal del usuario **CHARLES** conseguimos la primera **FLAG**:

![image.png](/assets/img/post-img/Pelican/image%206.png)
  


### SUID - ESCALADA DE PRIVILEGIOS

- Realizamos la enumeración para intentar escalar privilegios y vemos que **PKEXEC** tiene permisos **SUID**:

```bash
find / -perm -4000 2>/dev/null
```

![image.png](/assets/img/post-img/Pelican/image%207.png)
  


- Copiamos el código en **PYTHON** del **PWNKIT** y le damos permisos de ejecución:

[raw.githubusercontent.com](https://raw.githubusercontent.com/Almorabea/pkexec-exploit/refs/heads/main/CVE-2021-4034.py)

![image.png](/assets/img/post-img/Pelican/image%208.png)
  
  

- Ejecutamos **PWNKIT** y logramos escalar privilegios a **ROOT** y leer la FLAG:

![image.png](/assets/img/post-img/Pelican/image%209.png)