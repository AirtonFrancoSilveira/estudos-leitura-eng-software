# ⚡ Event-Driven Architecture - Guia Aprofundado

## 📚 **Objetivos de Aprendizagem**

Ao final deste módulo, você será capaz de:

-   ✅ **Dominar os conceitos fundamentais** da Arquitetura Orientada a Eventos (EDA).
-   ✅ **Implementar padrões essenciais** como Event Sourcing, Saga, e CQRS em Java.
-   ✅ **Projetar e construir sistemas EDA robustos e escaláveis** usando os principais serviços da AWS (EventBridge, SQS, SNS, Kinesis).
-   ✅ **Analisar os trade-offs** entre consistência, disponibilidade e performance em sistemas orientados a eventos.
-   ✅ **Resolver problemas complexos** de transações distribuídas e consistência de dados.
-   ✅ **Evoluir de uma arquitetura monolítica** para uma arquitetura de microserviços orientada a eventos.

---

## 🎯 **Conceito 1: A Mudança de Paradigma - De Requisição/Resposta para Eventos**

### **🤔 Por que Mudar? Os Limites do Modelo Síncrono**

O modelo tradicional de **Requisição/Resposta (Request/Response)**, onde um cliente envia uma requisição e espera por uma resposta, é simples e eficaz para muitas aplicações. No entanto, em sistemas distribuídos complexos, ele apresenta gargalos significativos:

-   **Acoplamento Forte:** O serviço A precisa conhecer e estar diretamente conectado ao serviço B. Se B estiver offline, A falha.
-   **Baixa Resiliência:** Uma falha em um único serviço pode causar uma cascata de falhas (efeito dominó).
-   **Escalabilidade Limitada:** É difícil escalar serviços de forma independente, pois eles estão amarrados uns aos outros.
-   **Latência Aumentada:** O cliente fica bloqueado, esperando a conclusão de toda a cadeia de chamadas síncronas.

### **✨ A Solução: Desacoplamento com Eventos**

A **Arquitetura Orientada a Eventos (EDA)** quebra essas amarras. Em vez de chamadas diretas, os serviços se comunicam de forma assíncrona através de **eventos**.

-   **Evento:** Um registro imutável de algo que **aconteceu** no passado. Ex: `PedidoCriado`, `PagamentoProcessado`, `UsuárioCadastrado`.

**O fluxo muda drasticamente:**

1.  **Produtor (Producer):** Um serviço realiza uma ação e **publica um evento** em um canal central (o "Event Broker"). Ele não sabe (e não se importa) quem vai consumir esse evento.
2.  **Event Broker (ou Message Broker):** Uma infraestrutura (como AWS EventBridge, SQS, ou Apache Kafka) que recebe, armazena e distribui os eventos.
3.  **Consumidor (Consumer):** Outros serviços **se inscrevem** nos eventos que lhes interessam e reagem a eles de forma independente.

| Característica | Requisição/Resposta (Síncrono) | Arquitetura Orientada a Eventos (Assíncrono) |
| :--- | :--- | :--- |
| **Comunicação** | Direta (ponto a ponto) | Indireta (via broker) |
| **Acoplamento** |  Forte (temporal e de localização) | Fraco (serviços não se conhecem) |
| **Disponibilidade**| Menor (falha em um afeta o outro) | **Maior** (serviços operam de forma independente) |
| **Escalabilidade** | Complexa (em bloco) | **Simplificada** (serviços escalam individualmente) |
| **Conhecimento** | O chamador precisa saber quem chamar | O produtor não sabe quem são os consumidores |
| **Exemplo** | `servicoPagamento.processar(pedido)` | `publicarEvento(new PedidoCriado(pedido))` |

---

## 🎯 **Conceito 2: Padrões Essenciais de Comunicação com Eventos**

### **1. Padrão Publish/Subscribe (Pub/Sub)**

#### **📖 O que é?**
O padrão Publish/Subscribe (ou Pub/Sub) é a espinha dorsal da maioria das arquiteturas orientadas a eventos. Ele desacopla completamente os produtores dos consumidores através de um canal chamado **Tópico (Topic)**.

