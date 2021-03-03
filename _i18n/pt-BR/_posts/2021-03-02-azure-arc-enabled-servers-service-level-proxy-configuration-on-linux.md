---
layout: single
title: "Azure Arc enabled Servers: Configuração de proxy a nível de serviço em Linux"
namespace: "azure-arc-enabled-servers-service-level-proxy-configuration-on-linux"
category: "Azure Arc"
tags: azure arc linux
date: 2021-03-02 18:00:00
---

Algumas vezes, durante engajamentos com clientes, recebo questionamentos de como configurar o proxy para acesso à internet apenas em serviços específicos e com o Azure Arc para Servidores Linux isso não é diferente. 

Neste post irei guiá-los quanto ao procedimento de configuração de proxy apenas para o Arc, sem definir o proxy de maneira global no Linux.

## Configuração default

Por padrão, quando especificamos um servidor proxy na instalação do agente do Azure Arc for Linux Server, o utilitário `azcmagent_proxy` é invocado para automatizar estas configurações, que possui as seguintes opções:

```bash
dcruz@vmlx02:~$ sudo azcmagent_proxy
Usage:  azcmagent_proxy add <URL> - to add URL as the proxy
        azcmagent_proxy remove - to delete configured proxy
dcruz@vmlx02:~$
```

Quando configuramos o proxy a partir deste método, dois arquivos são alterados, conforme podemos ver abaixo:

```bash
dcruz@vmlx02:~$ sudo azcmagent_proxy add http://vmlx01:3128
    No proxy previously configured
Removing proxy environment variable from file:  /opt/azcmagent/bin/azcmagent
    No proxy previously configured
Setting proxy environment variable to file:  /lib/systemd/system.conf.d/proxy.conf
Adding proxy environment variable to file:  /opt/azcmagent/bin/azcmagent
dcruz@vmlx02:~$
```

O primeiro deles, o ` /lib/systemd/system.conf.d/proxy.conf`, define o proxy a todos os serviços systemd de forma global, enquanto o arquivo `/opt/azcmagent/bin/azcmagent`, que é o wrapper do utilitário de linha de comando do Arc, também recebe esta configuração. 

## Alterando o proxy apenas para o Azure Arc

A fim de configurar os serviços para que tenham a conectividade necessária, sem ativar o proxy de forma global ao systemd é necessário editar cada um dos arquivos de unidade(*unity files*), conforme descrito abaixo:

- Se o proxy foi configurado utilizando o `azcmagent_proxy` e deseja remover as configurações feitas por ele, é necessário executar a linha de comando abaixo com o parâmetro **remove**

  ```bash
  $ sudo azcmagent_proxy remove
  ```

- Adicionalmente, para cada um dos arquivos dos 3 serviços utilizados pelo Azure Arc for Linux Servers (`himdsd.service` ,`gcad.service` ,`extd.service`), que estão localizados no diretório `/lib/systemd/system`, precisamos incluir um parâmetro na seção `[Service]` com a variável de ambiente **https_proxy**, conforme exemplo abaixo e que pode ser visto também na documentação do [**systemd**](https://www.freedesktop.org/software/systemd/man/systemd.service.html):

  ```ini
  [Service]
  # [...]
  Environment=https_proxy=http://vmlx01:3128
  ```

- Após alterar os arquivos mencionados, basta executar os comandos a seguir para realizar o reinício do systemd e reiniciar as dependências do Arc:

  ```bash
  $ sudo systemctl daemon-reexec
  $ sudo systemct	restart extd.service himdsd.service gcad.service
  ```

Após estas alterações, voce deverá ver a seguinte entrada nos arquivos de log, que comprovam que o serviço está funcionando corretamente e utilizando o proxy recém definido. O Log pode ser validado em `/var/opt/azcmagent/log/himds.log`

```
time="yyyy-MM-dd02T17:34:07Z" level=debug msg="Using Https Proxy: http://vmlx01:3128"
```

### Script

Para simplificar a configuração, criei um script inspirado no `azcmagent_proxy` para automatizar esta configuração para distribuições Linux que utilizem o **systemd**, disponível em  [Security/azcmagent_proxydaemon.sh at main · davi-cruz/Security (github.com)](https://github.com/davi-cruz/Security/blob/main/AzureArc/azcmagent_proxydaemon.sh).

Este script pode ser útil não apenas para definir as configurações de proxy para os servidores Linux habilitados com Azure Arc mas também para outros service unities em seus workloads. :smile:

## Nota

Como estas alterações não são globais, outros serviços instalados a partir de Azure Extensions (Nativas ou Customizadas) podem requerer alterações adicionais para que funcionem com o proxy padrão ou a mesma configuração deverá ser reproduzida para estes serviços

Espero que ajude!
