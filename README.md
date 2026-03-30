Arquitetura de Alta Disponibilidade para site do WordPress.

<img width="886" height="873" alt="image" src="https://github.com/user-attachments/assets/40e378e8-8e54-4bcf-a00b-9d95a6aec62d" />

Objetivos do projeto:
Criar uma infraestrutura cloud altamente disponível (99,99%);
Garantir baixa latência e carregamento rápido das páginas;
Implementar múltiplas camadas de segurança contra-ataques;
Projetar uma arquitetura escalável para suportar crescimento;
Aplicar boas práticas de arquitetura na AWS;

# 🚀 Arquitetura AWS Enterprise para WordPress (Alta Disponibilidade)

Este guia detalha a implementação de uma infraestrutura de alta disponibilidade, escalável e segura para WordPress na AWS. A arquitetura utiliza boas práticas de nível *Enterprise*, garantindo entrega de conteúdo global com baixa latência, proteção contra ataques e gestão segura de credenciais.

## 🗺️ Fluxo da Arquitetura

**Usuário na Internet**
↓
**Route 53** (Resolução de DNS)
↓
**CloudFront + WAF** (Cache na borda e bloqueio de ataques)
↓ *Se o conteúdo não estiver no cache:*
**Internet Gateway**
↓
**Application Load Balancer (ALB)** (Subnets Públicas)
↓
**Auto Scaling Group (EC2)** (Subnets Privadas)
↓
**ElastiCache Redis** (Cache de objetos em RAM, resposta de 1-2ms)
↓ *Se não estiver no Redis:*
**Amazon EFS** (Arquivos do WordPress compartilhado) & **Amazon RDS Multi-AZ** (Banco de dados)

---

## 🏗️ FASE 1 — Infraestrutura de Rede (VPC)
Antes de provisionar instâncias, é necessário criar a base de rede com foco em alta disponibilidade:
* **VPC:** Criar nova VPC.
* **Subnets Públicas:** 2 subnets (para o ALB).
* **Subnets Privadas:** 2 subnets para EC2 e 2 subnets para o RDS.
* **Gateways:** Internet Gateway (para as subnets públicas) e NAT Gateway (para as subnets privadas terem saída para a internet).

*Nota: As instâncias EC2 devem residir obrigatoriamente em subnets privadas.*

---

## 🗄️ FASE 2 — Banco de Dados (Amazon RDS)
Configuração do banco de dados relacional:
* **Engine:** MySQL 8
* **Multi-AZ:** Habilitado
* **DB Subnet Group:** Associar às 2 subnets privadas de banco de dados.
* **Public access:** NO
* **Security Group:** Liberar a porta `3306` com origem exclusiva do Security Group das instâncias EC2.

> **Importante:** Anote o Endpoint do RDS, Usuário e Senha gerados nesta etapa.

---

## 📂 FASE 3 — Sistema de Arquivos (Amazon EFS)
O EFS garantirá que todas as instâncias EC2 compartilhem os mesmos arquivos do WordPress.
* **Performance:** General Purpose
* **Mount targets:** Criar nas 2 Zonas de Disponibilidade (AZs) privadas.
* **Security Group:** Permitir NFS (porta `2049`) com origem exclusiva do Security Group das instâncias EC2.

> **Importante:** Anote o ID do EFS (Ex.: `fs-12345678`).

---

## 🖥️ FASE 4 — Instância Temporária (Setup Inicial do WordPress)
Para a primeira configuração, sobe-se uma EC2 temporária (Amazon Linux) na subnet privada, associada a uma IAM Role com permissão de SSM.

### 1. Instalar pacotes essenciais
```bash
sudo dnf update -y
sudo dnf install -y httpd php php-cli php-fpm php-mysqlnd php-gd php-curl php-xml php-mbstring php-zip php-intl amazon-efs-utils
```

### 2. Montar EFS
```bash
sudo mkdir -p /var/www/html
sudo mount -t efs fs-12345678:/ /var/www/html
echo "fs-12345678:/ /var/www/html efs defaults,_netdev 0 0" | sudo tee -a /etc/fstab
df -h # Verifica se o EFS está montado corretamente
```

### 3. Instalar WordPress
```bash
cd /var/www/html
sudo wget https://wordpress.org/latest.tar.gz
sudo tar -xzf latest.tar.gz
sudo mv wordpress/* .
sudo rm -rf wordpress latest.tar.gz
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
```

### 4. Configurar wp-config.php
```bash
cd /var/www/html
sudo cp wp-config-sample.php wp-config.php
sudo vim wp-config.php
```
*Edição recomendada:*
```php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'SenhaForte123!');
define('DB_HOST', 'endpoint-rds.amazonaws.com');

define('WP_HOME','https://SEU-DOMINIO.com');
define('WP_SITEURL','https://SEU-DOMINIO.com');

if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
   $_SERVER['HTTPS'] = 'on';
}

# Evitar que o WP-Admin quebre atrás do Load Balancer
define('CONCATENATE_SCRIPTS', false);
```

### 5. Configurar Apache
```bash
sudo vim /etc/httpd/conf/httpd.conf
```
*Altere para `AllowOverride ALL` dentro do bloco `<Directory "/var/www">`.*

### 6. Iniciar Apache e Finalizar
```bash
sudo systemctl enable httpd
sudo systemctl start httpd
```
Após finalizar a instalação do WordPress pelo navegador, os arquivos estarão persistidos no EFS. **A instância temporária pode ser destruída.**

