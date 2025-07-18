# ☁️ AWS - Serviços e Implementações - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Configurar e usar principais serviços AWS
- ✅ Implementar aplicações Java na nuvem
- ✅ Projetar arquiteturas escaláveis e resilientes
- ✅ Aplicar padrões de segurança e monitoramento
- ✅ Otimizar custos e performance

---

## 🎯 **Conceito 1: Compute Services**

### **🔍 O que são?**
Serviços de computação são como diferentes tipos de máquinas de trabalho: desde computadores pessoais (EC2) até escritórios compartilhados (Lambda) e fábricas automatizadas (ECS).

### **🎓 Conceitos Avançados:**

**1. Amazon EC2 - Elastic Compute Cloud**
```java
// Configuração de instância EC2 com Spring Boot
@Configuration
@Profile("aws")
public class EC2Configuration {
    
    @Bean
    public AmazonEC2 ec2Client() {
        return AmazonEC2ClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
    
    @Bean
    public InstanceMetadataService instanceMetadataService() {
        return new InstanceMetadataService() {
            @Override
            public String getInstanceId() {
                try {
                    return EC2MetadataUtils.getInstanceId();
                } catch (Exception e) {
                    return "local-instance";
                }
            }
            
            @Override
            public String getAvailabilityZone() {
                try {
                    return EC2MetadataUtils.getAvailabilityZone();
                } catch (Exception e) {
                    return "us-east-1a";
                }
            }
        };
    }
}

// Health Check personalizado para EC2
@Component
public class EC2HealthIndicator implements HealthIndicator {
    
    private final AmazonEC2 ec2Client;
    private final InstanceMetadataService metadataService;
    
    @Override
    public Health health() {
        try {
            String instanceId = metadataService.getInstanceId();
            
            DescribeInstancesRequest request = new DescribeInstancesRequest()
                .withInstanceIds(instanceId);
            
            DescribeInstancesResult result = ec2Client.describeInstances(request);
            
            if (!result.getReservations().isEmpty()) {
                Instance instance = result.getReservations().get(0).getInstances().get(0);
                
                return Health.up()
                    .withDetail("instanceId", instanceId)
                    .withDetail("instanceType", instance.getInstanceType())
                    .withDetail("state", instance.getState().getName())
                    .withDetail("availabilityZone", instance.getPlacement().getAvailabilityZone())
                    .build();
            }
            
            return Health.down().withDetail("reason", "Instance not found").build();
            
        } catch (Exception e) {
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```

**2. AWS Lambda - Serverless Computing**
```java
// Lambda Handler com Spring Boot
@Component
public class BankingLambdaHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    private final ObjectMapper objectMapper;
    private final TransactionService transactionService;
    
    public BankingLambdaHandler(ObjectMapper objectMapper, TransactionService transactionService) {
        this.objectMapper = objectMapper;
        this.transactionService = transactionService;
    }
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        try {
            // Log do evento
            context.getLogger().log("Received request: " + input.getBody());
            
            // Parse do request
            TransferRequest request = objectMapper.readValue(input.getBody(), TransferRequest.class);
            
            // Validações
            validateRequest(request);
            
            // Processar transação
            TransactionResult result = transactionService.processTransfer(request);
            
            // Log de sucesso
            context.getLogger().log("Transaction processed successfully: " + result.getTransactionId());
            
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(200)
                .withHeaders(Map.of(
                    "Content-Type", "application/json",
                    "Access-Control-Allow-Origin", "*"
                ))
                .withBody(objectMapper.writeValueAsString(result))
                .build();
                
        } catch (ValidationException e) {
            context.getLogger().log("Validation error: " + e.getMessage());
            
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(400)
                .withBody(createErrorResponse("VALIDATION_ERROR", e.getMessage()))
                .build();
                
        } catch (Exception e) {
            context.getLogger().log("Error processing request: " + e.getMessage());
            
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(500)
                .withBody(createErrorResponse("INTERNAL_ERROR", "Internal server error"))
                .build();
        }
    }
    
    private void validateRequest(TransferRequest request) {
        if (request.getAmount() == null || request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ValidationException("Amount must be positive");
        }
        
        if (StringUtils.isBlank(request.getFromAccountId()) || StringUtils.isBlank(request.getToAccountId())) {
            throw new ValidationException("Account IDs are required");
        }
    }
    
    private String createErrorResponse(String errorCode, String message) {
        try {
            Map<String, Object> error = Map.of(
                "errorCode", errorCode,
                "message", message,
                "timestamp", Instant.now()
            );
            return objectMapper.writeValueAsString(error);
        } catch (Exception e) {
            return "{\"error\":\"Internal server error\"}";
        }
    }
}

// Configuração do Lambda
@Configuration
@ComponentScan(basePackages = "com.bank.lambda")
public class LambdaConfig {
    
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
    
    @Bean
    @Primary
    public AmazonDynamoDB dynamoDBClient() {
        return AmazonDynamoDBClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
}
```

