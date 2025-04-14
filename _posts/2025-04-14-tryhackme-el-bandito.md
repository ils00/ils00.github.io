---
title: "El Bandito - THM | by ils00"
author: ils00
categories: [TryHackMe, Duro]
tags: [web, js, request_smuggling, burpsuie]
render_with_liquid: false
media_subpath: /img_p/el_bandito_thm
image:
  path: logo.png
---

Esta es una de las maneras de resolver la máquina **El Bandito** de **TryHackMe** con categoría **Dura** en la plataforma.

![](maquina.png)

## Conceptos Previos:

- Burp Suite
- Enumeración con Nmap y Gobuster
- Contrabando de Solicitudes HTTP

Después de tener los conocimientos vamos a comenzar!!!

****** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ******

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.png)

## Escaneo de Puertos:

Primeramente, lanzamos una traza ICMP hacia la máquina víctima para ver si responde.

![](ping.png)

Una vez que nos responde analizamos la traza y podemos ver el TTL que es igual a **63** por lo tanto estamos ante una máquina **Linux**.

## Escaneo de Puertos:

Lanzamos el primer comando de escaneo de **Nmap** que nos dará una vista de más o menos a que nos nos estamos enfrentando.

`nmap -p- --open -sS -n -Pn - min-rate 5000 -v [IP VICTIMA]`

![](nmap1.png)

Ahora que vemos los puertos abiertos vamos a escanearlos más en detalle.

`nmap -p22,80,631,8080 -sCV [IP VICTIMA] -oN Targeted`

