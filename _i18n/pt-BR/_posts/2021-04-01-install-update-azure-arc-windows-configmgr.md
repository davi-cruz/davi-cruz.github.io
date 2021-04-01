---
layout: single
title: "Azure Arc enabled Servers: Instalação e Atualização via Configuration Manager"
namespace: "install-update-azure-arc-windows-configmgr"
category: "Azure Arc"
tags: azure arc windows ConfigMgr MECM MEM
date: 2021-04-01 12:00:00
---

Neste post compartilho o procedimento de como realizar o deploy e atualização do Azure Arc for Servers (Windows) via Microsoft Endpoint Manager Configuration Manager - MECM (aka SCCM).

O processo é bastante simples, porém algumas pequenas dicas são compartilhadas para que possa obter o máximo do ConfigMgr para esta atividade :smile:.

:information_source: **Nota**: Os scripts compartilhados neste post também poderão ser utilizado a partir de outras ferramentas de gestão de configuração, porém ajustes podem ser necessários para que funcionem adequadamente de acordo com o método utilizado.
{: .notice--info}

## Azure Arc enabled Servers

O Azure Arc permite o gerenciamento de servidores físicos e virtuais fora do Azure a partir da console do Azure, permitindo que gerencie extensões, atualizações e guest policies de forma consistente do mesmo modo em que máquinas virtuais nativas do Azure são gerenciadas.

O Azure Arc é um asset bastante importante para utilização de features como o Vulnerability Assessment do Azure Defender, assim como a manutenção de algumas extensões como Microsoft Monitoring Agent (MMA) e o novo Azure Monitor Agent (AMA).

### Preço

