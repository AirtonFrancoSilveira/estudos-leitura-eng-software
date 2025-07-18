# 📊 Observabilidade e Monitoramento Distribuído

## 📚 **Objetivos de Aprendizagem**

Ao final deste módulo, você será capaz de:

-   ✅ **Diferenciar fundamentalmente Monitoramento de Observabilidade**.
-   ✅ **Instrumentar uma aplicação Java** com base nos três pilares: Logs, Métricas e Traces.
-   ✅ **Implementar Tracing Distribuído** usando OpenTelemetry para seguir uma requisição através de múltiplos microserviços.
-   ✅ **Utilizar o AWS X-Ray** para visualizar e depurar gargalos em sistemas serverless e baseados em contêineres.
-   ✅ **Criar dashboards eficazes no CloudWatch** que correlacionam métricas de negócio com métricas de sistema.
-   ✅ **Executar queries complexas com CloudWatch Logs Insights** para investigar problemas de produção rapidamente.

---

## 🎯 **Conceito 1: Por que Monitoramento Não é Suficiente**

Durante anos, o foco foi no **Monitoramento**: coletar dados pré-definidos (CPU, memória, latência) e criar alertas quando ultrapassam um limiar. Isso nos diz **SE** algo está errado.

A **Observabilidade** é a evolução. Ela nos dá a capacidade de **fazer perguntas arbitrárias sobre o nosso sistema sem precisar prever a pergunta com antecedência**. Ela nos ajuda a entender **POR QUE** algo está errado.

| Característica | Monitoramento (O QUE) | Observabilidade (PORQUÊ) |
| :--- | :--- | :--- |
| **Abordagem** | Reativa. "O sistema está lento?" | Proativa e Investigativa. "Quais usuários específicos estão vendo a maior latência e em qual chamada de serviço externa isso está acontecendo?"|
| **Dados** | Pré-agregados e conhecidos (métricas e "health checks"). | Ricos, de alta cardinalidade e não agregados (traces, logs estruturados). |
| **Foco** | Nos "conhecidos desconhecidos" (sabemos que a CPU pode subir). | Nos "desconhecidos desconhecidos" (erros que nunca imaginamos que poderiam ocorrer).|
| **Ferramentas** | Nagios, Zabbix, CloudWatch Alarms básicos. | OpenTelemetry, AWS X-Ray, Datadog, New Relic, Honeycomb. |

Em resumo: Monitoramento é olhar o painel do carro. Observabilidade é ter um mecânico de Fórmula 1 conectado ao motor, capaz de analisar qualquer aspecto do seu funcionamento em tempo real.

---

## 🎯 **Conceito 2: Os 3 Pilares da Observabilidade**

A observabilidade se apoia em três tipos principais de telemetria. Eles não são independentes; a mágica acontece quando eles estão **correlacionados**.

### **1. 🪵 Logs Estruturados**

Logs de texto plano (`System.out.println("Erro")`) são inúteis em escala. **Logs Estruturados** (geralmente em JSON) são a solução. Eles transformam cada linha de log em um objeto rico em dados que pode ser facilmente pesquisado, filtrado e agregado.

**Java (SLF4J + Logback):**
A melhor forma de fazer isso em Java é usar uma biblioteca como `logstash-logback-encoder`.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.2</version>
</dependency>
```

```xml
<!-- logback.xml -->
<configuration>
    <appender name="jsonConsoleAppender" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>

    <root level="INFO">
        <appender-ref ref="jsonConsoleAppender"/>
    </root>
</configuration>
```

```java
// No seu código
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import static net.logstash.logback.argument.StructuredArguments.*;

@Service
public class PagamentoService {
    private static final Logger logger = LoggerFactory.getLogger(PagamentoService.class);

