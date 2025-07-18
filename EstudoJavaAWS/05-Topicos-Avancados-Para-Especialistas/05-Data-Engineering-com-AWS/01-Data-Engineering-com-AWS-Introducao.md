# 🔍 Data Engineering com AWS - Uma Introdução para Desenvolvedores Java

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Compreender o que é Data Engineering** e por que ele é relevante para um desenvolvedor backend.
-   ✅ **Conhecer os principais serviços do ecossistema de Analytics da AWS:** Kinesis, Glue, Athena, e S3 como Data Lake.
-   ✅ **Entender o fluxo de um pipeline de dados moderno na AWS**, da ingestão à consulta.
-   ✅ **Escrever uma função Lambda em Java** para realizar um processo de ETL (Extract, Transform, Load) simples.
-   ✅ **Identificar oportunidades** para aplicar conceitos de Data Engineering em sistemas transacionais.

---

## 🎯 **Conceito 1: Onde o Backend Encontra os Dados (Analytics)**

Como desenvolvedor Java, você é um mestre na construção de **sistemas transacionais (OLTP - Online Transaction Processing)**. Seu foco é em baixa latência, alta consistência e no processamento de operações de negócio (criar pedido, atualizar usuário, etc.).

O **Data Engineering** foca em **sistemas analíticos (OLAP - Online Analytical Processing)**. O objetivo aqui é processar grandes volumes de dados para extrair insights, alimentar dashboards de Business Intelligence (BI) ou treinar modelos de Machine Learning. O foco é em alto throughput e na capacidade de realizar queries complexas sobre dados históricos.

**Por que isso importa para você?**
-   Os dados gerados pelos seus microserviços (logs, eventos, registros do banco) são o **combustível** para os pipelines de dados.
-   Cada vez mais, funcionalidades de backend precisam consumir insights gerados por processos de analytics (ex: um sistema de recomendação).
-   Conhecer o básico de Data Engineering permite que você construa serviços que se integram de forma mais eficiente e inteligente com o ecossistema de dados da empresa, tornando seu perfil profissional muito mais completo.

---

## 🎯 **Conceito 2: A Arquitetura de um Pipeline de Dados Moderno na AWS**

Um pipeline de dados típico na AWS pode ser dividido em quatro estágios principais:

1.  **Ingestão (Ingest):** Coletar os dados brutos de diversas fontes.
2.  **Armazenamento (Store):** Armazenar os dados de forma barata, durável e escalável em um **Data Lake**.
3.  **Processamento e Catálogo (Process & Catalog):** Transformar, limpar e enriquecer os dados, e criar um catálogo para que eles possam ser descobertos e consultados.
4.  **Análise e Visualização (Analyze & Visualize):** Executar queries ad-hoc, criar dashboards ou alimentar outras aplicações.

**Mapeando para Serviços AWS:**

| Estágio | Serviço AWS Principal | O que ele faz |
| :--- | :--- | :--- |
| **Ingestão** | **Amazon Kinesis** | Captura e processa streams de dados em tempo real (logs, eventos de clique, dados de IoT). |
| **Armazenamento** | **Amazon S3** | O coração do Data Lake. Armazena virtualmente qualquer volume de dados em seu formato bruto (JSON, CSV, Parquet).|
| **Processamento**| **AWS Glue** (ETL) | Um serviço de ETL serverless. Ele pode "rastrear" (crawl) seus dados no S3, inferir o schema, e executar jobs (em Spark ou Python) para transformar os dados. |
| **Catálogo** | **AWS Glue Data Catalog** | Um "índice" ou metarepositório dos seus dados. Ele armazena as definições das tabelas que apontam para os arquivos no S3. |
| **Análise** | **Amazon Athena** | Permite que você execute **queries SQL padrão diretamente nos arquivos do seu S3**, usando o schema do Glue Data Catalog. É totalmente serverless - você paga apenas pelas queries que executa. |