De acordo com a página [Azure Arc – Azure Management | Microsoft Azure](https://azure.microsoft.com/en-us/pricing/details/azure-arc/), features relacionadas ao control plane do Azure (Instalação de Extensões, Arm Templates, etc.) **são gratuitos**, assim como o uso do Azure Update Management.

Features relacionadas à Azure Policy Guest Configurations (Automações, inventários, state configuration) e outros serviços conectados via Arc (Azure Defender e Azure Monitor, por exemplo) são cobrados de acordo com suas respectivas tabelas de custos, as quais poderão ser verificadas diretamente na [calculadora do Azure](https://azure.microsoft.com/en-us/pricing/calculator/).

### Pré-requisitos

Alguns requisitos devem ser observados para viabilizar esta instalação, cujos detalhes podem ser observados no link [Overview of the Connected Machine agent - Azure Arc | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-arc/servers/agent-overview#prerequisites):

- Um Sistema Operacional Suportado.
- Liberação das URLs necessárias para o Serviço do Azure Arc, vide link acima citado.
  - Dependendo de como seu ambiente está configurado, o uso de um servidor proxy pode ser necessário.
  - Algumas extensões a serem instaladas via Arc podem necessitar de liberações adicionais para garantir seu funcionamento.
- Provisionamento de um Service Principal para que o onboarding do Arc seja feito de modo silencioso. O procedimento de criação deste objeto está disponível detalhado em [Connect hybrid machines to Azure at scale - Azure Arc | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-arc/servers/onboard-service-principal).
- Ativação dos resource providers **Microsoft.HybridCompute** e **Microsoft.GuestConfiguration**, conforme discutido em [Connect hybrid machine with Azure Arc enabled servers - Azure Arc | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-arc/servers/learn/quick-enable-hybrid-vm#register-azure-resource-providers).

## Gestão via Configuration Configuration Manager

Caso sua organização já faça uso do Configuration Manager para gestão de dispositivos, o mesmo também poderá ser utilizado para instalação e atualização do Azure Machine Connected Agent, utilizado pelo Azure Arc.

### Instalação do Agente

Utilizando o recurso de [Applications do Configuration Manager](https://docs.microsoft.com/en-us/mem/configmgr/apps/deploy-use/create-applications), temos a possibilidade de garantir que estes agentes estão instalados adequadamente em um dispositivo, o que permite uma melhor gestão do status de instalação do pacote após sua execução.

Abaixo descrevo o procedimento utilizado para criar este recurso:

- Download da última release do instalador do Arc para Windows: O binário de instalação pode ser baixado diretamente da URL encurtada `https://aka.ms/AzureConnectedMachineAgent`. A linha de PowerShell abaixo salva o arquivo em um diretório especificado.

  - O arquivo deverá ser copiado para um diretório a ser utilizado pelo MECM como content source

  ```powershell
  $destinationPath = "c:\path\to\file"
  Invoke-WebRequest -Uri https://aka.ms/AzureConnectedMachineAgent -OutFile "$destinationPath\AzureConnectedMachineAgent.msi"
  ```

- Baixar e editar o script [Install-AzureArcMECM.ps1](https://github.com/davi-cruz/Security/blob/main/AzureArc/Windows/Install-AzureArcMECM.ps1) e editar as variáveis conforme exemplo abaixo. Este script deverá ser colocado no mesmo caminho onde o arquivo *.msi baixado anteriormente está localizado.

  - Caso seu servidor necessite de um proxy para comunicação. especifique a variável `$proxyUrl`. Caso contrário, mantenha o valor **em branco**.
  - O script compartilhado, além de instalar o agente, realiza a configuração de proxy (se informado) e conecta o dispositivo ao serviço do Arc, conforme credenciais compartilhadas:

  ```powershell
  ## Variables
  $installLogFile = "c:\Windows\Temp\AzureArcSetup.log"
  
  $tenantID = "7482a3c1-7102-4a0d-9dfb-0aa2186ce450"
  $subscriptionID = "48a0ab65-bd82-4aac-9283-99344f387d9b"
  $ResourceGroupName = "rg-azurearc-windows"
  $serviceprincipalAppID = "909e06f7-577c-45b9-b49e-eff9e6560ef6"
  $serviceprincipalSecret = "f53f0a59-6db3-4ac7-b861-e760d29240d8"
  $resourceLocation = "eastus"
  $proxyUrl = "" # Format: http[s]://server.fqdn:port
  
  ### Sample data provided for illustration purposes
  ```

- Seguir com a criação do application no console do MECM acessando *Software Library* > *Application Management* > *Applications* e selecionar a opção *Create Application* no ribbon ou no menu de contexto de *Application*.

![Criando um Application no MECM](https://i.imgur.com/UTfhvOv.png)

- Selecione o arquivo caminho do `*.msi` no File Share onde foi armazenado, de acordo com o pré-estabelecido em seu ambiente.

![Create Application Wizard - General](https://i.imgur.com/dE4HPPW.png)

- Clique em *Next* para avançar após validar os dados do aplicativo a ser instalado.

![Create Application Wizard - Important Information](https://i.imgur.com/RIHhWc0.png)

- Em *General Information*, substitua a linha de comando pré-configurada com a linha de comando abaixo para que a instalação ocorra a partir do script PowerShell baixado e clique em *Next*.

  ```properties
  %windir%\System32\WindowsPowerShell\v1.0\powershell.exe -File .\Install-AzureArcMECM.ps1
  ```

  :bulb: **Dica**: Complemente nesta janela também as informações solicitadas para enriquecer os dados de catálogo para este aplicativo e simplificar sua gestão posterior.
  {: .notice--primary}

![Create Application Wizard - General Information](https://i.imgur.com/V6qUH4B.png)

- Revise as informações configuradas e clique em *Next* para criar o application.

![Create Application Wizard - Summary](https://i.imgur.com/tYcMhvu.png)

- Após concluída a criação, a seguinte mensagem de sucesso será exibida. Clique em *Close* para fechá-la.

![Create Application Wizard - Completion](https://i.imgur.com/93fBtqs.png)

Após criar o application, precisaremos ajudar alguns outros detalhes para que funcione de forma mais adequada, conforme abaixo:

- Acesse as propriedades do deployment type selecionando o aplicativo > clique na aba *Deployment Types* > clique com o botão direito no deployment existente e selecione a opção *Properties*.

![ConfigMgr Console - Deployment Type Properties](https://i.imgur.com/an2pGmr.png)

- Na aba *Detection Method*, selecione a opção *Use a custom script to detect the presence of this deployment type* e clique no botão *Edit*.

![Deployment Type Properties - Detection Method](https://i.imgur.com/QkRIfhc.png)

- Selecione a opção *Powershell* e cole o snippet abaixo, clicando em *OK* para concluir.

  - Este script de detecção verificará não apenas se o agente está devidamente instalado, mas se também estará conectado ao Azure Arc ao término do processo de instalação.

![Detection Method - Script Editor](https://i.imgur.com/BWSfWnQ.png)

  :warning: **Alerta**: É importante que o script apenas retorne algum output se o aplicativo estiver instalado. Qualquer output retornado o ConfigMgr interpreta como instalado.
  {: .notice--warning}

```powershell
  # Detection Method: Returns the installed version if properly installed
  
  try{
      $agentDetails = azcmagent show
      $agentStatus = ($agentDetails | Where-Object {$_ -like '*Agent Status*'}).Split(": ")[-1]
      
      if($agentStatus){
          Write-Output ($agentDetails | Where-Object {$_ -like '*Agent Version*'}).Split(": ")[-1]
      }
  }
  catch{
      # Returns nothing if not installed
  }
```

- Na aba *Requirements* é importante também configurarmos as versões de Sistema Operacional que desejamos que este pacote seja instalado. Isso também evita que deployments acidentais venham instalar o produto em desktops Windows, o que não é suportado.

![Deployment Type Properties - Requirements - Create Requirement](https://i.imgur.com/TMATvrP.png)

- Após ajustes, clicar em *OK* para finalizar a configuração.

Neste momento o application criado estará pronto para ser instalado nos servidores, onde poderá ser feito o deployment para uma collection, vide [referência oficial](https://docs.microsoft.com/en-us/mem/configmgr/apps/deploy-use/deploy-applications).

### Atualização do agente

De acordo com a [documentação](https://docs.microsoft.com/en-us/azure/azure-arc/servers/manage-agent#upgrading-agent), o Azure Arc pode ser atualizado de forma Manual ou via WSUS para Sistemas Operacionais Windows.

Neste caso temos duas opções a serem seguidas:

- **Criar outro application no Configuration Manager para a nova versão**:

  - Este mecanismo requer maior esforço administrativo por parte do ConfigMgr porém pode ser alcançado criando outro Application para a nova versão, conforme orientado neste post.
  - É importante que seja definido para este novo aplicativo uma **relação de substituição**, conforme pode ser visto em mais detalhes [neste link](https://docs.microsoft.com/en-us/mem/configmgr/apps/deploy-use/revise-and-supersede-applications#supersedence). Isso permite que novas instalações que referenciam o item anterior façam uso da versão mais recente, além de permitir a definição do comportamento a ser seguido para as máquinas que já possuem o produto instalado, que pode ser atualizar ou desinstalar.

- **Incluir os produtos e serviços no Catálogo de Software Update Point e processo de gestão de atualizações de sua organização (recomendado)**: Esta abordagem permite o uso do processo já implementado de gestão de atualizações via ConfigMgr para também atualizar o Azure Arc.

  - Os seguintes produtos e classificações devem estar selecionados para sincronismo do Software Update Point na infraestrutura. Os detalhes de como realizar esta ação estão disponíveis neste link [neste link](https://docs.microsoft.com/en-us/mem/configmgr/sum/get-started/configure-classifications-and-products):

    | Configuração  | Valor a ser selecionado                   |
    | ------------- | ----------------------------------------- |
    | Produtos      | Microsoft > Azure Connected Machine Agent |
    | Classificação | Critical Updates                          |

## Conclusão

Como pudemos ver o processo de instalação via Configuration Manager é bem simples, tendo apenas alguns ajustes para que possamos não apenas instalar o agente, mas também realizar o seu onboarding.

Na documentação oficial temos outros métodos de deployment disponíveis, que podem ser conferidos diretamente em seus respectivos artigos oficiais:

- [Install Connected Machine agent using Windows PowerShell DSC - Azure Arc | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-arc/servers/onboard-dsc)
- [Connect hybrid machines to Azure from Windows Admin Center - Azure Arc | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-arc/servers/onboard-windows-admin-center)
- [Connect hybrid machines to Azure by using PowerShell - Azure Arc | Microsoft Docs](https://docs.microsoft.com/en-us/azure/azure-arc/servers/onboard-powershell)

Espero que as orientações compartilhadas lhe sejam úteis para a criação deste deployment via ConfigMgr, permitindo a gestão destes dispositivos a partir da console do Azure.

Até o próximo post! :smile:
