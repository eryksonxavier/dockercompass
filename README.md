# Project Compass - AWS with Docker - PB NOV 2024 | DevSecOps

Este projeto apresenta uma implementação prática de alta disponibilidade e escalabilidade utilizando AWS. Nele, configuramos um ambiente WordPress com balanceamento de carga, escalabilidade automática e armazenamento persistente. A infraestrutura inclui instâncias EC2 em múltiplas zonas de disponibilidade (AZs), um Application Load Balancer (ALB), um Auto Scaling Group (ASG), um banco de dados gerenciado no Amazon RDS e armazenamento compartilhado via Amazon EFS. O objetivo é garantir uma aplicação resiliente e eficiente, explorando boas práticas de arquitetura na nuvem.

## Arquitetura 

![Image](https://github.com/user-attachments/assets/e744e105-76f4-426a-b045-8a3d378a1f84)

### Pré-requisitos
- **Conhecimentos em Docker**
- **Noções básicas em Linux**
- **Conta na AWS com permissões:**  
  - VPC 
  - NAT Gateway
  - EFS
  - RDS
  - EC2
  - Load Balancer
  - Auto Scaling Group


## Índice

 1. [Configuração da VPC](#1-configuração-da-vpc)
 2. [Configuração dos Security Groups](#2-configuração-dos-security-groups-grupos-de-segurança)
 3. [Configuração do File System](#3-configuração-do-file-system-sistema-de-arquivos)
 4. [Configuração do RDS](#4-configuração-do-rds)
 5. [Configuração do Load Balancer](#5-configuração-do-load-balancer-balanceador-de-carga)
 6. [Configuração do template EC2](#6-configuração-do-template-ec2-modelo-de-execução-dentro-do-auto-scaling-group)
 7. [Configuração do Auto Scaling Group](#7-configuração-do-auto-scaling-group)
 8. [Testes e Validação](#8-testes-e-validação)

## 1. Configuração da VPC
#### 1.1 Pesquise por VPC, nas opções à esquerda clique em **Your VPCs**.

1. Create VPC.
2. Selecione : `VPC e muito mais`.
3. Escolha uma name tag para sua vpc: `wp-docker`.
4. IPV4 CIDR block: `10.0.0.0/16`.
5. Número de zonas disponíveis: `2`.
6. Número de subredes públicas: `2`.
7. Número de subredes privadas: `2`.
8. Customize os CIDR Blocks das subredes: `x.x.x.x/24`.
9. NAT gateways: `Nenhuma`.
10. Criar VPC.

#### 1.2 Nas opções à esquerda, clique em **NAT gateways**.
1. Create NAT gateway.
2. Escolha um nome para sua NAT gateway: `wp-natgateway`.
3. Selecione uma subrede pública que pertence a sua VPC.
4. Clique em `Alocar IP elástico` para associar à um IP elástico.
5. Criar NAT gateway.

Aguarde o estado do seu NAT gateway ficar como **Available** para prosseguir. 

#### 1.3 Associe seu NAT gateway nas tabelas de rotas. 

Nas opções à esquerda, clique em **Tabela de Rotas**. Identifique as duas subredes privadas criadas pela sua VPC. Clique no ID de cada um vá em Editar rotas.

1. Adicionar rota
2. Destination: `0.0.0.0/0`
3. Target: `NAT Gateway` e o id da sua nat gateway

![Image](https://github.com/user-attachments/assets/2e2abe5d-41e2-4571-a63a-7fc570588e98)

## 2. Configuração dos Security Groups (Grupos de Segurança)

Navegue até EC2 e procure por **Grupos de Segurança**. Serão criados 4 grupos de segurança no total, para o **EC2**, **Load Balancer**, **EFS** e **RDS**.

#### 2.1 Grupo de Segurança EC2: SG-EC2

* Regras de Entrada

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| MySQL/Aurora  | 3306          | SG-RDS        |
| NFS           | 2049          | SG-EFS        |
| HTTP          | 80            | SG-LB         |
    
* Regras de Saída

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| Todo o tráfego   | All           | 0.0.0.0/0     |

#### 2.2 Grupo de Segurança Load Balancer: SG-LB

* Regras de Entrada

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| HTTP          | 80            | 0.0.0.0/0     |

* Regras de Saída

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| HTTP          | 80            | SG-EC2        |
    
#### 2.3 Grupo de Segurança EFS: SG-EFS

* Regras de Entrada

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| NFS           | 2049          | SG-EC2        |

* Regras de Saída

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| Todo o tráfego   | All           | 0.0.0.0/0     |

#### 2.4 Grupo de Segurança RDS: SG-RDS

* Regras de Entrada

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| MySQL/Aurora  | 3306          | SG-EC2        |

* Regras de Saída

| Type          | Port Range    | Source        |
| ------------- |:-------------:|:-------------:|
| MySQL/Aurora  | 3306          | 0.0.0.0/0     |


## 3. Configuração do File System (Sistema de Arquivos)
#### 3.1 Navegue até EFS e clique em **Create file system (Criar sistema de arquivos)**.

1. Customize
2. Escolha um nome para seu EFS: `wp-efs`.
3. Deixe como `Regional`.
4. Em **Network**, selecione sua VPC.
5. Em **Mount targets**, em cada zona de disponibilidade, selecione o `SG-EFS` no **Security groups**.
6. Create.

![Image](https://github.com/user-attachments/assets/a4558772-9e94-4f6d-8388-65255697b358)

## 4. Configuração do RDS
#### 4.1 Navegue até RDS, nas opções à esquerda clique em Databases e Create database.

1. `Standart create (Criação Padrão)`.
2. Selecione o `MySQL`.
3. Escolha uma versão compatível com WordPress: `MySQL 8.0.39`.
4.  **Templates**: `Free tier (Nível Gratuito)`.
5.  Escolha um identificador para o banco de dados.
6.  Escolha uma senha para o banco de dados.
7.  **Instance configuration (Configuração da Instância)**: `db.t3.micro`
8.  **Public Access (Acesso ao Público)**: `No`
9.  Selecione sua **VPC**.
10.  Selecione o grupo de segurança: `SG-RDS`
11.  **Availability Zone (Zona de Disponibilidade)**: `No preference`
12.  **Additional configuratioal (Configuração Adicional)**, escolha um nome para seu banco de dados.
13.  Criar banco de dados.

![Image](https://github.com/user-attachments/assets/30154f82-017f-4a92-be3b-8c5bc428d7c1)

## 5. Configuração do Load Balancer (Balanceador de Carga)
#### 5.1 Vá em EC2, nas opções à esquerda, em **Target Groups (Grupos de Destino)** clique em **Create target group (Criar grupos de destino)**.

1. Tipo: `Instances (Instâncias)`
2. **Protocol:Port** como `HTTP:80`
3. Selecione sua **VPC**.
4. Next (Próximo) e **Create target group (Criar Grupo de Destino)**

![Image](https://github.com/user-attachments/assets/38028bc2-2524-47b1-9d04-4fc29c85ce66)

#### 5.2 Agora vá em **Load Balancers** e **Create load balancer (Criar Load Balancer)**.

1. **Application Load Balancer** clique em **Create (Criar)**.
2. Escolha um nome do load balancer: `wp-alb`.
3. Escolher a opção `Internet-facing (Voltado para Internet)` e IPV4.
4. Escolha sua **VPC**.
5. Selecione as **2** zonas de disponibilidade e suas subredes públicas.
6. Selecione o grupo de segurança: SG-LB
7. Em **Listeners and routing (Listeners e roteamento)** selecione o grupo de destino que foi criado.
8. Criar load balancer.

![Image](https://github.com/user-attachments/assets/09f7a2e2-ca68-48ef-b60b-601d4b8c2e47)

## 6. Configuração do template EC2 (Modelo de Execução) dentro do Auto Scaling Group
#### 6.1Navegue até EC2, nas opções à esquerda, vá em Auto Scaling Group e clique em Create Auto Scaling Group.

1. Em Template (Modelo de Execução) clique em: `Criar um Template (Modelo de Execução)`.
2. Escolha um nome para o template (Modelo de Execução): `wp-dockerAMI`.
3. Escolha uma AMI: `Amazon Linux 2 Kernel`.
4. Tipo de instância: `t2.micro`.
5. Selecione/crie um par de chaves.
6. Não inclua uma subrede.
7. Selecione o grupo de segurança: `SG-EC2`
8. Adicione as tags de instância e volume (TAGS da Compass).
9. Em **Advanced details (Detalhes Avançados)**, vá até o **User data (Dados do Usuário)** e insira o arquivo user_data.sh .
10. Criar launch template (Modelo de Execução).

```bash
#!/bin/bash

# Variáveis
EFS_VOLUME="/mnt/efs"
WORDPRESS_VOLUME="/var/www/html"
DATABASE_HOST="wp-database.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com"
DATABASE_USER="admin"
DATABASE_PASSWORD="12345678"
DATABASE_NAME="wpdatabase"

# Atualização do sistema
sudo yum update -y

# Instalação do Docker e do utilitário EFS
sudo yum install docker -y
sudo yum install amazon-efs-utils -y

# Adição do usuário ao grupo Docker
sudo usermod -aG docker $(whoami)

# Inicialização e ativação do serviço Docker
sudo systemctl start docker
sudo systemctl enable docker

# Criação do ponto de montagem EFS
sudo mkdir -p $EFS_VOLUME

# Montagem do volume EFS
if ! mountpoint -q $EFS_VOLUME; then
  echo "Montando volume EFS..."
  sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-xxxxxxxxxxxxxxxxx.efs.us-east-1.amazonaws.com:/ $EFS_VOLUME
else
  echo "Volume EFS já montado."
fi

# Ajuste de permissões do EFS
sudo chown -R 33:33 $EFS_VOLUME   # Usuário do Apache/Nginx no container
sudo chmod -R 775 $EFS_VOLUME

# Download do Docker Compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /bin/docker-compose
chmod +x /bin/docker-compose

# Criação do arquivo docker-compose.yaml
cat <<EOL > /home/ec2-user/docker-compose.yaml
version: '3.8'
services:
  wordpress:
    image: wordpress:latest
    volumes:
      - $EFS_VOLUME$WORDPRESS_VOLUME:/$WORDPRESS_VOLUME
    ports:
      - 80:80
    environment:
      WORDPRESS_DB_HOST: $DATABASE_HOST
      WORDPRESS_DB_USER: $DATABASE_USER
      WORDPRESS_DB_PASSWORD: $DATABASE_PASSWORD
      WORDPRESS_DB_NAME: $DATABASE_NAME
EOL

# Inicialização do serviço WordPress
docker-compose -f /home/ec2-user/docker-compose.yaml up -d
```
#### Nota
Faça as seguintes substituições:

1. O DATABASE_NAME=`wpdatabase` pelo nome do seu database (Banco de Dados RDS)
2. O DATABASE_PASSWORD=`12345678` pela senha do seu banco de dados
3. O `fs-xxxxxxxxxxxxxxxxx.efs.us-east-1.amazonaws.com` pelo DNS do seu **EFS**
4. O `wp-database.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com` pelo endpoint de seu **RDS**.

## 7. Configuração do Auto Scaling Group
#### 7.1 Nas opções à esquerda, vá em **Auto Scaling Groups** e clique em **Create Auto Scaling group**.

1. Escolha um nome para seu **ASG**: `wp-asg`.
2. Escolha o Template (Modelo de Execução) que você criou anteriormente.
3. Selecione sua **VPC**.
4. Escolha as **2** subredes privadas da sua **VPC**.
5. **Attach to an existing load balancer (Anexar a um balanceador de carga existente)**.
6. Vincule o grupo de destino criado.
7. Habilite o health cheks ELB.
8. Capacidade desejada: `2`
9. Capacidade mínima: `2`
10. Capacidade máxima: `4`
11. **Target tracking scaling policy**
12. **Target tracking scaling policy** insira um valor desejado: `2`
13. **Criar Auto Scaling group (Grupo de Auto Scaling)**.

## 8. Testes e Validação
#### 8.1 Aguarde as instâncias serem inicializadas.

![Image](https://github.com/user-attachments/assets/aca4fd0e-3f19-4f5b-973e-3beddb6c2e5a)

#### 8.2 Em seguida vá para o Load Balancer e verifique se o estado do Load Balancer está como Ativo.

![Image](https://github.com/user-attachments/assets/15219359-5dfb-4f2f-89f1-6941f81e7187)

#### 8.2 Copie o DNS name do Load Balancer gerado e cole no navegador.

#### 8.3 Configure uma conta de login e instale o wordpress.
![Image](https://github.com/user-attachments/assets/0e9aa816-f206-4f92-96ed-417ac29b9d57)

![Image](https://github.com/user-attachments/assets/6e81eb89-32e6-4d5f-b37a-b1007332a69f)

![Image](https://github.com/user-attachments/assets/bc8a9d9c-7e2c-4fa1-a92d-85ee46020633)

## Conclusão
Com a conclusão deste projeto, foi possível configurar um ambiente WordPress altamente disponível na AWS, utilizando Load Balancer, Auto Scaling, RDS e EFS para garantir persistência de dados e escalabilidade. A implementação permite que novas instâncias EC2 sejam criadas automaticamente conforme a demanda, garantindo um ambiente robusto e sem pontos únicos de falha. Seguindo essa documentação, você será capaz de replicar essa infraestrutura para hospedar aplicações web escaláveis e resilientes na nuvem.

## Créditos
Projeto desenvolvido como parte do Project Compass - AWS with Docker - PB NOV 2024 | DevSecOps


