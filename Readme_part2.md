# Infraestrutura como Código (IaC) utilizando Terraform

## Descrição Técnica das Melhorias

### 1 - Variáveis e Recurso Grupo de Segurança
Foi incluído a variável **confiavel_ip** para que seja feito a definições de um IP estático para ser utilizado como parâmetro do **cidr_blocks** no recurso **aws_security_group** e aumentando a segurança 

    variable "confiavel_ip" {
      description = "Endereço IP confiável para acesso SSH"
      type        = string
      default     = "192.168.1.1/32"  #Substitua pelo seu IP
    }

    resource "aws_security_group" "main_sg" {
      name        = "${var.projeto}-${var.candidato}-sg"
      description = "Permitir SSH de IP confiável e tráfego HTTP/HTTPS"
      vpc_id      = aws_vpc.main_vpc.id
    
      # Regras de entrada
      ingress {
        description = "Allow SSH from trusted IP"
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = [var.confiavel_ip] #Alterado para receber o IP por variavel
      }

Além dessas alterações relacionadas ao IP também foi feito duas inclusões nas regras de entrada no recurso ** aws_security_group**, para que seja permitido o tráfego HTTP/HTTPS apenas nas portas 80 (HTTP) e 443(HTTPS) e assim garantindo que o tráfego web não tenha grande exposição ao possíveis ataques.

    ingress {
        description = "Allow HTTP"
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }
      
      ingress {
        description = "Allow HTTPS traffic"
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
      }

---
### 2 - Recurso Chave SSH
No recurso **tls_private_key** o parâmetro **rsa_bits** foi alterado para utilizar 4096 bits em vez de 2048, aumentando a segurança da criptografia.

    resource "tls_private_key" "ec2_key" {
      algorithm = "RSA"
      rsa_bits  = 4096
    }

---
### 4.1 - Recurso Instância EC2

No recurso ** aws_instance** e no bloco do **root_block_device**  foi adicionado o parâmetro **encrypted** fazendo com que o volume definido para a instância seja criptografado em repouso e garantindo que os dados fiquem seguros quando a instância estiver em repouso.

    root_block_device {
        volume_size           = 20
        volume_type           = "gp2"
        encrypted             = true  #Criptografia do volume EBS
        delete_on_termination = true
      }

---
### 4.2 - Recurso Instância EC2
No recurso **aws_instance** e dentro do bloco **user_data** foi realizado o a inclusão do script que irá instalar e inicializar de forma automática o Ngnix no boot da instância.

     user_data = <<-EOF
                  #!/bin/bash
                  set -e
                  apt-get update -y
                  apt-get upgrade -y
                  apt-get install -y nginx
                  systemctl start nginx
                  systemctl enable nginx
                  echo "<h1>Servidor Nginx Configurado com Sucesso</h1>" > /var/www/html/index.html
                  EOF


---
