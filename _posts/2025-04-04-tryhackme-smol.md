---
title: "Smol - THM | by ils00"
author: ils00
categories: [TryHackMe, Medium]
tags: [web, burpsuite, wordpress, lfi, rce, bruteforce, backdoor, reversing, pivoting]
render_with_liquid: false
media_subpath: /img_p/smol_thm
image:
  path: smol_logo.webp
---

Esta es una de las maneras de resolver la máquina **Smol** de **TryHackMe** con categoría **Media** en la plataforma.

![](maquina.png)

## Conocimientos Previos:

- Burp Suite
- Análisis de máquinas Linux
- Vulnerabilidades Wordpress
- LFI y RCE
- Herramientas de Fuerza Bruta (John de Ripper)
- BackDoor (básico)
- Reversing
- Escalada de privilegios y pivoting de usuarios

Después de revisar estos conceptos, comencemos!!!

**** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ****

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.webp)

## Escaneo de Puertos:

Primero le mandamos una traza ICMP hacia la máquina víctima para ver si nos responde.

![](ping.webp)

Una vez que nos responde analizamos la traza y podemos ver el TTL que es igual a **63** por lo tanto estamos ante una máquina **Linux**.

Lanzamos el primer comando de escaneo de **Nmap** que nos dirá una vista de más o menos a que nos estamos enfrentando.

![](nmap1.webp)

`namp -p- --open -sS -n -Pn --min-rate 5000 -v [IP VICTIMA]`

Ahora vamos a escanear los puertos de una manera más detallada.

![](nmap2.webp)

`nmap -p[PORTS] -sCV [IP VICTIMA] -oN [FICHERO]`

Ahora con este escaneo, podemos ver mejor a que nos estamos enfrentando. Análicemos los puertos:

- **Puerto 22** → Servicio SSH corriendo, y gracias a la versión de OpenSSH podemos saber presuntamente que estamos ante un **Ubuntu Focal**.
- **Puerto 80** → Vemos que tenemos un servidor web levantado y que se esta aplicando Virtual Hosting, por lo que vamos a tener que meter el dominio en el fichero hosts.

![](hosts.webp)

Ahora si podemos ver la página.

![](web1.webp)

Vamos a investigar a ver por dónde podríamos empezar.

## Análisis:

![](web2.webp)

Ya tenemos una pista de algo, es un WordPress.

![](web3.webp)

Aquí también podemos ver un correo.

Ya que sabemos que es un wordpress, vamos a utilizar una herramienta llamada **Wpscan** que nos podría reportar mucha información sobre este CMS.

`wpscan --url http://www.smol.thm`

![](wpscan.webp)

En un apartado nos da la versión que gracias a esto podemos buscar vulnerabilidades.

Un poco más abajo, nos reporta un plugin llamado **jsmol2wp**, con su respectiva versión **1.07**.

![](wpscan2.webp)

Ahora otra cosa que podemos hacer es buscar usuarios validos en el wordpress, con el siguiente comando:

`wpscan --url http://www.smol.thm --enumerate u`

![](wpscan3.webp)

Nos reportan estos usuarios, por lo que yo he optado por guardarlo en un fichero.

Ya que tenemos una lista de usuarios podemos probar a hacer un ataque de fuerza bruta.

`wpscan --url http://www.smol.thm --password pass.txt --usernames [LISTA CREADA]`

Después de estar un rato, no he sacado nada en claro.

## Explotación de Vulnerabilidad de un Plugin de WordPress:

Vamos a analizar por otro lado, como por ejemplo el plugin **jsmol2wp.**

Una de las primeras búsquedas:

![](vuln.webp)