    public void processar(Pagamento pagamento) {
        logger.info(
            "Processando pagamento",
            kv("pagamentoId", pagamento.getId()),
            kv("clienteId", pagamento.getClienteId()),
            kv("valor", pagamento.getValor()),
            kv("moeda", "BRL")
        );
        // ...
    }
}
```

**Saída JSON (Pronta para o CloudWatch Logs Insights):**
```json
{
  "@timestamp": "2023-10-27T10:00:00.123Z",
  "@version": "1",
  "message": "Processando pagamento",
  "logger_name": "com.example.PagamentoService",
  "thread_name": "task-1",
  "level": "INFO",
  "pagamentoId": "pay_12345",
  "clienteId": "cli_67890",
  "valor": 99.90,
  "moeda": "BRL"
}
```

### **2. 📈 Métricas (Metrics)**
Métricas são agregações numéricas ao longo do tempo. Elas são eficientes para armazenar e consultar, ideais para dashboards e alertas.

**Java (Micrometer):**
O Micrometer é o `SLF4J` das métricas. É uma fachada que se integra com dezenas de sistemas de monitoramento (Prometheus, CloudWatch, Datadog). O Spring Boot Actuator o utiliza por padrão.

```java
// pom.xml com Spring Boot Actuator e suporte ao CloudWatch
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-cloudwatch2</artifactId>
</dependency>
```

```java
@Service
public class PedidoService {
    private final Counter pedidosCriadosCounter;
    private final Timer processamentoPedidoTimer;

    public PedidoService(MeterRegistry meterRegistry) {
        // Métrica do tipo "Contador"
        this.pedidosCriadosCounter = meterRegistry.counter(
            "pedidos.criados.total",
            "tipo", "online" // "Tags" ou "Dimensões" permitem filtrar a métrica
        );
        // Métrica do tipo "Timer" para medir latência
        this.processamentoPedidoTimer = meterRegistry.timer("pedidos.processamento.latencia");
    }

    public void criarPedido(Pedido pedido) {
        processamentoPedidoTimer.record(() -> {
            // ... lógica de negócio ...
            pedidosCriadosCounter.increment();
        });
    }
}
```
Essas métricas podem ser enviadas automaticamente para o **AWS CloudWatch** para criar dashboards e alarmes.

### **3. ιχ Traces (Rastreamentos Distribuídos)**
Traces são o pilar mais importante para depurar microserviços. Um `trace` é a história completa de uma requisição enquanto ela viaja por múltiplos serviços. Cada `trace` é composto por `spans`.

-   **Trace:** Representa a transação inteira. Possui um `traceId` único.
-   **Span:** Representa uma única unidade de trabalho dentro do trace (ex: uma chamada de API, uma query no banco). Possui um `spanId` e herda o `traceId`. Spans podem ter `spans` filhos.

Juntos, eles formam uma árvore que mostra exatamente onde o tempo foi gasto e onde os erros ocorreram.

---

## 🎯 **Conceito 3: Implementando Tracing Distribuído com OpenTelemetry e AWS X-Ray**

**OpenTelemetry (OTel)** é uma iniciativa da Cloud Native Computing Foundation (CNCF) para padronizar a forma como coletamos e exportamos telemetria (traces, métricas e logs). Ao usar OTel, você evita o *vendor lock-in*, podendo trocar o backend de observabilidade (de X-Ray para Datadog, por exemplo) com uma simples mudança de configuração.

**AWS X-Ray** é o serviço de tracing distribuído da AWS. Ele coleta dados sobre as requisições, constrói um "mapa de serviço" visual e ajuda a identificar gargalos de performance e erros.

### **Passo a Passo: Instrumentando uma Aplicação Java**

#### **1. Adicionar Dependências do OpenTelemetry**

A forma mais fácil é usar o **Agente Java do OpenTelemetry**. É um único arquivo `.jar` que você anexa à sua aplicação na inicialização, sem precisar alterar seu código. Ele instrumenta automaticamente bibliotecas populares (Spring, JDBC, Apache HttpClient, etc.).

```bash
# Exemplo de como iniciar sua aplicação Java com o agente
java -javaagent:path/to/opentelemetry-javaagent.jar \
     -Dotel.service.name=meu-servico-de-pedidos \
     -Dotel.traces.exporter=otlp \
     -Dotel.exporter.otlp.protocol=http/protobuf \
     -Dotel.exporter.otlp.endpoint=http://localhost:4318 \
     -jar minha-aplicacao.jar
