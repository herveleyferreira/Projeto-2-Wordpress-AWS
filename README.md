# Implantação de Ambiente WordPress na AWS EC2

## Sobre o Projeto

Este projeto visa a criação de um ambiente WordPress hospedado na AWS, com foco em escalabilidade, segurança e alta disponibilidade. A infraestrutura foi planejada para suportar um tráfego crescente e garantir a estabilidade do sistema. Os principais componentes da arquitetura incluem:

- **Criação de VPC (Virtual Private Cloud)** para isolar a rede.
- **RDS (Relational Database Service)** com MySQL como banco de dados gerenciado.
- **EFS (Elastic File System)** para armazenamento compartilhado entre as instâncias.
- **Instâncias EC2 (Elastic Compute Cloud)** para hospedar o WordPress.
- **Balanceador de carga (Elastic Load Balancer)** para distribuir o tráfego entre as instâncias.
- **Dimensionamento automático (Auto Scaling)** para ajustar a capacidade das instâncias de acordo com a demanda.
- **Host Bastion** para acesso seguro às instâncias internas da rede.

## Arquitetura do Projeto

![Captura de tela 2025-01-21 132043](https://github.com/user-attachments/assets/6ab2dca6-5746-4fc9-8b99-52c2092601e0)

### Tecnologias Utilizadas
- **Console AWS**: Interface gráfica para gerenciamento dos recursos da AWS.
- **Shell Script**: Automação de tarefas e configurações na infraestrutura.
- **Linux**: Sistema operacional utilizado nas instâncias EC2.
- **Docker**: Contêinerização do WordPress para facilitar o gerenciamento e a escalabilidade.

### Objetivo
O objetivo principal deste projeto é criar um ambiente WordPress robusto e seguro, utilizando os serviços da AWS para garantir a performance e confiabilidade, seguindo as melhores práticas para infraestrutura escalável e segura.

---

## 1. Criação da VPC

A primeira etapa do projeto consistiu na criação de uma nova VPC (Virtual Private Cloud) para organizar os recursos de forma eficiente e segura dentro da AWS.

### Parâmetros Utilizados:
- **Bloco CIDR IPv4:** `10.0.0.0/16`
- **Zonas de Disponibilidade (AZs):** 2 zonas para garantir alta disponibilidade.
- **Sub-redes:** 2 sub-redes públicas e 2 sub-redes privadas para segregação de recursos.
- **NAT Gateway:** 1 NAT Gateway por Zona de Disponibilidade, permitindo a comunicação das instâncias privadas com a internet.

### Sobre o NAT Gateway
O **NAT Gateway** é uma solução que permite que instâncias localizadas em sub-redes privadas tenham acesso à internet para atualizações e outras operações necessárias, sem expô-las diretamente à rede pública. Ele garante a segurança, mantendo a comunicação unidirecional entre as instâncias privadas e a internet.

## 2. Configuração de Grupos de Segurança

Os **Grupos de Segurança** são responsáveis por definir as regras de entrada e saída para os recursos da AWS, proporcionando controle detalhado sobre o tráfego de rede.

### Grupos Criados:
- **Balanceador de Carga (GS-Load-Balancer):** 
  - Permitir tráfego de **HTTP** e **HTTPS** de qualquer endereço IPv4 (Anywhere IPv4).
  
- **Instâncias EC2 (GS-EC2):** 
  - **HTTP:** Redirecionar o tráfego para o Load Balancer.
  - **SSH:** Permitir acesso de qualquer IP (Anywhere IPv4).
  - **HTTPS:** Redirecionar o tráfego para o Load Balancer.
  
- **RDS (GS-RDS):** 
  - Permite tráfego **MySQL/Aurora** apenas da origem **MyGroup-ec2**.

- **EFS (GS-EFS):** 
  - Permitir tráfego **NFS** apenas da origem **MyGroup-ec2**.

## 3. Criação do RDS

O **RDS (Relational Database Service)** facilita a configuração, gerenciamento e escalabilidade de bancos de dados, sendo uma opção ideal para ambientes gerenciados.

### Configuração:
- **Tipo de Banco de Dados:** MySQL (Nível gratuito)
- **Identificador da Instância:** `wordpress-db`
- **Usuário Principal:** `admin`
- **Instância:** `db.t3.micro`
- **Backup e Criptografia:** Desativados (para ambiente de teste)
- **VPC:** VPC criada na etapa anterior
- **Grupo de Sub-redes:** Criado conforme a arquitetura
- **Acesso Público:** Desativado
- **Grupo de Segurança:** `GS-RDS`
- **Nome do Banco de Dados:** `wordpress`

Após a configuração do RDS, é necessário copiar o **IP** da instância do RDS e adicioná-lo ao arquivo `user_data.sh`, no parâmetro `WORDPRESS_DB_HOST`,na seção do Docker Compose. Além disso, preencha as variáveis `WORDPRESS_DB_USER`, `WORDPRESS_DB_PASSWORD` e `WORDPRESS_DB_NAME` com as informações configuradas. O nome do banco de dados é wordpress por padrão.

## 4. Criação do EFS

O **Amazon Elastic File System (EFS)** oferece uma solução de armazenamento escalável e de alta disponibilidade, permitindo o compartilhamento de arquivos entre várias instâncias EC2.

### Configuração:
- **Nome:** `EFS-projeto-wordpress`
- **VPC:** Selecionar a VPC criada anteriormente
- **Sub-redes Privadas:** Sub-rede privada 1 e Sub-rede privada 2
- **Grupo de Segurança:** `SG-EFS`

No script de inicialização `user_data.sh`, adicione os seguintes comandos para montar o EFS automaticamente nas instâncias EC2:

```sh
# Criar diretório para EFS
sudo mkdir -p /mnt/efs  

# Montar o EFS
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport <id-amazon>:/ /mnt/efs
```
## 5.Criação das Instâncias EC2

Para este projeto, criamos **duas instâncias EC2** com as seguintes configurações.

### Configurações principais:
- **Nome e tags**: Atribua o nome e as tags de acordo com o padrão estabelecido pela equipe.
- **Sistema operacional**: Utilizamos o **Ubuntu** como sistema operacional para as instâncias.
- **Tipo de instância**: O tipo de instância foi mantido como o padrão para atender às necessidades do projeto.
- **Par de chaves**: O par de chaves foi criado ou reutilizado, dependendo da disponibilidade do par existente, para permitir o acesso SSH às instâncias.
- **Sub-redes**:
  - **Instância 1**: Localizada na **Sub-rede Privada 1**.
  - **Instância 2**: Localizada na **Sub-rede Privada 2**.
- **Atribuição de IP público**: A opção de atribuição de IP público foi habilitada para permitir a comunicação externa, caso necessário.
- **Grupo de segurança**: O grupo de segurança `MyGroup-ec2` foi configurado para controlar o tráfego de entrada e saída das instâncias.
- **Configuração avançada (User Data)**: Foi incluído o script `user_data.sh` para garantir que as instâncias sejam inicializadas automaticamente com as configurações e serviços necessários, como o Docker e o WordPress.

Essas instâncias EC2 formam a base de nosso ambiente, permitindo a execução do WordPress e outros serviços essenciais do projeto.

## 6. Bastion Host

O **Bastion Host** foi criado para permitir o acesso às instâncias privadas via SSH de forma segura, utilizando uma instância EC2 em uma **sub-rede pública**.

### Passos para Criação e Acesso ao Bastion Host:

**Criar uma instância EC2** em uma **sub-rede pública**.
**Conectar ao Bastion Host** via SSH:

    ```sh
    ssh -i "sua-chave.pem" ubuntu@<ip-privado>
    ```
**Acessar as instâncias privadas** a partir do Bastion Host:

    ```sh
    ssh -i "sua-chave" ubuntu@<ip-privado>
    ```
**Aplicar permissões na chave**:

    ```sh
    chmod 400 sua-chave.pem
    ```

## Configuração do Load Balancer

O **Load Balancer** é responsável por distribuir o tráfego de forma equilibrada entre as instâncias EC2, garantindo alta disponibilidade e desempenho.

### Configuração:
- **Tipo**: Classic Load Balancer
- **Nome**: `MyLoadBalancer`
- **Sub-redes**: Sub-redes públicas para garantir acessibilidade externa.
- **Grupo de segurança**: `MyGroup-loadbalancer`, para controlar o tráfego de entrada e saída.

### Verificação de integridade:
- **Caminho de ping**: A URL `/wp-admin/install.php` foi configurada como caminho de verificação de integridade. O status esperado é `200`, indicando que a instância está saudável.
- **Adicionar instâncias privadas**: As instâncias EC2 privadas foram adicionadas ao Load Balancer para distribuir o tráfego entre elas.

Após a configuração, o Load Balancer gerará um DNS que pode ser utilizado para acessar a aplicação WordPress de forma balanceada e eficiente.

### Configuração do Auto Scaling

O **Auto Scaling** ajusta automaticamente o número de instâncias EC2 com base na demanda, garantindo que o ambiente seja dimensionado conforme necessário para manter a performance e a disponibilidade.

Para configurar, criamos um **modelo de execução** com o tipo de instância `t2.micro` e configuramos o script `user_data.sh` para garantir que as instâncias iniciem corretamente. As instâncias são distribuídas entre **sub-redes privadas** para maior resiliência.

A configuração também integra o Auto Scaling ao Load Balancer existente, garantindo que as novas instâncias sejam automaticamente registradas e passem a receber tráfego.

Após a configuração, o Auto Scaling cria e elimina instâncias automaticamente conforme a demanda, garantindo escalabilidade e eficiência.

## Conclusão
A configuração foi testada acessando o **DNS do Load Balancer**. A tela inicial do WordPress apareceu corretamente, indicando que o ambiente foi implementado com sucesso.

![Captura de tela 2025-02-04 154151](https://github.com/user-attachments/assets/beac7413-8f10-4bea-9fe0-93948edb49a1)

Este projeto proporcionou uma experiência prática valiosa na implementação de um ambiente WordPress escalável na AWS. Durante o processo, desafios foram enfrentados e superados, contribuindo significativamente para o aprendizado e aprimoramento dessas habilidades.

Com esta infraestrutura, é possível expandir o projeto para novos cenários, garantindo maior desempenho, segurança e confiabilidade.

