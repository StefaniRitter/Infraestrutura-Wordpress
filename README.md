# Infraestrutura-Wordpress

Este projeto tem como objetivo implantar o WordPress na AWS de forma escalável e altamente disponível. A ideia é simular um ambiente de produção real, utilizando serviços como EC2 com Auto Scaling, Load Balancer, RDS e EFS, garantindo que a aplicação continue funcionando mesmo em caso de falhas. Além de colocar em prática conceitos de arquitetura em nuvem, o projeto também ajuda a desenvolver habilidades em infraestrutura como código, provisionamento de recursos e resiliência de sistemas, utilizando os principais serviços gerenciados da AWS. 

## Ferramentas e dependências utilizadas: 
* **Docker e Docker Compose**: usados dentro das instâncias EC2 para executar o WordPress em containers.
* **MySQL**: banco de dados relacional utilizado pelo WordPress, provisionado no Amazon RDS em ambiente de produção.
* **WordPress**: plataforma CMS que foi implantada e configurada.
* **AWS EC2**: instâncias de servidor para hospedar a aplicação WordPress.
* **Amazon EFS**: sistema de arquivos compartilhado entre as instâncias EC2.
* **Application Load Balancer (ALB)**: balanceamento de tráfego entre múltiplas instâncias.
* **Auto Scaling Group (ASG)**: escalabilidade automática das instâncias EC2 conforme a demanda.
* **VPC personalizada**: rede virtual com subnets públicas e privadas, NAT Gateway e tabelas de rotas.
* **AWS CLI / Console AWS**: para criação, configuração e gerenciamento dos recursos.
* **Sistema Operacional da EC2**: Ubuntu 24.04 LTS.

## Etapa 1: Configuração do Ambiente

### 1.1. Configuração da Rede na AWS Console

* **Criação da VPC "vpc-wordpress"**: rede isolada criada na região us-east-1, onde toda a infraestrutura foi provisionada.
* **Criação de Sub-redes**:
  * **Zona de disponibilidade us-east-1a**: subrede-publica01 e subrede-privada01.
  * **Zona de disponibilidade us-east-1b**: subrede-publica02 e subrede-privada02.
* **Internet Gateway (igw-wordpress)**: criado e anexado à VPC para permitir a comunicação com a internet.
* **NAT Gateway (nat-wordpress)**: criado na subrede-publica01 com IP elástico associado, permitindo que instâncias em subnets privadas tenham acesso à internet.
* **Tabela de Rotas Pública (route-table-wordpress)**: associada às subnets públicas, com rota padrão 0.0.0.0/0 apontando para o Internet Gateway.
* **Tabela de Rotas Privada (route-table-privada)**: associada às subnets privadas, com rota padrão 0.0.0.0/0 apontando para o NAT Gateway, garantindo saída para a internet sem exposição direta.

## Etapa 2: Configuração do RDS e EFS

### 2.1. Criação do Banco de Dados (Amazon RDS)

* **Instância RDS MySQL**: criada no serviço Amazon RDS utilizando a engine MySQL.
* **Subnets privadas**: o banco foi provisionado em sub-redes privadas da VPC, garantindo maior segurança.
* **Parâmetros de acesso**: usuário administrador, senha e endpoint gerados no momento da criação. Esses dados são utilizados pelo WordPress para se conectar ao banco.
  
### 2.2. Criação do Sistema de Arquivos (Amazon EFS)

* **Sistema de Arquivos EFS**: criado para armazenar os arquivos de mídia do WordPress (imagens, vídeos e documentos).
* **Montagem via User Data**: configurado posteriormente para ser montado automaticamente em todas as instâncias EC2 no momento do boot.








# grupo de segurança rds e efs criados com a ec2 e seu grupo
* **Grupo de Segurança (sg-rds-wordpress)**: configurado para permitir conexões apenas das instâncias EC2.
  * **Porta liberada**: 3306 (MySQL).
  * **Origem**: Security Group das instâncias EC2.

    * **Grupo de Segurança (sg-efs-wordpress)**: configurado para permitir conexões apenas das instâncias EC2.

  * Porta liberada: 2049 (NFS).
  * Origem: Security Group das instâncias EC2.









