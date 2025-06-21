# Integração Multi-Cloud: Azure e AWS (VPN Site-to-Site)

Este documento descreve a criação de um ambiente multi-cloud entre Microsoft Azure e Amazon Web Services (AWS), com comunicação privada via VPN Site-to-Site.
O objetivo é permitir que duas máquinas virtuais (uma em cada nuvem) se comuniquem entre si apenas por IPs privados.

## Sumário

- [1. Visão Geral da Arquitetura](#1-visão-geral-da-arquitetura)
- [2. Arquitetura da Solução](#2-arquitetura-da-solução)
- [3. Provisionamento das Máquinas Virtuais](#3-provisionamento-das-máquinas-virtuais)
- [4. Configuração da VPN Site-to-Site](#4-configuração-da-vpn-site-to-site)
- [5. Configuração de Rede](#5-configuração-de-rede)
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
![Arquitetura MultiCloud](MultiCloud/)


## 3. Provisionamento das Máquinas Virtuais

### 3.1 - Configurações na Azure

Azure Serviço: Ubuntu <br>
Rede Virtual (VNet): vnet-azure (10.0.0.0/16) <br>
Sub-rede: subnet-azure-private (10.0.1.0/24) <br>
IP Privado: 10.0.1.4 <br>
Região: West US 2 <br>
Grupo de Recursos: gr-multicloud <br>
Grupo de Segurança de Rede (NSG): nsg-multicloud 

##

### 3.1.1  [Criação do Grupo de Recursos]

Um grupo de recursos é um contêiner lógico para os recursos do Azure.

1.  No portal do Azure, pesquise por "Grupos de recursos" e clique em **"Criar"**.
2.  Selecione sua **"Assinatura"** (ex: `Azure for Students`).
3.  Para **"Nome do grupo de recursos"**, digite `gr-multicloud`.
4.  Selecione a **"Região"** (ex: `(US) West US 2`).
5.  Clique em **"Revisar + criar"** e depois em **"Criar"**.

![Grupo de Recursos](MultiCloud/azure%20-%20gr.png)

### 3.1.2  [Criação da Virtual Network (VNet)]

Agora, criaremos a VNet que abrigará seus recursos no Azure.

1.  No portal do Azure, pesquise por "Redes virtuais" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome da rede virtual"**, digite `vnet-azure`.
4.  Selecione a mesma **"Região"** (`West US 2`).

![vnet0](MultiCloud/azure%20-%20vnet0.png)

7.  Vá para a aba **"Endereços IP"**.
8.  O espaço de endereço padrão é `10.0.0.0/16`. Mantenha-o.
9.  **Adicionar sub-rede**:
    * **"Nome da sub-rede"**: `subnet-azure-private`.
    * **"Intervalo de endereços IP"**: `10.0.1.0/24`.
10.  Clique em **"Adicionar"**.
11.  Clique em **"Revisar + criar"** e depois em **"Criar"**.

![vnet1](MultiCloud/azure%20-%20vnet1.png)

### 3.1.3  [Criação da Subnet para o Gateway]

O Gateway de Rede Virtual precisa de uma sub-rede dedicada chamada `GatewaySubnet`.

1.  Navegue até sua **VNet** (`vnet-azure`).
2.  No menu esquerdo, em **"Configurações"**, clique em **"Sub-redes"**.
3.  Clique em **"+ Sub-rede do gateway"**.
4.  O **"Nome"** será automaticamente `GatewaySubnet`.
5.  Para **"Intervalo de endereços IP"**, use um bloco adequado, como `10.0.2.224/27` (um /27 é o mínimo recomendado).
6.  Clique em **"Adicionar"**.

![GatewaySubnet](MultiCloud/azure%20-%20sub%20gtw.png)

### 3.1.4  [Criação do Network Security Group (NSG)]

1.  No portal do Azure, pesquise por "Grupos de segurança de rede" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome"**, digite `nsg-multicloud`.
4.  Selecione a mesma **"Região"** (`West US 2`).
5.  Clique em **"Revisar + criar"** e depois em **"Criar"**.

![nsg0](MultiCloud/azure%20-%20nsg0.png)

6.  Após a criação, navegue até o `nsg-multicloud`.
7.  No menu esquerdo, clique em **"Regras de segurança de entrada"**.
8.  Clique em **"+ Adicionar"** e adicione as seguintes regras:

  | Nome | Prioridade | Origem | Intervalos de IP de origem | Portas de Destino | Protocolo | Ação |
|------|------------|--------|----------------------------|-------------------|-----------|------|
| AWS-ICMP | 100 | Endereços IP | 172.16.0.0/16 | Qualquer | ICMP | Permitir |
| AWS-SSH-RDP | 110 | Endereços IP | 172.16.0.0/16 | 22 (SSH) ou 3389 (RDP) | TCP | Permitir |
| Meu-IP-SSH-RDP | 120 | Endereços IP | Seu IP Público | 22 (SSH) ou 3389 (RDP) | TCP | Permitir |
    
9.  Clique em **"Adicionar"** para cada regra.

![nsg1](MultiCloud/azure%20-%20nsg1.png)
![nsg2](MultiCloud/azure%20-%20nsg2.png)
![nsg3](MultiCloud/azure%20-%20nsg3.png)

### 3.1.5  [Criação da Máquina Virtual]

1.  No portal do Azure, pesquise por "Máquinas virtuais" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome da máquina virtual"**, digite `vm-azure`.
4.  Selecione a mesma **"Região"** (`West US 2`).
5.  **"Opções de segurança"**: Selecione `Standard`.
6.  **"Imagem"**: Escolha uma imagem (ex: `Ubuntu Server 24.04 LTS` ou `Windows Server 2019 Datacenter`).

![vm0](MultiCloud/azure%20-%20vm0.png)

7.  **"Tamanho"**: Selecione um tamanho (ex: `Standard_B1s` ou `Standard_B2as_v2`).
8.  **"Tipo de autenticação"**: `Chave pública SSH` ou `Senha`. Configure as credenciais de login.
9.  **"Regras da porta de entrada"**: `Nenhuma`. O NSG que criamos irá gerenciar isso.

![vm1](MultiCloud/azure%20-%20vm1.png)

10. **"Discos"**: Deixe o padrão ou configure conforme necessário.

![vm2](MultiCloud/azure%20-%20vm2.png)

11. **"Rede"**:
    * **"Rede virtual"**: Selecione `vnet-azure`.
    * **"Sub-rede"**: Selecione `subnet-azure-private` (`10.0.1.0/24`).
    * **"IP público"**: Selecione `Nenhum` (para forçar o tráfego via VPN). Se precisar de acesso público inicial, pode criar um e depois desassociá-lo.
    * **"Grupo de segurança de rede da NIC"**: Selecione `Grupo de segurança de rede avançado` e escolha `nsg-multicloud`.
12. Configure as demais abas (Gerenciamento, Monitoramento, Avançado, Marcas) conforme necessário.
13. Clique em **"Revisar + criar"** e depois em **"Criar"**.
14. Anote o **endereço IP privado** da VM do Azure (ex: `10.0.1.X`).

![vm3](MultiCloud/azure%20-%20vm3.png)

##
<br>

## 3.2 - Configurações na AWS

AWS Serviço: EC2 (Amazon Linux) <br>
VPC: vpc-aws (172.16.0.0/16) <br>
Sub-rede: subnet-aws-private (172.16.1.0/24) <br>
IP Privado: 172.16.1.4 <br>
Região: us-east-1 <br>
Security Group: multicloud-sg

##

### 3.2.1  [Criação da VPC]

Neste passo, criaremos a Virtual Private Cloud (VPC) que abrigará nossos recursos na AWS.

1.  No console da AWS, navegue até **VPC**.
2.  Clique em **"Create VPC"**.
3.  Selecione **"VPC only"**.
4.  Forneça um **"Name tag"**: `vpc-aws`.
5.  Em **"IPv4 CIDR block"**, insira `172.16.0.0/16`.
6.  Deixe as outras opções como padrão.
7.  Clique em **"Create VPC"**.

![vpc](MultiCloud/aws%20-%20vpc.png)

### 3.2.2  [Criação da Subnet]

Agora, criaremos uma sub-rede dentro da VPC recém-criada.

1.  No console da AWS, navegue até **VPC > Subnets**.
2.  Clique em **"Create subnet"**.
3.  Selecione a **"VPC ID"** (`vpc-aws`) que você acabou de criar.
4.  Para **"Subnet name"**, digite `subnet-aws-private`.
5.  Selecione uma **"Availability Zone"** (ex: `us-east-1a`).
6.  Em **"IPv4 subnet CIDR block"**, insira `172.16.1.0/24`.
7.  Clique em **"Create subnet"**.

![subnet](MultiCloud/aws%20-%20subnet.png)

### 3.2.3  [Criação do Security Group]

1.  No console da AWS, navegue até **EC2 > Security Groups**.
2.  Clique em **"Create security group"**.
3.  **"Security group name"**: `multicloud-sg`.
4.  **"Description"**: `liberar trafego do azure vnet para vpn`.
5.  **"VPC"**: Selecione a `vpc-aws`.
6.  **"Inbound rules"**: Adicione as seguintes regras:

| Type | Source | Description |
|------|--------|-------------|
| All ICMP - IPv4 | 10.0.0.0/16 | VNet CIDR do Azure | 
| SSH ou RDP | 10.0.0.0/16 | Acesso da VM do Azure a EC2 | 
| SSH ou RDP | My IP | Acesso inicial a EC2 | 

7.  **"Outbound rules"**: Deixe o padrão (All traffic, All IPs) ou restrinja, mas para teste, o padrão é suficiente.
8.  Clique em **"Create security group"**.

![sg](MultiCloud/aws%20-%20sg.png)

### 3.2.4  [Criação da Instância EC2]

1.  No console da AWS, navegue até **EC2 > Instances**.
2.  Clique em **"Launch instances"**.
3.  **"Name"**: `ec2-aws`.
4.  Selecione uma **"AMI"** (ex: Amazon Linux 2 AMI).

![instance](MultiCloud/aws%20-%20instance0.png)

5.  Selecione um **"Instance type"** (ex: `t2.micro`).
6.  **"Key pair (login)"**: Selecione um par de chaves existente ou crie um novo. (ex: `vockey`)

![instance](MultiCloud/aws%20-%20instance1.png)

7.  **"Network settings"**:
    * **"VPC"**: Selecione `vpc-aws`.
    * **"Subnet"**: Selecione `subnet-aws-private` (`172.16.1.0/24`).
    * **"Auto-assign public IP"**: `Disable` (para forçar o tráfego via VPN, ou `Enable` para testar conectividade pública separadamente).
    * **"Firewall (security groups)"**: Selecione `Select an existing security group` e escolha `multicloud-sg`.
8.  Configure o armazenamento e outras tags conforme necessário.
9.  Clique em **"Launch instance"**.
10. Anote o **endereço IP privado** da instância EC2 (ex: `172.16.1.X`).

![instance](MultiCloud/aws%20-%20instance2.png)


## 4. Configuração da VPN Site-to-Site

* **Azure (Gateway de VPN)**

### 4.1  [Criação do Gateway de Rede Virtual (VPN Gateway)]

Este é o ponto de extremidade VPN do lado do Azure.

1.  No portal do Azure, pesquise por "Gateways de rede virtual" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome"**, digite `vpn-gtw-azure`.
4.  Selecione a mesma **"Região"** (`West US 2`).
5.  Para **"Tipo de gateway"**, selecione `VPN`.
6.  Para **"SKU"**, selecione um SKU apropriado (ex: `VpnGw1`).
7.  Para **"Geração"**, selecione `Geração1`.
8.  Para **"Rede virtual"**, selecione `vnet-azure`. A sub-rede do gateway (`GatewaySubnet`) será preenchida automaticamente.

![VPN Gateway](MultiCloud/azure%20-%20vpn0.png)

9. Para **"Endereço IP público"**, selecione **"Criar novo"**.
10. Para **"Nome do endereço IP público"**, digite `pub-azure-aws`.
11. Para **"SKU do endereço IP público"**, selecione `Standard`.
12. Para **"Atribuição"**, selecione `Estático`.
13. Deixe **"Habilitar o modo ativo-ativo"** como `Desabilitado`.
14. Deixe **"Configurar BGP"** como `Desabilitado` (para VPN estática).
15. Clique em **"Revisar + criar"** e depois em **"Criar"**.

![VPN Gateway](MultiCloud/azure%20-%20vpn1.png)

* **A criação do Gateway VPN pode levar de 30 a 45 minutos.**

### 4.2  [Criação do Gateway de Rede Local]

O Gateway de Rede Local representa o Customer Gateway (a AWS VPC) para o Azure.

1.  No portal do Azure, pesquise por "Gateways de rede local" e clique em **"Criar"**.
2.  Selecione a **"Assinatura"** e o **"Grupo de recursos"** (`gr-multicloud`).
3.  Para **"Nome"**, digite `lgw-aws`.
4.  Para **"Região"**, selecione **"West US 2"**
5.  Para **"Ponto de extremidade"**, selecione **"Endereço IP"**.
6.  Para **"Endereço IP"**, insira o endereço IP público de um dos túneis do Gateway VPN da AWS (ex: Túnel 1 com `PSK`).
7.  Para **"Espaços de endereço"**, adicione o bloco CIDR da VPC da AWS (`172.16.0.0/16`).
8.  Clique em **"Revisar + criar"** e depois em **"Criar"**.

![Local Gateway](MultiCloud/azure%20-%20lgw.png)

### 4.3  [Criação da Conexão]

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

Inserir chave pré-compartilhada (PSK) igual à usada na AWS.



* **AWS (VPN Gateway e Customer Gateway)** 

### 4.4  [Criação do Customer Gateway]

1.  No console da AWS, navegue até **VPC > Customer Gateways**.
2.  Clique em **"Create Customer Gateway"**.
3.  **"Name"**: `ctm-gtw-aws`.
5.  **"IP Address"**: Insira o endereço IP público do Gateway VPN do Azure.
6.  Clique em **"Create Customer Gateway"**.

![Customer Gateway](MultiCloud/aws%20-%20customer.png)

### 4.5  [Criação do Virtual Private Gateway (VPG)]

1.  No console da AWS, navegue até **VPC > Virtual Private Gateways**.
2.  Clique em **"Create Virtual Private Gateway"**.
3.  **"Name tag"**: `vpg-aws`.
4.  Deixe o resto como padrão.
5.  Clique em **"Create Virtual Private Gateway"**.

![Virtual Private Gateway (VPG)](MultiCloud/aws%20-%20vpg0.png)

6.  Após a criação, selecione o VPG e clique em **"Actions > Attach to VPC"**.
7.  Selecione sua VPC (`vpc-aws`) e clique em **"Attach"**.

![Virtual Private Gateway (VPG)](MultiCloud/aws%20-%20vpg1.png)
![Virtual Private Gateway (VPG)](MultiCloud/aws%20-%20vpg2.png)

### 4.6  [Criação da Conexão VPN (Site-to-Site)]

1.  No console da AWS, navegue até **VPC > Site-to-Site VPN Connections**.
2.  Clique em **"Create VPN Connection"**.
3.  **"Name"**: `vpn-aws-azure`.
4.  **"Target Gateway Type"**: `Virtual Private Gateway`.
5.  **"Virtual Private Gateway"**: Selecione o `vpg-aws` criado anteriormente.
6.  **"Customer Gateway"**: `Existing`.
7.  **"Customer Gateway ID"**: Selecione o `ctm-gtw-aws` criado anteriormente.
8.  **"Routing Options"**: `Static`.

![VPN Site-to-Site](MultiCloud/aws%20-%20sts0.png)

9.  **"Static IP Prefixes"**: Adicione o bloco CIDR da VNet do Azure (`10.0.0.0/16`).
10. **"Local IPv4 Network CIDR"**: `172.16.0.0/16` (CIDR da sua VPC AWS).
11. **"Remote IPv4 Network CIDR"**: `10.0.0.0/16` (CIDR da sua VNet Azure).
12. **"Tunnel Options"**: Pode deixar como padrão ou personalizar (ex: chaves pré-compartilhadas: `Key_VPN_2025.`).
13. Clique em **"Create VPN Connection"**.

![VPN Site-to-Site](MultiCloud/aws%20-%20sts1.png)

Após a criação, você poderá "Download Configuration" para obter os detalhes do túnel, incluindo os IPs públicos dos endpoints da AWS VPN. Guarde esses IPs, pois você precisará deles para a configuração no Azure.

* **Dica: Certifique-se que ambas as VPNs estejam com IPsec/IKEv2 e mesmo algoritmo de criptografia (Ex: AES256/SHA256/DH14).**


## 5. Configuração de Rede

- Configurar a rota 172.16.1.0/24 (AWS) no Azure.
- Configurar a rota 10.0.1.0/24 (Azure) na tabela de rotas da AWS.

### Tabelas de Roteamento

#### Azure - Tabela de Rota 
| Nome | Prefixo | Próximo salto |
|-------|----------|----------------|
| rotaAWS | 172.16.1.0/24 | Gateway de VPN (Azure) |

#### AWS - Tabela de Rota 
| Nome | Prefixo | Próximo salto |
|------|--------|--------------|
| rotaAzure | 10.0.1.0/24 | Virtual Private Gateway |


[comment]: # (Marcar “Propagar rotas do gateway” na AWS para o VPN Gateway.)



## 6. Testes de Conectividade

Após a conclusão das configurações em ambas as nuvens:

1.  **Verifique o status da conexão VPN:**
   * No AWS: **VPC > Site-to-Site VPN Connections**. O status deve ser "UP".
   * No Azure: **Gateway de Rede Virtual > Conexões**. O status deve ser "Conectado".

2.  **Teste a conectividade:**

   * Do EC2, tente dar `ping` para o IP privado da VM do Azure.

#### SSH da EC2 → VM da Azure:

	ssh ubuntu@10.0.1.4

#### Ping da EC2 → VM da Azure:

	* ping 10.0.1.4 

 <br>   * Da VM do Azure, tente dar `ping` para o IP privado da instância EC2.

#### SSH da VM da Azure → EC2:

	ssh ec2-user@172.16.1.4

#### Ping da VM da Azure → EC2:

	ping 172.16.1.4

#### Verificar rotas com ip route ou traceroute:

	traceroute 10.0.1.4
	traceroute 172.16.1.4

Se a VPN estiver ativa e os firewalls corretamente configurados, a comunicação será bem-sucedida sem uso de IPs públicos.


## 7. Tabela de Configurações

 | Recurso | IP Privado | Sub-rede | CIDR | Região | 
|---------------|----------------|---------------|---------|------------|
|VM Azure|10.0.1.4|subnet-azure-private|10.0.1.0/24|West US 2|
|EC2 AWS|172.16.1.4|subnet-aws-private|172.16.1.0/24|us-east-1|
|GW Azure|Dinâmico|----|---|West US 2|
|GW AWS|Dinâmico|---|---|us-east-1|


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
