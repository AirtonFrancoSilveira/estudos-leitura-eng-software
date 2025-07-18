# 🚀 Orquestração com AWS ECS e Fargate

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Entender o que é um orquestrador de contêineres** e por que o `docker-compose` não é suficiente para produção.
-   ✅ **Dominar os quatro componentes centrais do AWS ECS:** Cluster, Task Definition, Task e Service.
-   ✅ **Diferenciar os launch types EC2 e Fargate**, compreendendo as vantagens do serverless com Fargate.
-   ✅ **Definir um serviço ECS completo como Infraestrutura como Código (IaC)** usando AWS CloudFormation ou Terraform.
-   ✅ **Integrar um serviço ECS com um Application Load Balancer** para distribuir tráfego.
-   ✅ **Configurar auto-scaling e logging** para um serviço em produção.

---

## 🎯 **Conceito 1: O Salto para a Produção - Por que Precisamos de um Orquestrador?**

O `docker-compose` é fantástico para o desenvolvimento local, mas ele não foi feito para ambientes de produção. Ele não consegue responder a perguntas cruciais como:

-   **Auto-cura (Self-healing):** O que acontece se um contêiner travar e morrer? Quem o reinicia?
-   **Escalabilidade (Scaling):** Como eu escalo minha aplicação de 1 para 10 contêineres se o tráfego aumentar? E como volto para 2 quando o tráfego diminuir?
-   **Distribuição de Tráfego (Load Balancing):** Como eu distribuo as requisições de forma inteligente entre esses 10 contêineres?
-   **Deploy sem Downtime (Zero-downtime Deployment):** Como eu atualizo minha aplicação para uma nova versão sem que nenhum usuário perceba uma interrupção?
-   **Gerenciamento de Recursos:** Como eu garanto que meus contêineres sejam alocados em servidores com CPU e memória suficientes?

Um **Orquestrador de Contêineres** (como AWS ECS, Kubernetes ou Nomad) é o "cérebro" que resolve todos esses problemas. Ele gerencia o ciclo de vida dos contêineres em um cluster de máquinas.

---

## 🎯 **Conceito 2: Os Componentes do AWS Elastic Container Service (ECS)**

O AWS ECS é o orquestrador nativo da AWS. Ele é conhecido por sua simplicidade e profunda integração com o ecossistema AWS. Seus conceitos principais são:

1.  **Task Definition (Definição de Tarefa):**
    -   **O que é:** O "blueprint" ou a "receita" do seu contêiner. É um arquivo JSON que descreve:
        -   Qual imagem Docker usar (ex: `123456789012.dkr.ecr.sa-east-1.amazonaws.com/minha-app:latest`).
        -   Quanta CPU e memória reservar.
        -   Variáveis de ambiente e segredos.
        -   Configurações de logging (para onde enviar os logs).
        -   Portas a serem expostas.
    -   **Analogia:** É como o `services` > `minha-app` dentro do seu `docker-compose.yml`.

2.  **Task (Tarefa):**
    -   **O que é:** Uma instância em execução de uma `Task Definition`. É o contêiner (ou grupo de contêineres) rodando.
    -   **Analogia:** Se a Task Definition é a classe, a Task é o objeto instanciado.

3.  **Service (Serviço):**
    -   **O que é:** O "cérebro" que mantém um número desejado de `Tasks` rodando e saudáveis. O Service é responsável por:
        -   **Manter o número de cópias:** Se você quer 3 `Tasks` rodando e uma morre, o Service automaticamente inicia uma nova.
        -   **Integração com Load Balancer:** Registra as `Tasks` em um Application Load Balancer para receber tráfego.
        -   **Estratégias de Deploy:** Controla como as atualizações são feitas (ex: Rolling Update, Blue/Green).
        -   **Auto-Scaling:** Aumenta ou diminui o número de `Tasks` com base em métricas (CPU, memória, etc.).

4.  **Cluster:**
    -   **O que é:** Um agrupamento lógico de recursos (CPU, memória) onde suas `Tasks` são executadas.
    -   **Analogia:** É o "chão de fábrica" onde seus robôs (Tasks) operam.

---

## 🎯 **Conceito 3: Fargate - O Fim do Gerenciamento de Servidores**

O ECS oferece dois "modos de lançamento" (Launch Types) para rodar suas `Tasks`:

1.  **Launch Type EC2:**
    -   **Como funciona:** Você é responsável por criar e gerenciar um grupo de servidores (instâncias EC2) que formarão seu cluster. Você precisa escolher o tipo de instância, atualizar o sistema operacional, aplicar patches de segurança e otimizar a alocação de contêineres nos servidores.
    -   **Vantagens:** Maior controle sobre a infraestrutura, pode ser mais barato para cargas de trabalho muito grandes e estáveis.
    -   **Desvantagens:** Muita sobrecarga de gerenciamento ("undifferentiated heavy lifting").

2.  **Launch Type Fargate (Serverless):**
    -   **Como funciona:** Você simplesmente entrega sua `Task Definition` para o ECS e diz "rode 3 cópias disso para mim". **A AWS cuida de encontrar os servidores, provisionar os recursos e gerenciar toda a infraestrutura subjacente**. Você não vê nem gerencia nenhum servidor.
    -   **Vantagens:** **Simplicidade radical**. Foco total na aplicação, não na infraestrutura. Escalabilidade quase instantânea. Modelo de pagamento por uso (paga pela CPU/memória que seu contêiner consome, por segundo).
    -   **Desvantagens:** Menos controle granular, pode ser um pouco mais caro para cargas de trabalho muito previsíveis e de longa duração.

Para a vasta maioria dos casos de uso, **Fargate é a escolha recomendada**. Ele acelera o desenvolvimento e reduz drasticamente a complexidade operacional.

---

## 🎯 **Conceito 4: Exemplo Prático com AWS CloudFormation**

Vamos definir um serviço completo usando Infraestrutura como Código. Este template define um serviço web que roda nossa imagem `minha-app`, expõe a porta 8080, e se registra em um Application Load Balancer para receber tráfego da internet.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: "Serviço Java rodando no ECS Fargate com Load Balancer"

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
  SubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
  ImageUrl:
    Type: String
    Description: "URL completa da imagem Docker no ECR (ex: 12345.dkr.ecr.region.amazonaws.com/myapp:latest)"

Resources:
  # 1. Cluster ECS (o ambiente lógico)
  MyCluster:
    Type: AWS::ECS::Cluster

  # 2. Definição da Tarefa (a receita do nosso contêiner)
  MyTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: "minha-app-task"
      Cpu: "256" # 0.25 vCPU
      Memory: "512" # 512 MB
      NetworkMode: "awsvpc"
      RequiresCompatibilities: ["FARGATE"]
      ExecutionRoleArn: !Ref MyEcsExecutionRole # Role para o ECS puxar a imagem e enviar logs
      ContainerDefinitions:
        - Name: "minha-app-container"
          Image: !Ref ImageUrl
          PortMappings:
            - ContainerPort: 8080
          LogConfiguration:
            LogDriver: "awslogs"
            Options:
              awslogs-group: !Ref MyLogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: "ecs"

  # 3. O Serviço (o cérebro que mantém as tarefas rodando)
  MyService:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !GetAtt MyCluster.Arn
      TaskDefinition: !Ref MyTaskDefinition
      DesiredCount: 2 # Queremos 2 cópias da nossa tarefa rodando
      LaunchType: "FARGATE"
      NetworkConfiguration:
        AwsvpcConfiguration:
          Subnets: !Ref SubnetIds
          SecurityGroups: [!Ref MyContainerSecurityGroup]
      LoadBalancers:
        - ContainerName: "minha-app-container"
          ContainerPort: 8080
          TargetGroupArn: !Ref MyTargetGroup

  # --- Componentes de Rede e Logging ---

  MyLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Type: "application"
      Subnets: !Ref SubnetIds

  MyListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: "forward"
          TargetGroupArn: !Ref MyTargetGroup
      LoadBalancerArn: !Ref MyLoadBalancer
      Port: 80
      Protocol: "HTTP"

  MyTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Port: 80
      Protocol: "HTTP"
      VpcId: !Ref VpcId
      HealthCheckPath: "/actuator/health"

  MyContainerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !Ref VpcId
      GroupDescription: "Permite tráfego de entrada na porta 8080 do Load Balancer"
      SecurityGroupIngress:
        - CidrIp: "0.0.0.0/0"
          IpProtocol: "tcp"
          FromPort: 8080
          ToPort: 8080

  MyLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: "/ecs/minha-app"
      RetentionInDays: 7

  MyEcsExecutionRole:
    Type: AWS::IAM::Role
    # ... (definição da Role com permissões para ECR e CloudWatch Logs)
```
Este template, embora longo, é declarativo. Ele define o **estado desejado** do nosso sistema, e o CloudFormation se encarrega de criar, conectar e configurar todos os recursos na ordem correta.

--- 