**3. Amazon ECS - Elastic Container Service**
```yaml
# task-definition.json
{
  "family": "banking-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "banking-app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/banking-app:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "SPRING_PROFILES_ACTIVE",
          "value": "aws"
        },
        {
          "name": "AWS_REGION",
          "value": "us-east-1"
        }
      ],
      "secrets": [
        {
          "name": "DATABASE_PASSWORD",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:banking-db-password"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/banking-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

### **💡 Caso de Uso Prático: Sistema Bancário Híbrido**

```java
// Arquitetura híbrida: EC2 + Lambda + ECS
@Service
public class HybridBankingService {
    
    private final LambdaInvokeService lambdaService;
    private final ECSService ecsService;
    private final LoadBalancerService loadBalancerService;
    
    public TransactionResult processTransaction(TransferRequest request) {
        // Decidir onde processar baseado na carga e tipo de transação
        if (isHighValueTransaction(request)) {
            // Transações de alto valor em EC2 para auditoria completa
            return processOnEC2(request);
        } else if (isSimpleTransaction(request)) {
            // Transações simples em Lambda para otimização de custo
            return processOnLambda(request);
        } else {
            // Transações complexas em ECS para escalabilidade
            return processOnECS(request);
        }
    }
    
    private TransactionResult processOnLambda(TransferRequest request) {
        InvokeRequest invokeRequest = InvokeRequest.builder()
            .functionName("banking-transaction-processor")
            .payload(objectMapper.writeValueAsString(request))
            .build();
        
        InvokeResponse response = lambdaService.invoke(invokeRequest);
        
        return objectMapper.readValue(response.payload().asUtf8String(), TransactionResult.class);
    }
    
    private TransactionResult processOnECS(TransferRequest request) {
        // Usar service discovery para encontrar instância disponível
        String serviceEndpoint = ecsService.discoverServiceEndpoint("banking-processor");
        
        return webClient.post()
            .uri(serviceEndpoint + "/process")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(TransactionResult.class)
            .block();
    }
    
    private boolean isHighValueTransaction(TransferRequest request) {
        return request.getAmount().compareTo(BigDecimal.valueOf(50000)) > 0;
    }
    
    private boolean isSimpleTransaction(TransferRequest request) {
        return request.getType() == TransactionType.SIMPLE_TRANSFER &&
               request.getAmount().compareTo(BigDecimal.valueOf(1000)) <= 0;
    }
}
```

---

## 🎯 **Conceito 2: Database Services**

### **🔍 O que são?**
Serviços de banco de dados são como diferentes tipos de arquivos: desde fichários tradicionais (RDS) até sistemas de cartões flexíveis (DynamoDB) e memória rápida (ElastiCache).

### **🎓 Conceitos Avançados:**

**1. Amazon RDS - Relational Database Service**
```java
// Configuração RDS com Spring Boot
@Configuration
@EnableTransactionManagement
public class RDSConfiguration {
    
    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://banking-db.cluster-xyz.us-east-1.rds.amazonaws.com:5432/banking");
        config.setUsername("banking_user");
        config.setPassword(getPasswordFromSecretsManager());
        
        // Configurações de pool
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setLeakDetectionThreshold(60000);
        
