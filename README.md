# AtividadeCompass

## Obejtivo: realizar teste de carga em um Auto Scaling Group (ASG) na AWS, o numero de instancia EC2 aumenta de 1 para 3 e diminui novamente na medida que o nivel de estress (requisições) diminui.

### Pré-requisitos

- ✅ Conta AWS
- ✅ Acesso ao EC2, Auto Scaling e ELB
- ✅ Conhecimento básico de linha de comando (SSH)

---

## Passo 1: Criar um Classic Load Balancer (CLB)

O Load Balancer distribui o tráfego entre as instâncias do Auto Scaling.

1. No Console AWS, vá para **EC2 > Load Balancers > Create Load Balancer**
2. Escolha **Classic Load Balancer**
3. Configuração básica:
   - **Nome:** `ASG-Test-LB`
   - **VPC/Sub-redes:** Escolha a mesma VPC e sub-redes onde suas instâncias estarão.
4. Configurar listeners:
   - Porta 80 (HTTP) → Mantenha o padrão.
5. Configurar Health Check:
   - **Path:** `/index.html` (para verificar se a instância está saudável).
6. Adicionar Security Group:
   - Libere HTTP (80) e SSH (22) se necessário.
7. Clique em **Create** e aguarde a criação.

---

## Passo 2: Criar um Launch Template (Modelo de Instância)

O Launch Template define qual AMI (imagem da máquina) será usada no Auto Scaling.

1. No Console EC2, vá para **Launch Templates > Create Launch Template**
2. **Nome:** `ASG-Template`
3. **AMI:** Amazon Linux 2 (gratuita)
4. **Instance Type:** `t2.micro` (free tier)
5. **Key Pair:** Selecione ou crie um par de chaves para SSH.
6. **Security Group:** Selecione o mesmo usado no Load Balancer (HTTP + SSH liberados).
7. **User Data (Script de Inicialização):**

    ```bash
    #!/bin/bash
    yum update -y
    yum install -y httpd
    echo "Hello World from $(hostname)" > /var/www/html/index.html
    systemctl start httpd
    systemctl enable httpd
    ```
    > Isso instala um servidor web básico.

8. Clique em **Create Launch Template**.

---

## Passo 3: Criar o Auto Scaling Group (ASG)

O ASG gerencia a escala automática das instâncias.

1. No Console EC2, vá para **Auto Scaling Groups > Create Auto Scaling Group**
2. **Nome:** `ASG-Test-Group`
3. **Launch Template:** Selecione o criado (`ASG-Template`).
4. **Configuração de rede:** Escolha a mesma VPC e sub-redes do Load Balancer.
5. **Configurar Load Balancer:** Selecione o Classic Load Balancer criado (`ASG-Test-LB`).
6. **Definir políticas de escala:**
   - **Tamanho do grupo:**
     - Mínimo: 1
     - Máximo: 3
     - Capacidade desejada: 1
   - **Configurar Scaling Policies:**
     - Métrica: CPU Utilization (ou RequestCount se preferir).
     - Scale Out: Adicionar 1 instância se CPU > 70% por 2 minutos.
     - Scale In: Remover 1 instância se CPU < 30% por 5 minutos.
7. Clique em **Create Auto Scaling Group**.

---

## Passo 4: Criar um Endpoint de Teste para Simular Carga

Vamos adicionar um script que simula processamento pesado para ativar o Auto Scaling.

1. Conecte-se via SSH em uma das instâncias:

    ```bash
    ssh -i "sua-chave.pem" ec2-user@IP-DA-INSTANCIA
    ```

2. Edite o arquivo `/var/www/html/teste`:

    ```bash
    sudo nano /var/www/html/teste
    ```

3. Cole o conteúdo abaixo:

    ```bash
    #!/bin/bash
    echo "Content-type: text/plain"
    echo ""
    echo "Processando requisição em $(hostname)"
    # Simula processamento pesado (1-3 segundos)
    sleep $((RANDOM % 3 + 1))
    ```

4. Dê permissão de execução:

    ```bash
    sudo chmod +x /var/www/html/teste
    ```

5. Reinicie o Apache:

    ```bash
    sudo systemctl restart httpd
    ```

---

## Passo 5: Testar o Auto Scaling Gerando Carga

Agora vamos forçar o ASG a escalar.

### Opção 1: Usando hey (Ferramenta de Teste de Carga)

1. Instale hey (na sua máquina local):

    ```bash
    go install github.com/rakyll/hey@latest
    ```

2. Execute um teste de carga:

    ```bash
    hey -z 5m -c 50 http://DNS-DO-LOAD-BALANCER/teste
    ```
    > Isso envia 50 requisições concorrentes por 5 minutos.

