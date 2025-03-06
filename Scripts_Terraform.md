
# Instruções de Uso do Terraform

## Pré-requisitos

Antes de rodar o Terraform, certifique-se de que você possui:

1.  **Conta na AWS** com credenciais configuradas.
2.  **Terraform instalado** no seu ambiente. Se não tiver, instale com:
    
    `sudo apt-get update && sudo apt-get install -y terraform`
    
    (No Windows, utilize o Chocolatey: `choco install terraform`)
3.  **AWS CLI instalada e configurada**. Configure as credenciais executando:
    
    `aws configure`
    
    **Dica:** Você pode verificar se a configuração está correta rodando:

	`aws sts get-caller-identity`

## Passos para Executar o Terraform

### 1 - Inicializar o Terraform

Navegue até a pasta onde está o código do Terraform e execute:

    terraform init
    
Isso irá baixar os provedores necessários.

---

### 2 - Validar o código

Antes de aplicar a configuração, verifique se há erros na sintaxe:

    terraform validate
---

### 3 - Visualizar o plano de execução

Veja o que será criado sem aplicar as mudanças:

    terraform plan

Isso ajudará a evitar surpresas antes da criação dos recursos.

----------

### 4 - Aplicar a configuração

Crie a infraestrutura na AWS com o comando:

    terraform apply

Será solicitado um **confirmar** digitando **yes**.

----------

### 5 - Acessar a instância EC2

Após a criação, pegue o IP público da máquina EC2 com:

    terraform output ec2_public_ip

Conecte-se via SSH (substitua `<IP_PUBLICO>` pelo IP retornado):

    ssh -i private_key.pem admin@<IP_PUBLICO>

**(Lembre-se de salvar a chave privada para acessar a instância!)**

----------

### 6 - Testar o Nginx

Abra o navegador e acesse:

    http://<IP_PUBLICO>


Você deverá ver a página com a mensagem **"Servidor Nginx Configurado com Sucesso"**.

----------

### 7 - Como Remover a Infraestrutura

Se quiser excluir tudo o que foi criado, execute:

    terraform destroy

Novamente, será necessário confirmar digitando **yes**