---

## 🧱 FASE 5 — Launch Template (Definitivo)
Configuração do modelo que o Auto Scaling usará para criar novas máquinas.
* **Rede:** Não adicionar IP público.
* **Security Group:** Permitir HTTP/HTTPS (origem: ALB) e NFS (origem: EFS).
* **IAM Role:** `AmazonS3FullAccess`, `AmazonSSMManagedInstanceCore` e `SecretsManagerReadWrite`.

### User Data
O script abaixo **não reinstala** o WordPress, apenas prepara a máquina, monta o EFS e inicia o servidor web.
```bash
#!/bin/bash
dnf update -y
dnf install -y httpd php php-cli php-fpm php-mysqlnd php-gd php-curl php-xml php-mbstring php-zip php-intl amazon-efs-utils

systemctl enable httpd

# Montar EFS existente
mkdir -p /var/www/html
mount -t efs fs-12345678:/ /var/www/html
echo "fs-12345678:/ /var/www/html efs defaults,_netdev 0 0" >> /etc/fstab

chown -R apache:apache /var/www/html
systemctl start httpd
```

---

## 📈 FASE 6 — Auto Scaling Group
* **Launch Template:** Selecionar o criado na Fase 5.
* **VPC e Subnets:** Escolher as 2 subnets privadas.
* **Capacidade:** Desired: 2 | Min: 2 | Max: 4.
* **Integração:** Associar ao Target Group do Application Load Balancer (ALB).

---

## 🌐 FASE 7 — Application Load Balancer (ALB)
* **Target Group:** Instâncias, Protocolo HTTP (porta 80). Health check no path `/`.
* **Load Balancer:** Esquema *Internet-facing*, selecionando as 2 subnets públicas.
* **Listeners:**
    * Porta `80` → Redirecionar para `443`.
    * Porta `443` → Encaminhar para o Target Group (Requer certificado do AWS Certificate Manager - ACM).

---

## 📦 FASE 8 — Armazenamento de Mídia (Amazon S3) - *Opcional*
Para otimizar o EFS e servir imagens mais rapidamente:
* Criar um bucket no Amazon S3.
* Instalar e configurar o plugin **WP Offload Media** no WordPress.

---

## 🔐 FASE 9 — CloudFront (CDN) e Certificado SSL
Garante cache global na borda e protege o ALB.

1. **AWS Certificate Manager (ACM):** Solicitar certificado público na região `us-east-1` (Norte da Virgínia) para `dominio.com` e `*.dominio.com`.
2. **CloudFront Distribution:**
    * **Origin domain:** Selecionar o ALB.
    * **Viewer Protocol Policy:** Redirect HTTP to HTTPS.
    * **Allowed HTTP Methods:** GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE.
    * **Cache Policy:** `CachingOptimized` | **Origin request policy:** `AllViewer`.
    * **WAF:** Habilitar proteções básicas.
    * **Custom SSL Certificate:** Selecionar o certificado ACM gerado.
3. **Behaviors do CloudFront (Evitar cache no admin do WP):**
    * Path `/wp-admin/*` → Cache Policy: `CachingDisabled` → Origin Request Policy: `AllViewer`.
    * Path `/wp-login.php` → Cache Policy: `CachingDisabled` → Origin Request Policy: `AllViewer`.

---

## 🗺️ FASE 10 — Route 53 (DNS)
1. Criar uma **Hosted Zone** pública para o domínio.
2. Criar um registro do tipo **A**.
3. Ativar a opção **Alias**.
4. Apontar (*Route traffic to*) para a distribuição do **CloudFront**.

---

## 💎 Configurações Extras (Enterprise)

### A. AWS Secrets Manager (Ocultar credenciais do RDS)
Remove as senhas em texto plano do `wp-config.php`.

1. **Criar Secret:** Do tipo *Credentials for RDS database* (Nome: `wordpress/rds/production`).
2. **Permissão:** Adicionar policy `secretsmanager:GetSecretValue` na IAM Role da EC2.
3. **Modificar User Data:** Adicionar script para ler o secret via AWS CLI e gerar o arquivo `/etc/wp-db-config.php`.
4. **Atualizar `wp-config.php`:** Fazer um `require_once '/etc/wp-db-config.php';` em vez de declarar as variáveis de banco diretamente.

### B. Amazon ElastiCache (Redis)
Reduz drasticamente as consultas diretas ao banco de dados.

1. **Criar Cluster Redis:** Na mesma VPC, com Security Group permitindo a porta `6379` a partir da EC2.
2. **User Data:** Adicionar a instalação do pacote `php-pecl-redis`.
3. **WordPress:** Instalar o plugin **Redis Object Cache**.
4. **`wp-config.php`:** Adicionar as constantes de conexão:
```php
define('WP_REDIS_HOST', 'seu-endpoint-redis.cache.amazonaws.com');
define('WP_REDIS_PORT', 6379);
define('WP_CACHE', true);
```

---

## 🎯 Resultados Alcançados
* **Performance:** Páginas ultrarrápidas distribuídas globalmente (CloudFront) e queries otimizadas em memória (Redis).
* **Segurança:** Banco de dados isolado em rede privada e credenciais protegidas via Secrets Manager.
* **Escalabilidade:** Arquitetura capaz de escalar horizontalmente e sem *downtime* (Auto Scaling + ALB + EFS + RDS Multi-AZ).