        // Configurações específicas do PostgreSQL
        config.addDataSourceProperty("tcpKeepAlive", "true");
        config.addDataSourceProperty("socketTimeout", "30");
        config.addDataSourceProperty("loginTimeout", "10");
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public DataSource readOnlyDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://banking-db-ro.cluster-xyz.us-east-1.rds.amazonaws.com:5432/banking");
        config.setUsername("banking_readonly");
        config.setPassword(getPasswordFromSecretsManager());
        config.setMaximumPoolSize(10);
        config.setReadOnly(true);
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }
    
    private String getPasswordFromSecretsManager() {
        AWSSecretsManager secretsManager = AWSSecretsManagerClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .build();
        
        GetSecretValueRequest request = new GetSecretValueRequest()
            .withSecretId("banking-db-password");
        
        GetSecretValueResult result = secretsManager.getSecretValue(request);
        
        return result.getSecretString();
    }
}

// Service com Read/Write separation
@Service
public class AccountService {
    
    @Autowired
    @Qualifier("dataSource")
    private DataSource writeDataSource;
    
    @Autowired
    @Qualifier("readOnlyDataSource")
    private DataSource readDataSource;
    
    @Transactional
    public Account createAccount(Account account) {
        // Operação de escrita no master
        return accountRepository.save(account);
    }
    
    @Transactional(readOnly = true)
    public Account getAccount(String accountId) {
        // Operação de leitura na réplica
        return accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
    
    @Transactional(readOnly = true)
    public List<Account> searchAccounts(AccountSearchCriteria criteria) {
        // Queries complexas na réplica para não impactar o master
        return accountRepository.findByCriteria(criteria);
    }
}
```

**2. Amazon DynamoDB - NoSQL Database**
```java
// Configuração DynamoDB
@Configuration
@EnableDynamoDBRepositories(basePackages = "com.bank.dynamodb.repository")
public class DynamoDBConfiguration {
    
    @Bean
    @Primary
    public AmazonDynamoDB amazonDynamoDB() {
        return AmazonDynamoDBClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
    
    @Bean
    public DynamoDBMapper dynamoDBMapper(AmazonDynamoDB amazonDynamoDB) {
        DynamoDBMapperConfig config = DynamoDBMapperConfig.builder()
            .withConsistentReads(DynamoDBMapperConfig.ConsistentReads.CONSISTENT)
            .withSaveBehavior(DynamoDBMapperConfig.SaveBehavior.UPDATE_SKIP_NULL_ATTRIBUTES)
            .build();
        
        return new DynamoDBMapper(amazonDynamoDB, config);
    }
}

// Entity para DynamoDB
@DynamoDBTable(tableName = "Transactions")
public class Transaction {
    
    private String transactionId;
    private String accountId;
    private String timestamp;
    private BigDecimal amount;
    private String type;
    private String status;
    private String description;
    private Map<String, String> metadata;
    
    @DynamoDBHashKey(attributeName = "transaction_id")
    public String getTransactionId() {
        return transactionId;
    }
    
    @DynamoDBRangeKey(attributeName = "timestamp")
    public String getTimestamp() {
        return timestamp;
    }
    
    @DynamoDBAttribute(attributeName = "account_id")
    @DynamoDBIndexHashKey(globalSecondaryIndexName = "AccountIndex")
    public String getAccountId() {
        return accountId;
    }
    
    @DynamoDBAttribute(attributeName = "amount")
    @DynamoDBTypeConverted(converter = BigDecimalTypeConverter.class)
    public BigDecimal getAmount() {
        return amount;
    }
    
    @DynamoDBAttribute(attributeName = "metadata")
    public Map<String, String> getMetadata() {
        return metadata;
    }
    
    // Setters...
}

// Repository para DynamoDB
@Repository
public class TransactionRepository {
    
    private final DynamoDBMapper dynamoDBMapper;
    
    public TransactionRepository(DynamoDBMapper dynamoDBMapper) {
        this.dynamoDBMapper = dynamoDBMapper;
    }
    
    public Transaction save(Transaction transaction) {
        dynamoDBMapper.save(transaction);
        return transaction;
    }
    
    public Transaction findById(String transactionId, String timestamp) {
        return dynamoDBMapper.load(Transaction.class, transactionId, timestamp);
    }
    