![AWS Analytics Pipeline](https://d1.awsstatic.com/product-marketing/Big-Data/Analytics_Pipeline_Diagram_updated.b8364c3c3a105220c35593c17885994e0781e64d.png)

---

## 🎯 **Conceito 3: Exemplo Prático - Criando um Mini-ETL com Lambda em Java**

Imagine que nosso serviço de pedidos publica um evento `PedidoCriado` em um tópico SNS. Queremos capturar esses eventos, enriquecê-los com dados do cliente (chamando outro serviço) e salvá-los em um formato otimizado (Parquet) no nosso Data Lake no S3.

```xml
<!-- pom.xml - Dependências para o Handler Lambda -->
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-core</artifactId>
    <version>1.2.1</version>
</dependency>
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-lambda-java-events</artifactId>
    <version>3.11.0</version>
</dependency>
<!-- SDK do S3 -->
<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>s3</artifactId>
    <version>2.17.290</version>
</dependency>
<!-- Biblioteca para trabalhar com JSON -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.13.4</version>
</dependency>
```

```java
// O Handler da nossa função Lambda
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.amazonaws.services.lambda.runtime.events.SNSEvent;
import software.amazon.awssdk.core.sync.RequestBody;
import software.amazon.awssdk.services.s3.S3Client;
import software.amazon.awssdk.services.s3.model.PutObjectRequest;

public class PedidoEtlHandler implements RequestHandler<SNSEvent, String> {

    private final S3Client s3Client = S3Client.builder().build();
    private final ObjectMapper objectMapper = new ObjectMapper();
    private final String BUCKET_NAME = System.getenv("DATA_LAKE_BUCKET");

    @Override
    public String handleRequest(SNSEvent event, Context context) {
        for (SNSEvent.SNSRecord record : event.getRecords()) {
            try {
                // 1. EXTRACT: Pega o dado bruto do evento SNS
                String jsonPayload = record.getSNS().getMessage();
                PedidoCriadoEvent pedidoEvent = objectMapper.readValue(jsonPayload, PedidoCriadoEvent.class);

                // 2. TRANSFORM: Enriquecer o evento
                // Em um caso real, chamaria o serviço de cliente para pegar mais dados
                ClienteInfo clienteInfo = buscarDadosCliente(pedidoEvent.getClienteId());
                
                PedidoEnriquecido pedidoEnriquecido = new PedidoEnriquecido(pedidoEvent, clienteInfo);

                // Converte o objeto enriquecido para JSON (ou Parquet/Avro em um caso real)
                String outputPayload = objectMapper.writeValueAsString(pedidoEnriquecido);

                // 3. LOAD: Salva o dado transformado no Data Lake (S3)
                String key = String.format("pedidos/ano=%d/mes=%d/dia=%d/pedido-%s.json",
                        LocalDate.now().getYear(),
                        LocalDate.now().getMonthValue(),
                        LocalDate.now().getDayOfMonth(),
                        pedidoEvent.getPedidoId());

                PutObjectRequest putObjectRequest = PutObjectRequest.builder()
                        .bucket(BUCKET_NAME)
                        .key(key)
                        .build();

                s3Client.putObject(putObjectRequest, RequestBody.fromString(outputPayload));
                
                context.getLogger().log("Pedido " + pedidoEvent.getPedidoId() + " processado e salvo no S3.");

            } catch (Exception e) {
                context.getLogger().log("Erro ao processar registro: " + e.getMessage());
                // Em um caso real, enviar para uma DLQ
            }
        }
        return "Processamento concluído";
    }
    
    private ClienteInfo buscarDadosCliente(String clienteId) {
        // Simulação de uma chamada a um microserviço de Clientes
        return new ClienteInfo(clienteId, "João da Silva", "São Paulo");
    }
}
```

**O que este código faz:**
-   É acionado por um evento (SNS).
-   **Extrai** o dado do evento.
-   **Transforma** o dado (enriquecendo-o).
-   **Carrega** o resultado no S3, particionando os dados por data (uma prática essencial em Data Lakes para otimizar queries).

Uma vez que os arquivos estão no S3, podemos usar o **AWS Glue Crawler** para catalogá-los e o **Amazon Athena** para consultá-los com SQL, sem precisar de nenhum servidor ou banco de dados.

--- 