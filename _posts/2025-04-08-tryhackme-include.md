---
title: "Include - THM | by ils00"
author: ils00
categories: [TryHackMe, Medium]
tags: [web, ssrf, api, lfi, bruteforce]
render_with_liquid: false
media_subpath: /img_p/include_thm
image:
  path: logo.png
---


Esta es una de las maneras de resolver la máquina **Include** de TryHackMe con categoría **Media** en la plataforma. Esta es una sala que se encuentra en la ruta de aprendizaje **Advanced Service-Side Attacks**.

![](maquina.webp){: width="700" height="400" }

## Conocimientos Previos:

- Server-Side Attacks (Avanzados)
- API
- Networking (Básico)
- LFI
- Código web (Básico)
- Fuerza Bruta (Básico)

Después de saber todo esto, comencemos!!

****** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ******

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.webp)

Le hacemos un Ping para ver si está levantada la máquina.

![](ping.webp)

Una vez que analizamos esta traza y podemos ver el TTL que es igual a **63** por lo tanto estamos ante una máquina **Linux.**

## Escaneo de Puertos:

Lanzamos el primer comando de escaneo de Nmap que nos dará una vista de más o menos a que nos estamos enfrentando.

nmap -p- --open -sS -n -Pn --min-rate 5000 -v [IP VICTIMA]

![](nmap1.webp)

Ahora vamos a escanear estos puertos de manera más detallada:

nmap -p22,25,110,143,993,995,4000,50000 -sCV [IP VICTIMA] -oN Targeted

![](nmap2.webp)

Aquí tenemos todos los puertos y servicios que están corriendo.

Puerto 22 SSH  
Puerto 25 SMTP  
Puerto 110 POP3  
Puerto 143 IMAP  
Puerto 993 IMAP/SSL  
Puerto 995 POP/SSL  
Puerto 4000 HTTP  
Puerto 50000 HTTP

En este caso nos vamos a centrar en los dos últimos que vemos que corre un servicio web por detrás.

## Análisis I:

Empezamos por el puerto 50000, como podemos ver, parece un portal el cual nos dice que tenemos el acceso restringido.

http://[IP]:50000

![](p50000.webp)

Seguidamente, vamos a realizar un ataque de fuerza bruta de directorios con la herramienta **gobuster**.

gobuster dir -u http://[IP]:50000 -w /diccionario.txt