    public List<Transaction> findByAccountId(String accountId) {
        Map<String, AttributeValue> eav = new HashMap<>();
        eav.put(":accountId", new AttributeValue().withS(accountId));
        
        DynamoDBQueryExpression<Transaction> queryExpression = new DynamoDBQueryExpression<Transaction>()
            .withIndexName("AccountIndex")
            .withConsistentRead(false)
            .withKeyConditionExpression("account_id = :accountId")
            .withExpressionAttributeValues(eav);
        
        return dynamoDBMapper.query(Transaction.class, queryExpression);
    }
    
    public List<Transaction> findByAccountIdAndDateRange(String accountId, String startDate, String endDate) {
        Map<String, AttributeValue> eav = new HashMap<>();
        eav.put(":accountId", new AttributeValue().withS(accountId));
        eav.put(":startDate", new AttributeValue().withS(startDate));
        eav.put(":endDate", new AttributeValue().withS(endDate));
        
        DynamoDBQueryExpression<Transaction> queryExpression = new DynamoDBQueryExpression<Transaction>()
            .withIndexName("AccountIndex")
            .withConsistentRead(false)
            .withKeyConditionExpression("account_id = :accountId")
            .withFilterExpression("#timestamp BETWEEN :startDate AND :endDate")
            .withExpressionAttributeNames(Map.of("#timestamp", "timestamp"))
            .withExpressionAttributeValues(eav);
        
        return dynamoDBMapper.query(Transaction.class, queryExpression);
    }
}
```

**3. Amazon ElastiCache - In-Memory Cache**
```java
// Configuração ElastiCache
@Configuration
@EnableCaching
public class ElastiCacheConfiguration {
    
    @Bean
    public JedisConnectionFactory jedisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName("banking-cache.abcdef.cache.amazonaws.com");
        config.setPort(6379);
        
        JedisConnectionFactory jedisConnectionFactory = new JedisConnectionFactory(config);
        jedisConnectionFactory.afterPropertiesSet();
        
        return jedisConnectionFactory;
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(JedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);
        
        // Configurar serializers
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        
        template.afterPropertiesSet();
        return template;
    }
    
    @Bean
    public CacheManager cacheManager(JedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(15))
            .disableCachingNullValues()
            .computePrefixWith(cacheName -> "banking:" + cacheName + ":")
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .transactionAware()
            .build();
    }
}

// Service com cache distribuído
@Service
public class CustomerService {
    
    private final CustomerRepository customerRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    
    @Cacheable(value = "customers", key = "#customerId")
    public Customer getCustomer(String customerId) {
        return customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
    }
    
    // Cache com pattern de cache-aside
    public CustomerProfile getCustomerProfile(String customerId) {
        String cacheKey = "customer:profile:" + customerId;
        
        // Tentar buscar no cache
        CustomerProfile cachedProfile = (CustomerProfile) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedProfile != null) {
            return cachedProfile;
        }
        
        // Buscar no banco se não estiver no cache
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        CustomerProfile profile = buildCustomerProfile(customer);
        
        // Cachear por 1 hora
        redisTemplate.opsForValue().set(cacheKey, profile, Duration.ofHours(1));
        
        return profile;
    }
    
    // Invalidação de cache inteligente
    @CacheEvict(value = "customers", key = "#customer.id")
    public Customer updateCustomer(Customer customer) {
        Customer updatedCustomer = customerRepository.save(customer);
        
        // Invalidar caches relacionados
        String profileCacheKey = "customer:profile:" + customer.getId();
        redisTemplate.delete(profileCacheKey);
        
        return updatedCustomer;
    }
}
```

### **💡 Caso de Uso Prático: Sistema de Score de Crédito**

```java
// Sistema híbrido usando RDS + DynamoDB + ElastiCache
@Service
public class CreditScoreService {
    
    private final CustomerRepository customerRepository; // RDS
    private final TransactionRepository transactionRepository; // DynamoDB
    private final RedisTemplate<String, Object> redisTemplate; // ElastiCache
    
    public CreditScoreResult calculateCreditScore(String customerId) {
        String cacheKey = "credit:score:" + customerId;
        
        // Verificar cache primeiro
        CreditScoreResult cachedScore = (CreditScoreResult) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedScore != null && !isScoreExpired(cachedScore)) {
            return cachedScore;
        }
        
        // Dados do cliente (RDS)
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        // Histórico de transações (DynamoDB)
        List<Transaction> transactions = transactionRepository.findByAccountIdAndDateRange(
            customer.getPrimaryAccountId(),
            LocalDate.now().minusMonths(12).toString(),
            LocalDate.now().toString()
        );
        
