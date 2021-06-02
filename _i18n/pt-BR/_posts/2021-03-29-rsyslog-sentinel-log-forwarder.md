---
layout: single
title: "Azure Sentinel: Configuração do Log Forwarder"
namespace: rsyslog-sentinel-log-forwarder
category: "Azure Sentinel"
tags: Sentinel Syslog Rsyslog Linux Forwarder Collector CEF
date: 2021-03-29 12:00:00 -03
last_modified_at: 2021-04-30 16:00:00 -03
toc: true
---

Frequentemente apoio clientes no deployment de *forwarders* (encaminhadores) de logs CEF/Syslog em seus ambientes para coletar informações de appliances de rede e/ou servidores e serviços para o Log Analytics, que consequentemente os disponibiliza para o Azure Sentinel.

Embora tenhamos uma diversidade de documentações em como fazer este deployment, assim como recursos de comunidade como o Tech Community e Webinars, compilei todos os pontos que normalmente reviso com meus clientes nestes engajamentos de deployment e revisão neste post para ajudar àqueles que estejam enfrentando algum desafio em seu ambiente.

{% capture changes %}
:newspaper: **Edits**:

- **30/04/2021** - Incluído seção [Redução de mensagens repetidas](#redução-de-mensagens-repetidas)
{% endcapture %}

<div>
{{ changes | markdownify }}
</div>{: .notice--success}

## Requisitos

De acordo com a [documentação oficial da Microsoft](https://docs.microsoft.com/en-us/azure/sentinel/connect-common-event-format), os requisitos para este recurso são bastante simples, conforme listado abaixo:

- 4 CPUs e 8GB RAM, que é capaz de processar cerca de 8500 EPS (eventos por segundo) utilizando o rsyslog, de acordo com a documentação.
  - Se estiver realizado este deployment no Azure você pode utilizar o SKU **Standard_F4s_v2** que é otimizado para processamento e recomendado neste caso.
- Uma distribuição Linux em que o Microsoft Monitoring Agent (também conhecido como omsagent) seja suportado.
- Python versão 2.7 ou 3.
- Um dos serviços de syslog suportados instalado. Na maioria dos cenários que trabalhei utilizei o **rsyslog**, uma vez que normalmente é o pacote padrão na maioria das distribuições. Abaixo a lista das versões suportadas, também de acordo com a documentação:
  - Rsyslog v8
  - Syslog-ng 2.1 - 3.22.1

## Arquitetura do Forwarder

![Forwarder Architecture](https://i.imgur.com/t2cvlm7.png)

A arquitetura da solução é bem simples:

- É composta por uma ou mais máquinas recebendo os logs em protocolo syslog via UDP, TCP ou TLS. Isso é alcançado a partir da configuração de listeners no rsyslog ou no syslog-ng, assim como quaisquer servidores syslog você tenha ou teve em execução em seu ambiente.
- As mensagens recebidas são encaminhadas para os listeners locais que são configurados no omsagent para ingestão como logs puros em Syslog (UDP/25224) ou CEF (TCP/25226).
- Mensagens são encaminhadas ao Log Analytics workspace de acordo com a sua configuração, acessíveis também ao Sentinel a partir da ativação da solução.

Agora vou orientá-los quando aos detalhes da configuração desta infraestrutura e compartilhar algumas dicas de configurações mais detalhadas para mitigar alguns problemas e reduzir ingestão desnecessária no seu ambiente.

### Considerações de Design

Antes de realizar o deploy/revisar a infraestrutura do seu forwarder, existem algumas perguntas que precisamos responder que auxiliarão na correta definição das suas configurações:

- **Este forwarder receberá apenas mensagens CEF, Syslog ou ambos?**
  - Embora ambos os tipos de mensagens sejam recebidos via protocolo syslog, as mensagens CEF são formatadas de acordo com seu padrão pelo omsagent, disponibilizando cada campo em uma coluna própria no Log Analytics, enquanto as mensagens Syslog puro são armazenadas em uma única coluna, o que irá requerer esforço adicional quando necessário realizar a análise destes dados em alguma regra.
  - Também, se você estiver utilizando o forwarder para ambos os tipos de logs, precisará impedir que mensagens enviadas do tipo CEF sejam armazenadas também como Syslog, o que irá incorrer em **custo duplicado** para sua workspace.
- **Quais são os facilities e severidades, assim como tipos de mensagens e fontes que serão encaminhados ao Sentinel?**
  - Envio de dados em excesso ao Sentinel pode gerar alto custo se não filtrados adequadamente.
  - Considere mapear os cenários de detecção de acordo com o [MITRE ATT&CK Matrix Techniques](https://attack.mitre.org/matrices/enterprise/) que tem interesse investigar antes de enviar os dados. Desta forma você conseguirá evitar o envio de dados sem um propósito ao seu SIEM, que não lhe trarão nenhum retorno sobre o investimento feito tanto em armazenamento quanto no Sentinel.
- **Quais são os requisitos de segurança para esta comunicação?**
  - Como descrito no post sobre [boas práticas](https://techcommunity.microsoft.com/t5/azure-sentinel/best-practices-for-common-event-format-cef-collection-in-azure/ba-p/969990) compartilhado por Cristhofer Romeo Muñoz, **devemos utilizar TCP como o protocolo padrão** para qualquer comunicação devido a sua confiabilidade, a menos que seu appliance apenas suporte UDP ou se necessite de criptografia na comunicação, adotando TLS quando o dado for transmitido pela internet ou se houver dados sensíveis a serem coletados.
  - Instalando o forwarder próximo da fonte também ajuda a reduzir a complexidade dos protocolos utilizados, assim como evita que tenhamos problemas de performance causados por alta latencia na comunicação entre origem e forwarder.

## Instalação

A instalação inicial pode ser facilmente realizada executando o script `cef_installer.py`, disponível no [repositório oficial do Github](https://github.com/Azure/Azure-Sentinel/blob/master/DataConnectors/CEF/cef_installer.py) utilizando a linha de comando abaixo, que pode ser obtida para seu ambiente acessando a página do conector **Common Event Format (CEF)** no Sentinel ou simplesmente substituir os dados `<workspace id>` e `<workspace secret>` pelos de sua workspace.

:bulb: **Nota**: Se não tiver o python 2.7 na máquina deverá substituir a instrução `python` por `python3` antes de executar a linha abaixo.
{: .notice--info}

```bash
sudo wget -O cef_installer.py https://raw.githubusercontent.com/Azure/Azure-Sentinel/master/DataConnectors/CEF/cef_installer.py && sudo python cef_installer.py <workspace id> <workspace secret>
```

Se seu forwarder necessita de um proxy para comunicar-se com a Internet, você precisará também atualizar tais configurações modificando o arquivo `/etc/opt/microsoft/omsagent/proxy.conf`:

```bash
proxyconf="http://user:password@proxyserver:3128" #user:password can be ignored if you proxy doesn't require authentication
sudo echo $proxyconf >>/etc/opt/microsoft/omsagent/proxy.conf
sudo chown omsagent:omiusers /etc/opt/microsoft/omsagent/proxy.conf

#restart service
sudo /opt/microsoft/omsagent/bin/service_control restart
```

Este script de instalação realiza as alterações listadas abaixo em seu sistema. A maioria delas necessita de conectividade com o Github uma vez que os assets utilizados estão armazenados no [repositório oficial do Azure Sentinel](https://github.com/Azure/Azure-Sentinel):

- Download e instalação do Microsoft Monitoring Agent para Linux.
- Configuração do omsagent para ouvir na porta TCP/25226 e realizar o parsing das mensagens CEF a partir da cópia do arquivo `security_events.conf` dentro do diretório de configuração da workspace (`/etc/opt/microsoft/omsagent/<workspaceID>/conf/omsagent.d`).
- Encaminha as mensagens que contenham ASA ou CEF para a porta TCP/25226 a partir da cópia do arquivo `security-config-omsagent.conf` para `/etc/rsyslog.d/` ou `/etc/syslog-ng/conf.d/`, dependendo do seu serviço de syslog.
- Habilita o serviço syslog para ouvir porta TCP/514 e UDP/514 a partir de edição do arquivo em `/etc/rsyslog.conf` ou `/etc/syslog-ng/syslog-ng.conf`.

Com esta configuração você já estará preparado para encaminhar as mensagens CEF para este servidor, que em breve estarão disponíveis na sua workspace de Log Analytics na tabela `CommonSecurityLog`, mas existem algumas outras alterações aqui descritas das quais poderá se beneficiar para coletar logs Syslog puro, assim como implementar filtros e ajustes na infraestrutura.

## Configuração

Como vimos a infraestrutura é baseada no serviço de syslog + omsagent. Uma vez que o **rsyslog** é um dos mais populares serviços utilizados e já disponível por padrão da maioria das distribuições atualmente, utilizarei todas as configurações aqui descritas para este serviço, embora você possa também ajustá-las para funcionamento com o syslog-ng caso necessário.

### Validação da versão e instalação do rsyslog

Como mencionado em documentação, precisamos utilizar o rsyslog versão 8. Para confirmar a versão em execução em seu systema você pode executar o comando `rsyslogd -v`, que exibirá um output como o exemplo abaixo:

```bash
$ rsyslogd -v
rsyslogd 8.32.0, compiled with:
        PLATFORM:                               x86_64-pc-linux-gnu
        PLATFORM (lsb_release -d):
        FEATURE_REGEXP:                         Yes
        GSSAPI Kerberos 5 support:              Yes
        FEATURE_DEBUG (debug build, slow code): No
        32bit Atomic operations supported:      Yes
        64bit Atomic operations supported:      Yes
        memory allocator:                       system default
        Runtime Instrumentation (slow code):    No
        uuid support:                           Yes
        systemd support:                        Yes
        Number of Bits in RainerScript integers: 64
```

De acordo com a [documentação do projeto](https://www.rsyslog.com/doc/v8-stable/installation/index.html), você não deve ter maiores problemas executando a versão do rsyslog disponível no repositório do mantenedor da distribuição a menos que eles não tenham portado alguma correção para ela. Neste caso você deverá baixar a última build para sua distribuição da página do projeto ou instalar a partir do código fonte.

### Aplicando as configurações do rsyslog

Após quaisquer alterações feitas no arquivo de configuração do rsyslog (`/etc/rsyslog.conf`) ou qualquer outro arquivo por ele importado (normalmente em `/etc/rsyslog.d/*.conf`) você precisará reiniciar o serviço para que as alterações tenham efeito. Isso pode ser alcançado a partir da execução da linha de comando abaixo:

```bash
sudo systemctl restart rsyslog.service
```

### Ajustando os listeners do rsyslog

Embora o `cef_installer.py` já defina os listeners para TCP/514 e UDP/514, talvez haja a necessidade de validar ou ajustar a sua configuração para criptografar a comunicação via TLS.

Sua configuração atual deve se parecer com o trecho abaixo, que pode ser visto no arquivo `/etc/rsyslog.conf`:

```conf
module(load="imuxsock") # provides support for local system logging
#module(load="immark")  # provides --MARK-- message capability

# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514") 

# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")
```

A configuração para comunicação via TLS requer alguns itens adicionais, onde é necessário emitir os certificados para servidores e clientes (caso não esteja utilizando uma infra de PKI) assim como alterar o arquivo de configuração do rsyslog para aceitar estas comunicações.

Maiores detalhes e exemplos de configurações podem ser encontrados na documentação oficial [neste link](https://www.rsyslog.com/doc/v8-stable/tutorials/tls_cert_summary.html), de onde retirei o trecho abaixo para que tenham uma ideia de como esta configuração se parece:

```conf
module(load="imuxsock") # local messages
module(load="imtcp" # TCP listener
    StreamDriver.Name="gtls"
    StreamDriver.Mode="1" # run driver in TLS-only mode
    StreamDriver.Authmode="anon"
    )

# make gtls driver the default and set certificate files
global(
    DefaultNetstreamDriver="gtls"
    DefaultNetstreamDriverCAFile="/path/to/contrib/gnutls/ca.pem"
    DefaultNetstreamDriverCertFile="/path/to/contrib/gnutls/cert.pem"
    DefaultNetstreamDriverKeyFile="/path/to/contrib/gnutls/key.pem"
    )

    # start up listener at port 6514
    input(
    type="imtcp"
    port="6514"
    )
```

### Evitando registro de eventos remotos em arquivos locais

Um ponto que é sempre questionado pelos clientes é que após configurar máquinas como forwarder para CEF e/ou Syslog os arquivos de log locais (frequentemente `/var/log/messages` ou `/var/log/syslog`) são sobrecarregados com mensagens oriundas de servidores remotos, e na maioria das vezes consumindo todo o espaço em disco disponível do volume onde `/var/log` reside.

Isso ocorre porque as definições padrão do rsyslog, listadas abaixo, enviam alguns facilities e severidades para arquivos locais sem validar a fonte destes eventos (comentários e linhas em branco removidas por simplicidade).

:bulb: **Nota**: O caminho onde encontrar estas configurações padrão variam de acordo com a distribuição Linux. Para Ubuntu existe um arquivo chamado `/etc/rsyslog.d/50-default.conf` enquanto para distribuições baseadas em RHEL (RHEL, CentOS, Fedora) está na configuração principal do `/etc/rsyslog.conf`.
{: .notice--info}

```conf
#Extracted from Ubuntu /etc/rsyslog.d/50-default.conf

auth,authpriv.*                 /var/log/auth.log
*.*;auth,authpriv.none          -/var/log/syslog
kern.*                          -/var/log/kern.log
mail.*                          -/var/log/mail.log
mail.err                        /var/log/mail.err
*.emerg                         :omusrmsg:*
```

Como podem ver, qualquer mensagem que coincida com alguma severidade contida nesta lista é enviada para um dos arquivos locais. Uma forma rápida de prevenir este comportamento é editar o arquivo de configuração para apenas aceitar mensagens geradas pelo host local (`127.0.0.1`):

```conf
if ($fromhost-ip == '127.0.0.1') then {
    auth,authpriv.*                 /var/log/auth.log
    *.*;auth,authpriv.none          -/var/log/syslog
    kern.*                          -/var/log/kern.log
    mail.*                          -/var/log/mail.log
    mail.err                        /var/log/mail.err
    *.emerg                         :omusrmsg:*
}
```

### Redução de mensagens repetidas

Conforme lembrado pelo meu colega [Flavio Honda](https://www.linkedin.com/in/flavio-honda/), o Rsyslog possui uma opção que pode auxiliar com a redução de mensagens repetidas.

Se você sofre com este tipo de problema ou quer prevenir antes mesmo que venha acontecer em seu ambiente, você pode habilitar a configuração abaixo no seu arquivo `/etc/rsyslog.conf` :smile:.

```ini
$RepeatedMsgReduction on
```

A referência oficial sobre esta feature na documentação do RSyslog pode ser encontrada [neste link](https://www.rsyslog.com/doc/master/configuration/action/rsconf1_repeatedmsgreduction.html).

### Fluxo de processamento de mensagens

No rsyslog, o primeiro arquivo a ser processado é o `rsyslog.conf`, que normalmente contém uma instrução para importar todos os arquivos `*.conf` localizados no diretório `/etc/rsyslog.d` e os arquivos são processados em ordem **alfabética**. Quando uma mensagem chega ou é gerada no sistema, é encaminhada e avaliada por cada uma das configurações nesta mesma ordem, a menos que alteremos explicitamente este comportamento.

Na configuração abaixo, vamos considerar que estamos encaminhando mensagens no formato CEF de um appliance que são enviadas com o facility `local4.info`:

<img src="https://i.imgur.com/IFykwc2.png" alt="processing - initial configuration" style="zoom:80%;" />{: .align-center}

- Se nenhuma alteração for realizada no `rsyslog.conf` ou `50-default.conf` para prevenir o registro de mensagens de um host remoto, estas mensagens serão registradas no arquivo `/var/log/syslog`.
- Também, se houver algum match no arquivo `95-omsagent.conf` esta mensagem será enviada para a workspace de Log Analytics como Syslog puro.
- E por fim, ele coincidirá com as regular expressions contidas no arquivo `security-config-omsagent.conf` e tratado como mensagem CEF para a tabela `CommonSecurityLog`.

Isso não apenas resultará no disco cheio, mas também em cobrança duplicada para este mesmo evento, que será armazenado duas vezes no Log Analytics.

Para evitar este comportamento, como Ofer Shezaf compartilhou em um dos [Webinars da Comunidade de Segurança](https://aka.ms/securitywebinars) (*Log Forwarder deep dive \| Filtering CEF and Syslog events*), vamos renomear o arquivo `security-config-omsagent.conf` para `60-cef.conf` de forma que ele seja processado **antes** do `95-omsagent.conf`.

Adicionalmente, precisamos garantir que uma vez que as mensagens sejam encaminhadas para o TCP/25226 e tratadas como CEF sejam descartadas, o que é alcançado pelo uso da instrução `stop`. O **Stop** remove as mensagens do processamento do rsyslog.

:bulb: **Nota**: Você poderá encontrar outras instruções que utilizem o stop como um til (`~`). Esta notação veio da syntax legado do syslogd
{: .notice--info}

Esta configuração ficaria da seguinte forma:

```bash
$ cat /etc/rsyslog.d/60-cef.conf
if ($rawmsg contains "CEF:") or ($rawmsg contains "ASA-") then {
        *.*     @@127.0.0.1:25226
        stop
}
```

<img src="https://i.imgur.com/cPteGGK.png" alt="processing - reordering" style="zoom:80%;" />{: .align-center}

### Ajustando o `95-omsagent.conf`

Após a instalação inicial, o agente do MMA criará um arquivo de configuração no diretório do serviço syslog (`/etc/rsyslog.d/95-omsagent.conf`) em sincronismo com as configurações definidas no portal do Azure, navegando em sua workspace de Log Analytics > Agents Configuration > Syslog:

![Log Analytics Agent Configurations - Syslog](https://i.imgur.com/UesHmCS.png)

Para a configuração acima o seguinte conteúdo será criado no arquivo, conforme abaixo:

```conf
# OMS Syslog collection for workspace <workspace id>
auth.=alert;auth.=crit;auth.=debug;auth.=emerg;auth.=err;auth.=info;auth.=notice;auth.=warning  @127.0.0.1:25224
authpriv.=alert;authpriv.=crit;authpriv.=debug;authpriv.=emerg;authpriv.=err;authpriv.=info;authpriv.=notice;authpriv.=warning  @127.0.0.1:25224
local3.=alert;local3.=crit;local3.=debug;local3.=emerg;local3.=err;local3.=info;local3.=notice;local3.=warning  @127.0.0.1:25224
local4.=alert;local4.=crit;local4.=debug;local4.=emerg;local4.=err;local4.=info;local4.=notice;local4.=warning  @127.0.0.1:25224
```

Mesmo que esta configuração seja extremamente simples de se fazer pelo portal, ela se aplica globalmente a todos os dispositivos Linux que possuem o omsagent conectado a esta workspace, ingerindo mais informação de que precisa dentro de sua workspace de Log Analytics se mais tipos de dados são selecionados que o necessário.

Se precisa definir uma configuração diferente para algumas máquinas como os seus forwarders, é recomendado que se edite diretamente o arquivo `95-omsagent.conf` adicionando ou removendo facilities e severidades dele, assim como construindo suas condições para que estes dados sejam enviados ao Azure mas precisamos impedir que este arquivo seja sincronizado com as definições no portal.

O comando abaixo "quebra" o vínculo existente entre este arquivo e a configuração da workspace:

```bash
sudo su omsagent -c 'python /opt/microsoft/omsconfig/Scripts/OMS_MetaConfigHelper.py --disable'
```

A única coisa que não pode ser alterada no arquivo `95-omsagent.conf` é a instrução de encaminhamento para `@127.0.0.1:25224`.

:bulb: **Nota**: Na sintaxe do syslogd uma arroba (`@`) significa UDP enquanto duas arrobas (`@@`) significam TCP.
{: .notice--info}

:warning: **Atenção**: No webinar citado anteriormente sobre um Dep Dive a respeito de CEF forwarder, Ofer também recomendou alterar as configurações do omsagent para aceitar as comunicações também via TCP para Syslog. Se seguir esta recomendação você deverá também ajustar qualquer instrução de encaminhamento para dois arrobas no arquivo `95-omsagent.conf`, caso contrário suas mensagens poderão não ser enviadas ao Log Analytics.
{: .notice--warning}

### Filtrando eventos

Se você ainda estiver notando a ingestão de eventos desnecessários em sua workspace, você pode implementar um filtro na fonte assim como no encaminhador, o que normalmente é mais simples já que muitos appliances não suportam grande granularidade em suas configurações.

Uma forma fácil de configurar esta filtragem é criando um arquivo a ser processando antes do `60-cef.conf` e `95-omsagent.conf`, neste exemplo chamado de `59-filter-out.conf`, mas poderia utilizar qualquer nome que desejar, desde seja processado na ordem correta.

<img src="https://i.imgur.com/tNsVYea.png" alt="filtering out messages" style="zoom:80%;" />{: .align-center}

Neste arquivo vamos utilizar novamente a instrução `stop`, removendo as mensagens que não gostaríamos que fossem encaminhadas seja como Syslog ou CEF. Neste exemplo qualquer mensagem que contenha a palavra "test" e tenha algum dos facilities/severidades mencionados serão removidas.

```conf
if ($rawmsg contains "test") and prifilt("auth,authpriv.*") then {
        stop
}
```

A documentação do rsyslog é muito rica em recursos mostrando [como você pode filtrar](https://www.rsyslog.com/doc/master/configuration/filters.html) estas mensagens, assim como [quais propriedades](https://www.rsyslog.com/doc/master/configuration/properties.html) se encontram disponíveis para que possa configurar estas instruções de forma mais avançada, incluindo dicas de [conversão das instruções](https://www.rsyslog.com/doc/v8-stable/configuration/converting_to_new_format.html) que utilizam sintaxe syslogd para o advanced, também conhecido como **RainerScript**.

Espero que este conteúdo tenha sido útil para aqueles que estão realizando o seu deployment ou manutenção de CEF/Syslog forwarders no seu ambiente!

Você tem alguma configuração específica ou desafio não discutido aqui? Sinta-se à vontade para compartilhar nos comentários para que possamos complementar o post com mais detalhes, podendo assim auxiliar outros que estejam enfrentando o mesmo! :smile:
