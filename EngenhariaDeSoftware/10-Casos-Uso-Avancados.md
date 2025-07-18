# 🎯 Casos de Uso Avançados - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Resolver problemas complexos de sistemas distribuídos
- ✅ Responder perguntas comportamentais de entrevista
- ✅ Projetar soluções para cenários reais
- ✅ Demonstrar conhecimento prático e teórico
- ✅ Discutir trade-offs e decisões arquiteturais

---

## 🎯 **Caso de Uso 1: Sistema de E-commerce com Alta Escala**

### **🔍 Cenário**
Você foi contratado para arquitetar um sistema de e-commerce que precisa suportar:
- 10 milhões de usuários ativos
- 100 mil transações por minuto no Black Friday
- Disponibilidade 99.99%
- Latência < 200ms
- Expansão global

### **🎓 Solução Arquitetural:**

**1. Arquitetura Geral**
```java
// Gateway de API com rate limiting
@RestController
@RequestMapping("/api/v1")
public class ECommerceGateway {
    
    private final OrderService orderService;
    private final ProductService productService;
    private final UserService userService;
    private final RateLimitService rateLimitService;
    
    @PostMapping("/orders")
    @RateLimited(requestsPerMinute = 10, keyGenerator = "userIdKeyGenerator")
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody @Valid CreateOrderRequest request,
            @RequestHeader("X-User-ID") String userId) {
        
        // Validar rate limit
        if (!rateLimitService.isAllowed(userId, "create-order")) {
            return ResponseEntity.status(429)
                .body(new OrderResponse("Rate limit exceeded"));
        }
        
        // Processar pedido de forma assíncrona
        OrderResult result = orderService.createOrderAsync(request, userId);
        
        return ResponseEntity.accepted()
            .body(new OrderResponse(result.getOrderId(), "Order being processed"));
    }
}

// Service de pedidos com padrão CQRS
@Service
public class OrderService {
    
    private final OrderCommandService commandService;
    private final OrderQueryService queryService;
    private final EventPublisher eventPublisher;
    
    public OrderResult createOrderAsync(CreateOrderRequest request, String userId) {
        // Criar comando
        CreateOrderCommand command = new CreateOrderCommand(
            OrderId.generate(),
            userId,
            request.getItems(),
            request.getShippingAddress()
        );
        
        // Publicar evento para processamento assíncrono
        eventPublisher.publishEvent(new OrderCreationRequestedEvent(command));
        
        return OrderResult.accepted(command.getOrderId());
    }
}

// Event Handler para processamento assíncrono
@Component
public class OrderEventHandler {
    
    private final InventoryService inventoryService;
    private final PaymentService paymentService;
    private final ShippingService shippingService;
    
    @EventListener
    @Async("orderProcessingExecutor")
    public void handleOrderCreationRequested(OrderCreationRequestedEvent event) {
        CreateOrderCommand command = event.getCommand();
        
        try {
            // Saga pattern para coordenar múltiplos serviços
            OrderSaga saga = new OrderSaga(command.getOrderId());
            
            // Reservar estoque
            InventoryReservationResult inventoryResult = inventoryService.reserveItems(
                command.getItems()
            );
            
            if (!inventoryResult.isSuccess()) {
                saga.fail("Estoque insuficiente");
                return;
            }
            
            // Processar pagamento
            PaymentResult paymentResult = paymentService.processPayment(
                command.getCustomerId(),
                command.getTotalAmount()
            );
            
            if (!paymentResult.isSuccess()) {
                // Compensar reserva de estoque
                inventoryService.releaseReservation(inventoryResult.getReservationId());
                saga.fail("Pagamento falhou");
                return;
            }
            
            // Criar pedido de envio
            ShippingResult shippingResult = shippingService.createShippingOrder(
                command.getShippingAddress(),
                command.getItems()
            );
            
            if (!shippingResult.isSuccess()) {
                // Compensar pagamento e estoque
                paymentService.refundPayment(paymentResult.getPaymentId());
                inventoryService.releaseReservation(inventoryResult.getReservationId());
                saga.fail("Erro no envio");
                return;
            }
            
            // Finalizar pedido
            Order order = new Order(
                command.getOrderId(),
                command.getCustomerId(),
                command.getItems(),
                OrderStatus.CONFIRMED
            );
            
            orderRepository.save(order);
            saga.complete();
            
            // Notificar sucesso
            eventPublisher.publishEvent(new OrderCompletedEvent(order));
            
        } catch (Exception e) {
            log.error("Erro ao processar pedido: {}", command.getOrderId(), e);
            eventPublisher.publishEvent(new OrderFailedEvent(command.getOrderId(), e.getMessage()));
        }
    }
}
```

**2. Estratégias de Escalabilidade**
```java
// Cache distribuído para dados frequentemente acessados
@Service
public class ProductCacheService {
    
    private final RedisTemplate<String, Product> productRedisTemplate;
    private final ProductRepository productRepository;
    
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }
    
    // Cache com warm-up para produtos populares
    @Scheduled(fixedRate = 300000) // 5 minutos
    public void warmUpPopularProducts() {
        List<String> popularProductIds = productRepository.findPopularProductIds(100);
        
        popularProductIds.parallelStream().forEach(productId -> {
            try {
                Product product = productRepository.findById(productId).orElse(null);
                if (product != null) {
                    String cacheKey = "products::" + productId;
                    productRedisTemplate.opsForValue().set(cacheKey, product, Duration.ofMinutes(30));
                }
            } catch (Exception e) {
                log.warn("Erro ao aquecer cache do produto {}: {}", productId, e.getMessage());
            }
        });
    }
}

// Padrão Circuit Breaker para serviços externos
@Component
public class PaymentServiceClient {
    
    private final CircuitBreaker circuitBreaker;
    private final RestTemplate restTemplate;
    
    public PaymentServiceClient() {
        this.circuitBreaker = CircuitBreaker.ofDefaults("payment-service");
        this.circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                log.info("Payment service circuit breaker state: {}", event.getStateTransition()));
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        return circuitBreaker.executeSupplier(() -> {
            try {
                ResponseEntity<PaymentResponse> response = restTemplate.postForEntity(
                    "/payments/process",
                    request,
                    PaymentResponse.class
                );
                
                return PaymentResult.success(response.getBody());
                
            } catch (HttpClientErrorException e) {
                if (e.getStatusCode().is4xxClientError()) {
                    return PaymentResult.failure("Erro de validação: " + e.getMessage());
                }
                throw e;
            }
        });
    }
}

// Database sharding por região
@Configuration
public class ShardedDataSourceConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        
        // Shard por região
        targetDataSources.put("us-east", createDataSource("us-east-db"));
        targetDataSources.put("us-west", createDataSource("us-west-db"));
        targetDataSources.put("eu-west", createDataSource("eu-west-db"));
        targetDataSources.put("asia-pacific", createDataSource("ap-db"));
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(targetDataSources);
        routingDataSource.setDefaultTargetDataSource(targetDataSources.get("us-east"));
        
        return routingDataSource;
    }
    
    private DataSource createDataSource(String region) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://" + region + ".amazonaws.com:5432/ecommerce");
        config.setUsername("app_user");
        config.setPassword(getPasswordFromVault(region));
        config.setMaximumPoolSize(50);
        config.setMinimumIdle(10);
        
        return new HikariDataSource(config);
    }
}
```