-   **Publisher (Produtor):** Publica eventos em um tópico específico, sem ter conhecimento de quem são os assinantes (subscribers).
-   **Topic (Tópico):** Um canal nomeado que atua como um quadro de avisos. Ele categoriza os eventos. Ex: `pedidos`, `pagamentos`.
-   **Subscriber (Consumidor):** Assina um ou mais tópicos de interesse. Ele recebe uma **cópia de todas as mensagens** publicadas no tópico após o momento de sua assinatura.

**Analogia:** Pense em um canal do YouTube. O criador de conteúdo (Publisher) posta um vídeo (evento) em seu canal (Topic). Todos os inscritos (Subscribers) são notificados e podem assistir ao vídeo. O criador não precisa enviar o vídeo para cada inscrito individualmente.

#### **💡 Implementação com Java (Spring Events)**

O Spring Framework oferece um mecanismo de Pub/Sub em memória, excelente para comunicação dentro de uma única aplicação (monolito) ou para desacoplar componentes internos.

```java
// Passo 1: Definir o Evento (um POJO imutável)
// Evento que representa a criação de um novo pedido.
public class PedidoCriadoEvent {
    private final String pedidoId;
    private final String clienteId;
    private final BigDecimal valor;

    public PedidoCriadoEvent(String pedidoId, String clienteId, BigDecimal valor) {
        this.pedidoId = pedidoId;
        this.clienteId = clienteId;
        this.valor = valor;
    }

    // Getters...
}

// Passo 2: Criar o Publisher (usando ApplicationEventPublisher do Spring)
@Service
public class PedidoService {

    private final ApplicationEventPublisher eventPublisher;

    @Autowired
    public PedidoService(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }

    public void criarPedido(Pedido pedido) {
        // ... lógica para salvar o pedido no banco de dados ...
        System.out.println("Pedido salvo no banco de dados. Publicando evento...");

        // Publica o evento para que outros componentes possam reagir.
        PedidoCriadoEvent event = new PedidoCriadoEvent(
            pedido.getId(), pedido.getClienteId(), pedido.getValor()
        );
        eventPublisher.publishEvent(event);

        System.out.println("Evento PedidoCriadoEvent publicado!");
    }
}

// Passo 3: Criar os Subscribers (usando a anotação @EventListener)

// Subscriber 1: Módulo de Notificação
@Component
class NotificacaoService {

    @EventListener
    @Async // Executa o listener em uma thread separada para não bloquear o publisher
    public void handlePedidoCriado(PedidoCriadoEvent event) {
        System.out.println(" Módulo de Notificação: Recebeu PedidoCriadoEvent. ");
        // Lógica para enviar um e-mail ou SMS de confirmação
        System.out.printf("Enviando e-mail para cliente %s sobre o pedido %s no valor de %.2f...%n",
            event.getClienteId(), event.getPedidoId(), event.getValor());
        // Simula o tempo de envio
        try {
            Thread.sleep(2000); 
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        System.out.println(" Módulo de Notificação: E-mail enviado. ");
    }
}

// Subscriber 2: Módulo de Logística/Estoque
@Component
class LogisticaService {

    @EventListener
    public void handlePedidoCriado(PedidoCriadoEvent event) {
        System.out.println(" Módulo de Logística: Recebeu PedidoCriadoEvent. ");
        // Lógica para reservar o estoque ou iniciar o processo de envio
        System.out.printf("Reservando estoque para o pedido %s...%n", event.getPedidoId());
         System.out.println(" Módulo de Logística: Estoque reservado. ");
    }
}

// Para habilitar o @Async, adicione @EnableAsync em uma classe de configuração
@Configuration
@EnableAsync
public class AsyncConfig {
}
```

#### **💡 Implementação com AWS (SNS - Simple Notification Service)**

O AWS SNS é um serviço de Pub/Sub totalmente gerenciado, ideal para comunicação entre microserviços, aplicações distribuídas ou para enviar notificações para múltiplos endpoints.

**Componentes:**
-   **SNS Topic:** O canal central para onde os eventos são publicados.
-   **Subscriptions:** As "assinaturas" do tópico. Cada assinatura pode ser um serviço diferente (SQS, Lambda, HTTP endpoint, etc.).

