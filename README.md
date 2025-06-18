# Integração Multi-Cloud: Azure e AWS (VPN Site-to-Site)

Este documento descreve a criação de um ambiente multi-cloud entre Microsoft Azure e Amazon Web Services (AWS), com comunicação privada via VPN Site-to-Site. 
O objetivo é permitir que duas máquinas virtuais (uma em cada nuvem) se comuniquem entre si apenas por IPs privados.

## Sumário

- [1. Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
- [2. Arquitetura da Solução](#2-arquitetura-da-solução)
- [3. Provisionamento das Máquinas Virtuais](#3-provisionamento-das-máquinas-virtuais)
- [4. Configuração da VPN Site-to-Site](#4-configuração-da-vpn-site-to-site)
- [5. Configuração de Rede e Segurança](#5-configuração-de-rede-e-segurança)
- [6. Testes de Conectividade](#6-testes-de-conectividade)
- [7. Tabelas de Configuração](#7-tabelas-de-configuração)
- [8. Resolução de Problemas Comuns](#8-resolução-de-problemas-comuns)
- [9. Considerações Finais](#9-considerações-finais)
- [10. Referências](#10-referências)


## 1. Visão Geral da Arquitetura

A integração será estabelecida através de uma VPN Site-to-Site. 
A AWS VPC terá um bloco CIDR "172.16.0.0/16" e a Azure VNet terá "10.0.0.0/16". 
Cada nuvem configura um Gateway VPN que se conecta ao outro, permitindo que as sub-redes em ambas as extremidades se comuniquem.

## 2. Arquitetura da Solução


(Figura ilustrativa com Azure, AWS, VPN Gateway, e VMs em sub-redes privadas)


## 3. Provisionamento das Máquinas Virtuais

## 3.1 - Azure

Azure Serviço: Ubuntu <br>
Rede Virtual (VNet): vnet-azure <br>
Sub-rede: subnet-azure-private <br>
IP Privado: 10.0.1.4 <br>
Região: West us 2 <br>
Grupo de Recursos: gr-multicloud <br>

## 3.1.1 [Configurações na Azure]

## 3.1.2  [Criação do Grupo de Recursos]

Um grupo de recursos é um contêiner lógico para os recursos do Azure.

1.  No portal do Azure, pesquise por "Grupos de recursos" e clique em **"Criar"**.
2.  Selecione sua **"Assinatura"** (ex: `Azure for Students`).
3.  Para **"Nome do grupo de recursos"**, digite `gr-multicloud`.
4.  Selecione a **"Região"** (ex: `(US) West US 2`).
5.  Clique em **"Revisar + criar"** e depois em **"Criar"**.

![Grupo de Recursos](MultiCloud/1az%20-%20Grupo%20de%20Recursos.png)

## 3.1.3  [Criação da Virtual Network (VNet)]

Agora, criaremos a VNet que abrigará seus recursos no Azure.

1.  No portal do Azure, pesquise por "Redes virtuais" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome da rede virtual"**, digite `vnet-azure`.
4.  Selecione a mesma **"Região"** (`West US 2`).
5.  Vá para a aba **"Endereços IP"**.
6.  O espaço de endereço padrão é `10.0.0.0/16`. Mantenha-o.
7.  **Adicionar sub-rede**:
    * **"Nome da sub-rede"**: `subnet-azure-private`.
    * **"Intervalo de endereços IP"**: `10.0.1.0/24`.
8.  Clique em **"Adicionar"**.
9.  Clique em **"Revisar + criar"** e depois em **"Criar"**.

![VNet](MultiCloud/2az%20-%20Vnet.png)
![VNet](MultiCloud/2.1az%20-%20Vnet.png)

## 3.1.4  [Criação da Subnet para o Gateway]

O Gateway de Rede Virtual precisa de uma sub-rede dedicada chamada `GatewaySubnet`.

1.  Navegue até sua **VNet** (`vnet-azure`).
2.  No menu esquerdo, em **"Configurações"**, clique em **"Sub-redes"**.
3.  Clique em **"+ Sub-rede do gateway"**.
4.  O **"Nome"** será automaticamente `GatewaySubnet`.
5.  Para **"Intervalo de endereços IP"**, use um bloco adequado, como `10.0.2.224/27` (um /27 é o mínimo recomendado).
6.  Clique em **"Adicionar"**.

![GatewaySubnet](MultiCloud/2.2az%20-%20sub%20gtw.png)

## 3.1.5  [Criação do Gateway de Rede Virtual (VPN Gateway)]

Este é o ponto de extremidade VPN do lado do Azure.

1.  No portal do Azure, pesquise por "Gateways de rede virtual" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome"**, digite `vpn-gtw-azure`.
4.  Selecione a mesma **"Região"** (`West US 2`).
5.  Para **"Tipo de gateway"**, selecione `VPN`.
6.  Para **"Tipo de VPN"**, selecione `Baseado em rota` (Route-based) ou `Baseado em política` (Policy-based). Para integração com AWS, geralmente `Baseado em rota` é a melhor escolha.
7.  Para **"SKU"**, selecione um SKU apropriado (ex: `VpnGw1`).
8.  Para **"Geração"**, selecione `Geração1`.
9.  Para **"Rede virtual"**, selecione `vnet-azure`. A sub-rede do gateway (`GatewaySubnet`) será preenchida automaticamente.
10. Para **"Endereço IP público"**, selecione **"Criar novo"**.
11. Para **"Nome do endereço IP público"**, digite `pub-azure-aws`.
12. Para **"SKU do endereço IP público"**, selecione `Standard`.
13. Para **"Atribuição"**, selecione `Estático`.
14. Deixe **"Habilitar o modo ativo-ativo"** como `Desabilitado`.
15. Deixe **"Configurar BGP"** como `Desabilitado` (para VPN estática).
16. Clique em **"Revisar + criar"** e depois em **"Criar"**.

![VPN Gateway](MultiCloud/3az%20-%20vpn.png)
![VPN Gateway](MultiCloud/3az%20-%20vpn2.png)
A criação do Gateway VPN pode levar de 30 a 45 minutos.

## 3.1.6  [Criação do Gateway de Rede Local]

O Gateway de Rede Local representa o Customer Gateway (a AWS VPC) para o Azure.

1.  No portal do Azure, pesquise por "Gateways de rede local" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome"**, digite `lgw-aws`.
4.  Para **"Ponto de extremidade"**, selecione **"Endereço IP"**.
5.  Para **"Endereço IP"**, insira o endereço IP público de um dos túneis do Gateway VPN da AWS (obtido no passo 2.5).
6.  Para **"Espaços de endereço"**, adicione o bloco CIDR da VPC da AWS (`172.16.0.0/16`).
7.  Clique em **"Revisar + criar"** e depois em **"Criar"**.


## 3.1.7  [Criação da Conexão]

Finalmente, crie a conexão entre o Gateway de Rede Virtual do Azure e o Gateway de Rede Local.

1.  Navegue até seu **Gateway de Rede Virtual** (`vpn-gtw-azure`).
2.  No menu esquerdo, em **"Configurações"**, clique em **"Conexões"**.
3.  Clique em **"+ Adicionar"**.
4.  Para **"Nome"**, digite `conn-azure-aws`.
5.  Para **"Tipo de conexão"**, selecione `Site-to-site (IPsec)`.
6.  Para **"Gateway de rede virtual"**, selecione `vpn-gtw-azure`.
7.  Para **"Gateway de rede local"**, selecione `lgw-aws`.
8.  Para **"Chave pré-compartilhada (PSK)"**, insira a mesma chave pré-compartilhada que você definiu ou obteve da configuração da VPN na AWS.
9.  Deixe **"BGP"** como `Desabilitado`.
10. Clique em **"OK"**.



## 3.2 AWS

AWS Serviço: EC2 (Amazon Linux) <br>
VPC: vpc-aws <br>
Sub-rede: subnet-aws-private <br>
IP Privado: 10.0.2.4 <br>
Região: us-east-1 <br>

## 3.2.1 - Configurações na AWS

## 3.2.2  [Criação da VPC]

Neste passo, criaremos a Virtual Private Cloud (VPC) que abrigará nossos recursos na AWS.

1.  No console da AWS, navegue até **VPC**.
2.  Clique em **"Create VPC"**.
3.  Selecione **"VPC only"**.
4.  Forneça um **"Name tag"**: `vpc-aws`.
5.  Em **"IPv4 CIDR block"**, insira `172.16.0.0/16`.
6.  Deixe as outras opções como padrão.
7.  Clique em **"Create VPC"**.

![VPC AWS](MultiCloud/1aws%20-%20vpc.png)

## 3.2.3  [Criação da Subnet]

Agora, criaremos uma sub-rede dentro da VPC recém-criada.

1.  No console da AWS, navegue até **VPC > Subnets**.
2.  Clique em **"Create subnet"**.
3.  Selecione a **"VPC ID"** (`vpc-aws`) que você acabou de criar.
4.  Para **"Subnet name"**, digite `subnet-aws-private`.
5.  Selecione uma **"Availability Zone"** (ex: `us-east-1a`).
6.  Em **"IPv4 subnet CIDR block"**, insira `172.16.1.0/24`.
7.  Clique em **"Create subnet"**.

![Subnet AWS](MultiCloud/2aws%20-%20subnet.png)

## 3.2.4   [Criação do Customer Gateway]

1.  No console da AWS, navegue até **VPC > Customer Gateways**.
2.  Clique em **"Create Customer Gateway"**.
3.  **"Name"**: `cgw-azure`.
4.  **"Routing"**: `Static` (para simplificar, mas BGP é recomendado para produção).
5.  **"IP Address"**: Insira o endereço IP público do Gateway VPN do Azure (será obtido após a criação do Gateway no Azure).
6.  Clique em **"Create Customer Gateway"**.

![Customer Gateway](MultiCloud/3aws%20-%20customer.png)

## 3.2.5   [Criação do Virtual Private Gateway (VPG)]

1.  No console da AWS, navegue até **VPC > Virtual Private Gateways**.
2.  Clique em **"Create Virtual Private Gateway"**.
3.  **"Name tag"**: `vpg-aws`.
4.  Deixe o resto como padrão.
5.  Clique em **"Create Virtual Private Gateway"**.
6.  Após a criação, selecione o VPG e clique em **"Actions > Attach to VPC"**.
7.  Selecione sua VPC (`vpc-aws`) e clique em **"Attach"**.

![Virtual Private Gateway (VPG)](MultiCloud/4aws%20-%20vpg.png)
![Virtual Private Gateway (VPG)](MultiCloud/4aws%20-%20vpg2.png)
![Virtual Private Gateway (VPG)](MultiCloud/4aws%20-%20vpg3.png)

## 3.2.6   [Criação da Conexão VPN (Site-to-Site)]

1.  No console da AWS, navegue até **VPC > Site-to-Site VPN Connections**.
2.  Clique em **"Create VPN Connection"**.
3.  **"Name"**: `vpn-aws-azure`.
4.  **"Target Gateway Type"**: `Virtual Private Gateway`.
5.  **"Virtual Private Gateway"**: Selecione o `vpg-aws` criado anteriormente.
6.  **"Customer Gateway"**: `Existing`.
7.  **"Customer Gateway ID"**: Selecione o `cgw-azure` criado anteriormente.
8.  **"Routing Options"**: `Static`.
9.  **"Static IP Prefixes"**: Adicione o bloco CIDR da VNet do Azure (`10.0.0.0/16`).
10. **"Tunnel Inside IP Version"**: `IPv4`.
11. **"Local IPv4 Network CIDR"**: `172.16.0.0/16` (CIDR da sua VPC AWS).
12. **"Remote IPv4 Network CIDR"**: `10.0.0.0/16` (CIDR da sua VNet Azure).
13. **"Tunnel Options"**: Pode deixar como padrão ou personalizar (ex: chaves pré-compartilhadas).
14. Clique em **"Create VPN Connection"**.

Após a criação, você poderá "Download Configuration" para obter os detalhes do túnel, incluindo os IPs públicos dos endpoints da AWS VPN. Guarde esses IPs, pois você precisará deles para a configuração no Azure.



## 4. Configuração da VPN Site-to-Site Etapa 1: AWS (VPN Gateway e Customer Gateway) Criar Customer Gateway com o IP público do Azure VPN Gateway.

Criar Virtual Private Gateway e anexar à VPC.

Criar a Conexão VPN com IP e PSK gerados.

Configurar a rota 10.1.1.0/24 (Azure) na tabela de rotas da AWS.

Etapa 2: Azure (Gateway de VPN) Criar o Gateway de VPN (VPN Gateway, tipo: Route-based).

Criar Conexão de VPN para o endereço do AWS VPN Gateway.

Inserir chave pré-compartilhada (PSK) igual à usada na AWS.

Configurar a rota 10.2.1.0/24 (AWS) no Azure.

Dica: Certifique-se que ambas as VPNs estejam com IPsec/IKEv2 e mesmo algoritmo de criptografia (Ex: AES256/SHA256/DH14).


## 5. Configuração de Rede e Segurança Grupos de Segurança / Firewall Azure: liberar ICMP (ping) e SSH da sub-rede 10.2.1.0/24
AWS: liberar ICMP (ping) e SSH da sub-rede 10.1.1.0/24

Tabelas de Roteamento Azure - Tabela de Rota Nome Prefixo Próximo salto rotaAWS 10.2.1.0/24 Gateway de VPN (Azure)

AWS - Tabela de Rota Nome Prefixo Próximo salto rotaAzure 10.1.1.0/24 Virtual Private Gateway

Marcar “Propagar rotas do gateway” na AWS para o VPN Gateway.

## 6. Testes de Conectividade

Após a conclusão das configurações em ambas as nuvens:

1.  **Verifique o status da conexão VPN:**
    * No AWS: **VPC > Site-to-Site VPN Connections**. O status deve ser "UP".
    * No Azure: **Gateway de Rede Virtual > Conexões**. O status deve ser "Conectado".

2.  **Lance instâncias de teste:**
    * Crie uma instância EC2 na AWS na `subnet-aws-private` (`172.16.1.x`). Certifique-se de que seu Security Group permite tráfego ICMP (ping) e SSH/RDP de `10.0.0.0/16`.
    * Crie uma máquina virtual (VM) no Azure na `subnet-azure-private` (`10.0.1.x`). Certifique-se de que o Network Security Group (NSG) permite tráfego ICMP (ping) e SSH/RDP de `172.16.0.0/16`.

3.  **Teste a conectividade:**
    * Do EC2, tente dar `ping` para o IP privado da VM do Azure.

SSH da EC2 → VM da Azure:

	ssh ubuntu@172.16.2.4

Ping da EC2 → VM da Azure:

	* ping 10.1.1.4

* Da VM do Azure, tente dar `ping` para o IP privado da instância EC2.

SSH da VM da Azure → EC2:

	ssh ec2-user@10.2.1.4

Ping da VM da Azure → EC2:

	ping 172.16.2.4

Verificar rotas com ip route ou traceroute:

	traceroute 10.2.1.4
	traceroute 172.16.2.4

Se a VPN estiver ativa e os firewalls corretamente configurados, a comunicação será bem-sucedida sem uso de IPs públicos.

## 7. Tabelas de Configuração 

 | Recurso | IP Privado | Sub-rede | CIDR | Região | 
|---------------|----------------|---------------|---------|------------|
|VM Azure|10.0.1.4|subnet-azure|10.0.1.0/24|West us 2|
|EC2 AWS|172.16.1.4|subnet-aws|172.16.1.0/24|us-east-1|
|GW Azure|Dinâmico|----|---|b|
|GW AWS|Dinâmico|---|---|b|

## 8. Resolução de Problemas Comuns

* **Firewalls/Security Groups/NSGs:** Certifique-se de que as regras de entrada e saída permitem o tráfego necessário entre os blocos CIDR da AWS e do Azure.
* **Chave Pré-Compartilhada (PSK):** Verifique se a PSK é idêntica em ambos os lados da conexão VPN.
* **Blocos CIDR:** Confirme que os blocos CIDR configurados nos Gateways de Rede Local e Conexões VPN estão corretos para cada extremidade.
* **Rotas:** Para VPNs baseadas em políticas, as regras de tráfego devem estar corretas. Para VPNs baseadas em rota, as tabelas de rota (especialmente as de roteamento de gateway) devem propagar as rotas corretas. Na AWS, verifique a tabela de rotas associada à sua VPC e à Subnet.
* **Status da VPN:** Se a VPN não estiver "UP", verifique os logs do Gateway (se disponíveis) e revise cada passo da configuração.
* **Endereços IP Públicos:** Verifique se os IPs públicos dos Gateways estão corretos e acessíveis.


## 9. Considerações Finais 
A criação de uma VPN site-to-site entre Azure e AWS é um passo importante para ambientes corporativos que desejam trabalhar em infraestruturas híbridas ou multi-cloud. Esta configuração demonstrou ser eficaz para comunicação segura e privada entre dois ambientes distintos, sem necessidade de IPs públicos para testes.

## 10. Referências

Microsoft Learn - [Conectar Azure a outra nuvem via VPN](https://learn.microsoft.com/pt-br/azure/vpn-gateway/)

AWS Documentation - [Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html)

RFC 4301 - [IPsec Architecture](https://datatracker.ietf.org/doc/html/rfc4301)

