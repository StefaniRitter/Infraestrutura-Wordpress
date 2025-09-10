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
* **Conexão com EC2**: a opção selecionada foi 'Não se conectar a um recurso de computação do EC2', pois isso será feito mais adiante.
* * **Grupo de Segurança**: um novo grupo de segurança identificado como `sec-group-wordpress` foi criado (regras de entrada e saída padrão foram mantidas neste momento).
* **Parâmetros de acesso**: usuário administrador (`admin`), senha, nome do banco (`wordpress_db`) e endpoint foram gerados no momento da criação. Esses dados serão utilizados posteriormente pelo WordPress para se conectar ao banco.
  
### 2.2. Criação do Sistema de Arquivos (Amazon EFS):
criado para armazenar os arquivos de mídia do WordPress (imagens, vídeos e documentos)

* **Criação do Grupo de Segurança**: antes da criação do EFS, um grupo de segurança chamado `sec-group-efs` foi criado (regras de entrada e saída padrão foram mantidas neste momento).
* **Sistema de Arquivos EFS**: em Amazon EFS -> Sistemas de Arquivos -> Criar -> foi definido o nome `wordpress-efs` e avançado para a próxima etapa.
* **VPC escolhida**: Na etapa de rede, foi selecionado `vpc-wordpress`.
* **Destinos de Montagem**:
  * **Zona de Disponibilidade**: 1-east-1a;   **Tipo de endereço IP**: Somente IPv4;   **Sub-rede**: `subnet-privada01`;   **Grupos de segurança**: `sec-group-efs`.
  * **Zona de Disponibilidade**: 1-east-1b;   **Tipo de endereço IP**: Somente IPv4;   **Sub-rede**: `subnet-privada02`;   **Grupos de segurança**: `sec-group-efs`  
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

### 3.3. Criando a Role (função):

* Na aba Funções da seção IAM, a opçaõ 'Criar perfil' foi selecionada.
* **Tipo de entidade confiável**: Serviço da AWS.
* **Caso de uso**: EC2.
* **Políticas de permissões**: nesta etapa foi selecionada a política `SecretsManagerWordpressRead`, criada anteriormente.
* **Nome definido**: `EC2SecretsWordpressRole`.

## Etapa 4: Criação das Instâncias EC2

### 4.1. Configuração da Instância Bastion Host:

* **Lançamento da Instância EC2**: Uma instância EC2 (distribuição Ubuntu 24.04 LTS) foi lançada na **`subrede-publica01`** da **`vpc-wordpress`** através do console da AWS, para testes iniciais.
* **Tipo de intância**: t2.micro.
* **Par de Chaves SSH**: Durante o processo de lançamento, um par de chaves SSH já existente (`projetoLinux.pem`) foi atribuído à instância para permitir acesso seguro via SSH.
* **Configuração do Grupo de Segurança (`sec-group-bastion`)**: Um novo Grupo de Segurança foi criado e configurado com as seguintes regras de entrada ('Inbound Rules'):
    * **SSH (Porta 22)**: Permitido para o endereço IP `<IP_AUTORIZADO_LOCAL>/32` (opção 'MEU IP').
    * **HTTP (Porta 80)**: Permitido para `0.0.0.0/0` (acesso de qualquer IP na internet).
    * As regras de saída (`Outbound Rules`) foram mantidas como padrão (permitindo todo o tráfego para `0.0.0.0/0`).
* **Associação do Grupo de Segurança à Instância**: O Grupo de Segurança criado (`sec-group-bastion`) foi associado à instância EC2, garantindo que as regras de firewall definidas fossem aplicadas corretamente ao tráfego de rede da instância.

### 4.2. Configuração da Instância Privada:

* **Lançamento da Instância EC2**: Uma instância EC2 (distribuição Ubuntu 24.04 LTS) foi lançada na **`subrede-privada01`** da **`vpc-wordpress`**.
* **Tipo de intância**: t2.micro.
* **Par de Chaves SSH**: Durante o processo de lançamento, o mesmo par de chaves anterior (`projetoLinux.pem`) foi atribuído à instância.
* **Configuração do Grupo de Segurança (`ec2-security-group`)**: Um novo Grupo de Segurança foi criado e configurado com as seguintes regras de entrada (`Inbound Rules`):
    * **SSH (Porta 22)**: Permitido para o Grupo de segurança da instância Bastion Host criado anteriormente (`sec-group-bastion`).
    * **HTTP (Porta 80)**: Permitido para `sec-group-bastion`.
    * As regras de saída (`Outbound Rules`) foram mantidas como padrão (permitindo todo o tráfego para `0.0.0.0/0`).