**3. Estratégias de Deploy para Black Friday**
```java
// Configuração de auto-scaling
@Component
public class AutoScalingManager {
    
    private final CloudWatchClient cloudWatchClient;
    private final ECSClient ecsClient;
    private final ApplicationLoadBalancer loadBalancer;
    
    @Scheduled(fixedRate = 30000) // 30 segundos
    public void checkAndScale() {
        // Métricas de CPU e memória
        double avgCpuUtilization = getAverageCpuUtilization();
        double avgMemoryUtilization = getAverageMemoryUtilization();
        
        // Métricas de negócio
        int requestsPerSecond = getCurrentRequestsPerSecond();
        double errorRate = getCurrentErrorRate();
        
        // Métricas de latência
        double avgResponseTime = getAverageResponseTime();
        
        // Decisão de scaling
        if (shouldScaleUp(avgCpuUtilization, requestsPerSecond, avgResponseTime)) {
            scaleUp();
        } else if (shouldScaleDown(avgCpuUtilization, requestsPerSecond)) {
            scaleDown();
        }
    }
    
    private boolean shouldScaleUp(double cpuUtilization, int requestsPerSecond, double responseTime) {
        return cpuUtilization > 70 || 
               requestsPerSecond > 1000 || 
               responseTime > 200;
    }
    
    private void scaleUp() {
        int currentInstances = getCurrentInstanceCount();
        int maxInstances = getMaxInstanceCount();
        
        if (currentInstances < maxInstances) {
            int newInstanceCount = Math.min(currentInstances + 2, maxInstances);
            updateDesiredCapacity(newInstanceCount);
            
            log.info("Scaling up from {} to {} instances", currentInstances, newInstanceCount);
        }
    }
}

// Preparação pré-Black Friday
@Component
public class BlackFridayPreparation {
    
    @EventListener
    public void prepareForBlackFriday(BlackFridayPreparationEvent event) {
        // Warm-up de caches
        warmUpCaches();
        
        // Pré-carregamento de dados frequentes
        preloadFrequentData();
        
        // Scaling preventivo
        scaleUpInfrastructure();
        
        // Configuração de rate limiting mais restritiva
        updateRateLimits();
        
        // Ativar modo de alta disponibilidade
        enableHighAvailabilityMode();
    }
    
    private void warmUpCaches() {
        // Cache de produtos mais vendidos
        productCacheService.warmUpPopularProducts();
        
        // Cache de categorias
        categoryCacheService.warmUpCategories();
        
        // Cache de promoções ativas
        promotionCacheService.warmUpActivePromotions();
    }
    
    private void preloadFrequentData() {
        // Carregar dados de estoque em memória
        inventoryService.preloadInventoryData();
        
        // Carregar configurações de preços
        pricingService.preloadPricingRules();
    }
    
    private void scaleUpInfrastructure() {
        // Aumentar instâncias de aplicação
        applicationAutoScaler.setDesiredCapacity(50);
        
        // Aumentar instâncias de banco
        databaseAutoScaler.setDesiredCapacity(10);
        
        // Aumentar instâncias de cache
        cacheAutoScaler.setDesiredCapacity(20);
    }
}
```

---

## 🎯 **Caso de Uso 2: Sistema de Notificações em Tempo Real**

### **🔍 Cenário**
Implementar sistema de notificações que precisa:
- Enviar 1 milhão de notificações por minuto
- Suportar múltiplos canais (email, SMS, push, WhatsApp)
- Priorização de mensagens
- Retry inteligente
- Auditoria completa

### **🎓 Solução Técnica:**