        // Calcular score
        CreditScoreResult score = performCreditScoreCalculation(customer, transactions);
        
        // Cachear por 6 horas
        redisTemplate.opsForValue().set(cacheKey, score, Duration.ofHours(6));
        
        return score;
    }
    
    private CreditScoreResult performCreditScoreCalculation(Customer customer, List<Transaction> transactions) {
        CreditScoreCalculator calculator = new CreditScoreCalculator();
        
        // Análise de dados do cliente
        calculator.addComponent(analyzeCustomerData(customer), 0.3);
        
        // Análise de histórico de transações
        calculator.addComponent(analyzeTransactionHistory(transactions), 0.4);
        
        // Análise de comportamento
        calculator.addComponent(analyzeBehaviorPattern(transactions), 0.3);
        
        return calculator.calculate();
    }
    
    private boolean isScoreExpired(CreditScoreResult score) {
        return score.getCalculatedAt().isBefore(Instant.now().minus(Duration.ofHours(6)));
    }
}
```

---

## 🎯 **Conceito 3: Integration Services**

### **🔍 O que são?**
Serviços de integração são como sistemas de comunicação: desde correios (SQS) até rádio (SNS) e operadoras telefônicas (API Gateway).

### **🎓 Conceitos Avançados:**

**1. Amazon SQS - Simple Queue Service**
```java
// Configuração SQS
@Configuration
@EnableSqs
public class SQSConfiguration {
    
    @Bean
    public AmazonSQSAsync amazonSQS() {
        return AmazonSQSAsyncClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
    
    @Bean
    public QueueMessagingTemplate queueMessagingTemplate(AmazonSQSAsync amazonSQS) {
        return new QueueMessagingTemplate(amazonSQS);
    }
}

// Producer
@Service
public class TransactionEventPublisher {
    
    private final QueueMessagingTemplate queueMessagingTemplate;
    
    @Value("${aws.sqs.transaction-queue}")
    private String transactionQueue;
    
    @Value("${aws.sqs.notification-queue}")
    private String notificationQueue;
    
    public void publishTransactionEvent(TransactionEvent event) {
        try {
            Map<String, Object> headers = new HashMap<>();
            headers.put("eventType", event.getType());
            headers.put("timestamp", System.currentTimeMillis());
            headers.put("source", "banking-service");
            
            queueMessagingTemplate.convertAndSend(transactionQueue, event, headers);
            
            log.info("Transaction event published: {}", event.getTransactionId());
            
        } catch (Exception e) {
            log.error("Failed to publish transaction event: {}", event.getTransactionId(), e);
            throw new MessagePublishException("Failed to publish event", e);
        }
    }
    
    public void publishNotificationEvent(NotificationEvent event) {
        try {
            // Usar delay para notificações programadas
            Map<String, Object> headers = new HashMap<>();
            headers.put("delaySeconds", event.getDelaySeconds());
            
            queueMessagingTemplate.convertAndSend(notificationQueue, event, headers);
            
        } catch (Exception e) {
            log.error("Failed to publish notification event", e);
        }
    }
}

// Consumer
@Component
public class TransactionEventConsumer {
    
    private final TransactionService transactionService;
    private final NotificationService notificationService;
    
    @SqsListener("${aws.sqs.transaction-queue}")
    public void handleTransactionEvent(TransactionEvent event, @Header Map<String, Object> headers) {
        try {
            log.info("Processing transaction event: {}", event.getTransactionId());
            
            switch (event.getType()) {
                case TRANSACTION_CREATED:
                    handleTransactionCreated(event);
                    break;
                case TRANSACTION_COMPLETED:
                    handleTransactionCompleted(event);
                    break;
                case TRANSACTION_FAILED:
                    handleTransactionFailed(event);
                    break;
                default:
                    log.warn("Unknown event type: {}", event.getType());
            }
            
        } catch (Exception e) {
            log.error("Error processing transaction event: {}", event.getTransactionId(), e);
            throw e; // Rejeitar mensagem para reprocessamento
        }
    }
    