```
*Observação: O endpoint `localhost:4318` aponta para um **Coletor OpenTelemetry**, que veremos a seguir.*

#### **2. Coletar e Exportar Dados com o AWS Distro for OpenTelemetry (ADOT) Collector**

Você não envia dados diretamente da sua aplicação para o X-Ray. A melhor prática é usar um **Coletor**. O **ADOT Collector** é a implementação da AWS para o Coletor OTel. Ele roda como um serviço separado (um sidecar em contêineres ou um processo no EC2) e age como um "hub" de telemetria.

**Por que usar um Coletor?**
-   **Desacoplamento:** Sua aplicação só precisa saber como enviar dados para o coletor local. O coletor cuida de autenticação, retries e exportação para a AWS.
-   **Processamento:** O coletor pode enriquecer, filtrar ou agregar dados antes de exportá-los.
-   **Eficiência:** Agrupa dados de múltiplas fontes e os envia em batch, otimizando custos e performance.

**Exemplo de configuração do ADOT Collector (`collector.yaml`):**

```yaml
receivers:
  otlp: # Recebe dados da sua aplicação via protocolo OTLP
    protocols:
      grpc:
      http:

processors:
  batch: # Agrupa dados em lotes para envio eficiente

exporters:
  awsxray: # Exportador que envia os traces para o AWS X-Ray
    region: "sa-east-1"
  awsemf: # Exportador que envia métricas para o CloudWatch como Embedded Metric Format
    region: "sa-east-1"
    namespace: "MeuApp/Microsservicos"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsxray] # Pipeline de traces: OTLP -> Batch -> X-Ray
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [awsemf] # Pipeline de métricas: OTLP -> Batch -> CloudWatch EMF
```

Este coletor está configurado para receber traces e métricas da sua aplicação e exportá-los para X-Ray and CloudWatch, respectivamente.

#### **3. Análise Visual no AWS X-Ray**

Uma vez que os dados de trace chegam ao X-Ray, você obtém superpoderes:

-   **Mapa de Serviço (Service Map):** Uma visualização em tempo real de como seus serviços se conectam, incluindo latência média e taxas de erro para cada conexão.
    ![AWS X-Ray Service Map](https://miro.medium.com/max/1400/1*L8a5B4G8Xy7r9b3Z2A_f9w.png)

-   **Análise de Traces (Traces Analytics):** Você pode mergulhar em um trace individual para ver a "cascata" de chamadas (waterfall view), identificando exatamente qual span está lento ou retornando erro.
    ![AWS X-Ray Trace Waterfall](https://docs.aws.amazon.com/xray/latest/devguide/images/traces-console-timeline.png)

### **Correlacionando os 3 Pilares**

A verdadeira mágica acontece quando você conecta tudo. O OpenTelemetry faz isso injetando o `traceId` e o `spanId` automaticamente nos seus logs.

**Log Estruturado com `traceId`:**
```json
{
  "timestamp": "...",
  "level": "INFO",
  "message": "Processando pagamento",
  "pagamentoId": "pay_12345",
  "trace_id": "1-5f8f8b8a-1b1b1b1b1b1b1b1b1b1b1b1b", // <-- A MÁGICA!
  "span_id": "2c2c2c2c2c2c2c2c"
}
```
Com isso, dentro do X-Ray, você pode clicar em um span com erro e ver **exatamente os logs** gerados durante aquela operação específica. No CloudWatch, você pode encontrar um log de erro e, usando o `trace_id`, pular diretamente para a visualização completa do trace no X-Ray.

---

## 🎯 **Conceito 4: Dashboards e Investigações Práticas no CloudWatch**

Coletar telemetria é apenas metade do trabalho. A outra metade é saber como visualizá-la e extrair insights práticos.

### **1. CloudWatch Logs Insights: O "SQL" para seus Logs**

O CloudWatch Logs Insights oferece uma linguagem de consulta poderosa para analisar seus logs estruturados em tempo real. É a ferramenta mais importante para investigações de causa raiz.

**Cenário:** *Houve um pico de erros no nosso serviço de pagamento. Queremos saber quais clientes foram mais afetados e quais os tipos de erro.*

```sql
# Exemplo de Query no Logs Insights