```java
// Sistema de notificações com padrão Strategy
@Service
public class NotificationService {
    
    private final Map<NotificationChannel, NotificationHandler> handlers;
    private final NotificationQueue notificationQueue;
    private final NotificationAuditService auditService;
    
    public NotificationService(List<NotificationHandler> handlers,
                              NotificationQueue notificationQueue,
                              NotificationAuditService auditService) {
        this.handlers = handlers.stream()
            .collect(Collectors.toMap(
                NotificationHandler::getChannel,
                Function.identity()
            ));
        this.notificationQueue = notificationQueue;
        this.auditService = auditService;
    }
    
    public NotificationResult sendNotification(NotificationRequest request) {
        try {
            // Validar request
            validateNotificationRequest(request);
            
            // Criar notificação
            Notification notification = createNotification(request);
            
            // Aplicar regras de prioridade
            notification = applyPriorityRules(notification);
            
            // Verificar se deve ser enviada imediatamente ou enfileirada
            if (notification.getPriority() == NotificationPriority.IMMEDIATE) {
                return sendImmediately(notification);
            } else {
                return enqueueForProcessing(notification);
            }
            
        } catch (Exception e) {
            log.error("Erro ao enviar notificação: {}", request, e);
            return NotificationResult.failure("Erro interno: " + e.getMessage());
        }
    }
    
    private NotificationResult sendImmediately(Notification notification) {
        NotificationHandler handler = handlers.get(notification.getChannel());
        
        if (handler == null) {
            throw new UnsupportedNotificationChannelException(
                "Canal não suportado: " + notification.getChannel());
        }
        
        // Auditoria - tentativa de envio
        auditService.logNotificationAttempt(notification);
        
        try {
            NotificationResult result = handler.send(notification);
            
            if (result.isSuccess()) {
                auditService.logNotificationSuccess(notification, result);
            } else {
                auditService.logNotificationFailure(notification, result);
                
                // Retry se necessário
                if (shouldRetry(notification, result)) {
                    scheduleRetry(notification);
                }
            }
            
            return result;
            
        } catch (Exception e) {
            auditService.logNotificationError(notification, e);
            throw e;
        }
    }
    
    private boolean shouldRetry(Notification notification, NotificationResult result) {
        return notification.getRetryCount() < getMaxRetries(notification.getChannel()) &&
               isRetryableError(result.getErrorCode());
    }
    
    private void scheduleRetry(Notification notification) {
        notification.incrementRetryCount();
        
        // Backoff exponencial
        long delayMinutes = (long) Math.pow(2, notification.getRetryCount());
        LocalDateTime retryAt = LocalDateTime.now().plusMinutes(delayMinutes);
        
        notificationQueue.scheduleNotification(notification, retryAt);
    }
}

// Handlers específicos para cada canal
@Component
public class EmailNotificationHandler implements NotificationHandler {
    
    private final EmailService emailService;
    private final EmailTemplateService templateService;
    
    @Override
    public NotificationResult send(Notification notification) {
        try {
            // Preparar dados do template
            Map<String, Object> templateData = notification.getTemplateData();
            
            // Renderizar template
            String subject = templateService.renderSubject(
                notification.getTemplate(), templateData);
            String body = templateService.renderBody(
                notification.getTemplate(), templateData);
            
            // Criar email
            EmailMessage email = EmailMessage.builder()
                .to(notification.getRecipient())
                .subject(subject)
                .body(body)
                .priority(mapPriority(notification.getPriority()))
                .build();
            
            // Enviar
            EmailResult result = emailService.sendEmail(email);
            
            return NotificationResult.success(result.getMessageId());
            
        } catch (Exception e) {
            return NotificationResult.failure("Erro no envio de email: " + e.getMessage());
        }
    }
    
    @Override
    public NotificationChannel getChannel() {
        return NotificationChannel.EMAIL;
    }
}

@Component
public class PushNotificationHandler implements NotificationHandler {
    
    private final FCMService fcmService;
    private final APNSService apnsService;
    private final DeviceService deviceService;
    
    @Override
    public NotificationResult send(Notification notification) {
        try {
            // Buscar dispositivos do usuário
            List<Device> devices = deviceService.getUserDevices(notification.getRecipient());
            
            if (devices.isEmpty()) {
                return NotificationResult.failure("Nenhum dispositivo encontrado");
            }
            
            List<NotificationResult> results = new ArrayList<>();
            
            for (Device device : devices) {
                NotificationResult result = sendToDevice(notification, device);
                results.add(result);
            }
            
            // Considerar sucesso se pelo menos um dispositivo recebeu
            boolean hasSuccess = results.stream().anyMatch(NotificationResult::isSuccess);
            
            if (hasSuccess) {
                return NotificationResult.success("Enviado para pelo menos um dispositivo");
            } else {
                return NotificationResult.failure("Falha em todos os dispositivos");
            }
            
        } catch (Exception e) {
            return NotificationResult.failure("Erro no envio push: " + e.getMessage());
        }
    }
    
    private NotificationResult sendToDevice(Notification notification, Device device) {
        try {
            switch (device.getPlatform()) {
                case IOS:
                    return sendToiOS(notification, device);
                case ANDROID:
                    return sendToAndroid(notification, device);
                default:
                    return NotificationResult.failure("Platform não suportada");
            }
        } catch (Exception e) {
            return NotificationResult.failure("Erro no dispositivo: " + e.getMessage());
        }
    }
}

// Processamento em lote para alta performance
@Component
public class BatchNotificationProcessor {
    
    private final List<NotificationHandler> handlers;
    private final NotificationQueue notificationQueue;
    
    @Scheduled(fixedDelay = 1000) // 1 segundo
    public void processBatch() {
        // Buscar notificações para processamento
        List<Notification> notifications = notificationQueue.getNotificationsForProcessing(100);
        
        if (notifications.isEmpty()) {
            return;
        }
        
        // Agrupar por canal para processamento em lote
        Map<NotificationChannel, List<Notification>> groupedNotifications = 
            notifications.stream()
                .collect(Collectors.groupingBy(Notification::getChannel));
        
        // Processar cada grupo em paralelo
        groupedNotifications.entrySet().parallelStream()
            .forEach(entry -> {
                NotificationChannel channel = entry.getKey();
                List<Notification> channelNotifications = entry.getValue();
                
                NotificationHandler handler = getHandler(channel);
                if (handler instanceof BatchNotificationHandler) {
                    ((BatchNotificationHandler) handler).sendBatch(channelNotifications);
                } else {
                    // Processar individualmente
                    channelNotifications.forEach(notification -> {
                        try {
                            handler.send(notification);
                        } catch (Exception e) {
                            log.error("Erro ao processar notificação {}: {}", 
                                notification.getId(), e.getMessage());
                        }
                    });
                }
            });
    }
}
```

---

## 🎯 **Caso de Uso 3: Sistema de Análise de Dados em Tempo Real**

### **🔍 Cenário**
Construir plataforma de analytics que processa:
- 10 TB de dados por dia
- Análise em tempo real (< 1 segundo)
- Machine Learning para previsões
- Dashboards interativos
- Alertas automáticos

### **🎓 Arquitetura de Dados:**

```java
// Stream Processing com Kafka Streams
@Component
public class RealTimeAnalyticsProcessor {
    
    @StreamListener("user-events")
    public void processUserEvents(UserEvent event) {
        // Enriquecer evento com dados do usuário
        EnrichedUserEvent enrichedEvent = enrichUserEvent(event);
        
        // Detectar padrões em tempo real
        PatternDetectionResult patterns = detectPatterns(enrichedEvent);
        
        // Atualizar métricas em tempo real
        updateRealTimeMetrics(enrichedEvent);
        
        // Verificar alertas
        checkAlerts(enrichedEvent, patterns);
        
        // Salvar para análise histórica
        saveToDataLake(enrichedEvent);
    }
    
    private EnrichedUserEvent enrichUserEvent(UserEvent event) {
        // Buscar dados do usuário do cache
        UserProfile profile = userProfileCache.get(event.getUserId());
        
        // Buscar dados de sessão
        SessionData session = sessionDataService.getSession(event.getSessionId());
        
        // Enriquecer evento
        return EnrichedUserEvent.builder()
            .originalEvent(event)
            .userProfile(profile)
            .sessionData(session)
            .geoLocation(geoLocationService.getLocation(event.getIpAddress()))
            .deviceInfo(deviceInfoService.getDeviceInfo(event.getUserAgent()))
            .build();
    }
    
    private PatternDetectionResult detectPatterns(EnrichedUserEvent event) {
        List<Pattern> detectedPatterns = new ArrayList<>();
        
        // Detectar comportamento anômalo
        if (anomalyDetector.isAnomaly(event)) {
            detectedPatterns.add(new Pattern(PatternType.ANOMALY, "Comportamento anômalo detectado"));
        }
        
        // Detectar padrão de churn
        if (churnPredictor.isChurnRisk(event)) {
            detectedPatterns.add(new Pattern(PatternType.CHURN_RISK, "Risco de churn detectado"));
        }
        
        // Detectar oportunidade de upsell
        if (upsellDetector.hasUpsellOpportunity(event)) {
            detectedPatterns.add(new Pattern(PatternType.UPSELL_OPPORTUNITY, "Oportunidade de upsell"));
        }
        
        return new PatternDetectionResult(detectedPatterns);
    }
}

// Machine Learning Pipeline
@Service
public class MLPipelineService {
    
    private final FeatureStore featureStore;
    private final ModelRegistry modelRegistry;
    private final PredictionCache predictionCache;
    
    public PredictionResult predict(String modelName, Map<String, Object> features) {
        try {
            // Buscar modelo
            MLModel model = modelRegistry.getModel(modelName);
            
            if (model == null) {
                throw new ModelNotFoundException("Modelo não encontrado: " + modelName);
            }
            
            // Verificar cache de predições
            String cacheKey = generateCacheKey(modelName, features);
            PredictionResult cachedResult = predictionCache.get(cacheKey);
            
            if (cachedResult != null) {
                return cachedResult;
            }
            
            // Preparar features
            FeatureVector featureVector = prepareFeatures(features, model.getFeatureSchema());
            
            // Fazer predição
            PredictionResult result = model.predict(featureVector);
            
            // Cachear resultado
            predictionCache.put(cacheKey, result, Duration.ofMinutes(5));
            
            // Log para monitoramento
            logPrediction(modelName, features, result);
            
            return result;
            
        } catch (Exception e) {
            log.error("Erro na predição do modelo {}: {}", modelName, e.getMessage());
            throw new PredictionException("Erro na predição", e);
        }
    }
    
    // Retreinamento automático
    @Scheduled(cron = "0 0 2 * * *") // Todo dia às 2h
    public void retrainModels() {
        List<MLModel> models = modelRegistry.getAllModels();
        
        for (MLModel model : models) {
            try {
                if (shouldRetrain(model)) {
                    retrainModel(model);
                }
            } catch (Exception e) {
                log.error("Erro ao retreinar modelo {}: {}", model.getName(), e.getMessage());
            }
        }
    }
    
    private boolean shouldRetrain(MLModel model) {
        // Verificar drift nos dados
        DataDriftResult driftResult = dataDriftDetector.checkDrift(model);
        
        if (driftResult.hasDrift()) {
            return true;
        }
        
        // Verificar performance do modelo
        ModelPerformance performance = performanceMonitor.getPerformance(model);
        
        return performance.getAccuracy() < model.getMinAccuracy();
    }
    
    private void retrainModel(MLModel model) {
        log.info("Iniciando retreinamento do modelo: {}", model.getName());
        
        // Buscar dados de treino
        TrainingData trainingData = featureStore.getTrainingData(
            model.getName(), 
            LocalDateTime.now().minusDays(30)
        );
        
        // Treinar novo modelo
        MLModel newModel = modelTrainer.train(model.getConfiguration(), trainingData);
        
        // Validar novo modelo
        ValidationResult validation = modelValidator.validate(newModel, trainingData);
        
        if (validation.isValid()) {
            // Substituir modelo em produção
            modelRegistry.replaceModel(model.getName(), newModel);
            
            log.info("Modelo {} retreinado com sucesso", model.getName());
        } else {
            log.warn("Retreinamento do modelo {} falhou na validação", model.getName());
        }
    }
}
```

---

## 🎯 **Perguntas Comportamentais e Técnicas**

### **1. "Conte sobre um projeto desafiador que você liderou"**

**Estrutura da Resposta (STAR):**
- **Situação**: Descreva o contexto e o desafio
- **Tarefa**: Explique sua responsabilidade
- **Ação**: Detalhe o que você fez
- **Resultado**: Mostre o impacto

**Exemplo de Resposta:**
```
"Liderei a migração de um sistema monolítico de um banco para microserviços, que 
processava 100 mil transações por dia. O sistema estava enfrentando problemas de 
escalabilidade e disponibilidade.

Minha responsabilidade era arquitetar a nova solução e liderar uma equipe de 8 
desenvolvedores. Criei uma estratégia de migração gradual usando o padrão Strangler 
Fig, implementei Event Sourcing para auditoria, e estabelecemos CI/CD com deploy 
automatizado.

O resultado foi uma redução de 60% no tempo de resposta, aumento da disponibilidade 
para 99.9%, e capacidade de processar 500 mil transações por dia."
```

### **2. "Como você lidaria com um problema de performance em produção?"**

**Abordagem Estruturada:**
1. **Identificação**: Usar ferramentas de monitoramento
2. **Análise**: Investigar métricas e logs
3. **Diagnóstico**: Encontrar a causa raiz
4. **Solução**: Implementar correção
5. **Validação**: Verificar resolução
6. **Prevenção**: Implementar melhorias

**Exemplo de Resposta:**
```java
// Processo de investigação
@Component
public class PerformanceInvestigator {
    
    public void investigatePerformanceIssue(PerformanceAlert alert) {
        // 1. Coletar métricas
        SystemMetrics metrics = metricsCollector.collect();
        
        // 2. Analisar logs
        List<LogEntry> errorLogs = logAnalyzer.getErrorLogs(
            alert.getStartTime(), 
            alert.getEndTime()
        );
        
        // 3. Verificar database
        DatabaseMetrics dbMetrics = databaseMonitor.getMetrics();
        
        // 4. Analisar traces distribuídos
        List<Trace> traces = tracingService.getSlowTraces();
        
        // 5. Criar relatório
        PerformanceReport report = createPerformanceReport(
            metrics, errorLogs, dbMetrics, traces
        );
        
        // 6. Sugerir ações
        List<ActionItem> actions = generateActionItems(report);
        
        // 7. Implementar correções
        implementCorrections(actions);
    }
}
```

