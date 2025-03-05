# Infraestrutura como Código (IaC) utilizando Terraform

## Análise técnica:
Abaixo está descrito uma análise técnica do **main.trf** com o máximo de detalhamento possível para a implementação de uma Infraestrutura como Código (IaC) utilizando o Terraform.

---
### 1 - Provedor
Primeiramente iniciamos a configuração definindo o provedor que iremos utilizar, que será o AWS, conforme demonstrado  o código abaixo:

    provider "aws" { 
    region = "us-east-1" 
    }

Nessa configuração será definido em qual região que iremos trabalhar, que será o **Norte da Virgínia (us-east-1)**. A região definida pode ser alterada conforme a necessidade de ser um local em que o valor seja mais atrativo ou em que a latência da rede seja menor.

---
### 2 - Variáveis
Após a configuração do provedor iremos realizar a declaração das variáveis de entrada para que possam ser utilizadas nos recursos seguintes para a configuração de nossa infraestrutura, tornando o código mais modular e reutilizáveis para a infraestrutura mais eficiente: 

       variable "projeto" {
      description = "Nome do projeto"
      type        = string
      default     = "VExpenses"
    }
    
    variable "candidato" {
      description = "Nome do candidato"
      type        = string
      default     = "SeuNome"
    }

No momento estamos utilizando apenas duas variáveis sendo uma **projeto**  e a outra **candidato** e ambas são do mesmo tipo **String**


---
### 3.1 - Recurso Chave SSH
Agora será feito a configuração dos recursos do projeto e é aonde temos as partes mais importantes de todo o projeto por representar os objetos de infraestrutura, que seriam as configurações feitas na infraestrutura sendo desde a criação de algum recurso ou a alteração de algo já existente.

Como o nosso projeto está usando módulos iremos criar um recurso para **tls_private_key** e **aws_key_pair**, aonde nos ajudará a criar uma chave privada TLS para que seja possível conectar com a máquina virtual criada posteriormente no AWS.

    resource "tls_private_key" "ec2_key" {
      algorithm = "RSA"
      rsa_bits  = 2048
    }
	resource "aws_key_pair" "ec2_key_pair" { 
	key_name = "${var.projeto}-${var.candidato}-key"
	public_key = tls_private_key.ec2_key.public_key_openssh 
	}
	
Nossa chave irá receber o nome através da concatenação das variáveis do projeto e do candidato e além do algoritmo RSA, você pode usar outros algoritmos como ECDSA P384 e ED25519 para gerar a chave.

---
### 3.2 - Recurso VPC
A arquitetura de rede de qualquer serviço baseado em nuvem é necessário que se seja baseada em uma nuvem privada virtual (VPC) e então será criado o recurso **aws_vpc** para o VPC.

    resource "aws_vpc" "main_vpc" {
      cidr_block           = "10.0.0.0/16"
      enable_dns_support   = true
      enable_dns_hostnames = true
    
      tags = {
        Name = "${var.projeto}-${var.candidato}-vpc"
      }
    }

Começaremos com a especificação do intervalo de endereços IP em um VPC com o bloco CIDR fornecido e também habilitaremos o suporte de DNS e HOSTNAMES e por fim criaremos uma tag Name para o VPC.

---
### 3.3 - Recurso Sub-rede e acesso a Internet
Após a criação do VPC iremos realizar a configuração da sub-rede empregando o recurso **aws_subnet**

    resource "aws_subnet" "main_subnet" {
      vpc_id            = aws_vpc.main_vpc.id
      cidr_block        = "10.0.1.0/24"
      availability_zone = "us-east-1a"
    
      tags = {
        Name = "${var.projeto}-${var.candidato}-subnet"
      }
    }
Nesse recurso estamos referenciando a VPC criada anteriormente pelo **vpc_id** para indicarmos que a sub-rede será criada dentro da VPC e estamos definindo o intervalo de endereços de IP para a sub-rede que será a **10.0.1.0/24** e toda essa sub-rede estará disponível na mesma região do provedor **us-east-1a**.

Após a definição da sub-rede iremos disponibilizar a nossa VPC para conectar à internet utilizando o recurso **aws_internet_gateway** 

    resource "aws_internet_gateway" "main_igw" {
      vpc_id = aws_vpc.main_vpc.id
    
      tags = {
        Name = "${var.projeto}-${var.candidato}-igw"
      }
    }