```yaml
# Template do AWS CloudFormation (ou SAM) para criar a infraestrutura

AWSTemplateFormatVersion: '2010-09-09'
Resources:
  # O Tópico SNS que funcionará como nosso "quadro de avisos" de pedidos
  PedidosTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: pedidos-topic

  # A Fila SQS para o serviço de notificação.
  # Ela assina o tópico e recebe uma cópia de cada evento.
  NotificacoesQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: notificacoes-queue

  # A Fila SQS para o serviço de logística.
  # Também assina o mesmo tópico.
  LogisticaQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: logistica-queue

  # A "assinatura" que conecta o Tópico à fila de Notificações
  SnsToNotificacoesSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: sqs
      Endpoint: !GetAtt NotificacoesQueue.Arn
      TopicArn: !Ref PedidosTopic
      RawMessageDelivery: 'true' # Envia a mensagem original, sem o encapsulamento do SNS

  # A "assinatura" que conecta o Tópico à fila de Logística
  SnsToLogisticaSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      Protocol: sqs
      Endpoint: !GetAtt LogisticaQueue.Arn
      TopicArn: !Ref PedidosTopic
      RawMessageDelivery: 'true'

  # Política que permite ao Tópico SNS enviar mensagens para as filas SQS
  SnsToSqsPolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref NotificacoesQueue
        - !Ref LogisticaQueue
      PolicyDocument:
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: SQS:SendMessage
            Resource: '*'
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PedidosTopic
```

```python
# Exemplo de código Python para o Publisher (ex: uma função Lambda)
import boto3
import json
import os

sns_client = boto3.client('sns')
PEDIDOS_TOPIC_ARN = os.environ['PEDIDOS_TOPIC_ARN']

def pedido_service_handler(event, context):
    # ... Lógica para criar o pedido ...
    pedido_id = event['pedido_id']
    
    message = {
        "pedidoId": pedido_id,
        "clienteId": "cliente-123",
        "valor": 99.90
    }

    # Publica a mensagem no Tópico SNS
    sns_client.publish(
        TopicArn=PEDIDOS_TOPIC_ARN,
        Message=json.dumps(message),
        MessageAttributes={
            'eventType': {
                'DataType': 'String',
                'StringValue': 'PedidoCriado'
            }
        }
    )
    return {"status": "sucesso", "pedidoId": pedido_id}
```

#### **⚖️ Trade-offs do Pub/Sub:**

| Prós (👍) | Contras (👎) |
| :--- | :--- |
| **Desacoplamento Máximo:** Produtores e consumidores são totalmente independentes. | **Garantias de Entrega:** O que acontece se um consumidor estiver offline? Ele perde a mensagem? (Depende do broker). |
| **Escalabilidade Horizontal:** Adicionar novos consumidores não afeta os existentes. | **Complexidade de Monitoramento:** Rastrear um evento através de múltiplos sistemas pode ser difícil (requer tracing distribuído). |
| **Flexibilidade:** Fácil de adicionar novas funcionalidades (basta um novo subscriber). | **Ordenação de Eventos:** A ordem de recebimento das mensagens não é garantida entre diferentes consumidores. |
| **Resiliência:** Uma falha em um consumidor não impacta os outros. | **Mensagens Duplicadas:** Consumidores devem ser **idempotentes**, pois podem receber a mesma mensagem mais de uma vez. |

---

### **2. Padrão Message Queue**

#### **📖 O que é?**
Enquanto o Pub/Sub é sobre "transmitir para todos", o Message Queue é sobre "processar um de cada vez". Uma fila (Queue) armazena mensagens em ordem e garante que **apenas um consumidor processe cada mensagem**. Isso é ideal para distribuir tarefas e garantir que o trabalho seja feito, mesmo que lentamente.

-   **Sender (Produtor):** Envia uma mensagem para uma fila específica.
-   **Queue (Fila):** Armazena as mensagens de forma durável (persiste em disco) até que um consumidor esteja pronto.
-   **Receiver (Consumidor):** Puxa (polls) uma mensagem da fila, a processa e, em seguida, a **exclui** para que não seja processada novamente.

**Analogia:** Pense em uma fila de caixa de supermercado. Os clientes (mensagens) entram na fila. Cada caixa (consumidor) atende um cliente por vez. Se um caixa fecha, os clientes simplesmente esperam na fila até que outro caixa fique disponível. A loja não perde o cliente.

**Diferença Chave: Pub/Sub vs. Message Queue**
-   **Pub/Sub (SNS):** 1 mensagem é copiada para **N** consumidores (broadcast).
-   **Message Queue (SQS):** 1 mensagem é processada por **apenas 1** consumidor (distribuição de trabalho).