### Opção 2: Usando stress (Dentro da Instância)

1. Instale stress:

    ```bash
    sudo yum install -y stress
    ```

2. Simule carga na CPU:

    ```bash
    stress --cpu 2 --timeout 300s
    ```
    > Isso força 100% de CPU por 5 minutos.

---

## Passo 6: Monitorar o Auto Scaling

1. No Console AWS, vá para **Auto Scaling Groups**:
   - Veja se novas instâncias estão sendo criadas.
2. Verifique o **CloudWatch**:
   - Métricas como CPUUtilization e RequestCount.
3. Acesse o endpoint:

    ```bash
    curl http://DNS-DO-LOAD-BALANCER/teste
    ```
    > Você verá respostas de diferentes instâncias conforme o ASG escala.

---

## ✅ Resultado Esperado

- ✔ Ao gerar carga, o ASG adiciona instâncias (até 3).
- ✔ Quando a carga diminui, o ASG remove instâncias (mínimo 1).
- ✔ O Load Balancer distribui o tráfego entre todas as instâncias.

Pronto! Agora você tem um sistema escalável automaticamente na AWS. 🚀

---

# Como devo configurar a VPC?

## Configuração da VPC para Auto Scaling e Load Balancer na AWS

Para que o Auto Scaling Group (ASG) e o Load Balancer (CLB) funcionem corretamente, sua VPC deve estar configurada com:

- ✅ Sub-redes públicas (para instâncias acessíveis pela internet)
- ✅ Tabelas de rotas (Route Tables) corretas
- ✅ Internet Gateway (IGW) para acesso à internet
- ✅ Security Groups liberando tráfego HTTP (80) e SSH (22)

### Passo a Passo: Configurar a VPC

1. **Criar uma VPC (se não existir)**
   - Console AWS: VPC > Your VPCs > Create VPC
   - Nome: `ASG-VPC`
   - CIDR Block: `10.0.0.0/16`
   - Clique em **Create VPC**

2. **Criar Sub-redes Públicas (para ASG e Load Balancer)**
   - Subnets > Create Subnet
   - VPC: Selecione `ASG-VPC`
   - Subnet 1:
     - Nome: `Public-Subnet-1a`
     - Availability Zone: `us-east-1a`
     - CIDR Block: `10.0.1.0/24`
   - Subnet 2:
     - Nome: `Public-Subnet-1b`
     - Availability Zone: `us-east-1b`
     - CIDR Block: `10.0.2.0/24`
   - Clique em **Create**
   - Habilite "Enable auto-assign public IPv4 address" em cada subnet

3. **Criar e Configurar um Internet Gateway (IGW)**
   - Internet Gateways > Create Internet Gateway
   - Nome: `ASG-IGW`
   - Clique em **Create**
   - Anexe à VPC: Actions > Attach to VPC > Escolha `ASG-VPC`

4. **Configurar a Tabela de Rotas (Route Table) Pública**
   - Route Tables > Localize a tabela padrão da VPC
   - Editar rotas: Routes > Edit routes
   - Adicione uma rota:
     - Destination: `0.0.0.0/0`
     - Target: `ASG-IGW`
   - Associe às sub-redes públicas: Subnet Associations > Edit subnet associations > Selecione `Public-Subnet-1a` e `Public-Subnet-1b`

5. **Criar Security Groups (Grupos de Segurança)**

   - **Para o Load Balancer (CLB):**
     - Nome: `ASG-LB-SG`
     - Descrição: "Libera HTTP para o Load Balancer"
     - VPC: `ASG-VPC`
     - Regras de entrada:
       | Tipo | Protocolo | Porta | Origem         |
       |------|-----------|-------|----------------|
       | HTTP | TCP       | 80    | 0.0.0.0/0      |

   - **Para as Instâncias do ASG:**
     - Nome: `ASG-Instances-SG`
     - VPC: `ASG-VPC`
     - Regras de entrada:
       | Tipo | Protocolo | Porta | Origem         |
       |------|-----------|-------|----------------|
       | HTTP | TCP       | 80    | ASG-LB-SG ou 0.0.0.0/0 |
       | SSH  | TCP       | 22    | Seu IP         |

---

### Resumo da Configuração da VPC

| Recurso         | Configuração                                      |
|-----------------|--------------------------------------------------|
| VPC             | ASG-VPC (10.0.0.0/16)                            |
| Sub-redes       | Public-Subnet-1a (10.0.1.0/24) e Public-Subnet-1b (10.0.2.0/24) |
| Internet Gateway| ASG-IGW anexado à VPC                            |
| Route Table     | Rota 0.0.0.0/0 → Internet Gateway                |
| Security Groups | ASG-LB-SG (HTTP 80) e ASG-Instances-SG (HTTP 80 + SSH 22) |