![](https://miro.medium.com/v2/resize:fit:700/1*6xae9VFgzAHH9_pwaEfHPQ.png)

Vemos varios directorios, vamos a ir mirando que contienen.

Al parecer estos directorios no contienen nada que interesante.

## Análisis II:

En otro portal ubicado en el puerto **4000** podemos ver un inicio de sesión y como podemos observar parece que nos dan la contraseña de invitado.

![](https://miro.medium.com/v2/resize:fit:621/1*2RGKNz4QXic43DsY55ZXjw.png)

Una vez iniciamos podemos ver la web.

![](https://miro.medium.com/v2/resize:fit:700/1*OwMzqOIi7YJNAhVqLZxGsw.png)

Observemos que hay un perfil de invitado (que es el nuestro) y dos amigos, vamos a ver que contienen.

Revisando todos los perfiles vemos que todos incluso el mío propio de invitado tienen las mismas características.

![](https://miro.medium.com/v2/resize:fit:700/1*CSEej6Mv2KYUlfHEbxgTww.png)

Al revisar con detalle, me llama la atención el campo **isAdmin: false** esto quiere decir que no tiene permisos de administrador.

Pero y si intento cambiarlo, en la parte de abajo vamos a introducir: **isAdmin** y **True**.

![](https://miro.medium.com/v2/resize:fit:700/1*DwycaYTBar7xCTgw4wfimw.png)

![](https://miro.medium.com/v2/resize:fit:223/1*9AdtV3PEMZ0ht7BE51iLyQ.png)

Y vemos que si se cambia, sabiendo esto puedo intentarlo hacer con mi perfil a ver si me da algunos permisos especiales.

Y efectivamente, nos da unos permisos especiales, nos desbloquea unos apartados en la parte superior izquierda del índice.

![](https://miro.medium.com/v2/resize:fit:535/1*5hDIJVcHI601yeLfGOIO_w.png)

Entramos en el apartado de **API** y podemos ver que tenemos información interesante.

![](https://miro.medium.com/v2/resize:fit:700/1*TRVib4ZW_QE0rsQOuF08AQ.png)

Investigando un poco entre en el apartado **Settings** y podemos ver algo como esto:

![](https://miro.medium.com/v2/resize:fit:700/1*MJvDpyvVwYLCvgB-x2D_DQ.png)

Obervamos que podemos poner aqui una imagen para un “Banner” de la aplicación. Si esto no está bien sanitizado, es decir, que no tiene filtros, podemos probar a subir el enlace de la API a ver si obtenemos respuesta, ya que forma parte de la red interna.

![](https://miro.medium.com/v2/resize:fit:700/1*O3_JwB8ls_tKG8pwHEWhAw.png)

La introducimos y el resultado que vemos es el siguiente:

 data:application/json; charset=utf-8;base64,eyJSZXZpZXdBcHBVc2VybmFtZSI6ImFkbWluIiwiUmV2aWV3QXBwUGFzc3dvcmQiOiJhZG1pbkAhISEiLCJTeXNNb25BcHBVc2VybmFtZSI6ImFkbWluaXN0cmF0b3IiLCJTeXNNb25BcHBQYXNzd29yZCI6IlMkOSRxazZkIyoqTFFVIn0=

Decodificamos el contenido en **base64** mediante el comando:

echo "[CONTENT]" | base64 -d

![](https://miro.medium.com/v2/resize:fit:700/1*LYbht6QXfOmaNuONk8S7bw.png)

El resultado es parecido a algo que hemos visto antes:

![](https://miro.medium.com/v2/resize:fit:568/1*kb8pYgNcPIxvSNOLOt-VuA.png)

Ahora ponemos el usuario y la contraseña que hemos encontrado para el **SysMon** y podemos contestar a la primera pregunta.

![](https://miro.medium.com/v2/resize:fit:382/1*c1nPWXjcEHhuZsdKNHdctA.png)

## Explotación:

Tras un rato dando vueltas por la página web me puse a mirar el código y encontré lo siguiente:

![](https://miro.medium.com/v2/resize:fit:700/1*YLPhIrvSF-3iekSmYX-Law.png)

Una imagen con una ruta un poco rara, la abrí y pensé que esto podría acceder a archivos confidenciales del sistema.

Me puse a intentar acceder explotando una vulnerabilidad **LFI**.

![](https://miro.medium.com/v2/resize:fit:612/1*fz4XapQCcCh6cIIEw9O0EA.png)

**Intento 1:**

../../../../../../../etc/passwd

SIN ÉXITO

**Intento 2:**

Luego pensé que tendrían algo que estuviera bloqueando la combinación de caracteres ../, por lo tanto lo hice doble.

....//....//....//....//etc/passwd

SIN ÉXITO

**Intento 3:**

Seguidamente probé a urlencodearlo por si había otro tipo de filtro.

....%2F%2F....%2F%2F....%2F%2F....%2F%2F....%2F%2F....%2F%2F....%2F%2F....%2F%2F....%2F%2F....%2F%2Fetc%2Fpasswd

Y BINGOO!!

![](https://miro.medium.com/v2/resize:fit:700/1*u8Zg0yoCVDARMk0pMt0eYw.png)

![](https://miro.medium.com/v2/resize:fit:700/1*L_uD98zNqB-IUGQ6nvjxUw.png)

Podemos ver dos usuarios, vamos a probar a acceder a ellos ya que hemos visto en el escaneo que el puerto **SSH** está abierto.

Para esta explotación de fuerza bruta la realizaremos con la herramienta **hydra.**

Probaremos los dos usuarios con la misma lista de contraseñas y BINGOO!

hydra -l joshua -P [DICCIONARIO] ssh://[IP]

hydra -l charles -P [DICCIONARIO] ssh://[IP]

Después de acceder a un usuario podremos contestar a la segunda pregunta.

![](https://miro.medium.com/v2/resize:fit:700/1*fUeLeZXKOgMhXRPVNVGoXw.png)

Y ya lo tenemos.

En esta máquina debido a que es una practica para practicar otro tipo de aptitudes no hay escalada a root.

## Conclusión:

Es una máquina que toca algunos temas que son complicados debido a que hay que buscar bastante, tema codigo y tema puntos finales. Por mi parte, me ha llevado **2h** realizar esta máquina y he tenido que buscar algo de información en los apuntes debido a que el principio tiene algo de complejidad.