```console
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 d8:50:06:c9:71:dd:34:17:fc:d9:0b:d2:2b:70:ff:a3 (RSA)
|   256 36:ff:60:e2:aa:6d:ae:5b:56:24:39:84:47:8f:97:64 (ECDSA)
|_  256 41:67:dc:b0:80:a3:09:fa:80:f0:fa:aa:31:c7:a4:5c (ED25519)
80/tcp   open  ssl/http El Bandito Server
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
| ssl-cert: Subject: commonName=localhost
| Subject Alternative Name: DNS:localhost
| Not valid before: 2021-04-10T06:51:56
|_Not valid after:  2031-04-08T06:51:56
|_ssl-date: TLS randomness does not represent time
|_http-server-header: El Bandito Server
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 404 NOT FOUND
|     Date: Fri, 11 Apr 2025 11:50:33 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 207
|     Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: SAMEORIGIN
|     X-XSS-Protection: 1; mode=block
|     Feature-Policy: microphone 'none'; geolocation 'none';
|     Age: 0
|     Server: El Bandito Server
|     Connection: close
|     <!doctype html>
|     <html lang=en>
|     <title>404 Not Found</title>
|     <h1>Not Found</h1>
|     <p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>
|   GetRequest: 
|     HTTP/1.1 200 OK
|     Date: Fri, 11 Apr 2025 11:49:41 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 58
|     Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: SAMEORIGIN
|     X-XSS-Protection: 1; mode=block
|     Feature-Policy: microphone 'none'; geolocation 'none';
|     Age: 0
|     Server: El Bandito Server
|     Accept-Ranges: bytes
|     Connection: close
|     nothing to see <script src='/static/messages.js'></script>
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Date: Fri, 11 Apr 2025 11:49:41 GMT
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 0
|     Allow: OPTIONS, POST, HEAD, GET
|     Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none';
|     X-Content-Type-Options: nosniff
|     X-Frame-Options: SAMEORIGIN
|     X-XSS-Protection: 1; mode=block
|     Feature-Policy: microphone 'none'; geolocation 'none';
|     Age: 0
|     Server: El Bandito Server
|     Accept-Ranges: bytes
|     Connection: close
|   RTSPRequest: 
|_    HTTP/1.1 400 Bad Request
631/tcp  open  ipp      CUPS 2.4
|_http-title: Forbidden - CUPS v2.4.7
|_http-server-header: CUPS/2.4 IPP/2.1
8080/tcp open  http     nginx
|_http-title: Site doesn't have a title (application/json;charset=UTF-8).
|_http-favicon: Spring Java Framework
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port80-TCP:V=7.94SVN%T=SSL%I=7%D=4/11%Time=67F901D6%P=x86_64-pc-linux-g
SF:nu%r(GetRequest,1E5,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Fri,\x2011\x20A
SF:pr\x202025\x2011:49:41\x20GMT\r\nContent-Type:\x20text/html;\x20charset
SF:=utf-8\r\nContent-Length:\x2058\r\nContent-Security-Policy:\x20default-
SF:src\x20'self';\x20script-src\x20'self';\x20object-src\x20'none';\r\nX-C
SF:ontent-Type-Options:\x20nosniff\r\nX-Frame-Options:\x20SAMEORIGIN\r\nX-
SF:XSS-Protection:\x201;\x20mode=block\r\nFeature-Policy:\x20microphone\x2
SF:0'none';\x20geolocation\x20'none';\r\nAge:\x200\r\nServer:\x20El\x20Ban
SF:dito\x20Server\r\nAccept-Ranges:\x20bytes\r\nConnection:\x20close\r\n\r
SF:\nnothing\x20to\x20see\x20<script\x20src='/static/messages\.js'></scrip
SF:t>")%r(HTTPOptions,1CB,"HTTP/1\.1\x20200\x20OK\r\nDate:\x20Fri,\x2011\x
SF:20Apr\x202025\x2011:49:41\x20GMT\r\nContent-Type:\x20text/html;\x20char
SF:set=utf-8\r\nContent-Length:\x200\r\nAllow:\x20OPTIONS,\x20POST,\x20HEA
SF:D,\x20GET\r\nContent-Security-Policy:\x20default-src\x20'self';\x20scri
SF:pt-src\x20'self';\x20object-src\x20'none';\r\nX-Content-Type-Options:\x
SF:20nosniff\r\nX-Frame-Options:\x20SAMEORIGIN\r\nX-XSS-Protection:\x201;\
SF:x20mode=block\r\nFeature-Policy:\x20microphone\x20'none';\x20geolocatio
SF:n\x20'none';\r\nAge:\x200\r\nServer:\x20El\x20Bandito\x20Server\r\nAcce
SF:pt-Ranges:\x20bytes\r\nConnection:\x20close\r\n\r\n")%r(RTSPRequest,1C,
SF:"HTTP/1\.1\x20400\x20Bad\x20Request\r\n\r\n")%r(FourOhFourRequest,26C,"
SF:HTTP/1\.1\x20404\x20NOT\x20FOUND\r\nDate:\x20Fri,\x2011\x20Apr\x202025\
SF:x2011:50:33\x20GMT\r\nContent-Type:\x20text/html;\x20charset=utf-8\r\nC
SF:ontent-Length:\x20207\r\nContent-Security-Policy:\x20default-src\x20'se
SF:lf';\x20script-src\x20'self';\x20object-src\x20'none';\r\nX-Content-Typ
SF:e-Options:\x20nosniff\r\nX-Frame-Options:\x20SAMEORIGIN\r\nX-XSS-Protec
SF:tion:\x201;\x20mode=block\r\nFeature-Policy:\x20microphone\x20'none';\x
SF:20geolocation\x20'none';\r\nAge:\x200\r\nServer:\x20El\x20Bandito\x20Se
SF:rver\r\nConnection:\x20close\r\n\r\n<!doctype\x20html>\n<html\x20lang=e
SF:n>\n<title>404\x20Not\x20Found</title>\n<h1>Not\x20Found</h1>\n<p>The\x
SF:20requested\x20URL\x20was\x20not\x20found\x20on\x20the\x20server\.\x20I
SF:f\x20you\x20entered\x20the\x20URL\x20manually\x20please\x20check\x20you
SF:r\x20spelling\x20and\x20try\x20again\.</p>\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Puerto **22** SSH 

Puerto **80** HTTP → Este puerto en esta máquina es protocolo seguro es algo raro en este puerto.

Puerto **631** IPP → Este puerto es el Protocolo de Impresión de Internet. Este protocolo permite imprimir documentos de forma segura a través de la red TCP/IP.

Puerto **8080** HTTP-PROXY → Proxy 

## Análisis (I):

He abierto primeramente la web por el puerto **80** y me ha saltado este error:

![](web.png)

Por lo que me he puesto primero con el puerto **8080**.

![](web1.png)

Ahora es el momento de analizar a ver que podemos sacar.

Lo primero que he hecho ha sido analizar posibles directorios que me interesen, esto lo haremos mediante la herramienta **gobuster**.

`gobuster dir -u http://IP_VICTIMA:8080 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -o dir_8080.txt`