---

## Qual IP devo colocar em SSH?

### Opções para acessar via SSH as instâncias do Auto Scaling

#### Opção 1: Usar o IP Público da Instância (não recomendado para ASG)

- Problema: Se o ASG substituir a instância, o IP muda.
- Como encontrar:
  - EC2 Console > Instances > IPv4 Public IP
- Conecte via SSH:
    ```bash
    ssh -i "sua-chave.pem" ec2-user@IP_PUBLICO_DA_INSTANCIA
    ```
- ⚠️ Não recomendado para ASG, pois o IP pode mudar após scaling.

#### Opção 2: Usar um Bastion Host (Jump Server)

- Crie uma instância EC2 (Bastion) em sub-rede pública.
- Security Group liberando SSH (22) apenas para seu IP.
- Ajuste o Security Group das instâncias do ASG para liberar SSH (22) apenas para o Bastion.
- Conecte-se via SSH:
    ```bash
    # Primeiro acesse o Bastion
    ssh -i "sua-chave.pem" ec2-user@IP_DO_BASTION

    # Dentro do Bastion, acesse uma instância do ASG
    ssh ec2-user@IP_PRIVADO_DA_INSTANCIA_ASG
    ```
- ✅ Vantagem: Mais seguro, ideal para ambientes com sub-redes privadas.

#### Opção 3: Usar Session Manager (AWS Systems Manager)

- Ative o AWS Systems Manager (SSM) na instância (role AmazonSSMManagedInstanceCore).
- Conecte-se via Session Manager:
    ```bash
    aws ssm start-session --target ID_DA_INSTANCIA
    ```
- ✅ Melhor opção: Sem exposição de portas, sem necessidade de IP fixo.

#### Opção 4: Usar Elastic IP (não recomendado para ASG)

- Aloque um Elastic IP no EC2 Console > Elastic IPs.
- Associe manualmente à instância.
- Use-o no SSH:
    ```bash
    ssh -i "sua-chave.pem" ec2-user@ELASTIC_IP
    ```
- ⚠️ O ASG não gerencia Elastic IPs automaticamente.

---

### Resumo: Qual IP usar?

| Método                 | Quando Usar                  | Vantagens                | Desvantagens                  |
|------------------------|-----------------------------|--------------------------|-------------------------------|
| IP Público da Instância| Testes rápidos              | Simples                  | IP muda com scaling           |
| Bastion Host           | Sub-redes privadas          | Seguro                   | Requer servidor extra         |
| Session Manager (SSM)  | Melhor prática              | Mais seguro, sem IP fixo | Requer configuração de IAM    |
| Elastic IP             | Necessidade de IP fixo      | Estável                  | Não escalável com ASG         |

**Recomendação Final:**  
Para produção: Use Session Manager (SSM) ou Bastion Host.  
Para testes rápidos: Use o IP público temporário, mas lembre-se que ele pode mudar.

---

## Solução para o Erro na Configuração de Rede do Auto Scaling

Se aparecer mensagens como "Remover sub-rede para modelos do EC2 Auto Scaling" ou "Interfaces de rede existentes não são recomendadas":

### Solução Recomendada

**No Launch Template:**
- Deixe o campo Sub-rede em branco.
- O Auto Scaling distribuirá automaticamente as instâncias pelas sub-redes definidas no Auto Scaling Group.
- Em "Grupos de segurança": selecione pelo menos um Security Group que permita SSH (22) e HTTP (80).
- Em "Atribuir IP público automaticamente": marque "Habilitar" se precisar de acesso à internet.

**No Auto Scaling Group:**
- Especifique pelo menos 2 sub-redes em diferentes zonas de disponibilidade.
- Não tente anexar interfaces de rede existentes.

---

## Solução para o Erro "Modelo de execução inválido" no Auto Scaling

Se aparecer o erro:  
`Modelo de execução inválido especificado na Etapa 1: You are not authorized to perform this operation`

### Solução Passo a Passo

1. **Verificar Permissões do IAM**
   - Console IAM > Roles > Localize a função usada pelo Auto Scaling (ex: AWSServiceRoleForAutoScaling)
   - Se não existir, crie:
     - Create role > AWS service > Auto Scaling > AutoScalingServiceRolePolicy

2. **Atualizar Permissões do Usuário**
   - Adicione as políticas AmazonEC2FullAccess e AutoScalingFullAccess ao usuário/grupo.

3. **Tentar Novamente a Criação do Auto Scaling Group**
   - Selecione a opção "Melhor esforço equilibrado" e continue.

4. **Possíveis Causas Adicionais**
   - Verifique se o Launch Template existe e está completo.
   - Confirme que você tem permissão para usá-lo.
   - Verifique limites de instâncias EC2.
   - Certifique-se de que todos os recursos estão na mesma região AWS.

---

## Como criar um endpoint para se conectar à EC2 via terminal pela AWS

### 1. Método Tradicional (SSH direto com IP Público)

⚠️ Requer: IP público + Security Group liberando SSH

```bash
ssh -i "sua-chave.pem" ec2-user@IP_PUBLICO_DA_EC2
```

Problema: Instâncias em Auto Scaling podem ter IPs variáveis.

---

### 2. Método Recomendado (AWS Systems Manager - SSM)

✅ Mais seguro (sem IP público necessário)

**Passo a Passo:**

- Adicione a política AmazonSSMManagedInstanceCore à role da instância

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:StartSession"
      ],
      "Resource": "*"
    }
  ]
}
```

- Instale o AWS CLI e plugin SSM:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb
```

- Conecte-se:

```bash
aws ssm start-session --target ID_DA_INSTANCIA
```

---

### 3. Bastion Host (Jump Server)

🔐 Ideal para instâncias em sub-redes privadas

- Crie uma EC2 pública (t2.micro) com Security Group liberando SSH apenas para seu IP
- Na instância privada: libere SSH apenas para o Security Group do Bastion
- Conecte-se em 2 etapas:

```bash
ssh -i "chave-bastion.pem" ec2-user@IP_BASTION
ssh ec2-user@IP_PRIVADO_DA_EC2
```

---

### Comparação dos Métodos

| Método      | Segurança | Complexidade | Custo         | Melhor Para         |
|-------------|-----------|--------------|---------------|---------------------|
| SSH Direto  | ⭐⭐        | ⭐           | $0            | Testes rápidos      |
| AWS SSM     | ⭐⭐⭐⭐      | ⭐⭐          | $0.005/sessão | Produção (recomendado) |
| Bastion     | ⭐⭐⭐       | ⭐⭐⭐         | Custo EC2     | Redes privadas      |

> **Dica profissional:** Para ambientes de produção, use sempre o AWS Systems Manager (SSM).

---

## Solução para Erro de Conexão SSH em Instâncias EC2

Se você está recebendo o erro "Failed to connect to your instance" ao tentar acessar sua instância EC2 via SSH, siga este guia:

### Diagnóstico Inicial

- Verifique o estado da instância (Running)
- Verifique o status de verificação de status (Status Checks)
- Confirme o método de conexão (SSH direto, Session Manager ou Bastion Host)

---

### Soluções Comuns

#### 1. Problemas com Security Groups

- Acesse EC2 > Security Groups
- Localize o security group associado à sua instância
- Adicione uma regra de entrada (Inbound):
  - Tipo: SSH
  - Protocolo: TCP
  - Porta: 22
  - Origem: Seu IP público (ou 0.0.0.0/0 para testes)

#### 2. Problemas com Par de Chaves

- Verifique se está usando o par de chaves correto
- O nome do arquivo .pem corresponde ao key pair da instância?
- As permissões do arquivo estão corretas? (`chmod 400 sua-chave.pem`)
- Se perdeu a chave:
  1. Criar novo par de chaves
  2. Parar a instância
  3. Alterar o key pair associado
  4. Reiniciar a instância

#### 3. Instância sem IP Público

- Para instâncias em sub-rede privada:
    ```bash
    aws ssm start-session --target i-1234567890abcdef0
    ```
- Ou configure um Bastion Host

#### 4. Problemas de Rede

- Verifique:
  - A sub-rede tem rota para internet? (Internet Gateway/NAT)
  - O NACL permite tráfego SSH (porta 22)?

---

### Solução Rápida para Testes

- Adicione temporariamente ao Security Group:
  - Tipo: All traffic
  - Origem: 0.0.0.0/0

- Conecte via:

    ```bash
    ssh -i "sua-chave.pem" ec2-user@IP_PUBLICO -v
    ```
    > O `-v` mostra logs detalhados

---

### Verificação Final

- Aguarde 1-2 minutos após mudanças no Security Group
- Tente conectar novamente
- Verifique os logs do sistema da instância via console EC2 > Actions > Monitor and troubleshoot > Get system log

---

## Se o problema persistir, forneça:

- A configuração completa do Security Group
- A saída do comando SSH com `-v`
- A configuração de rede da VPC/sub-rede