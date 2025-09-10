# Infraestrutura-Wordpress

Este projeto tem como objetivo implantar o WordPress na AWS de forma escalável e altamente disponível. A ideia é simular um ambiente de produção real, utilizando serviços como EC2 com Auto Scaling, Load Balancer, RDS e EFS, garantindo que a aplicação continue funcionando mesmo em caso de falhas. Além de colocar em prática conceitos de arquitetura em nuvem, o projeto também ajuda a desenvolver habilidades em infraestrutura como código, provisionamento de recursos e resiliência de sistemas, utilizando os principais serviços gerenciados da AWS. 

## Ferramentas e dependências utilizadas
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

### 1.1. Configuração da Rede na AWS Console:

* **Criação da VPC (`vpc-wordpress`)**: rede isolada criada na região **us-east-1**, onde toda a infraestrutura foi provisionada.
* **Criação de Sub-redes**:
  * **Zona de disponibilidade us-east-1a**: `subnet-publica01` e `subnet-privada01`.
  * **Zona de disponibilidade us-east-1b**: `subnet-publica02` e `subnet-privada02`.
* **Internet Gateway (`igw-wordpress`)**: criado e anexado à VPC para permitir a comunicação com a internet.
* **NAT Gateway (`nat-wordpress`)**: criado na `subnet-publica01` com IP elástico associado, permitindo que instâncias em subnets privadas tenham acesso à internet.
* **Tabela de Rotas Pública (`route-table-wordpress`)**: associada às subnets públicas, com rota padrão 0.0.0.0/0 apontando para o Internet Gateway.
* **Tabela de Rotas Privada (`route-table-privada`)**: associada às subnets privadas, com rota padrão 0.0.0.0/0 apontando para o NAT Gateway, garantindo saída para a internet sem exposição direta.

## Etapa 2: Configuração do RDS e EFS

### 2.1. Criação do Banco de Dados (Amazon RDS):

* **Instância RDS MySQL**: uma intância single-az db3t.micro foi criada no serviço Amazon RDS utilizando a engine MySQL e atribuída à `vpc-wordpress`.
* **Subnets privadas**: o banco foi provisionado em um subnet-group `subnet-group-privado` contendo apenas as subnets privadas `subnet-privada01` e `subnet-privada02` da VPC, garantindo maior segurança.
* **Parâmetros de acesso**: usuário administrador (`admin`), senha, nome do banco (`wordpress_db`) e endpoint foram gerados no momento da criação. Esses dados serão utilizados posteriormente pelo WordPress para se conectar ao banco.
  
### 2.2. Criação do Sistema de Arquivos (Amazon EFS):

* **Sistema de Arquivos EFS**: sistema criado para armazenar os arquivos de mídia do WordPress (imagens, vídeos e documentos), usando as configurações padrão.
* **Montagem via User Data**: será configurado posteriormente para ser montado automaticamente em todas as instâncias EC2 no momento do boot.

## Etapa 3: Configuração do AWS Secrets Manager:
O AWS Secrets Manager foi utilizado para armazenar e proteger as credenciais de acesso ao banco de dados MySQL do WordPress. Dessa forma, não é necessário deixar usuário e senha expostos diretamente no script user-data.

### 3.1. Criação do Segredo:
* No Console da AWS, foi selecionada a opção 'Armazenar um novo segredo', dentro do serviço Secrets Manager.
* **Tipo de segredo selecionado**: 'Outro tipo de segredo'.
* **Campos informados**:
  * **username**: usuário administrador do RDS (`admin`).
  * **password**: senha definida no momento de criação do banco.
  * **host**: endpoint do banco RDS (disponível na seção de RDS).
  * **dbname**: nome do banco usado pelo WordPress (`wordpress_db`).
* **Nome dado ao segredo**: `/prod/wordpress/db`.
* **VPC**: `vpc-wordpress`.

### 3.2. Definindo Permissões de Acesso:

* Na aba Políticas da seção IAM, a opçaõ 'Criar política' foi selecionada.
* **Serviço escolhido**: EC2.
* **Política definida em 'Editor de políticas -> JSON'**:
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret"
            ],
            "Resource": "arn:aws:secretsmanager:us-east-1:<ID_CONTA>:secret:/prod/wordpress/db-*"
        }
    ]
}
```
* **Política salva com o nome `SecretsManagerWordpressRead`**.

### 3.2. Criando a Role (função):

* Na aba Funções da seção IAM, a opçaõ 'Criar perfil' foi selecionada.
* **Tipo de entidade confiável**: Serviço da AWS.
* **Caso de uso**: EC2.
* **Políticas de permissões**: nesta etapa foi selecionada a política `SecretsManagerWordpressRead`, criada anteriormente.
* **Nome definido**: `EC2SecretsWordpressRole`.












# grupo de segurança rds e efs criados com a ec2 e seu grupo
* **Grupo de Segurança (sg-rds-wordpress)**: configurado para permitir conexões apenas das instâncias EC2.
  * **Porta liberada**: 3306 (MySQL).
  * **Origem**: Security Group das instâncias EC2.

    * **Grupo de Segurança (sg-efs-wordpress)**: configurado para permitir conexões apenas das instâncias EC2.

  * Porta liberada: 2049 (NFS).
  * Origem: Security Group das instâncias EC2.