### **3. "Explique como você projetaria um sistema de cache distribuído"**

**Resposta Técnica:**
```java
// Arquitetura de cache distribuído
@Service
public class DistributedCacheService {
    
    private final List<CacheNode> cacheNodes;
    private final ConsistentHashRing hashRing;
    private final ReplicationManager replicationManager;
    
    public <T> T get(String key, Class<T> valueType) {
        // Determinar nó usando consistent hashing
        CacheNode primaryNode = hashRing.getNode(key);
        
        try {
            // Tentar buscar no nó primário
            T value = primaryNode.get(key, valueType);
            
            if (value != null) {
                return value;
            }
            
            // Buscar em réplicas se não encontrar
            List<CacheNode> replicas = replicationManager.getReplicas(primaryNode);
            
            for (CacheNode replica : replicas) {
                value = replica.get(key, valueType);
                if (value != null) {
                    // Reparar dado no nó primário
                    primaryNode.put(key, value);
                    return value;
                }
            }
            
            return null;
            
        } catch (Exception e) {
            // Failover para réplicas
            return getFromReplicas(key, valueType, primaryNode);
        }
    }
    
    public void put(String key, Object value, Duration ttl) {
        CacheNode primaryNode = hashRing.getNode(key);
        
        // Escrever no nó primário
        primaryNode.put(key, value, ttl);
        
        // Replicar para outros nós
        replicationManager.replicate(primaryNode, key, value, ttl);
    }
}
```

### **4. "Como você garantiria a consistência de dados em um sistema distribuído?"**

**Estratégias:**
1. **ACID vs BASE**
2. **Eventual Consistency**
3. **Saga Pattern**
4. **Two-Phase Commit**
5. **Event Sourcing**

**Exemplo Prático:**
```java
// Implementação de Saga Pattern
@Service
public class OrderSagaOrchestrator {
    
    public void processOrder(Order order) {
        SagaTransaction saga = new SagaTransaction(order.getId());
        
        try {
            // Etapa 1: Reservar estoque
            saga.addStep(
                () -> inventoryService.reserveItems(order.getItems()),
                () -> inventoryService.releaseReservation(order.getId())
            );
            
            // Etapa 2: Processar pagamento
            saga.addStep(
                () -> paymentService.processPayment(order.getPayment()),
                () -> paymentService.refundPayment(order.getId())
            );
            
            // Etapa 3: Criar envio
            saga.addStep(
                () -> shippingService.createShipment(order.getShipping()),
                () -> shippingService.cancelShipment(order.getId())
            );
            
            // Executar saga
            saga.execute();
            
        } catch (Exception e) {
            // Compensar em caso de falha
            saga.compensate();
            throw new OrderProcessingException("Falha no processamento", e);
        }
    }
}
```

### **5. "Quando você usaria NoSQL vs SQL?"**

**Critérios de Decisão:**
- **Estrutura dos dados**: Relacional vs Documento/Chave-Valor
- **Escalabilidade**: Vertical vs Horizontal
- **Consistência**: ACID vs Eventual Consistency
- **Consultas**: Complexas vs Simples
- **Performance**: Transações vs Throughput

**Exemplo de Decisão:**
```java
// Sistema híbrido
@Service
public class HybridDataService {
    
    // SQL para dados transacionais
    @Autowired
    private PostgreSQLRepository transactionalRepository;
    
    // NoSQL para dados de sessão
    @Autowired
    private RedisRepository sessionRepository;
    
    // NoSQL para dados de auditoria
    @Autowired
    private MongoRepository auditRepository;
    
    // NoSQL para dados de analytics
    @Autowired
    private CassandraRepository analyticsRepository;
    
    public void processTransaction(Transaction transaction) {
        // Salvar transação no SQL (ACID)
        transactionalRepository.save(transaction);
        
        // Salvar eventos no NoSQL (Performance)
        auditRepository.save(transaction.getAuditEvents());
        
        // Atualizar métricas (Analytics)
        analyticsRepository.updateMetrics(transaction);
    }
}
```

