---
title: "Bounty Hacker - THM | by ils00"
author: ils00
categories: [TryHackMe, Easy]
tags: [web, ftp, bruteforce]
render_with_liquid: false
media_subpath: /img_p/bounty_hacker_thm
image:
  path: bounty_logo.jpeg
---



Esta es una de las maneras de resolver la máquina **Bounty Hacker** de **TryHackMe** con categoría **Fácil** en la plataforma.

<img title="" src="maquina.png" alt="" width="684">

## **Conocimientos Previos:**

- Escaneo de puertos con NMAP
- Conceptos básicos de Linux
- Herramientas de fuerza bruta (Hydra)
- Escalada de privilegios básica

Después de saber esto, vamos a comenzar!!

****** ESPERAR 3 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ******

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.webp)

## Escaneo de Puertos:

Primeramente, lanzamos una traza ICMP hacia la máquina víctima para ver si llegamos a ella.

![](ping.webp)

> *Si cambia la IP en el WriteUp es debido a que la VPN tuvo irregularidades y tuve que reiniciar.*

Una vez nos responde podemos ver el **TTL** de la máquina que es **63** por lo que nos estamos enfrentando a una máquina **Linux**.

Lanzamos el primer comando de escaneo con **Nmap** que nos dará una vista general de a que nos enfrentamos.

![](https://miro.medium.com/v2/resize:fit:618/1*NVbvFPnJbgGkaY7Mejx47A.png)

`namp -p- --open -sS -n -Pn --min-rate 5000 -v [IP VISTIMA]`

Ahora que tenemos los puertos abiertos, vamos a lanzar otro comando para poder analizarlos más en profundidad.

![](nmap2.webp)

`namp -p21,22,80 -sCV [IP VICTIMA] -oN Objetivo`

Con este comando estaremos lanzando varios scripts básicos de reconocimiento y finalmente todo el resultado lo va a meter el fichero **Objetivo**, por si lo necesitamos consultar más tarde.

En el resultado podemos observar varias cosas:

- **Puerto 21 FTP** → La versión no es vulnerable a nada que nos pueda interesar, pero vemos algo que si es interesante.

![](ftp.webp)

Podemos conectarnos sin usuario, lo podemos hacer de manera anónima.

- **Puerto 22 SSH** → Gracias a la versión del OpenSSH podemos saber presuntamente a que distribución nos estamos enfrentando, en este caso a **Ubuntu Xenial**
- Puerto 80 HTTP → Aquí podemos ver que tenemos un servicio web activo, la versión no es vulnerable a nada que nos interese.

## Investigación FTP:

Hemos visto que tenemos acceso al protocolo FTP de la máquina de forma anónima, pues vamosa ver que contiene:

`ftp [IP VICTIMA]  `
name: anonymous

![](ftp2.webp)

Ya estamos dentro ahora con el comando **dir** podemos listar ficheros de dentro del servicio.

![](ftp3.webp)

Vemos que tenemos dos fichero **locks.txt** y **task.txt.** Me los voy a traer a mi máquina para poder verlos.

`get locks.txt ||| get task.txt`

Leemos el de **locks.txt.**

![](ftp4.webp)

Al parecer parece un fichero de contraseñas.

Vemos el otro:

![](cat.webp)

Tiene pinta de lista, y podemos observar que lo ha firmado **lin** que puede ser perfectamente un usuario del sistema.

Para ello vamos a comprobarlo, vamos a intentar explotar el protocolo SSH que hemos visto que está abierto mediante una herramienta de fuerza bruta.

## Explotación SSH:

La herramienta que vamos a utilizar es **hydra.**

`hydra -l lin -P [DICCIONARO ENCONTRADO] ssh://[IP VICTUMA]`

![](hydra.webp)

Podemos ver que una vez lanzado nos da la contraseña del este usuario para el protocolo SSH.

Vamos a entrar!!

![](ssh.webp)

Una vez dentro, podemos ver la bandera de usuario.

## Escalada de Privilegios:

Ahora que estamos dentro de la máquina vamos a ver si podemos escalar a root de una manera sencilla, investiguemos:

Una de las primeras cosas que pruebo es a ver si tengo permisos de sudo con algún comando, y BINGO!!

Con el comando **/bin/tar**

![](bintar.webp)

Entonces haciendo una búsqueda en la página **GTFOBins** en el apartado **SUDO** podemos ver el payload.

`sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh`



[gtfobins.github.io](https://gtfobins.github.io/gtfobins/tar/#sudo)



Lo ejecutamos y…

![](root.webp)

Somos ROOT!!!

## Conclusión

Es una máquina de nivel fácil muy básica, se puede hacer en aproximadamente **30min** con conocimiento y sin mucho conocimiento en **1h.**

Un saludo, GOOD HACKING!!!

***XIII***