Muitas vezes, eles são usados juntos no padrão **Fan-Out**: um evento do SNS (Pub/Sub) é enviado para múltiplas filas SQS (Message Queues), permitindo que diferentes grupos de consumidores processem o mesmo evento de forma independente e resiliente.

#### **💡 Implementação com Java (JMS / RabbitMQ)**
Para comunicação entre diferentes serviços, usamos brokers de mensagens dedicados como RabbitMQ ou ActiveMQ.

```java
// Exemplo com Spring AMQP para se conectar a um RabbitMQ

// Passo 1: Configurar a conexão e a Fila
@Configuration
public class RabbitMQConfig {

    public static final String PAGAMENTOS_QUEUE = "pagamentos-queue";

    @Bean
    public Queue pagamentosQueue() {
        // durable=true garante que a fila sobreviva a reinicializações do broker
        return new Queue(PAGAMENTOS_QUEUE, true);
    }

    //... outras configurações de connection factory, template, etc.
}

// Passo 2: O Sender (Produtor) envia a mensagem para a fila
@Service
public class FaturamentoService {

    private final RabbitTemplate rabbitTemplate;

    @Autowired
    public FaturamentoService(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;
    }

    public void solicitarProcessamentoPagamento(Pagamento pagamento) {
        System.out.println("Enviando solicitação de pagamento para a fila...");
        // Converte o objeto Pagamento para JSON e o envia
        rabbitTemplate.convertAndSend(RabbitMQConfig.PAGAMENTOS_QUEUE, pagamento);
        System.out.println("Solicitação enviada com sucesso!");
    }
}

// Passo 3: O Receiver (Consumidor) escuta a fila
@Component
public class PagamentoConsumer {

    @RabbitListener(queues = RabbitMQConfig.PAGAMENTOS_QUEUE)
    public void processarPagamento(Pagamento pagamento) {
        System.out.printf("Recebido pedido de pagamento para o valor %.2f.%n", pagamento.getValor());
        // Lógica de negócio para processar o pagamento...
        // Ex: chamar a API de um gateway de pagamento, atualizar o banco de dados.
        // Se ocorrer uma exceção aqui, a mensagem NÃO é removida da fila e pode ser reprocessada.
        System.out.println("Pagamento processado com sucesso!");
    }
}
```

#### **💡 Implementação com AWS (SQS - Simple Queue Service)**
O SQS é o serviço de filas gerenciado da AWS. É altamente escalável, durável e uma peça central em arquiteturas desacopladas.

**Componentes:**
-   **Standard Queue:** Oferece máxima throughput, "at-least-once delivery" (entrega pelo menos uma vez) e ordenação de melhor esforço.
-   **FIFO Queue (First-In, First-Out):** Garante a ordem e "exactly-once processing" (processamento exatamente uma vez), mas com menor vazão.
-   **Dead-Letter Queue (DLQ):** Uma fila secundária para onde as mensagens são enviadas após falharem no processamento várias vezes. Isso evita que mensagens "venenosas" travem a fila principal.

```yaml
# Template do AWS CloudFormation (ou SAM)

AWSTemplateFormatVersion: '2010-09-09'
Resources:
  # A Fila de Destino para onde as mensagens "venenosas" irão
  ProcessamentoPagamentosDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: processamento-pagamentos-dlq

  # A Fila Principal
  ProcessamentoPagamentosQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: processamento-pagamentos-queue
      # Tempo que a mensagem fica invisível após ser lida por um consumidor.
      # Deve ser maior que o tempo de processamento.
      VisibilityTimeout: 300 
      # Política de Redrive: após 3 falhas, move a mensagem para a DLQ.
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt ProcessamentoPagamentosDLQ.Arn
        maxReceiveCount: 3

  # Função Lambda que será o nosso consumidor
  PagamentoProcessorFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: pagamento-processor
      Handler: index.handler
      Runtime: python3.9
      Timeout: 280 # Menor que o VisibilityTimeout
      # O "gatilho": a Lambda é invocada quando há mensagens na fila.
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt ProcessamentoPagamentosQueue.Arn
            BatchSize: 10 # Processa até 10 mensagens por invocação

      Policies:
        # Permissões para a Lambda ler e deletar mensagens da SQS
        - SQSProcessMessagePolicy:
            QueueName: !GetAtt ProcessamentoPagamentosQueue.Name

```