---

## 🎯 **Simulação de Entrevista Técnica**

### **Exercício Final: Sistema de Streaming de Vídeo**

**Problema:**
"Projete um sistema como Netflix que precisa servir vídeos para 100 milhões de usuários, com recomendações personalizadas e qualidade adaptativa."

**Pontos a Abordar:**
1. **Arquitetura geral**
2. **CDN e distribuição de conteúdo**
3. **Sistema de recomendação**
4. **Qualidade adaptativa**
5. **Monitoramento e analytics**
6. **Escalabilidade e custos**

**Solução Esperada:**
```java
// Arquitetura de streaming
@RestController
@RequestMapping("/api/streaming")
public class StreamingController {
    
    private final VideoService videoService;
    private final RecommendationService recommendationService;
    private final CDNService cdnService;
    
    @GetMapping("/video/{videoId}/stream")
    public ResponseEntity<StreamingResponse> getVideoStream(
            @PathVariable String videoId,
            @RequestParam String quality,
            @RequestHeader("User-Agent") String userAgent,
            @RequestHeader("X-Real-IP") String clientIp) {
        
        // Determinar melhor CDN baseado na localização
        CDNEndpoint cdnEndpoint = cdnService.getBestEndpoint(clientIp);
        
        // Obter URL do vídeo com qualidade adaptativa
        VideoStreamInfo streamInfo = videoService.getStreamInfo(videoId, quality, userAgent);
        
        // Registrar evento de visualização
        viewingEventService.recordViewingStart(videoId, userId, quality);
        
        return ResponseEntity.ok(StreamingResponse.builder()
            .streamUrl(cdnEndpoint.getBaseUrl() + streamInfo.getPath())
            .qualities(streamInfo.getAvailableQualities())
            .subtitles(streamInfo.getSubtitles())
            .build());
    }
    
    @GetMapping("/recommendations/{userId}")
    public ResponseEntity<List<VideoRecommendation>> getRecommendations(
            @PathVariable String userId) {
        
        // Buscar recomendações personalizadas
        List<VideoRecommendation> recommendations = recommendationService
            .getPersonalizedRecommendations(userId);
        
        return ResponseEntity.ok(recommendations);
    }
}
```

---

## 🎯 **Checklist Final de Preparação**

### **Conhecimentos Técnicos**
- [ ] Arquitetura de sistemas distribuídos
- [ ] Padrões de design (SOLID, DDD, CQRS)
- [ ] Tecnologias Java e Spring
- [ ] Serviços AWS
- [ ] Banco de dados (SQL e NoSQL)
- [ ] Cache distribuído
- [ ] Mensageria e eventos
- [ ] Monitoramento e observabilidade
- [ ] Segurança e compliance
- [ ] Performance e escalabilidade

### **Soft Skills**
- [ ] Comunicação clara e objetiva
- [ ] Análise de trade-offs
- [ ] Pensamento crítico
- [ ] Resolução de problemas
- [ ] Liderança técnica
- [ ] Trabalho em equipe
- [ ] Adaptabilidade
- [ ] Aprendizado contínuo

### **Dicas para a Entrevista**
1. **Escute atentamente** - Entenda o problema antes de responder
2. **Faça perguntas** - Clarifique requisitos e restrições
3. **Pense em voz alta** - Mostre seu raciocínio
4. **Considere trade-offs** - Discuta prós e contras
5. **Seja prático** - Use exemplos reais
6. **Admita limitações** - Seja honesto sobre o que não sabe
7. **Demonstre curiosidade** - Faça perguntas sobre a empresa

---

## 🚀 **Próximos Passos**

1. **Pratique regularmente** - Implemente os exemplos
2. **Simule entrevistas** - Pratique com colegas
3. **Estude casos reais** - Analise arquiteturas conhecidas
4. **Mantenha-se atualizado** - Acompanhe tendências
5. **Construa portfólio** - Documente seus projetos

**Lembre-se:** A entrevista é uma conversa técnica, não um exame. Seja você mesmo e demonstre sua paixão por tecnologia!

**Boa sorte! 🎯** 