    private void handleTransactionCreated(TransactionEvent event) {
        // Enviar notificação de criação
        notificationService.sendTransactionCreatedNotification(event);
        
        // Iniciar processo de validação
        transactionService.startValidationProcess(event.getTransactionId());
    }
    
    private void handleTransactionCompleted(TransactionEvent event) {
        // Enviar notificação de sucesso
        notificationService.sendTransactionCompletedNotification(event);
        
        // Atualizar métricas
        metricsService.incrementTransactionCounter("completed");
    }
    
    private void handleTransactionFailed(TransactionEvent event) {
        // Enviar notificação de erro
        notificationService.sendTransactionFailedNotification(event);
        
        // Processar rollback se necessário
        transactionService.processRollback(event.getTransactionId());
    }
}
```

**2. Amazon SNS - Simple Notification Service**
```java
// Configuração SNS
@Configuration
public class SNSConfiguration {
    
    @Bean
    public AmazonSNS amazonSNS() {
        return AmazonSNSClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
}

// Service para múltiplos canais de notificação
@Service
public class NotificationService {
    
    private final AmazonSNS snsClient;
    private final EmailService emailService;
    private final SMSService smsService;
    
    @Value("${aws.sns.transaction-topic}")
    private String transactionTopic;
    
    public void sendTransactionNotification(TransactionEvent event) {
        try {
            // Criar mensagem estruturada
            Map<String, Object> messageData = Map.of(
                "transactionId", event.getTransactionId(),
                "customerId", event.getCustomerId(),
                "amount", event.getAmount(),
                "type", event.getType(),
                "timestamp", event.getTimestamp()
            );
            
            String message = objectMapper.writeValueAsString(messageData);
            
            // Publicar no tópico SNS
            PublishRequest publishRequest = new PublishRequest()
                .withTopicArn(transactionTopic)
                .withMessage(message)
                .withMessageAttributes(Map.of(
                    "eventType", new MessageAttributeValue()
                        .withDataType("String")
                        .withStringValue(event.getType().name()),
                    "priority", new MessageAttributeValue()
                        .withDataType("String")
                        .withStringValue(event.getPriority().name())
                ));
            
            PublishResult result = snsClient.publish(publishRequest);
            
            log.info("Transaction notification published: messageId={}, transactionId={}", 
                result.getMessageId(), event.getTransactionId());
            
        } catch (Exception e) {
            log.error("Failed to publish transaction notification: {}", event.getTransactionId(), e);
            
            // Fallback para notificação direta
            sendDirectNotification(event);
        }
    }
    
    private void sendDirectNotification(TransactionEvent event) {
        try {
            // Enviar email como fallback
            emailService.sendTransactionNotification(event);
            
            // Enviar SMS para transações de alto valor
            if (event.getAmount().compareTo(BigDecimal.valueOf(5000)) > 0) {
                smsService.sendTransactionAlert(event);
            }
            
        } catch (Exception e) {
            log.error("Failed to send direct notification: {}", event.getTransactionId(), e);
        }
    }
}
```

**3. Amazon API Gateway**
```java
// Configuração para integração com API Gateway
@RestController
@RequestMapping("/api/gateway")
public class APIGatewayController {
    
    private final TransactionService transactionService;
    private final AuthenticationService authService;
    
    // Endpoint para API Gateway com validação de API Key
    @PostMapping("/transactions")
    @PreAuthorize("hasRole('API_CLIENT')")
    public ResponseEntity<TransactionResponse> processTransaction(
            @Valid @RequestBody TransactionRequest request,
            @RequestHeader("X-API-Key") String apiKey,
            @RequestHeader("X-Client-ID") String clientId,
            HttpServletRequest httpRequest) {
        
        try {
            // Validar API Key
            if (!authService.validateApiKey(apiKey, clientId)) {
                return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body(new TransactionResponse("UNAUTHORIZED", "Invalid API key"));
            }
            
            // Log da requisição
            log.info("API Gateway request: clientId={}, sourceIp={}, userAgent={}", 
                clientId, 
                httpRequest.getRemoteAddr(), 
                httpRequest.getHeader("User-Agent"));
            
            // Processar transação
            TransactionResult result = transactionService.processTransactionFromAPI(request, clientId);
            
            return ResponseEntity.ok(new TransactionResponse(result));
            
        } catch (ValidationException e) {
            log.warn("Validation error for API client {}: {}", clientId, e.getMessage());
            return ResponseEntity.badRequest()
                .body(new TransactionResponse("VALIDATION_ERROR", e.getMessage()));
                
        } catch (Exception e) {
            log.error("Error processing API Gateway request for client {}: {}", clientId, e.getMessage(), e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new TransactionResponse("INTERNAL_ERROR", "Internal server error"));
        }
    }
    