[pentest-tools.com](https://pentest-tools.com/vulnerabilities-exploits/wordpress-jsmol2wp-107-local-file-inclusion_2654)

Si seguimos investigando podemos ver que mediante una consulta por la URL podemos ver el contenido de los ficheros. Lo que es un **LFI.**

En esta página podemos ver la query.

[wpscan.com](https://wpscan.com/vulnerability/0bbf1542-6e00-4a68-97f6-48a7790d1c3e/))

`http://localhost:8080/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php`

Así que para hacerlo vamos a enviar una petición simple al Burp Suite desde esta parte de la página. Como en la foto.

![](vuln2.webp)

Una vez enviado al Repeater editamos y ponemos la query correcta para poder acceder a la config.

![](burp.webp)

De esta manera, y BINGO!!! Tenemos acceso a ficheros del sistema.

![](burp2.webp)

**wp-config.php**

Vamos a acceder al **/etc/passwd** para ver los usuarios del sistema y a ver si nos podemos aprovechar de ver sus claves **id_rsa**.

![](burp3.webp)

**/etc/passwd**

Podemos probar médiate el **LFI** a ver si podemos acceder a su **.ssh/id_rsa** pero lo he estado mirando y **NADA.** Pero sería una posibilidad.

Ahora que hemos visto todo esto en el fichero **wp-config.php** hemos visto una contraseña (kbLSF2Vop#lw3rjDZ629*Z%G). Vamos a probar a ponerla como el usuario **wpuser.**

Y BINGO!!!

![](dashboard.webp)

Estamos dentro del **dashboard**.

**RCE Wordpress (Hello Dolly):**

Vamos a analizar el dashboard en busca de algo que sea interesante.

![](dashboard2.webp)

Si entramos dentro podemos editarlo y ver que hay una lista, en la cual hay algo que me llama la atención.

![](https://miro.medium.com/v2/resize:fit:875/1*7fbpkmMQ4nyYkx0rscXMtw.png)

Vamos a buscar eso de “Hello Dolly” y en todos los artículos que he visto pone que es un plugin de wordpress que siempre tiene que estar desactivado…

Viendo un poco por internet, me he dado cuenta que el fichero de este plugin se llama **hello.php**.

Entonces vamos a buscarlo con la vulnerabilidad LFI encontrada anteriormente.

![](burp4.webp)

![](burp5.webp)

Aquí tenemos la BackDoor que hablaba en la lista. Esta línea contiene la función **eval()**. Esta función permite ejecutar código arbitrario, lo que la convierte en un riesgo de seguridad significativo. Lo que da esto a una posible vulnerabilidad **RCE**.

Decodifiquemos lo que está en base64 dentro de la función.

Podemos hacerlo con [CyberChef](https://gchq.github.io/CyberChef/) o por comandos:

`echo "[CONTENIDO BASE64]" | base64 -d`

![](burp6.webp)

Explicación del código:

- Verifica si existe un parámetro en la URL llamado **cmd** ($_GET[“cmd”])
- Si el parámetro está presente, la función system() ejecuta el contenido cmd como un comando en el servidor.

Lo que hay dentro de los corchetes es la palabra “cmd” en hexadecimal.

Por lo tanto en la URL podemos introducir:

`&cmd=[COMANDO]`

![](dashboard3.webp)

Y podemos ver que tenemos ejecución remota de comando en el servidor.

Ya que poseemos de un RCE, vamos a intentar colar una reverse shell a ver si podemos acceder a la máquina.

## Acceso a la Máquina:

Primeramente, generamos un fichero de reverse shell, yo en este caso no he usado oneliner he usado una reverse shell de php.

[pentestmonkey_shell](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php)

Modificamos los campos como la **IP** y el **PUERTO**.

Una vez la tengamos tenemos que llevarla al servidor, yo he optado por levantarme un servidor en **python** y desde el RCE me lo he traído con **wget**.

![](shell.webp)

Luego, simplemente ejecutamos la shell y…

![](shell2.webp)

ESTAMOS DENTRO!!!

Vamos a tratar la **TTY** para poder trabajar de manera cómoda.

```
script /dev/null -c bash  
Ctrl + Z  
(En la máquina de atacante) --> stty raw -echo; fg  
(Se queda pillado) --> reset xterm  
export TERM=xterm (Para poder limpiar)  
stty rows 44 columns 184
```

Con estos comandos podremos tener una TTY para poder trabajar como si estuviéramos por SSH.

## Escalada de Privilegios (I)

Vamos a intentar escalar de www-data que es un usuario sin ningún tipo de privilegio a un usuario del sistema que tenga algo de permisos.

![](escalada1.webp)

Estos son todos los usuarios, podemos ver que no podemos acceder como **www-data** por lo que tenemos que ver que lo único que tienen en común es el grupo **internal**.

Como no podemos hacer nada, vamos a mirar la info que tenemos y teníamos claves de acceso para una base de datos.

![](burp7.webp)

Así que vamos a empezar por ahí.

![](mysql.webp)

Con este comando podemos acceder a la base de datos **wordpress**.

mysql -u wpuser -p -h localhost wordpress

Hacemos una consulta SQL a la tabla **wp_users**.

`show tables  `
`select * from wp_users;`

![](mysql2.webp)

Podemos ver los hashes de las contraseñas de usuario.

Vamos a ver si podemos romper alguno de los hashes, me he creado un fichero y lo voy a pasar por **John The Ripper**.

![](hashes.webp)

Voy a lanzar este comando, que especifica el formato (phpass), lo he sacado debido al inicio del hash **($P$)**.

`john --format=phpass --wordlist=[DICCIONARIO] [TU FICHERO CON HASHES]`

![](john.webp)

Y OJO, tenemos la contraseña de uno de los usuarios, **diego**.

Vamos a acceder a el que también tiene la bandera de usuario.

![](escalada2.webp)

## Escalada de Privilegios (II)

![](escalada3.webp)

Como hemos visto antes todos los usuarios están metidos dentro del grupo **internal**, por tanto, como el usuario **diego** que también está en ese grupo, podemos acceder a todos los usuarios.

Vamos a ver que cosas interesantes contienen:

En uno de los usuarios, en este caso, **think** podemos observar que tiene el directorio **.ssh** por lo tanto posee una clave **id_rsa.** Vamos a usarla para pivotar a otro usuario.

La leemos mediante el comando:

`cat /home/think/.ssh/id_rsa `

La copiamos en en la máquina atacante creamos esa clave le damos permisos y nos metemos como este usuario.

`chmod 600 id_rsa_think`  
`ssh think@www.smol.thm -i id_rsa_think`

![](ssh.webp)

ESTAMOS DENTRO!!!

## Escalada de Privilegios (III)

Ahora vamos a ver si podemos acceder al usuario **gege** porque al parecer tiene un comprimido en **zip** el cual solamente puede manipular el, ya que por el nombre me interesa poder verlo ya que es una copia de wordpress antigua y puede haber pistas interesantes.

Trás hacer muchas pruebas, una de ellas fue poner: **su gege** y directamente se me auténtico como ese usuario. ¿Porqué pasa eso?

![](gege.webp)

## Mala configuración del fichero PAM:

He investigado y es por la mala configuración del fichero **pam.d** que se ubica en **/etc/pam.d/su** y dentro de este fichero hay una línea la cual permite al usuario root o a un usuario con ciertos privilegios poder cambiar a este usuario sin contraseña.

![](pam.webp)

La línea del fichero

## **Escalada de privilegios (IV):**

Una vez convertido en este usuario me he llevado el fichero a mi máquina para poder analizarlo con detenimiento.

![](zip.webp)

Cuando lo tengo para descomprimir me he dado cuenta que tiene contraseña. Pues vamos a ver si puedo romper esta contraseña.

![](zip2.webp)

Mediante este comando voy a sacar el hash de la contraseña del fichero y almacenarlo en un fichero para mi.

`zip2john wordpress.old.zip > hash_zip.txt`

Luego, cuando tengamos este fichero podemos probar a romperlo.

`john hash_zip.txt --wordlist=rockyou.txt `

![](hash_zip.webp)

Y tenemos la contraseña, entonces ya podemos descomprimirlo.

Una vez descomprimido, vamos a buscar en el **wp_config.php** porque a lo mejor al ser una copia antigua puede contemplar a otro usuario.

Y efectivamente.

![](wp_config.webp)

Vamos a probar a loguear con el usuario **xavi**.

## Escalada de rooteo:

Somos xavi, asi que como ya hemos tocado todos los usuarios vamos a ver si con este podemos llegara a root.

Realizamos una pruebas simples para ver si tenemos alguna pista y…

A la segunda, **sudo -l**

![](root.webp)

Vemos que podemos hacer todo, así que..

**SUDO SU o SUDO BASH**

![](root2.webp)

Ya somos ROOT!!!!

## Conclusión

Es una máquina de nivel medio la cual se necesita algo de conocimiento y algo de investigación. Yo he tardado en resolverla **3h** aproximadamente de trabajo activo, ya que a la hora de pivotar necesitas encontrar la manera de poder saltar.

Un saludo, GOOD HACKING!!!!!

~~XIII~~