```python
# Exemplo de código Python para o Consumidor (Lambda)
import json

def handler(event, context):
    for record in event['Records']:
        try:
            # O corpo da mensagem SQS
            payload = json.loads(record['body'])
            print(f"Processando pagamento: {payload}")
            # ... Lógica de processamento ...
            # Se a função Lambda executar com sucesso, a mensagem é automaticamente removida da fila.
        except Exception as e:
            # Se a função Lambda gerar uma exceção, a mensagem não é removida e
            # voltará a ficar visível na fila para uma nova tentativa.
            print(f"Erro ao processar a mensagem: {e}")
            raise e # Importante relançar a exceção para o SQS saber que falhou!
    
    return {"status": "sucesso"}
```

#### **⚖️ Trade-offs das Message Queues:**

| Prós (👍) | Contras (👎) |
| :--- | :--- |
| **Durabilidade e Resiliência:** Mensagens são persistidas e sobrevivem a falhas dos consumidores e do broker. | **Latência:** A comunicação é inerentemente mais lenta que uma chamada direta. |
| **Balanceamento de Carga (Load Balancing):** Distribui o trabalho uniformemente entre múltiplos consumidores. | **Complexidade de Gerenciamento:** Requer configuração de DLQs, visibility timeouts, e monitoramento de tamanho da fila. |
| **Controle de Fluxo (Throttling):** A fila age como um buffer, protegendo sistemas downstream de picos de tráfego. | **Ordenação Estrita:** Garantir a ordem (Filas FIFO) geralmente vem com um custo de performance significativo. |
| **Processamento Assíncrono:** Permite que o produtor continue seu trabalho sem esperar pelo processamento da tarefa. | **Não é ideal para comunicação Request/Reply:** Embora possível, é complexo implementar um fluxo de resposta. |

---

### **3. Padrão Event Bus com AWS EventBridge**

#### **📖 O que é?**
Se o SNS é um quadro de avisos e o SQS é uma fila de caixa, o **AWS EventBridge** é um **roteador de eventos inteligente e centralizado**. Ele permite que diferentes partes do seu sistema (incluindo serviços AWS, suas próprias aplicações e até integrações com parceiros SaaS como Zendesk ou Stripe) se comuniquem através de um **Event Bus** compartilhado.

Sua principal vantagem é o **roteamento baseado em conteúdo**. Em vez de apenas publicar em um "tópico" genérico, você envia eventos com uma estrutura rica (JSON), e o EventBridge usa **regras (Rules)** para filtrar e enviar esses eventos apenas para os alvos (Targets) relevantes.

**Fluxo com EventBridge:**
1.  **Event Source (Fonte):** Qualquer serviço (AWS ou custom) envia um evento para o Event Bus.
2.  **Event Bus:** O barramento central que recebe todos os eventos.
3.  **Rules (Regras):** Você cria regras que examinam o **conteúdo** do evento. Ex: `Se "eventType" for "PedidoCriado" E "valor" > 1000`.
4.  **Targets (Alvos):** Se um evento corresponde a uma regra, o EventBridge o envia para um ou mais alvos, como Lambdas, filas SQS, Step Functions, etc.

**EventBridge vs. SNS + SQS:**
-   **SNS:** Filtra mensagens principalmente pelos `MessageAttributes` (metadados), não pelo corpo da mensagem. O roteamento é mais simples (Tópico -> Assinatura).
-   **EventBridge:** Permite **filtragem complexa e roteamento avançado diretamente sobre o conteúdo (payload) do evento**. É o serviço preferencial para orquestração de eventos entre microserviços.

#### **💡 Implementação com AWS**

Imagine um cenário de e-commerce onde pedidos de alto valor ou de clientes VIP precisam de tratamento especial.