![](gobuster.png)

Tenemos varios directorios que investigar, al parecer en la gran mayoría de directorios por código de estado refleja que no tenemos permisos para entrar.

Navegando por la página en la parte superior derecha podemos ver un apartado que nos lleva a un HTML diferente al índice llamado **burn.html**.

![](burn_token.png)

![](web2.png)

Esta es la página mencionada anteriormente, me puse a analizarla y entrando el código pude ver que utilizaba **WebSocket**. 

El WebSocket es un protocolo de comunicación informática que proporciona un canal de comunicación bidireccional simultánea a través de una única conexión TCP.

![](codigo.png)

##Contrabando de solicitudes (I): 

Lo más seguro que tengamos que hacer un contrabando de solicitudes ua que este es un desafío posterior a ver todo este contenido.

Vamos a analizar con Burp Suite la solicitud a la parte de la página **burn.html**.

![](burp.png)

Podemos ver una solicitud HTTP/1.1 WebSocket.

Ahora me quedé pensando un poco en como poder seguir y retrocedí un poco y me puse a ver otra vez el análisis de nmap a ver si podía sacar algo. Y si pude ver algo:

![](nmap_extracto.png)

Podemos ver a qué Framework nos estamos enfrentando 

Al buscar información sobre esto encontré la siguiente información:

