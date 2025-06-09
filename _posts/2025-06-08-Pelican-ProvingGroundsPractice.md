---
layout: post
title: Pelican (Intermediate) - Proving Grounds Practice
date: 08.06.2025
categories: [Linux, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Linux, Privilege Escalation, PKEXEC, PWNKIT, Zookeeper 3.4.6-1569965, Zookeeper, Exhibitor,GTFOBINS,GCORE, UNIX PASSWORD MANAGER, SUID, SUDO, SUDOERS, Intermediate]
---

# Pelican (Easy) - Proving Grounds Practtice

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- --open -v -n 192.168.229.98 -oG allPorts
```

![image.png](assets/img/post-img/Pelican/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p22,139,445,631,2181,2222,8080,8081,46295 --min-rate 5000 192.168.229.98 -oN targeted
```

![image.png](assets/img/post-img/Pelican/image%201.png)

![image.png](assets/img/post-img/Pelican/image%202.png)

### EXPLOITDB

- Analizando el **OUTPUT** de **NMAP** podemos ver varios puertos y servicios diferentes. Después de analizar algunas opciones realizamos una búsqueda en **GOOGLE** de “***Zookeeper 3.4.6-1569965***” ya que vemos que parece ejecutarse en el puerto **2181** y encontramos una entrada que hace referencia a **EXHIBITOR** que también los vemos en el puerto **8081** seguido de una **URL** que por la ruta parece ser un **ENPOINT** de alguna **API**.

[Exhibitor Web UI 1.7.1 - Remote Code Execution](https://www.exploit-db.com/exploits/48654)

## EXPLOTACIÓN

- Leyendo la información parece que se trata de un **RCE** que se realiza desde la **INTERFAZ GRÁFICA** de esta aplicación. Además, también vemos una opción para realizar el mismo proceso desde **CURL** donde vemos una **RUTA** con una estructura muy parecida a la que vimos en el **OUTPUT** de **NMAP**:

![image.png](assets/img/post-img/Pelican/image%203.png)

- Seguimos los pasos que nos indica desde la **INTERFAZ GRÁFICA** y nos ponemos en escucha con **NC** en nuestra **MÁQUINA ATACANTE**:

![image.png](assets/img/post-img/Pelican/image%204.png)

- Una vez hacemos el **COMMIT** recibimos la conexión:

```bash
nc -lvnp 4444
```

![image.png](assets/img/post-img/Pelican/image%205.png)

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

![image.png](assets/img/post-img/Pelican/image%206.png)

## ESCALADA DE PRIVILEGIOS

> Como suele ocurrir con muchas máquinas que se diseñaron antes de que se descubriera la vulnerabilidad **PSEXEC** tenemos dos formas de escalar privilegios, la del propio **PWNKIT** y una escalada de privilegios aprovechando unos permisos mal configurados que nos permiten ejecutar **GCORE** con privilegios de **ROOT** y aprovecharnos de esa ejecución para volcar la información del **PROCESO** de **PASSWOR-STORE** que se está ejecutando.
> 

### GCORE (SUDO) - ESCALADA DE PRIVILEGIOS

- Encontramos que podemos ejecutar **GCORE** con privilegios de **ROOT**. Después de informarnos vemos que es ua herramienta para **VOLCAR EL CONTENIDO DEL CORE DE UN PROCESO EN EJECUCIÓN**.

```bash
sudo -l
```

![image.png](assets/img/post-img/Pelican/image%207.png)

- Buscando un poco de información en **GTFOBINS** vemos que simplemente tenemos que ejecutar el **GCORE** pasándole el **PID** del proceso al que queremos apuntar para que nos vuelque el **CORE**.

[gcore | GTFOBins](https://gtfobins.github.io/gtfobins/gcore/#sudo)

![image.png](assets/img/post-img/Pelican/image%208.png)

- Enumeramos los procesos que hay en ejecución y al filtrar por la palabra **PASSWORD** vemos que solo hay un proceso que se está ejecutando y parece ser un **GESTOR DE CONTRASEÑAS** estándar de **UNIX**, **PASSWORD-STORE**:

```bash
ps -aux | grep password
```

![image.png](assets/img/post-img/Pelican/image%209.png)

- Ahora que tenemos el **PID** de un proceso que parece interesante vamos a lanzar **GCORE** como **SUDO**. Esto nos **VOLCARÁ** la información en un archivo **CORE.PID**:

```bash
sudo /usr/bin/gcore 490
```

![image.png](assets/img/post-img/Pelican/image%2010.png)

- Después de lanzar **CAT** y no poder leer nada probamos con lanzando un **STRINGS** y logramos ver las cadenas imprimibles entre las que se encuentra una posible **CONTRASEÑA** para el usuario **ROOT**:

```bash
strings core.490
```

![image.png](assets/img/post-img/Pelican/image%2011.png)

![image.png](assets/img/post-img/Pelican/image%2012.png)

> **User: root 

> Password: ClogKingpinInning731**
 

- Accedemos como **ROOT** usando la contraseña que acabamos de encontrar:

```bash
su root
```

![image.png](assets/img/post-img/Pelican/image%2013.png)

- Ahora ya podemos leer la **FLAG**:

![image.png](assets/img/post-img/Pelican/image%2014.png)

### SUID - ESCALADA DE PRIVILEGIOS

- Realizamos la enumeración para intentar escalar privilegios y vemos que **PKEXEC** tiene permisos **SUID**:

```bash
find / -perm -4000 2>/dev/null
```

![image.png](assets/img/post-img/Pelican/image%2015.png)

- Copiamos el código en **PYTHON** del **PWNKIT** y le damos permisos de ejecución:

[raw.githubusercontent.com](https://raw.githubusercontent.com/Almorabea/pkexec-exploit/refs/heads/main/CVE-2021-4034.py)

![image.png](assets/img/post-img/Pelican/image%2016.png)

- Ejecutamos **PWNKIT** y logramos escalar privilegios a **ROOT** y leer la **FLAG**:

![image.png](assets/img/post-img/Pelican/image%2017.png)