Assim como a sub-rede, temos que vincular o Getway com a nossa VPC utilizando o **vpc_id** e então isso fará com que o VPC se comunique com a internet e para esse recurso também teremos a tag Name para definir um nome dinâmico atribuído pelas variáveis.

---
### 3.4 - Recurso Tabela de Rotas
Nesse recurso iremos definir a tabela de roteamento para que o VPC possa encaminhar os pacotes devidamente e evitando a perda trafego e para isso definiremos o recurso **aws_route_table**

    resource "aws_route_table" "main_route_table" {
      vpc_id = aws_vpc.main_vpc.id
    
      route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.main_igw.id
      }
    
      tags = {
        Name = "${var.projeto}-${var.candidato}-route_table"
      }
    }
Nessa tabela de roteamento estamos referenciando nossa VPC pelo **vpc_id** e após essa referência iremos definir as regras de roteamento no bloco **route** aonde o **cidr_block** indica qual é a rota que será aplicada, mas como estamos utilizando o **0.0.0.0/0** estamos indicando que a rota será direcionada para qualquer IP na internet e também teremos a tag Name para definir um nome dinâmico atribuído pelas variáveis.

---
### 3.5 - Recurso Associação de Sub-rede e Tabela de Rotas
Como já realizamos a criação de uma sub-rede e a tabela de rotas, agora será necessário realizar a associação de ambas configurações dentro da VPC e para isso teremos que definir o recurso **aws_route_table_association**

    resource "aws_route_table_association" "main_association" {
      subnet_id      = aws_subnet.main_subnet.id
      route_table_id = aws_route_table.main_route_table.id
    
      tags = {
        Name = "${var.projeto}-${var.candidato}-route_table_association"
      }
    }

Para este recurso temos que referenciar a sub-rede já criada pelo **subnet_id** e a tabela de rotas pelo **route_table_id** e então teremos a associação dos recursos dentro da VPC e por fim teremos a tag Name para para definir um nome dinâmico atribuído pelas variáveis.

---
### 3.6 - Recurso Grupo de Segurança
Iremos definir um recurso para que seja possível criar e gerenciar os grupos de segurança na AWS e para isso utilizaremos o recurso **aws_security_group**. Esses grupos serão configurado para permitir o trafego de entrada via SSH e também o trafego de saída.

    resource "aws_security_group" "main_sg" {
      name        = "${var.projeto}-${var.candidato}-sg"
      description = "Permitir SSH de qualquer lugar e todo o tráfego de saída"
      vpc_id      = aws_vpc.main_vpc.id
    
      # Regras de entrada
      ingress {
        description      = "Allow SSH from anywhere"
        from_port        = 22
        to_port          = 22
        protocol         = "tcp"
        cidr_blocks      = ["0.0.0.0/0"]
        ipv6_cidr_blocks = ["::/0"]
      }
    
      # Regras de saída
      egress {
        description      = "Allow all outbound traffic"
        from_port        = 0
        to_port          = 0
        protocol         = "-1"
        cidr_blocks      = ["0.0.0.0/0"]
        ipv6_cidr_blocks = ["::/0"]
      }
    
      tags = {
        Name = "${var.projeto}-${var.candidato}-sg"
      }
    }

Definiremos o nome através das variáveis declaradas **projeto** e **candidato** e também faremos a referência de qual VPC que iremos disponibilizar o grupo de segurança pelo **vpc_id** e depois definiremos os parâmetros das regras de entrada
-   **description** -> Fornece uma explicação clara sobre o que a regra faz.
-   **from_port** -> Permite tráfego de partida apenas na porta 22.
-   **to_port** -> E como a porta de destino é a mesma que a porta de partida (22), significa que a regra se aplica apenas à porta 22.
-   **protocol** -> O protocolo especificado é o TCP, que é o protocolo utilizado pelo SSH.
-   **cidr_blocks** -> Como está definida com o IP **0.0.0.0/0**,  a regra permite conexões de qualquer pessoa da internet pode tentar fazer conexões SSH na instância.
-   **ipv6_cidr_blocks** -> Assim como na regra acima o IPV6 foi definido **(::/0)** para que qualquer dispositivo com um endereço IPv6 também pode tentar fazer uma conexão SSH.