    // Health check para API Gateway
    @GetMapping("/health")
    public ResponseEntity<Map<String, Object>> health() {
        Map<String, Object> health = Map.of(
            "status", "UP",
            "timestamp", Instant.now(),
            "version", "1.0.0"
        );
        
        return ResponseEntity.ok(health);
    }
}
```

### **💡 Caso de Uso Prático: Sistema de Notificações Bancárias**

```java
// Sistema completo de notificações usando SQS + SNS
@Service
public class BankingNotificationSystem {
    
    private final QueueMessagingTemplate queueTemplate;
    private final AmazonSNS snsClient;
    private final CustomerPreferenceService preferenceService;
    
    public void processTransactionNotification(TransactionEvent event) {
        try {
            // Buscar preferências do cliente
            CustomerNotificationPreferences preferences = 
                preferenceService.getPreferences(event.getCustomerId());
            
            // Criar notificação baseada no tipo de transação
            NotificationMessage notification = createNotificationMessage(event, preferences);
            
            // Decidir canal de notificação
            if (isUrgentNotification(event)) {
                // Notificação urgente - usar SNS para múltiplos canais
                publishUrgentNotification(notification);
            } else {
                // Notificação normal - usar SQS para processamento assíncrono
                queueNormalNotification(notification);
            }
            
        } catch (Exception e) {
            log.error("Failed to process transaction notification: {}", event.getTransactionId(), e);
        }
    }
    
    private void publishUrgentNotification(NotificationMessage notification) {
        // Publicar no tópico SNS para notificação imediata
        PublishRequest request = new PublishRequest()
            .withTopicArn("arn:aws:sns:us-east-1:123456789012:urgent-notifications")
            .withMessage(notification.getMessage())
            .withMessageAttributes(Map.of(
                "notificationType", new MessageAttributeValue()
                    .withDataType("String")
                    .withStringValue("URGENT"),
                "customerId", new MessageAttributeValue()
                    .withDataType("String")
                    .withStringValue(notification.getCustomerId())
            ));
        
        snsClient.publish(request);
    }
    
    private void queueNormalNotification(NotificationMessage notification) {
        // Enfileirar para processamento em lote
        queueTemplate.convertAndSend("normal-notifications-queue", notification);
    }
    
    private boolean isUrgentNotification(TransactionEvent event) {
        return event.getAmount().compareTo(BigDecimal.valueOf(10000)) > 0 ||
               event.getType() == TransactionType.SUSPICIOUS_ACTIVITY ||
               event.getPriority() == Priority.HIGH;
    }
}
```

---

## 🎯 **Exercícios Práticos**

### **Exercício 1: Arquitetura Multi-Tier**
**Cenário:** Projetar sistema bancário com alta disponibilidade.

**Sua tarefa:**
- Usar EC2 com Auto Scaling
- RDS Multi-AZ
- ElastiCache para sessões
- S3 para documentos

### **Exercício 2: Sistema Serverless**
**Cenário:** API de consulta de saldo com milhões de requisições.

**Sua tarefa:**
- Lambda + API Gateway
- DynamoDB para dados
- CloudWatch para monitoramento
- Otimização de cold start

### **Exercício 3: Event-Driven Architecture**
**Cenário:** Sistema de notificações bancárias em tempo real.

**Sua tarefa:**
- SQS para filas
- SNS para broadcast
- Lambda para processamento
- Dead Letter Queues para falhas

---

## 🚀 **Próximos Passos**

1. **Pratique com LocalStack** - Simule AWS localmente
2. **Estude DevOps** - Próximo módulo: 04-DevOps-CICD.md
3. **Implemente um projeto** - Combine todos os serviços
4. **Otimize custos** - Aprenda billing e cost optimization

**Lembre-se:** AWS é sobre escolher o serviço certo para o problema certo! 