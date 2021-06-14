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

Olá pessoal!

A máquina desta semana será **Delivery**, outra máquina Linux classificada como fácil do [Hack The Box](https://www.hackthebox.eu), criada por [ippsec](https://app.hackthebox.eu/users/3769).<!--more-->

:information_source: **Info**: Write-ups para máquinas do Hack The Box são postados assim que as respectivas máquinas são aposentadas
{: .notice--info}

![HTB Delivery](https://i.imgur.com/7C0GCed.png){: .align-center}

A resolução desta máquina foi bem interessante, onde tive a oportunidade de aprender a crackear senhas utilizando variações de dicionários utilizando o `hashcat`, além de vários pivoteamentos, porém simples, até que chegássemos nas credenciais de user e logo após sua obtenção, root.

## Enumeração

Como de costume, iniciado com scan rápido do `nmap` para que possamos identificar os serviços publicados nesta máquina.

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

### 80/TCP - Serviço HTTP

Acessando a página e inspecionando o código-fonte, identifiquei que existe um link para **helpdesk.delivery.htb**. Adicionada entrada encontrada, assim como dominio `delivery.htb`, no `/etc/hosts` da máquina para permitir o acesso aos websites publicados nesta máquina.

![HTB Delivery - Website](https://i.imgur.com/DFYYSkN.png){: .align-center}

Ao clicar no link *Contact us*, a seguinte informação é exibida, informando que assim que tivermos um e-mail delivery.htb, poderemos acessar a plataforma **MatterMost**, que, conforme links enumerados, funciona na porta TCP 8065.

> ## CONTACT US
> For unregistered users, please use our HelpDesk to get in touch with our team. Once you have an @delivery.htb email address, you'll be able to have access to our MatterMost server.

Seguindo com a tentativa de conseguir um e-mail com o HelpDesk, aberto um ticket com conteúdo de mentira enquanto inspecionava as requisições via *BurpSuite*.

Após concluída a abertura do ticket, recebida seguinte informação, algo que foi crucial para obter acesso ao MatterMost: um endereço de e-mail do domínio **@delivery.htb**!

![HTB Delivery - Helpdesk Website](https://i.imgur.com/Zh6GeDh.png){: .align-center}

Acessando a plataforma do Mattermost criado uma conta e informado o e-mail recém obtido. Desta forma, caso precisemos receber alguma validação do serviço, os dados estarão disponíveis como comentários no ticket que acabamos de abrir :smile:.

![HTB Delivery - Mattermost](https://i.imgur.com/6zMqj4N.png){: .align-center}

Como esperado, ao verificar o status do ticket criado, comprovamos que nos comentários tínhamos um e-mail de confirmação do Mattermost, enviado para **8024065@delivery.htb**, onde pudemos obter o link de verificação para o serviço.

![HTB Delivery - Email comment](https://i.imgur.com/TLIBKIn.png){: .align-center}

Após verificada a conta, pudemos acessar o portal com as credenciais previamente utilizadas e obter acesso a um time chamado **Internal**.

![HTB Delivery - Mattermost - Account Confirmed](https://i.imgur.com/qN2YraR.png){: .align-center}

Dentre as mensagens neste time, algumas chamaram atenção, que continham informações importantes para a resolução desta máquina:

- Credenciais para o osTicket são **maildeliverer:Youve_G0t_Mail!**.
- Desenvolvedores utilizavam com frequência *variações de **PleaseSubscribe!*** e deveriam deixar de fazê-lo.
  - Estas variações poderiam ser facilmente craqueadas utilizando **`hashcat` rules**, que foi uma dica bastante valiosa em como quebrar a senha de root ou algo em seu caminho.

![HTB Delivery - Mattermost Internal Channel](https://i.imgur.com/8KP9NRl.png){: .align-center}

Buscando a página de administração do osTicket, fiz logoff da conta *Guest User*, do lado esquerdo superior e, durante o processo de sign-in, selecionei a opção **I'm an agent - sign in here**, que me levou à URL `http://helpdesk.delivery.htb/scp/login.php`, de onde é feita a administração dos incidentes gerados na plataforma.

![HTB Delivery - OsTicket Admin Portal](https://i.imgur.com/JsI5fmb.png){: .align-center}

Ao acessá-la com as credenciais do usuário **maildeliverer**, pude confirmar acesso ao serviço e validar a versão em execução do osTicket, que é a **1.15.1**, onde vamos buscar um acesso inicial a partir de alguma vulnerabilidade ou funcionalidade existente na plataforma.

![HTB Delivery - OsTicket Version](https://i.imgur.com/vYiQjKN.png){: .align-center}

Navegando pela console, encontrei uma API, onde menciona a execução de tarefas usando `cron`, que poderia eventualmente ser utilizado pra algum acesso inicial na máquina. Vendo a documentação encontrei [este link](https://docs.osticket.com/en/latest/Developer%20Documentation/API/Tasks.html?highlight=cron) que menciona o arquivo `scripts\rcron.php` o qual é utilizado para manipular as chamadas.

Segui então com a criação de uma API key para o endereço IP da minha máquina e baixei o script [deste link](https://raw.githubusercontent.com/osTicket/osTicket/1.15.x/setup/scripts/rcron.php) para inspecionar a execução, **porém em nada disso obtive o sucesso esperado**.

## Acesso inicial e User flag

Embora tenha analisado algumas possibilidades via osTicket API, acabei voltando atrás e pensando simples, me perguntando: Será que a conta **maildeliverer** é um usuário desta máquina? Ao tentar conectar via SSH usando as mesmas credenciais do osTicket consegui o acesso inicial com esta conta :man_facepalming:. De quebra, dentro do diretório deste usuário havia o arquivo `user.txt` o qual permitiu a leitura da flag.

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

Iniciado com a execução do `linpeas.sh` para que pudesse realizar a enumeração mais rapidamente, onde os seguintes pontos chamaram atenção:

- Processo `python3 /root/py-smtp.py` em execução pelo usuário **root**. Pode eventualmente permitir algum tipo de path hijack dependendo de quais binários são chamados por este script, caso tenhamos acesso leitura no mesmo.
- Também é executado por **root** um script em `/root/mail.sh`, podendo ser eventualmente explorado conforme item anterior.
- Serviço MySQL em execução na máquina na porta padrão (3306/TCP).

### osTicket

Embora pudessem ser caminhos promissores, não foi possível ler os arquivos citados, logo parti pra outra abordagem que seria buscar por credencias para conexão no MySQL. Analisando os arquivos do osTicket, encontrei em `/var/www/osticket/upload/include/ost-config.php` as credenciais para o usuário **ost_user**:

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

Entretanto ao se conectar no banco de dados, não encontrado nenhuma informação útil, onde decidi seguir para a aplicação Mattermost.

### Mattermost

De acordo com os processos em execução, a aplicação é executada a partir do diretório `/opt/mattermost/config/config.json` encontrei outra credencial para conexão ao mysql.

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

Utilizando as credenciais **mmuser:Crack_The_MM_Admin_PW** acessei o database **mattermost** e na tabela users, encontrei os seguintes usuários e hashes, onde temos o usuário **root** com privilégios de **system_admin**

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

Uma vez que no Mattermost o usuário root disse que a senha não estaria no dicionário, mas que poderia ser uma variação de `PleaseSubscribe!`, pesquisando sobre algumas formas de fazer o cracking desta senha encontrei a seguinte forma usando o `hashcat`:

- `basicpwd.txt` contém a senha `PleaseSubscribe!` conforme mencionado no Mattermost;
- O arquivo `best64.rule` contém algumas das mais populares formas de variação de senha.

Executando selecionando o tipo de hash correto (3200 - bcrypt), encontramos a senha **PleaseSubscribe!21**

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

Utilizando a senha encontrada foi possível obter a última flag :smiley:

```bash
maildeliverer@Delivery:/opt/mattermost/config$ su root
Password:
root@Delivery:/opt/mattermost/config# cd /root/
root@Delivery:~# cat root.txt
<redacted>
root@Delivery:~#
```

Além do root flag, no arquivo `note.txt` ippsec deixa uma mensagem:

> I hope you enjoyed this box, the attack may seem silly but it demonstrates a pretty high risk vulnerability I've seen several times.  The inspiration for the box is here:
>
> \- `https://medium.com/intigriti/how-i-hacked-hundreds-of-companies-through-their-helpdesk-b7680ddc2d4c`
>
> Keep on hacking! And please don't forget to subscribe to all the security streamers out there.
>
> \- ippsec

Espero que tenham gostado!

Até a próxima! :smile:
