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

