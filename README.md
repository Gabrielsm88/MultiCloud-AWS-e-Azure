# Integração Multi-Cloud Azure + AWS via VPN Site-to-Site


Sumário
Introdução

Arquitetura da Solução

Provisionamento das Máquinas Virtuais

Configuração da VPN Site-to-Site

Configuração de Rede e Segurança

Testes de Conectividade

Tabelas de Configuração

Considerações Finais

Referências

1. Introdução
Este documento descreve a criação de um ambiente multi-cloud entre Microsoft Azure e Amazon Web Services (AWS), com comunicação privada via VPN Site-to-Site. O objetivo é permitir que duas máquinas virtuais (uma em cada nuvem) se comuniquem entre si apenas por IPs privados.

<div>2. Arquitetura da Solução</div>
 

(Figura ilustrativa com Azure, AWS, VPN Gateway, e VMs em sub-redes privadas)

3. Provisionamento das Máquinas Virtuais
Azure
Serviço: Máquina Virtual (Ubuntu Server 22.04)

Rede Virtual (VNet): vnet-azure

Sub-rede: subnet-azure-private

IP Privado: 10.1.1.4

Região: Brazil South

Grupo de Recursos: rg-multicloud

AWS
Serviço: EC2 (Amazon Linux 2023)

VPC: vpc-aws

Sub-rede: subnet-aws-private

IP Privado: 10.2.1.4

Região: us-east-1

4. Configuração da VPN Site-to-Site
Etapa 1: AWS (VPN Gateway e Customer Gateway)
Criar Customer Gateway com o IP público do Azure VPN Gateway.

Criar Virtual Private Gateway e anexar à VPC.

Criar a Conexão VPN com IP e PSK gerados.

Configurar a rota 10.1.1.0/24 (Azure) na tabela de rotas da AWS.

Etapa 2: Azure (Gateway de VPN)
Criar o Gateway de VPN (VPN Gateway, tipo: Route-based).

Criar Conexão de VPN para o endereço do AWS VPN Gateway.

Inserir chave pré-compartilhada (PSK) igual à usada na AWS.

Configurar a rota 10.2.1.0/24 (AWS) no Azure.

 Dica: Certifique-se que ambas as VPNs estejam com IPsec/IKEv2 e mesmo algoritmo de criptografia (Ex: AES256/SHA256/DH14).

5. Configuração de Rede e Segurança
Grupos de Segurança / Firewall
Azure: liberar ICMP (ping) e SSH da sub-rede 10.2.1.0/24

AWS: liberar ICMP (ping) e SSH da sub-rede 10.1.1.0/24

Tabelas de Roteamento
Azure - Tabela de Rota
Nome	Prefixo	Próximo salto
rotaAWS	10.2.1.0/24	Gateway de VPN (Azure)

AWS - Tabela de Rota
Nome	Prefixo	Próximo salto
rotaAzure	10.1.1.0/24	Virtual Private Gateway

 Marcar “Propagar rotas do gateway” na AWS para o VPN Gateway.

6. Testes de Conectividade
Passos:
SSH da VM da Azure → EC2:

bash
Copiar
Editar
ssh ec2-user@10.2.1.4
Ping da VM da Azure → EC2:

bash
Copiar
Editar
ping 10.2.1.4
Ping da EC2 → VM da Azure:

bash
Copiar
Editar
ping 10.1.1.4
Verificar rotas com ip route ou traceroute:

bash
Copiar
Editar
traceroute 10.2.1.4
 Se a VPN estiver ativa e os firewalls corretamente configurados, a comunicação será bem-sucedida sem uso de IPs públicos.

7. Tabelas de Configuração
Tabela de IPs e Sub-redes
Recurso	IP Privado	Sub-rede CIDR	Região
VM Azure	10.1.1.4	10.1.1.0/24	Brazil South
EC2 AWS	10.2.1.4	10.2.1.0/24	us-east-1
Gateway Azure	Dinâmico	-	Brazil South
Gateway AWS	Dinâmico	-	us-east-1

8. Considerações Finais
A criação de uma VPN site-to-site entre Azure e AWS é um passo importante para ambientes corporativos que desejam trabalhar em infraestruturas híbridas ou multi-cloud. Esta configuração demonstrou ser eficaz para comunicação segura e privada entre dois ambientes distintos, sem necessidade de IPs públicos para testes.