Por fim, esse recurso também recebe a tag Name, como uma forma de organização, e para definir um nome dinâmico atribuído pelas variáveis.

---
### 3.7 - Recurso de imagem Debian (AMI)
Agora iremos definir um recurso para realizar a busca de uma imagem que é o recurso **data** e que colocaremos algumas condições. Esse recurso é diferente dos demais, pois não irá criar algo e sim buscar, no Terraform, e para isso utilizaremos o **aws_ami** denominado como **debian12**.

    data "aws_ami" "debian12" {
      most_recent = true
    
      filter {
        name   = "name"
        values = ["debian-12-amd64-*"]
      }
    
      filter {
        name   = "virtualization-type"
        values = ["hvm"]
      }
    
      owners = ["679593333241"]
    }
Nas busca da imagem foi definido através do parâmetro **most_recent** que seja trazido a imagem mais recente e os filtros para essa busca foi definido que no primeiro filtro o **values** seja uma imagem que tenha o nome comece com **debian-12-amd64** e que possa ter qualquer informação após o **asterisco (*)**. Já no segundo filtro foi definido que será um buscado um tipo de virtualização parametrizado pelo **name** e que seja uma virtualização **HVM (Hardware Virtual Machine)** para que nossa imagem seja compatível os novas instâncias geradas na AWS e por fim foi definido o parametro para restringir a consulta para imagens publicadas pelo proprietários de ID 679593333241 **(owners = ["679593333241"])**.

---
### 3.8 - Recurso Instância EC2
Com os recursos necessários já definidos agora iremos utilizar o recurso **aws_instance** para realizar a criação de uma nova instância denominada como **debian_ec2** no AWS.

    resource "aws_instance" "debian_ec2" {
      ami             = data.aws_ami.debian12.id
      instance_type   = "t2.micro"
      subnet_id       = aws_subnet.main_subnet.id
      key_name        = aws_key_pair.ec2_key_pair.key_name
      security_groups = [aws_security_group.main_sg.name]
    
      associate_public_ip_address = true
    
      root_block_device {
        volume_size           = 20
        volume_type           = "gp2"
        delete_on_termination = true
      }
    
      user_data = <<-EOF
                  #!/bin/bash
                  apt-get update -y
                  apt-get upgrade -y
                  EOF
    
      tags = {
        Name = "${var.projeto}-${var.candidato}-ec2"
      }
    }

Através do parâmetro **ami** é feito a referência da imagem a ser utilizada e que foi definida no recurso anterior e o tipo da instância a ser criada será o **t2.micro** definido no parâmetro **instance_type**. A sub-rede dessa instância será utilizada a que foi definida no recurso **aws_subnet** sendo referenciada pelo parâmetro **subnet_id**. A chave SSH a ser utilizada pela instância será a que foi criada pelo recurso **aws_key_pair** denominado **ec2_key_pair** e a última associação utilizada de um recurso já criado é o grupo de segurança criado pelo recurso **aws_security_group**, denomidado **main_sg**.

No parâmetro **associate_public_ip_address** está sendo ativado como **true**, para que a instância seja associada a um IP público e atribuído automaticamente e facilitando o acesso via internet.

No bloco de parâmetros **root_block_device** iremos definir algumas características, como o tipo de volume de armazenamento **(volume_type)** que terá o tipo gp2 que faz referência ao SSD e o tamanho desse volume **(volume_size)** que terá 20 GB e com último parâmetro **delete_on_termination** aonde definimos que a instância será deletada automaticamente ao ser terminada, evitando cobranças excessivas e desperdiço. 

No bloco de parâmetros **user_data** e fornecido um script para ser executado automaticamente quando a instância for inicializada e esse script será executado com privilégios de administrador:
**apt-get update -y**: Atualiza a lista de pacotes disponíveis.
**apt-get upgrade -y**: Atualiza todos os pacotes instalados para a versão mais recente disponível.


---
### 4 - Output
Para que durante o processo de execução do Terraform seja exibindo algumas informações necessário foi criado os recursos de **output** sendo o **private_key** e o **ec2_public_ip**

    output "private_key" {
      description = "Chave privada para acessar a instância EC2"
      value       = tls_private_key.ec2_key.private_key_pem
      sensitive   = true
    }
    
    output "ec2_public_ip" {
      description = "Endereço IP público da instância EC2"
      value       = aws_instance.debian_ec2.public_ip
    }