```yaml
# Template do AWS CloudFormation (ou SAM)

AWSTemplateFormatVersion: '2010-09-09'
Resources:
  # O Event Bus principal para todos os eventos de pedidos
  PedidosEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: pedidos-event-bus

  # Alvo 1: Uma Lambda para processar TODOS os pedidos
  ProcessadorGeralLambda:
    Type: AWS::Lambda::Function
    # ... (definição da Lambda)

  # Alvo 2: Uma Fila SQS para o time de "Contas Especiais"
  ContasEspeciaisQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: contas-especiais-queue

  # Alvo 3: Uma máquina de estados para pedidos fraudulentos
  AnaliseDeFraudeStepFunction:
    Type: AWS::StepFunctions::StateMachine
    # ... (definição da Step Function)


  # --- REGRAS DE ROTEAMENTO ---

  # Regra 1: Captura todos os eventos de "PedidoCriado" e envia para o processador geral
  RegraTodosPedidos:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref PedidosEventBus
      EventPattern:
        source:
          - "com.meuecommerce.pedidos"
        detail-type:
          - "PedidoCriado"
      Targets:
        - Arn: !GetAtt ProcessadorGeralLambda.Arn
          Id: "ProcessadorGeralTarget"

  # Regra 2: Captura pedidos de ALTO VALOR e envia para a fila de Contas Especiais
  RegraPedidosAltoValor:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref PedidosEventBus
      EventPattern:
        source:
          - "com.meuecommerce.pedidos"
        detail-type:
          - "PedidoCriado"
        detail:
          valor:
            - numeric: [ ">=", 1000 ] # Filtra pelo CONTEÚDO do evento!
      Targets:
        - Arn: !GetAtt ContasEspeciaisQueue.Arn
          Id: "ContasEspeciaisTarget"

  # Regra 3: Captura pedidos com suspeita de fraude e dispara a análise
  RegraSuspeitaFraude:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref PedidosEventBus
      EventPattern:
        source:
          - "com.meuecommerce.pedidos"
        detail-type:
          - "PedidoCriado"
        detail:
          analiseFraude:
            - "requerida"
      Targets:
        - Arn: !Ref AnaliseDeFraudeStepFunction
          Id: "AnaliseFraudeTarget"
          RoleArn: !GetAtt EventBridgeToStepFunctionsRole.Arn

  # Permissão para o EventBridge invocar a Lambda da Regra 1
  PermissaoInvocacaoLambda:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref ProcessadorGeralLambda
      Action: "lambda:InvokeFunction"
      Principal: "events.amazonaws.com"
      SourceArn: !GetAtt RegraTodosPedidos.Arn

# ... (outras permissões necessárias, como para a Role da Regra 3) ...
```

```python
# Exemplo de código Python para o Produtor enviar um evento para o EventBridge
import boto3
import json

client = boto3.client('events')

def criar_pedido_handler(event, context):
    # ...
    novo_pedido = {
        "pedidoId": "12345",
        "clienteId": "cliente-vip-678",
        "valor": 1500.50,
        "itens": [...],
        "analiseFraude": "requerida" # Campo que será usado na filtragem
    }

    # Monta a entrada para o EventBridge
    response = client.put_events(
        Entries=[
            {
                'Source': 'com.meuecommerce.pedidos', # Boa prática usar um namespace reverso
                'DetailType': 'PedidoCriado', # O tipo do evento
                'Detail': json.dumps(novo_pedido), # O payload completo
                'EventBusName': 'pedidos-event-bus'
            }
        ]
    )
    # ...
```
Este evento será capturado por **TRÊS** regras e enviado para **TRÊS** alvos diferentes, de forma paralela e desacoplada.

#### **⚖️ Trade-offs do EventBridge:**

| Prós (👍) | Contras (👎) |
| :--- | :--- |
| **Roteamento Inteligente:** A lógica de roteamento fica centralizada no broker, não no código dos produtores ou consumidores. | **Descoberta de Eventos:** Pode ser difícil saber quais eventos existem e qual o seu schema. O [Schema Registry](https://aws.amazon.com/eventbridge/schema-registry/) do EventBridge ajuda a mitigar isso. |
| **Desacoplamento Avançado:** Os serviços não precisam saber nada sobre a lógica de negócio uns dos outros. | **Depuração (Debugging):** Rastrear por que um evento não chegou a um alvo pode ser complexo. Requer o uso do CloudWatch e X-Ray. |
| **Integração Nativa:** Conecta-se facilmente com mais de 200 serviços AWS e parceiros SaaS. | **Custo:** Embora barato, é um serviço pago por evento publicado no barramento. |
| **Escalabilidade e Resiliência:** Totalmente gerenciado pela AWS, com retries e DLQs para os alvos. | **Não é uma Fila:** O EventBridge não armazena eventos por longos períodos. Se não houver uma regra correspondente, o evento é descartado. Por isso é comum ter uma regra "catch-all" que envia tudo para um S3 ou CloudWatch Logs para arquivamento. |

--- 