[https://hacktricks.boitatech.com.br/pentesting/pentesting-web/spring-actuatorso](https://hacktricks.boitatech.com.br/pentesting/pentesting-web/spring-actuators)

Leyendo en esta web podemos saber como empezar el ataque.

![](info.png)

![](gobuster2.png)

Aquí podemos confirmar con mi fichero extraído del análisis de **gobuster**.

Ahora me puse a buscar como poder explotar esto y encontré esta forma:

## Explotación (I):

En las salas de explicación previa a esta práctica nos dicen que podemos habilitar el contrabando de solicitudes mediante las actualizaciones de **WebSocket**. Crearemos una solicitud malformada para que el proxy asuma que se ha realizando una actualización **WebSocket**, pero al versión del backend se encontrará igual.

El servidor parece estar protegido y verifica si la actualización de WebSocket se realizó correctamente. Por ejemplo, no podemos acceder a **/env**.

![](burp1.png)

Aquí estamos enviando dos solicitudes y hemos cambiando la versión del WebSocket a una versión diferente a la que tiene y nos responde de esta manera.

![](burp2.png)

No podemos contrabandear solicitudes con solo una simple solicitud malformada, necesitamos la forma de engañar al proxy para que crea que se esta estableciendo una conexión WebSocket válida.

Buscando un poco por la web me encontré esto:

![](web3.png)

Esto nos indica el estado del servicio de cada componente dentro del ecosistema de Bandit Token.

Vamos a analizarla con Burp Suite

![](burp3.png)

Ojo, que con esta solicitud de aquí podemos falsificar la solicitud del lado del servidor y apuntar a nuestro servidor como atacante.

Esto lo haremos cambiando la ruta a nuestro servidor de atacante.

![](burp4.png)

```console
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler

if len(sys.argv)-1 != 1:
    print("""
Usage: {} 
    """.format(sys.argv[0]))
    sys.exit()

class Redirect(BaseHTTPRequestHandler):
   def do_GET(self):
       self.protocol_version = "HTTP/1.1"
       self.send_response(101)
       self.end_headers()

HTTPServer(("", int(sys.argv[1])), Redirect).serve_forever()
```

Vamos a reutilizar este script obtenido de la sala del módulo de TryHackMe (URL)

Lo ejecutamos…

![](script.png)

Ahora que lo tenemos levantado vamos a enviar la solicitud a nuestro servidor, editando esta solicitud.

Cuando le demos a enviar en Burp Suite veremos la respuesta en nuestro servidor.

![](script1.png)

Ahora si que podemos ver el contenido del directorio **/env**, unificando dichas solicitudes.

![](burp5.png)

Y en la respuesta nos da el resultado.

![](burp6.png)

Si podemos ver el contenido de **/env** lo podremos ver de los demás.

Vamos a ver primero el directorio **/trace.**

![](burp7.png)

Vemos que tenemos dos directorios mas con nombres bastente interseantes.

Por lo que si los abrimos tendremos la bandera y información de un usuario y una contraseña.

**/admin-creds**

![](burp8.png)

**/admin-flag**

![](burp9.png)

#### Análisis (II):

![](web4.png)

He conseguido abrir la página que antes me daba error. Era debido al protocolo https, es decir, conexión segura pero por el puerto 80.

Es algo raro pero probable.

Podemos ver este mensaje y si vamos al código vemos esto.

![](codigo1.png)

Entramos al script y lo leemos:

Este script carga y muestra mensajes de un chat. Puede enviar mensajes, y si pones ?msg=X en la URL, el mensaje se marca como enviado por “Bot”.

Usa **/getMessage** y **/send_message** para comunicarse con el servidor.

También hay dos usuarios **JACK** y **OLIVER.**

Ahora con el escaneo de gobuster si que puedo realizar debido a que me he dado cuanta que funciona con certificado.

Dentro de este escaneo podemos ver **/access**.

![](gobuster3.png)

Vamos a entrar:

![](login.png)

Aquí tenemos un login.

Lo primero que voy a probar van a ser el usuario y la contraseña que me he guardado antes.

![](login2.png)

![](web5.png)

Ya tenemos acceso a la web, y podemos seleccionar los chats de Jack y Oliver.

## Explotación (II):

Vamos a jugar un poco con Burp Suite para ver como se envían los mensajes. Por lo que he decidido enviar una solicitud de todo.

1. Enviar un mensaje → **/send_message**
2. Recargar la página general →** /message**
3. Acceder al **/getMessage**

Para hacer el contrabando lo haremos será:

Enviaremos una solicitud de contrabando incompleta al punto final **/send_message** con un encabezado **Content-Length** demasiado largo y nuestra cookie, ya que se necesita autorización para enviar mensajes.

Con esta solicitud, cualquier otra que siga a la nuestra se agregará a nuestra solicitud de contrabando y se interpretará con el parámetro **data** en una solicitud al punto final **/send_message**.

![](https://cdn-images-1.medium.com/max/880/1*qUkjxfIcQ40Bod1qdplAKw.png)

Después de enviar la solicitud se queda congelado durante unos segundos y al poco recibiremos un mensaje.

![](burp11.png)

Ya tenemos la segunda bandera.

## Conclusión:

En conclusión esta máquina es bastante avanzada, por mi parte me ha llevado bastante tiempo para poder realizarla debido a que no es muy estable. Me ha llevado alrededor de 5h poder realizarla con ayuda en algunos casos y documentarme sobre el temario aprendido.