# Campos que queremos exibir na tabela de resultados
fields @timestamp, @message, pagamentoId, clienteId, error.code
# Filtra logs do nosso serviço, que sejam do nível ERROR, nos últimos 30 minutos
| filter logger_name == 'com.example.PagamentoService' and @logLevel == 'ERROR'
| filter @timestamp > now() - 30m
# Agrupa os resultados por tipo de erro e conta as ocorrências
| stats count(*) as totalErros by error.code
# Ordena para mostrar os erros mais comuns primeiro
| sort totalErros desc
```

**Resultado:**
Uma tabela mostrando exatamente quais códigos de erro (`gateway_timeout`, `insufficient_funds`) aconteceram com mais frequência, permitindo que você foque sua investigação.

### **2. Criando Dashboards Eficazes**

Um bom dashboard conta uma história. Evite dashboards com dezenas de gráficos de CPU. Em vez disso, foque em métricas que reflitam a **saúde do negócio e a experiência do usuário**.

**O Método das 4 Métricas de Ouro (Google SRE):**
Um ótimo ponto de partida para qualquer serviço:

1.  **Latência (Latency):** O tempo que leva para servir uma requisição. (Ex: `pedidos.processamento.latencia` do nosso exemplo com Micrometer).
2.  **Tráfego (Traffic):** A quantidade de demanda que seu sistema está recebendo. (Ex: Requisições por segundo, `pedidos.criados.total`).
3.  **Erros (Errors):** A taxa de requisições que estão falhando. (Ex: Métrica de `http_server_requests` com `status=5xx`).
4.  **Saturação (Saturation):** Quão "cheio" seu serviço está. Representa a utilização do seu recurso mais restrito (CPU, memória, I/O, tamanho da fila).

**Exemplo de Widget para um Dashboard no CloudWatch (Terraform):**

```terraform
resource "aws_cloudwatch_dashboard" "app_dashboard" {
  dashboard_name = "MinhaApp-VisaoGeral"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric",
        width  = 12,
        height = 6,
        properties = {
          title  = "Latência de Processamento de Pedidos (p95)",
          region = "sa-east-1",
          stat   = "p95", # O percentil 95 é mais útil que a média!
          metrics = [
            ["MeuApp/Microsservicos", "pedidos.processamento.latencia", "ServiceName", "servico-de-pedidos"]
          ]
        }
      },
      {
        type   = "metric",
        width  = 12,
        height = 6,
        properties = {
          title  = "Taxa de Erros no Gateway (5xx)",
          region = "sa-east-1",
          stat   = "Sum",
          metrics = [
            ["AWS/ApiGateway", "5XXError", "ApiName", "meu-gateway"]
          ]
        }
      },
      {
        type = "log",
        width = 24,
        height = 8,
        properties = {
            title = "Logs de Erro Recentes (Pagamentos)",
            region = "sa-east-1",
            # Query do Logs Insights diretamente no dashboard!
            query = "SOURCE '/aws/lambda/servico-de-pagamentos' | fields @timestamp, @message | filter @logLevel = 'ERROR' | sort @timestamp desc | limit 20"
        }
      }
    ]
  })
}
```
Este dashboard simples já é extremamente poderoso, pois mostra a **experiência do usuário** (latência), a **saúde do sistema** (erros) e fornece **acesso direto aos logs de erro** para investigação imediata.

---

## 🚀 **Resumo do Módulo e Próximos Passos**

Neste módulo, você aprendeu que:
- Observabilidade é sobre fazer perguntas, não apenas ver painéis.
- Os 3 pilares (Logs, Métricas, Traces) são a base de qualquer sistema observável.
- Logs devem ser **estruturados** para permitir consultas complexas.
- Tracing distribuído com **OpenTelemetry** e **AWS X-Ray** é essencial para depurar microserviços.
- A mágica acontece quando os pilares são **correlacionados** (ex: `traceId` nos logs).
- Dashboards devem focar em **métricas de negócio e experiência do usuário**, não apenas em CPU/memória.

O próximo passo lógico na sua jornada de especialista é aprender a **testar** esses sistemas distribuídos complexos. 