* **Associação do Grupo de Segurança à Instância**: O Grupo de Segurança criado (`ec2-security-group`) foi associado à instância EC2 privada no momento da criação.
* **Script User-data**: Ainda na etapa de criação da EC2, em detalhes avançados, o seguinte script foi anexado:

```
#!/bin/bash
set -uo pipefail
export DEBIAN_FRONTEND=noninteractive

# ----- Variáveis -----
EFS_ID="<ID_SISTEMA_DE_ARQUIVOS>"
SECRET_ID="/prod/wordpress/db"
WP_HOST_DIR="/efs_data"
WP_UPLOADS_DIR="${WP_HOST_DIR}/wp-content/uploads"

# ----- Instalar dependências -----
apt-get update -y
apt-get install -y docker.io docker-compose git nfs-common jq curl mysql-client python3-pip
systemctl enable --now docker
usermod -aG docker ubuntu || true
pip3 install awscli --break-system-packages

# ----- Obter região da instância -----
for i in $(seq 1 10); do
    TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
        -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    [ -n "$TOKEN" ] && break
    sleep 5
done
AWS_REGION=$(curl -sH "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/dynamic/instance-identity/document | jq -r .region)
AWS_REGION=${AWS_REGION:-us-east-1}

# ----- Obter segredo do Secrets Manager -----
for i in $(seq 1 5); do
    SECRET_JSON=$(aws secretsmanager get-secret-value \
        --region "$AWS_REGION" --secret-id "$SECRET_ID" \
        --query SecretString --output text 2>/dev/null)
    [ -n "$SECRET_JSON" ] && break
    sleep 5
done

[ -n "$SECRET_JSON" ] || exit 1

DB_USER=$(echo "$SECRET_JSON" | jq -r '.username')
DB_PASS=$(echo "$SECRET_JSON" | jq -r '.password')
RDS_ENDPOINT=$(echo "$SECRET_JSON" | jq -r '.host')
DB_NAME=$(echo "$SECRET_JSON" | jq -r '.dbname')

# ----- Configurar EFS -----
mkdir -p "${WP_UPLOADS_DIR}"
for i in $(seq 1 10); do
    mount -t nfs4 -o nfsvers=4.1 "${EFS_ID}.efs.${AWS_REGION}.amazonaws.com":/ "${WP_HOST_DIR}" && break
    sleep 5
done
chown -R 33:33 "${WP_HOST_DIR}"
chmod -R 775 "${WP_HOST_DIR}"
grep -q "${EFS_ID}" /etc/fstab || \
    echo "${EFS_ID}.efs.${AWS_REGION}.amazonaws.com:/ ${WP_HOST_DIR} nfs4 defaults,_netdev 0 0" >> /etc/fstab

# ----- Criar banco de dados -----
for i in $(seq 1 5); do
    mysql --host="${RDS_ENDPOINT}" --user="${DB_USER}" --password="${DB_PASS}" \
        -e "CREATE DATABASE IF NOT EXISTS \`${DB_NAME}\`;" && break
    sleep 5
done

# ----- Criar docker-compose.yml -----
cat > /home/ubuntu/docker-compose.yml <<EOF
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: ${RDS_ENDPOINT}
      WORDPRESS_DB_USER: ${DB_USER}
      WORDPRESS_DB_PASSWORD: ${DB_PASS}
      WORDPRESS_DB_NAME: ${DB_NAME}
    volumes:
      - ${WP_UPLOADS_DIR}:/var/www/html/wp-content/uploads
EOF

# ----- Iniciar container -----
cd /home/ubuntu
docker-compose up -d
```
  
## Etapa 5: Criação/configuração dos demais Grupos de Segurança

### 5.1. Configuração do Grupo de Segurança do EFS (`sec-group-efs`):

*  Para o Grupo de Segurança `sec-group-efs`, criado na etapa de criação do EFS, a seguinte regra de entrada (`Inbound Rules`) foi definida:
  * **Porta liberada**: 2049 (NFS).
  * **Origem**: Grupo de segurança da intância EC2 `ec2-security-group`.
* As regras de saída (`Outbound Rules`) padrão foram mantidas (permitindo todo o tráfego para `0.0.0.0/0`).

### 5.2. Configuração do Grupo de Segurança do RDS (`sec-group-wordpress`):

*  Para o Grupo de Segurança `sec-group-wordpress`, criado na etapa de criação do RDS, a seguinte regra de entrada (`Inbound Rules`) foi definida:
  * **Porta liberada**: 3306 (MYSQL).
  * **Origem**: Grupo de segurança da intância EC2 `ec2-security-group`.
* As regras de saída (`Outbound Rules`) foram mantidas.










