# 🏗️ System Design Patterns - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Aplicar padrões de System Design em sistemas distribuídos
- ✅ Escolher o padrão adequado para cada problema
- ✅ Implementar padrões de resiliência e escalabilidade
- ✅ Projetar sistemas de alta disponibilidade
- ✅ Resolver problemas de consistência e performance

---

## 🎯 **Conceito 1: Padrões de Resiliência**

### **🔍 O que são?**
Padrões de resiliência são como sistemas de segurança de um prédio: protegem contra falhas, isolam problemas e garantem que o sistema continue funcionando mesmo quando parte dele falha.

### **🎓 Padrões Fundamentais:**

**1. Circuit Breaker Pattern**

### **📖 Introdução Conceitual:**
O Circuit Breaker é como um disjuntor elétrico em sua casa. Quando detecta uma sobrecarga (muitas falhas), ele "desarma" temporariamente para proteger o sistema, impedindo que falhas se propaguem. Após um tempo, ele testa novamente se o problema foi resolvido. É essencial para prevenir cascatas de falhas em sistemas distribuídos.

**Estados do Circuit Breaker:**
- **🟢 CLOSED:** Funcionamento normal (disjuntor "ligado")
- **🔴 OPEN:** Falhas detectadas, bloqueando chamadas (disjuntor "desligado")  
- **🟡 HALF-OPEN:** Testando se o serviço voltou a funcionar

### **🎯 Quando Usar:**
- ✅ **Chamadas para serviços externos** (APIs de terceiros, bancos, gateways de pagamento)
- ✅ **Operações que podem falhar em cascata** (microserviços interdependentes)
- ✅ **Sistemas com SLA críticos** (banking, e-commerce, sistemas de emergência)
- ✅ **Quando você precisa de fallback rápido** (evitar timeouts longos)

### **🚫 Quando NÃO Usar:**
- ❌ **Operações internas simples** (validações, cálculos locais)
- ❌ **Sistemas com baixo volume** (menos de 100 requests/minuto)
- ❌ **Quando não há estratégia de fallback** (sem plano B)

### **💡 Casos de Uso Reais:**

**🏦 Banking - Verificação de CPF**
```java
// Cenário: Validação de CPF via Serasa/SPC
@Service
public class CpfValidationService {
    
    private final CircuitBreaker serasaCircuitBreaker;
    private final CircuitBreaker spcCircuitBreaker;
    
    public CpfValidationResult validateCpf(String cpf) {
        // Tentar Serasa primeiro
        try {
            return serasaCircuitBreaker.execute(
                () -> serasaClient.validateCpf(cpf),
                () -> tryAlternativeValidation(cpf)
            );
        } catch (Exception e) {
            // Fallback para SPC
            return spcCircuitBreaker.execute(
                () -> spcClient.validateCpf(cpf),
                () -> CpfValidationResult.temporarilyUnavailable()
            );
        }
    }
}
```

**🛒 E-commerce - Processamento de Pagamento**
```java
// Cenário: Gateway de pagamento pode ficar indisponível
@Service
public class PaymentGatewayService {
    
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentCircuitBreaker.execute(
            () -> primaryGateway.processPayment(request),
            () -> {
                // Fallback strategies
                if (request.getAmount().compareTo(BigDecimal.valueOf(100)) < 0) {
                    return processViaSecondaryGateway(request);
                } else {
                    return PaymentResult.queueForLaterProcessing(request);
                }
            }
        );
    }
}
```

**🚀 Streaming - Recomendações**
```java
// Cenário: Sistema de ML para recomendações pode falhar
@Service
public class RecommendationService {
    
    public List<Content> getRecommendations(String userId) {
        return mlCircuitBreaker.execute(
            () -> mlRecommendationEngine.getPersonalizedContent(userId),
            () -> {
                // Fallback para recomendações populares
                return contentService.getPopularContent(
                    userService.getUserPreferences(userId)
                );
            }
        );
    }
}
```

### **⚖️ Trade-offs:**
- **✅ Prós:** Previne falhas em cascata, melhora disponibilidade, reduz latência em falhas
- **❌ Contras:** Complexidade adicional, pode mascarar problemas reais, configuração delicada

```java
// Implementação avançada do Circuit Breaker
@Component
public class CircuitBreaker {
    
    private final AtomicInteger failureCount = new AtomicInteger(0);
    private final AtomicInteger requestCount = new AtomicInteger(0);
    private final AtomicLong lastFailureTime = new AtomicLong(0);
    private final AtomicReference<CircuitState> state = new AtomicReference<>(CircuitState.CLOSED);
    
    private final int failureThreshold;
    private final int minimumRequests;
    private final long timeoutDuration;
    private final double failureRateThreshold;
    
    public CircuitBreaker(int failureThreshold, int minimumRequests, 
                         long timeoutDuration, double failureRateThreshold) {
        this.failureThreshold = failureThreshold;
        this.minimumRequests = minimumRequests;
        this.timeoutDuration = timeoutDuration;
        this.failureRateThreshold = failureRateThreshold;
    }
    
    public <T> T execute(Supplier<T> operation, Supplier<T> fallback) {
        if (state.get() == CircuitState.OPEN) {
            if (canAttemptReset()) {
                state.compareAndSet(CircuitState.OPEN, CircuitState.HALF_OPEN);
            } else {
                log.warn("Circuit breaker is OPEN, executing fallback");
                return fallback.get();
            }
        }
        
        try {
            T result = operation.get();
            onSuccess();
            return result;
            
        } catch (Exception e) {
            onFailure();
            if (state.get() == CircuitState.OPEN) {
                log.warn("Circuit breaker opened due to failures, executing fallback");
                return fallback.get();
            }
            throw e;
        }
    }
    
    private void onSuccess() {
        requestCount.incrementAndGet();
        failureCount.set(0);
        
        if (state.get() == CircuitState.HALF_OPEN) {
            state.compareAndSet(CircuitState.HALF_OPEN, CircuitState.CLOSED);
            log.info("Circuit breaker transitioned to CLOSED");
        }
    }
    
    private void onFailure() {
        int failures = failureCount.incrementAndGet();
        int requests = requestCount.incrementAndGet();
        
        lastFailureTime.set(System.currentTimeMillis());
        
        if (requests >= minimumRequests) {
            double failureRate = (double) failures / requests;
            
            if (failureRate >= failureRateThreshold || failures >= failureThreshold) {
                state.compareAndSet(CircuitState.CLOSED, CircuitState.OPEN);
                log.warn("Circuit breaker opened: failureRate={}, failures={}", 
                    failureRate, failures);
            }
        }
    }
    
    private boolean canAttemptReset() {
        return System.currentTimeMillis() - lastFailureTime.get() > timeoutDuration;
    }
    
    public CircuitState getState() {
        return state.get();
    }
    
    public enum CircuitState {
        CLOSED, OPEN, HALF_OPEN
    }
}

// Uso em serviços
@Service
public class PaymentService {
    
    private final CircuitBreaker circuitBreaker;
    private final PaymentGateway paymentGateway;
    
    public PaymentService() {
        this.circuitBreaker = new CircuitBreaker(
            5,      // 5 falhas
            10,     // em 10 requests
            60000,  // timeout de 60 segundos
            0.5     // 50% de taxa de falha
        );
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        return circuitBreaker.execute(
            () -> paymentGateway.processPayment(request),
            () -> PaymentResult.fallback("Payment service unavailable")
        );
    }
}
```

**2. Bulkhead Pattern**

### **📖 Introdução Conceitual:**
O Bulkhead é inspirado nos compartimentos estanques dos navios Titanic. Se um compartimento é danificado, os outros permanecem intactos, evitando que o navio afunde completamente. Em sistemas, separamos recursos (threads, conexões, memória) em "compartimentos" isolados, garantindo que uma falha em um não afete os outros.

**Tipos de Isolamento:**
- **🧵 Thread Pools:** Pools separados para diferentes operações
- **🔌 Connection Pools:** Conexões de banco segregadas por função
- **💾 Memory Partitions:** Alocação de memória por criticidade
- **⚡ CPU/Rate Limits:** Limites de processamento por serviço

### **🎯 Quando Usar:**
- ✅ **Operações com diferentes prioridades** (críticas vs. não-críticas)
- ✅ **Recursos limitados** (threads, conexões de banco, memória)
- ✅ **Isolamento de falhas** (uma operação não pode afetar outras)
- ✅ **SLAs diferentes** (operações com requisitos de performance distintos)

### **🚫 Quando NÃO Usar:**
- ❌ **Sistemas com recursos abundantes** (sub-utilização de recursos)
- ❌ **Operações homogêneas** (todas têm a mesma prioridade)
- ❌ **Sistemas simples** (overhead desnecessário)

### **💡 Casos de Uso Reais:**

**🏦 Banking - Segregação de Operações**
```java
// Cenário: Banco com operações críticas e não-críticas
@Configuration
public class BankingBulkheadConfig {
    
    @Bean("transactionExecutor")
    public Executor transactionExecutor() {
        // Pool dedicado para transações financeiras
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("banking-transaction-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
    
    @Bean("reportingExecutor")
    public Executor reportingExecutor() {
        // Pool separado para relatórios (não-crítico)
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("banking-report-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        return executor;
    }
    
    @Bean("notificationExecutor")
    public Executor notificationExecutor() {
        // Pool para notificações (baixa prioridade)
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("banking-notification-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardOldestPolicy());
        return executor;
    }
}

@Service
public class BankingOperationsService {
    
    @Async("transactionExecutor")
    public CompletableFuture<TransactionResult> processPixTransfer(PixTransferRequest request) {
        // Operação crítica - pool dedicado de alta prioridade
        return CompletableFuture.completedFuture(pixService.processTransfer(request));
    }
    
    @Async("reportingExecutor")
    public CompletableFuture<Report> generateAccountStatement(String accountId) {
        // Relatório - pool separado, não afeta transações
        return CompletableFuture.completedFuture(reportService.generateStatement(accountId));
    }
    
    @Async("notificationExecutor")
    public CompletableFuture<Void> sendTransactionNotification(String customerId, TransactionEvent event) {
        // Notificação - pool de baixa prioridade
        notificationService.sendNotification(customerId, event);
        return CompletableFuture.completedFuture(null);
    }
}
```

**🛒 E-commerce - Isolamento por Função**
```java
// Cenário: E-commerce com diferentes tipos de carga
@Service
public class EcommerceService {
    
    @Async("orderProcessingExecutor")
    public CompletableFuture<OrderResult> processOrder(OrderRequest request) {
        // Processamento de pedidos - crítico
        return CompletableFuture.completedFuture(orderService.processOrder(request));
    }
    
    @Async("inventoryUpdateExecutor")
    public CompletableFuture<Void> updateInventory(InventoryUpdateRequest request) {
        // Atualização de estoque - importante mas não crítico
        inventoryService.updateStock(request);
        return CompletableFuture.completedFuture(null);
    }
    
    @Async("analyticsExecutor")
    public CompletableFuture<Void> trackUserBehavior(UserBehaviorEvent event) {
        // Analytics - pode ser processado com delay
        analyticsService.track(event);
        return CompletableFuture.completedFuture(null);
    }
}
```

**🏥 Healthcare - Isolamento por Criticidade**
```java
// Cenário: Sistema hospitalar com diferentes níveis de urgência
@Service
public class HospitalService {
    
    @Async("emergencyExecutor")
    public CompletableFuture<EmergencyResponse> handleEmergency(EmergencyRequest request) {
        // Emergências - máxima prioridade
        return CompletableFuture.completedFuture(emergencyService.handleEmergency(request));
    }
    
    @Async("appointmentExecutor")
    public CompletableFuture<AppointmentResult> scheduleAppointment(AppointmentRequest request) {
        // Agendamentos - prioridade média
        return CompletableFuture.completedFuture(appointmentService.schedule(request));
    }
    
    @Async("billingExecutor")
    public CompletableFuture<Void> processBilling(BillingRequest request) {
        // Faturamento - pode ser processado offline
        billingService.processBilling(request);
        return CompletableFuture.completedFuture(null);
    }
}
```

### **⚖️ Trade-offs:**
- **✅ Prós:** Isolamento de falhas, priorização de recursos, SLAs diferenciados
- **❌ Contras:** Complexidade de configuração, possível sub-utilização de recursos, overhead de gerenciamento

```java
// Implementação do padrão Bulkhead
@Configuration
public class BulkheadConfiguration {
    
    @Bean("criticalOperationsExecutor")
    public Executor criticalOperationsExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("critical-ops-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean("backgroundOperationsExecutor")
    public Executor backgroundOperationsExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("background-ops-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        executor.initialize();
        return executor;
    }
    
    @Bean("reportingOperationsExecutor")
    public Executor reportingOperationsExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(50);
        executor.setThreadNamePrefix("reporting-ops-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class BankingService {
    
    @Async("criticalOperationsExecutor")
    public CompletableFuture<TransactionResult> processTransfer(TransferRequest request) {
        // Operação crítica - pool dedicado
        return CompletableFuture.completedFuture(
            transactionProcessor.processTransfer(request)
        );
    }
    
    @Async("backgroundOperationsExecutor")
    public CompletableFuture<Void> sendNotification(NotificationRequest request) {
        // Operação de background - pool separado
        notificationService.send(request);
        return CompletableFuture.completedFuture(null);
    }
    
    @Async("reportingOperationsExecutor")
    public CompletableFuture<Report> generateReport(ReportRequest request) {
        // Operação de relatório - pool dedicado
        return CompletableFuture.completedFuture(
            reportGenerator.generate(request)
        );
    }
}
```

**3. Retry Pattern com Backoff**

### **📖 Introdução Conceitual:**
O Retry Pattern é como tentar ligar para alguém que está ocupado - você não fica tentando incessantemente, mas espera um pouco e tenta novamente. O "backoff" é o tempo de espera que aumenta progressivamente (1s, 2s, 4s, 8s...), evitando sobrecarregar um serviço que já está com problemas. É crucial para lidar com falhas temporárias em sistemas distribuídos.

**Estratégias de Backoff:**
- **📈 Exponential:** Tempo dobra a cada tentativa (1s, 2s, 4s, 8s...)
- **📊 Linear:** Tempo aumenta linearmente (1s, 2s, 3s, 4s...)
- **🎲 Random Jitter:** Adiciona aleatoriedade para evitar thundering herd
- **🔄 Fixed Interval:** Mesmo intervalo entre tentativas

### **🎯 Quando Usar:**
- ✅ **Falhas transientes** (timeouts de rede, indisponibilidade temporária)
- ✅ **Operações idempotentes** (podem ser repetidas sem efeitos colaterais)
- ✅ **Serviços externos instáveis** (APIs de terceiros, infraestrutura cloud)
- ✅ **Operações críticas** (não podem falhar por problemas temporários)

### **🚫 Quando NÃO Usar:**
- ❌ **Operações não-idempotentes** (transferências bancárias, criação de recursos)
- ❌ **Erros permanentes** (401 Unauthorized, 404 Not Found, validation errors)
- ❌ **Operações em tempo real** (quando delay não é aceitável)
- ❌ **Recursos sob pressão** (pode piorar a situação)

### **💡 Casos de Uso Reais:**

**🏦 Banking - Consulta ao Banco Central**
```java
// Cenário: Consulta à API do BACEN para validar instituições
@Service
public class BacenIntegrationService {
    
    private final RetryTemplate retryTemplate;
    
    public BankInstitution getBankInfo(String bankCode) {
        RetryPolicy policy = RetryPolicy.exponentialBackoff(
            3,      // 3 tentativas
            2000,   // 2 segundos inicial
            2.0     // dobrar a cada tentativa
        );
        
        return retryTemplate.execute(
            () -> {
                ResponseEntity<BankInstitution> response = bacenRestTemplate.getForEntity(
                    "/api/v1/institutions/{code}", BankInstitution.class, bankCode);
                
                if (response.getStatusCode().is5xxServerError()) {
                    throw new BacenUnavailableException("BACEN API returned: " + response.getStatusCode());
                }
                
                return response.getBody();
            },
            policy
        );
    }
}
```

**📱 Mobile Banking - Sincronização de Dados**
```java
// Cenário: App mobile sincronizando dados com servidor
@Service
public class MobileSyncService {
    
    public SyncResult syncTransactionHistory(String accountId, String lastSyncTimestamp) {
        RetryPolicy policy = RetryPolicy.exponentialBackoff(
            5,      // 5 tentativas (mobile pode ter rede instável)
            1000,   // 1 segundo inicial
            1.5     // crescimento mais suave
        );
        
        return retryTemplate.execute(
            () -> {
                SyncRequest request = new SyncRequest(accountId, lastSyncTimestamp);
                return bankingApiClient.syncTransactions(request);
            },
            policy
        );
    }
}
```

**🛒 E-commerce - Processamento de Pedidos**
```java
// Cenário: Integração com fornecedores pode falhar temporariamente
@Service
public class SupplierIntegrationService {
    
    public ProductAvailability checkProductAvailability(String productId) {
        RetryPolicy policy = RetryPolicy.exponentialBackoff(
            3,      // 3 tentativas
            500,    // 500ms inicial (operação rápida)
            2.0
        );
        
        return retryTemplate.execute(
            () -> {
                try {
                    return supplierApiClient.checkAvailability(productId);
                } catch (SocketTimeoutException e) {
                    throw new SupplierTemporaryException("Supplier timeout", e);
                } catch (ConnectException e) {
                    throw new SupplierTemporaryException("Connection refused", e);
                }
            },
            policy
        );
    }
}
```

**☁️ Cloud Services - Upload de Arquivos**
```java
// Cenário: Upload para S3 pode falhar por questões de rede
@Service
public class FileUploadService {
    
    public UploadResult uploadToS3(FileUploadRequest request) {
        RetryPolicy policy = RetryPolicy.exponentialBackoff(
            4,      // 4 tentativas (uploads podem ser lentos)
            3000,   // 3 segundos inicial
            1.5     // crescimento moderado
        );
        
        return retryTemplate.execute(
            () -> {
                try {
                    return s3Client.uploadFile(request);
                } catch (AmazonS3Exception e) {
                    if (e.getStatusCode() >= 400 && e.getStatusCode() < 500) {
                        // Erro do cliente - não retry
                        throw new NonRetryableException("Client error: " + e.getMessage(), e);
                    }
                    // Erro do servidor - retry
                    throw new S3TemporaryException("S3 server error", e);
                }
            },
            policy
        );
    }
}
```

**🔄 Microservices - Comunicação Entre Serviços**
```java
// Cenário: Serviços podem estar temporariamente indisponíveis
@Service
public class UserServiceClient {
    
    public UserProfile getUserProfile(String userId) {
        RetryPolicy policy = RetryPolicy.exponentialBackoff(
            3,      // 3 tentativas
            1000,   // 1 segundo inicial
            2.0
        );
        
        return retryTemplate.execute(
            () -> {
                ResponseEntity<UserProfile> response = userServiceRestTemplate.getForEntity(
                    "/users/{id}/profile", UserProfile.class, userId);
                
                if (response.getStatusCode().is5xxServerError()) {
                    throw new UserServiceTemporaryException("User service unavailable");
                }
                
                if (response.getStatusCode() == HttpStatus.NOT_FOUND) {
                    throw new UserNotFoundException("User not found: " + userId);
                }
                
                return response.getBody();
            },
            policy
        );
    }
}
```

### **⚖️ Trade-offs:**
- **✅ Prós:** Aumenta confiabilidade, lida com falhas transientes, melhora experiência do usuário
- **❌ Contras:** Pode mascarar problemas reais, aumenta latência, pode sobrecarregar recursos

```java
// Implementação avançada do padrão Retry
@Component
public class RetryTemplate {
    
    public <T> T execute(Supplier<T> operation, RetryPolicy retryPolicy) {
        Exception lastException = null;
        
        for (int attempt = 0; attempt < retryPolicy.getMaxAttempts(); attempt++) {
            try {
                return operation.get();
                
            } catch (Exception e) {
                lastException = e;
                
                if (attempt < retryPolicy.getMaxAttempts() - 1) {
                    if (retryPolicy.shouldRetry(e)) {
                        long delay = retryPolicy.calculateDelay(attempt);
                        
                        log.warn("Attempt {} failed, retrying in {} ms: {}", 
                            attempt + 1, delay, e.getMessage());
                        
                        try {
                            Thread.sleep(delay);
                        } catch (InterruptedException ie) {
                            Thread.currentThread().interrupt();
                            throw new RuntimeException("Retry interrupted", ie);
                        }
                    } else {
                        log.error("Non-retryable exception: {}", e.getMessage());
                        throw e;
                    }
                }
            }
        }
        
        log.error("All retry attempts failed");
        throw new RuntimeException("Operation failed after " + retryPolicy.getMaxAttempts() + " attempts", lastException);
    }
}

// Políticas de retry
public class RetryPolicy {
    
    private final int maxAttempts;
    private final long initialDelay;
    private final long maxDelay;
    private final double multiplier;
    private final Set<Class<? extends Exception>> retryableExceptions;
    private final Set<Class<? extends Exception>> nonRetryableExceptions;
    
    public static RetryPolicy exponentialBackoff(int maxAttempts, long initialDelay, double multiplier) {
        return new RetryPolicy(maxAttempts, initialDelay, 30000, multiplier, 
            Set.of(IOException.class, SocketTimeoutException.class),
            Set.of(IllegalArgumentException.class, SecurityException.class));
    }
    
    public static RetryPolicy fixedDelay(int maxAttempts, long delay) {
        return new RetryPolicy(maxAttempts, delay, delay, 1.0,
            Set.of(IOException.class, SocketTimeoutException.class),
            Set.of(IllegalArgumentException.class, SecurityException.class));
    }
    
    public boolean shouldRetry(Exception e) {
        if (nonRetryableExceptions.contains(e.getClass())) {
            return false;
        }
        
        return retryableExceptions.isEmpty() || 
               retryableExceptions.contains(e.getClass());
    }
    
    public long calculateDelay(int attempt) {
        long delay = (long) (initialDelay * Math.pow(multiplier, attempt));
        return Math.min(delay, maxDelay);
    }
}

// Uso prático
@Service
public class ExternalServiceClient {
    
    private final RetryTemplate retryTemplate;
    
    public CustomerData getCustomerData(String customerId) {
        RetryPolicy policy = RetryPolicy.exponentialBackoff(3, 1000, 2.0);
        
        return retryTemplate.execute(
            () -> {
                ResponseEntity<CustomerData> response = restTemplate.getForEntity(
                    "/customers/{id}", CustomerData.class, customerId);
                
                if (!response.getStatusCode().is2xxSuccessful()) {
                    throw new ExternalServiceException("Service returned: " + response.getStatusCode());
                }
                
                return response.getBody();
            },
            policy
        );
    }
}
```

**4. Rate Limiting Pattern**

### **📖 Introdução Conceitual:**
O Rate Limiting é como um porteiro de balada que controla quantas pessoas entram por minuto. Ele protege o sistema limitando quantas requisições um cliente pode fazer em um período específico, evitando sobrecarga e garantindo uso justo dos recursos. É essencial para prevenir abuso, ataques DDoS e garantir SLAs.

**Algoritmos Principais:**
- **🪣 Token Bucket:** Balde com tokens que se reenchem ao longo do tempo
- **🪟 Sliding Window:** Janela deslizante que conta requisições
- **⏰ Fixed Window:** Janelas fixas de tempo (ex: 100 req/minuto)
- **⛽ Leaky Bucket:** Requisições "vazam" do balde em taxa constante

### **🎯 Quando Usar:**
- ✅ **Proteção contra abuso** (ataques DDoS, spam, scrapers)
- ✅ **Recursos limitados** (APIs custosas, processamento intensivo)
- ✅ **Fair use policy** (garantir acesso equitativo aos recursos)
- ✅ **Monetização** (limites por plano de assinatura)

### **🚫 Quando NÃO Usar:**
- ❌ **Sistemas internos confiáveis** (overhead desnecessário)
- ❌ **Recursos abundantes** (quando não há limitação real)
- ❌ **Operações críticas** (que não podem ser limitadas)

### **💡 Casos de Uso Reais:**

**🏦 Banking - Proteção de APIs**
```java
// Cenário: Banco com diferentes limites por tipo de operação
@Component
public class BankingRateLimitingInterceptor implements HandlerInterceptor {
    
    private final RateLimiter rateLimiter;
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                           Object handler) throws Exception {
        
        String customerId = extractCustomerId(request);
        String endpoint = request.getRequestURI();
        String method = request.getMethod();
        
        // Diferentes limites por tipo de operação
        RateLimitConfig config = getRateLimitConfig(endpoint, method);
        String key = customerId + ":" + endpoint + ":" + method;
        
        if (!rateLimiter.isAllowed(key, config.getMaxRequests(), config.getWindow())) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setHeader("X-RateLimit-Limit", String.valueOf(config.getMaxRequests()));
            response.setHeader("X-RateLimit-Window", config.getWindow().toString());
            response.getWriter().write("Rate limit exceeded for " + endpoint);
            return false;
        }
        
        return true;
    }
    
    private RateLimitConfig getRateLimitConfig(String endpoint, String method) {
        // Transferências PIX: 50 por minuto
        if (endpoint.startsWith("/api/pix/transfer")) {
            return new RateLimitConfig(50, Duration.ofMinutes(1));
        }
        
        // Consultas de saldo: 200 por minuto
        if (endpoint.startsWith("/api/accounts/balance")) {
            return new RateLimitConfig(200, Duration.ofMinutes(1));
        }
        
        // Relatórios: 10 por hora
        if (endpoint.startsWith("/api/reports")) {
            return new RateLimitConfig(10, Duration.ofHours(1));
        }
        
        // Autenticação: 5 tentativas por minuto
        if (endpoint.startsWith("/api/auth/login")) {
            return new RateLimitConfig(5, Duration.ofMinutes(1));
        }
        
        // Default: 1000 por hora
        return new RateLimitConfig(1000, Duration.ofHours(1));
    }
}
```

**🛒 E-commerce - Proteção Contra Scrapers**
```java
// Cenário: E-commerce protegendo catálogo contra bots
@Service
public class ProductCatalogService {
    
    private final RateLimiter rateLimiter;
    
    public ProductListResponse getProducts(ProductSearchRequest request, String clientId) {
        
        // Diferentes limites por tipo de cliente
        RateLimitConfig config = getClientRateLimit(clientId);
        
        if (!rateLimiter.isAllowed(clientId + ":product_search", 
                                 config.getMaxRequests(), 
                                 config.getWindow())) {
            throw new RateLimitExceededException("Too many product searches");
        }
        
        return productRepository.searchProducts(request);
    }
    
    private RateLimitConfig getClientRateLimit(String clientId) {
        ClientType type = clientService.getClientType(clientId);
        
        switch (type) {
            case PREMIUM:
                return new RateLimitConfig(10000, Duration.ofHours(1));
            case BUSINESS:
                return new RateLimitConfig(5000, Duration.ofHours(1));
            case REGULAR:
                return new RateLimitConfig(1000, Duration.ofHours(1));
            case SUSPECTED_BOT:
                return new RateLimitConfig(100, Duration.ofHours(1));
            default:
                return new RateLimitConfig(500, Duration.ofHours(1));
        }
    }
}
```

**📱 API Gateway - Proteção de Microserviços**
```java
// Cenário: API Gateway protegendo microserviços
@Component
public class ApiGatewayRateLimiter {
    
    private final RateLimiter rateLimiter;
    
    public boolean checkRateLimit(String serviceName, String clientId, String plan) {
        
        // Rate limiting por serviço e plano
        String key = String.format("%s:%s:%s", serviceName, clientId, plan);
        RateLimitConfig config = getServiceRateLimit(serviceName, plan);
        
        return rateLimiter.isAllowed(key, config.getMaxRequests(), config.getWindow());
    }
    
    private RateLimitConfig getServiceRateLimit(String serviceName, String plan) {
        Map<String, RateLimitConfig> serviceLimits = Map.of(
            "user-service", Map.of(
                "basic", new RateLimitConfig(100, Duration.ofMinutes(1)),
                "premium", new RateLimitConfig(500, Duration.ofMinutes(1)),
                "enterprise", new RateLimitConfig(2000, Duration.ofMinutes(1))
            ),
            "payment-service", Map.of(
                "basic", new RateLimitConfig(10, Duration.ofMinutes(1)),
                "premium", new RateLimitConfig(50, Duration.ofMinutes(1)),
                "enterprise", new RateLimitConfig(200, Duration.ofMinutes(1))
            ),
            "notification-service", Map.of(
                "basic", new RateLimitConfig(50, Duration.ofMinutes(1)),
                "premium", new RateLimitConfig(200, Duration.ofMinutes(1)),
                "enterprise", new RateLimitConfig(1000, Duration.ofMinutes(1))
            )
        );
        
        return serviceLimits.getOrDefault(serviceName, Map.of())
                          .getOrDefault(plan, new RateLimitConfig(100, Duration.ofMinutes(1)));
    }
}
```

**🔐 Security - Proteção Contra Ataques**
```java
// Cenário: Sistema de autenticação com proteção contra brute force
@Service
public class AuthenticationService {
    
    private final RateLimiter rateLimiter;
    
    public AuthenticationResult authenticate(String username, String password, String clientIp) {
        
        // Rate limiting por IP
        String ipKey = "auth_attempts:ip:" + clientIp;
        if (!rateLimiter.isAllowed(ipKey, 20, Duration.ofMinutes(1))) {
            throw new TooManyAttemptsException("Too many authentication attempts from IP");
        }
        
        // Rate limiting por username
        String userKey = "auth_attempts:user:" + username;
        if (!rateLimiter.isAllowed(userKey, 5, Duration.ofMinutes(1))) {
            throw new TooManyAttemptsException("Too many authentication attempts for user");
        }
        
        // Autenticação mais restritiva após falhas
        String userFailureKey = "auth_failures:user:" + username;
        Long failureCount = rateLimiter.getCount(userFailureKey);
        
        if (failureCount > 3) {
            // Após 3 falhas, só permite 1 tentativa por minuto
            if (!rateLimiter.isAllowed(userFailureKey, 1, Duration.ofMinutes(1))) {
                throw new AccountTemporarilyLockedException("Account temporarily locked");
            }
        }
        
        AuthenticationResult result = performAuthentication(username, password);
        
        if (!result.isSuccess()) {
            // Incrementar contador de falhas
            rateLimiter.increment(userFailureKey, Duration.ofMinutes(15));
        } else {
            // Resetar contador em caso de sucesso
            rateLimiter.reset(userFailureKey);
        }
        
        return result;
    }
}
```

**🎮 Gaming - Proteção Contra Cheating**
```java
// Cenário: Jogo online com proteção contra spam de ações
@Service
public class GameActionService {
    
    private final RateLimiter rateLimiter;
    
    public ActionResult performAction(String playerId, GameAction action) {
        
        // Diferentes limites por tipo de ação
        RateLimitConfig config = getActionRateLimit(action.getType());
        String key = playerId + ":" + action.getType();
        
        if (!rateLimiter.isAllowed(key, config.getMaxRequests(), config.getWindow())) {
            return ActionResult.rateLimited("Action rate limit exceeded");
        }
        
        return gameEngine.processAction(playerId, action);
    }
    
    private RateLimitConfig getActionRateLimit(ActionType actionType) {
        switch (actionType) {
            case MOVE:
                return new RateLimitConfig(100, Duration.ofSeconds(1)); // 100 movimentos por segundo
            case ATTACK:
                return new RateLimitConfig(10, Duration.ofSeconds(1));  // 10 ataques por segundo
            case CHAT_MESSAGE:
                return new RateLimitConfig(5, Duration.ofSeconds(1));   // 5 mensagens por segundo
            case ITEM_USE:
                return new RateLimitConfig(20, Duration.ofSeconds(1));  // 20 itens por segundo
            default:
                return new RateLimitConfig(50, Duration.ofSeconds(1));
        }
    }
}
```

### **⚖️ Trade-offs:**
- **✅ Prós:** Proteção contra abuso, controle de recursos, fair use, monetização
- **❌ Contras:** Pode impactar usuários legítimos, complexidade de configuração, overhead de processamento

```java
// Implementação de Rate Limiting
@Component
public class RateLimiter {
    
    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();
    private final RedisTemplate<String, Object> redisTemplate;
    
    public boolean isAllowed(String key, int maxRequests, Duration window) {
        String redisKey = "rate_limit:" + key;
        
        // Sliding window com Redis
        long currentTime = System.currentTimeMillis();
        long windowStart = currentTime - window.toMillis();
        
        // Remover requests antigas
        redisTemplate.opsForZSet().removeRangeByScore(redisKey, 0, windowStart);
        
        // Contar requests atuais
        Long currentCount = redisTemplate.opsForZSet().count(redisKey, windowStart, currentTime);
        
        if (currentCount < maxRequests) {
            // Adicionar request atual
            redisTemplate.opsForZSet().add(redisKey, UUID.randomUUID().toString(), currentTime);
            redisTemplate.expire(redisKey, window);
            return true;
        }
        
        return false;
    }
    
    // Token bucket implementation
    public boolean isAllowedTokenBucket(String key, int maxTokens, Duration refillRate) {
        TokenBucket bucket = buckets.computeIfAbsent(key, k -> new TokenBucket(maxTokens, refillRate));
        return bucket.tryConsume();
    }
}

public class TokenBucket {
    
    private final int maxTokens;
    private final long refillRateMillis;
    private final AtomicInteger tokens;
    private final AtomicLong lastRefillTime;
    
    public TokenBucket(int maxTokens, Duration refillRate) {
        this.maxTokens = maxTokens;
        this.refillRateMillis = refillRate.toMillis();
        this.tokens = new AtomicInteger(maxTokens);
        this.lastRefillTime = new AtomicLong(System.currentTimeMillis());
    }
    
    public boolean tryConsume() {
        refillTokens();
        
        while (true) {
            int currentTokens = tokens.get();
            if (currentTokens > 0) {
                if (tokens.compareAndSet(currentTokens, currentTokens - 1)) {
                    return true;
                }
            } else {
                return false;
            }
        }
    }
    
    private void refillTokens() {
        long currentTime = System.currentTimeMillis();
        long lastRefill = lastRefillTime.get();
        
        if (currentTime - lastRefill >= refillRateMillis) {
            if (lastRefillTime.compareAndSet(lastRefill, currentTime)) {
                tokens.set(maxTokens);
            }
        }
    }
}

// Interceptor para Rate Limiting
@Component
public class RateLimitingInterceptor implements HandlerInterceptor {
    
    private final RateLimiter rateLimiter;
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                           Object handler) throws Exception {
        
        String clientId = extractClientId(request);
        String endpoint = request.getRequestURI();
        String key = clientId + ":" + endpoint;
        
        // Diferentes limites por endpoint
        int maxRequests = getMaxRequestsForEndpoint(endpoint);
        Duration window = getWindowForEndpoint(endpoint);
        
        if (!rateLimiter.isAllowed(key, maxRequests, window)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.getWriter().write("Rate limit exceeded");
            return false;
        }
        
        return true;
    }
    
    private int getMaxRequestsForEndpoint(String endpoint) {
        if (endpoint.startsWith("/api/auth")) return 10;
        if (endpoint.startsWith("/api/payments")) return 100;
        if (endpoint.startsWith("/api/transfers")) return 50;
        return 1000; // Default
    }
    
    private Duration getWindowForEndpoint(String endpoint) {
        if (endpoint.startsWith("/api/auth")) return Duration.ofMinutes(1);
        if (endpoint.startsWith("/api/payments")) return Duration.ofMinutes(5);
        return Duration.ofHours(1); // Default
    }
}
```

---

## 🎯 **Conceito 2: Padrões de Escalabilidade**

### **🔍 O que são?**
Padrões de escalabilidade são como sistemas de transporte urbano: permitem que mais "passageiros" (requests) sejam atendidos eficientemente, seja aumentando a capacidade ou otimizando o fluxo.

### **🎓 Padrões Fundamentais:**

**1. Database Sharding Pattern**

### **📖 Introdução Conceitual:**
Database Sharding é como dividir uma biblioteca gigante em várias bibliotecas menores por tema. Cada "shard" (fragmento) contém uma parte dos dados, distribuídos por diferentes servidores. Isso permite que o sistema escale horizontalmente, suportando mais dados e usuários. A chave é escolher uma boa estratégia de particionamento que distribua a carga uniformemente.

**Estratégias de Sharding:**
- **🔢 Range-based:** Dividir por faixas (IDs 1-1000, 1001-2000...)
- **🎲 Hash-based:** Usar função hash para distribuir dados
- **📍 Geographic:** Dividir por localização geográfica
- **👥 Directory-based:** Usar um serviço de lookup para localizar shards

### **🎯 Quando Usar:**
- ✅ **Banco de dados muito grande** (TBs de dados, bilhões de registros)
- ✅ **Alta concorrência** (milhares de writes simultâneos)
- ✅ **Crescimento horizontal** (quando scaling vertical não é viável)
- ✅ **Dados geograficamente distribuídos** (latência regional)

### **🚫 Quando NÃO Usar:**
- ❌ **Bancos pequenos** (menos de 100GB, queries simples)
- ❌ **Muitas queries cross-shard** (joins complexos entre shards)
- ❌ **Transações distribuídas** (operações ACID entre shards)
- ❌ **Equipe pequena** (complexidade de manutenção)

### **💡 Casos de Uso Reais:**

**🏦 Banking - Sharding por Região**
```java
// Cenário: Banco nacional com milhões de clientes
@Component
public class BankingShardingStrategy {
    
    private final Map<String, DataSource> regionalShards;
    
    public BankingShardingStrategy() {
        this.regionalShards = Map.of(
            "SOUTHEAST", createDataSource("bank_southeast_db"),
            "NORTHEAST", createDataSource("bank_northeast_db"),
            "SOUTH", createDataSource("bank_south_db"),
            "NORTH", createDataSource("bank_north_db"),
            "MIDWEST", createDataSource("bank_midwest_db")
        );
    }
    
    public DataSource getShardForCustomer(String customerId) {
        // Usar primeiros dígitos do CPF para determinar região
        String cpf = customerService.getCpf(customerId);
        String region = getRegionByCpf(cpf);
        
        return regionalShards.get(region);
    }
    
    private String getRegionByCpf(String cpf) {
        // Algoritmo baseado nos primeiros 3 dígitos do CPF
        int code = Integer.parseInt(cpf.substring(0, 3));
        
        if (code >= 011 && code <= 139) return "SOUTHEAST";
        if (code >= 140 && code <= 229) return "NORTHEAST";
        if (code >= 230 && code <= 289) return "SOUTH";
        if (code >= 290 && code <= 329) return "NORTH";
        return "MIDWEST";
    }
}

@Repository
public class CustomerRepository {
    
    private final BankingShardingStrategy shardingStrategy;
    
    public Customer findByCustomerId(String customerId) {
        DataSource shard = shardingStrategy.getShardForCustomer(customerId);
        JdbcTemplate template = new JdbcTemplate(shard);
        
        return template.queryForObject(
            "SELECT * FROM customers WHERE customer_id = ?",
            new CustomerRowMapper(),
            customerId
        );
    }
    
    // Query cross-shard para compliance/auditoria
    public List<Customer> findCustomersByNameAcrossAllShards(String name) {
        List<Customer> allCustomers = new ArrayList<>();
        
        // Executar em paralelo em todos os shards
        List<CompletableFuture<List<Customer>>> futures = 
            shardingStrategy.getAllShards().values().stream()
                .map(dataSource -> CompletableFuture.supplyAsync(() -> {
                    JdbcTemplate template = new JdbcTemplate(dataSource);
                    return template.query(
                        "SELECT * FROM customers WHERE name LIKE ?",
                        new CustomerRowMapper(),
                        "%" + name + "%"
                    );
                }))
                .collect(Collectors.toList());
        
        // Aguardar e combinar resultados
        futures.forEach(future -> {
            try {
                allCustomers.addAll(future.get());
            } catch (Exception e) {
                log.error("Error querying shard", e);
            }
        });
        
        return allCustomers;
    }
}
```

**🛒 E-commerce - Sharding por Categoria**
```java
// Cenário: Marketplace com milhões de produtos
@Component
public class ProductShardingStrategy {
    
    private final Map<String, DataSource> categoryShards;
    
    public ProductShardingStrategy() {
        this.categoryShards = Map.of(
            "ELECTRONICS", createDataSource("products_electronics_db"),
            "CLOTHING", createDataSource("products_clothing_db"),
            "BOOKS", createDataSource("products_books_db"),
            "HOME", createDataSource("products_home_db"),
            "SPORTS", createDataSource("products_sports_db")
        );
    }
    
    public DataSource getShardForProduct(String productId) {
        // Usar hash do ID para distribuir uniformemente
        String category = productService.getCategory(productId);
        return categoryShards.get(category);
    }
}

@Service
public class ProductService {
    
    private final ProductShardingStrategy shardingStrategy;
    
    public Product findProduct(String productId) {
        DataSource shard = shardingStrategy.getShardForProduct(productId);
        JdbcTemplate template = new JdbcTemplate(shard);
        
        return template.queryForObject(
            "SELECT * FROM products WHERE product_id = ?",
            new ProductRowMapper(),
            productId
        );
    }
    
    // Search cross-shard para busca global
    public List<Product> searchProducts(String searchTerm) {
        List<Product> allProducts = new ArrayList<>();
        
        // Buscar em paralelo em todos os shards
        List<CompletableFuture<List<Product>>> futures = 
            shardingStrategy.getAllShards().values().stream()
                .map(dataSource -> CompletableFuture.supplyAsync(() -> {
                    JdbcTemplate template = new JdbcTemplate(dataSource);
                    return template.query(
                        "SELECT * FROM products WHERE name LIKE ? OR description LIKE ? LIMIT 100",
                        new ProductRowMapper(),
                        "%" + searchTerm + "%",
                        "%" + searchTerm + "%"
                    );
                }))
                .collect(Collectors.toList());
        
        // Combinar e ordenar resultados
        futures.forEach(future -> {
            try {
                allProducts.addAll(future.get());
            } catch (Exception e) {
                log.error("Error searching in shard", e);
            }
        });
        
        // Ordenar por relevância e limitar
        return allProducts.stream()
            .sorted((p1, p2) -> compareRelevance(p1, p2, searchTerm))
            .limit(50)
            .collect(Collectors.toList());
    }
}
```

**📱 Social Media - Sharding por Usuário**
```java
// Cenário: Rede social com bilhões de posts
@Component
public class SocialMediaShardingStrategy {
    
    private final ConsistentHashRing hashRing;
    private final Map<String, DataSource> userShards;
    
    public SocialMediaShardingStrategy() {
        this.userShards = Map.of(
            "shard_001", createDataSource("social_shard_001"),
            "shard_002", createDataSource("social_shard_002"),
            "shard_003", createDataSource("social_shard_003"),
            "shard_004", createDataSource("social_shard_004"),
            "shard_005", createDataSource("social_shard_005")
        );
        
        this.hashRing = new ConsistentHashRing(userShards.keySet());
    }
    
    public DataSource getShardForUser(String userId) {
        String shardKey = hashRing.getNode(userId);
        return userShards.get(shardKey);
    }
}

@Service
public class PostService {
    
    private final SocialMediaShardingStrategy shardingStrategy;
    
    public Post createPost(String userId, CreatePostRequest request) {
        DataSource shard = shardingStrategy.getShardForUser(userId);
        JdbcTemplate template = new JdbcTemplate(shard);
        
        String postId = UUID.randomUUID().toString();
        template.update(
            "INSERT INTO posts (id, user_id, content, created_at) VALUES (?, ?, ?, ?)",
            postId, userId, request.getContent(), Instant.now()
        );
        
        return findPost(postId, userId);
    }
    
    public List<Post> getUserPosts(String userId) {
        DataSource shard = shardingStrategy.getShardForUser(userId);
        JdbcTemplate template = new JdbcTemplate(shard);
        
        return template.query(
            "SELECT * FROM posts WHERE user_id = ? ORDER BY created_at DESC LIMIT 50",
            new PostRowMapper(),
            userId
        );
    }
    
    // Timeline agregado de múltiplos usuários
    public List<Post> getTimelinePosts(String userId, List<String> followingUserIds) {
        // Agrupar usuários por shard
        Map<String, List<String>> usersByShards = followingUserIds.stream()
            .collect(Collectors.groupingBy(
                followingUserId -> shardingStrategy.getShardKey(followingUserId)
            ));
        
        List<Post> timelinePosts = new ArrayList<>();
        
        // Buscar posts em paralelo por shard
        List<CompletableFuture<List<Post>>> futures = usersByShards.entrySet().stream()
            .map(entry -> CompletableFuture.supplyAsync(() -> {
                DataSource shard = shardingStrategy.getShardByKey(entry.getKey());
                JdbcTemplate template = new JdbcTemplate(shard);
                
                String userIds = entry.getValue().stream()
                    .map(id -> "'" + id + "'")
                    .collect(Collectors.joining(","));
                
                return template.query(
                    "SELECT * FROM posts WHERE user_id IN (" + userIds + ") " +
                    "ORDER BY created_at DESC LIMIT 20",
                    new PostRowMapper()
                );
            }))
            .collect(Collectors.toList());
        
        // Combinar e ordenar por data
        futures.forEach(future -> {
            try {
                timelinePosts.addAll(future.get());
            } catch (Exception e) {
                log.error("Error fetching timeline posts", e);
            }
        });
        
        return timelinePosts.stream()
            .sorted(Comparator.comparing(Post::getCreatedAt).reversed())
            .limit(50)
            .collect(Collectors.toList());
    }
}
```

### **⚖️ Trade-offs:**
- **✅ Prós:** Escalabilidade horizontal, performance, distribuição geográfica
- **❌ Contras:** Complexidade alta, queries cross-shard custosas, rebalanceamento difícil

```java
// Implementação de Database Sharding
@Component
public class ShardingStrategy {
    
    private final Map<String, DataSource> shards;
    private final ConsistentHashRing hashRing;
    
    public ShardingStrategy(Map<String, DataSource> shards) {
        this.shards = shards;
        this.hashRing = new ConsistentHashRing(shards.keySet());
    }
    
    public String getShardKey(String partitionKey) {
        return hashRing.getNode(partitionKey);
    }
    
    public DataSource getShardDataSource(String partitionKey) {
        String shardKey = getShardKey(partitionKey);
        return shards.get(shardKey);
    }
}

@Repository
public class ShardedCustomerRepository {
    
    private final ShardingStrategy shardingStrategy;
    private final JdbcTemplate jdbcTemplate;
    
    public Customer findByCustomerId(String customerId) {
        DataSource dataSource = shardingStrategy.getShardDataSource(customerId);
        JdbcTemplate shardJdbcTemplate = new JdbcTemplate(dataSource);
        
        return shardJdbcTemplate.queryForObject(
            "SELECT * FROM customers WHERE customer_id = ?",
            new CustomerRowMapper(),
            customerId
        );
    }
    
    public List<Customer> findByRegion(String region) {
        // Query cross-shard para dados por região
        List<Customer> allCustomers = new ArrayList<>();
        
        // Executar em paralelo em todos os shards
        List<CompletableFuture<List<Customer>>> futures = shards.values()
            .stream()
            .map(dataSource -> CompletableFuture.supplyAsync(() -> {
                JdbcTemplate shardTemplate = new JdbcTemplate(dataSource);
                return shardTemplate.query(
                    "SELECT * FROM customers WHERE region = ?",
                    new CustomerRowMapper(),
                    region
                );
            }))
            .collect(Collectors.toList());
        
        // Aguardar todas as consultas
        futures.forEach(future -> {
            try {
                allCustomers.addAll(future.get());
            } catch (Exception e) {
                log.error("Error querying shard", e);
            }
        });
        
        return allCustomers;
    }
}

// Consistent Hash Ring
public class ConsistentHashRing {
    
    private final SortedMap<Integer, String> ring = new TreeMap<>();
    private final int virtualNodes = 100;
    
    public ConsistentHashRing(Set<String> nodes) {
        for (String node : nodes) {
            for (int i = 0; i < virtualNodes; i++) {
                int hash = hash(node + ":" + i);
                ring.put(hash, node);
            }
        }
    }
    
    public String getNode(String key) {
        if (ring.isEmpty()) {
            return null;
        }
        
        int hash = hash(key);
        
        // Encontrar o primeiro nó >= hash
        SortedMap<Integer, String> tailMap = ring.tailMap(hash);
        
        if (tailMap.isEmpty()) {
            // Wrap around para o primeiro nó
            return ring.get(ring.firstKey());
        }
        
        return tailMap.get(tailMap.firstKey());
    }
    
    private int hash(String key) {
        return key.hashCode();
    }
}
```

**2. CQRS com Read Replicas**
```java
// Implementação avançada de CQRS
@Component
public class CQRSDataSourceRouter {
    
    private final DataSource writeDataSource;
    private final List<DataSource> readDataSources;
    private final AtomicInteger readIndex = new AtomicInteger(0);
    
    public DataSource getWriteDataSource() {
        return writeDataSource;
    }
    
    public DataSource getReadDataSource() {
        if (readDataSources.isEmpty()) {
            return writeDataSource;
        }
        
        // Round-robin entre read replicas
        int index = readIndex.getAndIncrement() % readDataSources.size();
        return readDataSources.get(index);
    }
}

// Command Side
@Service
@Transactional
public class CustomerCommandService {
    
    private final CQRSDataSourceRouter dataSourceRouter;
    private final EventPublisher eventPublisher;
    
    public CustomerCommandResult createCustomer(CreateCustomerCommand command) {
        DataSource writeDS = dataSourceRouter.getWriteDataSource();
        JdbcTemplate writeTemplate = new JdbcTemplate(writeDS);
        
        // Executar comando na base de escrita
        String customerId = UUID.randomUUID().toString();
        writeTemplate.update(
            "INSERT INTO customers (id, name, email, created_at) VALUES (?, ?, ?, ?)",
            customerId, command.getName(), command.getEmail(), Instant.now()
        );
        
        // Publicar evento para atualizar read models
        CustomerCreatedEvent event = new CustomerCreatedEvent(
            customerId, command.getName(), command.getEmail()
        );
        eventPublisher.publishEvent(event);
        
        return CustomerCommandResult.success(customerId);
    }
}

// Query Side
@Service
@Transactional(readOnly = true)
public class CustomerQueryService {
    
    private final CQRSDataSourceRouter dataSourceRouter;
    private final RedisTemplate<String, Object> cacheTemplate;
    
    public CustomerView getCustomer(String customerId) {
        // Tentar cache primeiro
        String cacheKey = "customer:" + customerId;
        CustomerView cached = (CustomerView) cacheTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            return cached;
        }
        
        // Buscar na read replica
        DataSource readDS = dataSourceRouter.getReadDataSource();
        JdbcTemplate readTemplate = new JdbcTemplate(readDS);
        
        CustomerView customer = readTemplate.queryForObject(
            "SELECT * FROM customer_views WHERE id = ?",
            new CustomerViewRowMapper(),
            customerId
        );
        
        // Cachear resultado
        cacheTemplate.opsForValue().set(cacheKey, customer, Duration.ofMinutes(10));
        
        return customer;
    }
    
    public List<CustomerView> searchCustomers(CustomerSearchQuery query) {
        DataSource readDS = dataSourceRouter.getReadDataSource();
        JdbcTemplate readTemplate = new JdbcTemplate(readDS);
        
        StringBuilder sql = new StringBuilder("SELECT * FROM customer_views WHERE 1=1");
        List<Object> params = new ArrayList<>();
        
        if (query.getName() != null) {
            sql.append(" AND name LIKE ?");
            params.add("%" + query.getName() + "%");
        }
        
        if (query.getEmail() != null) {
            sql.append(" AND email LIKE ?");
            params.add("%" + query.getEmail() + "%");
        }
        
        if (query.getRegion() != null) {
            sql.append(" AND region = ?");
            params.add(query.getRegion());
        }
        
        sql.append(" ORDER BY created_at DESC LIMIT ? OFFSET ?");
        params.add(query.getLimit());
        params.add(query.getOffset());
        
        return readTemplate.query(sql.toString(), new CustomerViewRowMapper(), params.toArray());
    }
}

// Event Handler para sincronizar read models
@Component
public class CustomerReadModelHandler {
    
    private final CQRSDataSourceRouter dataSourceRouter;
    private final RedisTemplate<String, Object> cacheTemplate;
    
    @EventListener
    @Async
    public void handleCustomerCreated(CustomerCreatedEvent event) {
        // Atualizar read model
        DataSource readDS = dataSourceRouter.getReadDataSource();
        JdbcTemplate readTemplate = new JdbcTemplate(readDS);
        
        readTemplate.update(
            "INSERT INTO customer_views (id, name, email, region, created_at) VALUES (?, ?, ?, ?, ?)",
            event.getCustomerId(),
            event.getName(),
            event.getEmail(),
            extractRegion(event.getEmail()),
            event.getOccurredAt()
        );
        
        // Invalidar cache
        cacheTemplate.delete("customer:" + event.getCustomerId());
    }
    
    @EventListener
    @Async
    public void handleCustomerUpdated(CustomerUpdatedEvent event) {
        // Atualizar read model
        DataSource readDS = dataSourceRouter.getReadDataSource();
        JdbcTemplate readTemplate = new JdbcTemplate(readDS);
        
        readTemplate.update(
            "UPDATE customer_views SET name = ?, email = ?, updated_at = ? WHERE id = ?",
            event.getName(),
            event.getEmail(),
            event.getOccurredAt(),
            event.getCustomerId()
        );
        
        // Invalidar cache
        cacheTemplate.delete("customer:" + event.getCustomerId());
    }
}
```

**3. Cache-Aside Pattern**

### **📖 Introdução Conceitual:**
O Cache-Aside é como ter uma mesa de cabeceira ao lado da cama - você coloca ali os itens que usa com mais frequência para não precisar ir até o armário toda vez. A aplicação gerencia o cache manualmente: primeiro verifica o cache, se não encontrar, busca na fonte original e guarda no cache para próximas consultas. É o padrão mais comum e flexível de cache.

**Fluxo Cache-Aside:**
1. **📖 Read:** Verificar cache → Se miss, buscar no DB → Guardar no cache
2. **✍️ Write:** Escrever no DB → Invalidar/atualizar cache
3. **🗑️ Eviction:** Cache remove itens menos usados automaticamente

```java
// Implementação do padrão Cache-Aside
@Service
public class CacheAsideService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final CustomerRepository customerRepository;
    
    public Customer getCustomer(String customerId) {
        String cacheKey = "customer:" + customerId;
        
        // 1. Tentar buscar no cache
        Customer cached = (Customer) redisTemplate.opsForValue().get(cacheKey);
        
        if (cached != null) {
            log.debug("Cache hit for customer: {}", customerId);
            return cached;
        }
        
        // 2. Buscar no banco de dados
        log.debug("Cache miss for customer: {}", customerId);
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        // 3. Salvar no cache
        redisTemplate.opsForValue().set(cacheKey, customer, Duration.ofMinutes(30));
        
        return customer;
    }
    
    public Customer updateCustomer(String customerId, CustomerUpdateRequest request) {
        // 1. Atualizar no banco
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        customer.setName(request.getName());
        customer.setEmail(request.getEmail());
        customer.setUpdatedAt(Instant.now());
        
        Customer updatedCustomer = customerRepository.save(customer);
        
        // 2. Invalidar cache
        String cacheKey = "customer:" + customerId;
        redisTemplate.delete(cacheKey);
        
        // 3. Ou atualizar cache (write-through)
        // redisTemplate.opsForValue().set(cacheKey, updatedCustomer, Duration.ofMinutes(30));
        
        return updatedCustomer;
    }
}

// Cache warming strategy
@Component
public class CacheWarmingService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final CustomerRepository customerRepository;
    
    @Scheduled(fixedRate = 300000) // 5 minutos
    public void warmPopularCustomers() {
        log.info("Starting cache warming for popular customers");
        
        // Buscar clientes mais acessados
        List<String> popularCustomerIds = getPopularCustomerIds();
        
        popularCustomerIds.parallelStream().forEach(customerId -> {
            try {
                String cacheKey = "customer:" + customerId;
                
                // Verificar se já está no cache
                if (!redisTemplate.hasKey(cacheKey)) {
                    Customer customer = customerRepository.findById(customerId).orElse(null);
                    if (customer != null) {
                        redisTemplate.opsForValue().set(cacheKey, customer, Duration.ofMinutes(30));
                        log.debug("Warmed cache for customer: {}", customerId);
                    }
                }
            } catch (Exception e) {
                log.warn("Failed to warm cache for customer: {}", customerId, e);
            }
        });
        
        log.info("Cache warming completed for {} customers", popularCustomerIds.size());
    }
    
    private List<String> getPopularCustomerIds() {
        // Buscar IDs dos clientes mais acessados no último dia
        return customerRepository.findMostAccessedCustomers(
            LocalDateTime.now().minusDays(1), 
            LocalDateTime.now(), 
            100
        );
    }
}
```

---

## 🎯 **Conceito 3: Padrões de Distribuição**

### **🔍 O que são?**
Padrões de distribuição são como sistemas de logística: organizam como dados e operações são distribuídos entre múltiplos serviços e localizações.

### **🎓 Padrões Fundamentais:**

**1. API Gateway Pattern**

### **📖 Introdução Conceitual:**
O API Gateway é como uma recepcionista de um grande edifício corporativo - ela é o ponto central que direciona as pessoas (requisições) para os departamentos corretos (microserviços). Ela cuida da autenticação, autorização, rate limiting, logging e pode até agregar respostas de múltiplos serviços. É fundamental em arquiteturas de microserviços para simplificar a comunicação cliente-servidor.

**Responsabilidades do API Gateway:**
- **🚪 Ponto de entrada único:** Centralize todas as chamadas externas
- **🔐 Autenticação/Autorização:** Valide usuários e permissões
- **📊 Rate Limiting:** Controle de tráfego e proteção contra abuso
- **📝 Logging/Monitoring:** Observabilidade centralizada
- **🔄 Request/Response Transformation:** Adaptação de formatos

```java
// Implementação avançada de API Gateway
@RestController
@RequestMapping("/gateway")
public class ApiGateway {
    
    private final Map<String, ServiceRoute> routes;
    private final LoadBalancer loadBalancer;
    private final RateLimiter rateLimiter;
    private final AuthenticationService authService;
    
    @PostMapping("/{service}/**")
    public ResponseEntity<?> routeRequest(
            @PathVariable String service,
            HttpServletRequest request,
            HttpServletResponse response,
            @RequestBody(required = false) String body) {
        
        try {
            // 1. Validar serviço
            ServiceRoute route = routes.get(service);
            if (route == null) {
                return ResponseEntity.notFound().build();
            }
            
            // 2. Autenticação
            AuthenticationResult authResult = authService.authenticate(request);
            if (!authResult.isAuthenticated()) {
                return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
            }
            
            // 3. Autorização
            if (!route.hasPermission(authResult.getUser(), request.getMethod())) {
                return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
            }
            
            // 4. Rate limiting
            String clientId = authResult.getUser().getId();
            if (!rateLimiter.isAllowed(clientId, route.getRateLimit())) {
                return ResponseEntity.status(HttpStatus.TOO_MANY_REQUESTS).build();
            }
            
            // 5. Load balancing
            ServiceInstance instance = loadBalancer.selectInstance(service);
            
            // 6. Request transformation
            String targetUrl = buildTargetUrl(instance, request);
            HttpHeaders headers = transformHeaders(request, authResult);
            
            // 7. Circuit breaker
            return executeWithCircuitBreaker(targetUrl, request.getMethod(), headers, body);
            
        } catch (Exception e) {
            log.error("Error routing request to service: {}", service, e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }
    
    private ResponseEntity<?> executeWithCircuitBreaker(String url, String method, 
                                                      HttpHeaders headers, String body) {
        return circuitBreaker.execute(
            () -> {
                HttpEntity<String> entity = new HttpEntity<>(body, headers);
                return restTemplate.exchange(url, HttpMethod.valueOf(method), entity, String.class);
            },
            () -> ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                .body("Service temporarily unavailable")
        );
    }
}

// Service Discovery
@Component
public class ServiceRegistry {
    
    private final Map<String, List<ServiceInstance>> services = new ConcurrentHashMap<>();
    private final Map<String, HealthChecker> healthCheckers = new ConcurrentHashMap<>();
    
    public void registerService(String serviceName, ServiceInstance instance) {
        services.computeIfAbsent(serviceName, k -> new ArrayList<>()).add(instance);
        
        // Iniciar health checking
        HealthChecker healthChecker = new HealthChecker(instance);
        healthCheckers.put(instance.getId(), healthChecker);
        healthChecker.start();
    }
    
    public List<ServiceInstance> getHealthyInstances(String serviceName) {
        return services.getOrDefault(serviceName, Collections.emptyList())
            .stream()
            .filter(ServiceInstance::isHealthy)
            .collect(Collectors.toList());
    }
    
    public void deregisterService(String serviceName, String instanceId) {
        List<ServiceInstance> instances = services.get(serviceName);
        if (instances != null) {
            instances.removeIf(instance -> instance.getId().equals(instanceId));
        }
        
        HealthChecker healthChecker = healthCheckers.remove(instanceId);
        if (healthChecker != null) {
            healthChecker.stop();
        }
    }
}

// Load Balancer
@Component
public class LoadBalancer {
    
    private final ServiceRegistry serviceRegistry;
    private final Map<String, AtomicInteger> roundRobinCounters = new ConcurrentHashMap<>();
    
    public ServiceInstance selectInstance(String serviceName) {
        List<ServiceInstance> instances = serviceRegistry.getHealthyInstances(serviceName);
        
        if (instances.isEmpty()) {
            throw new ServiceUnavailableException("No healthy instances for service: " + serviceName);
        }
        
        // Round-robin load balancing
        AtomicInteger counter = roundRobinCounters.computeIfAbsent(serviceName, k -> new AtomicInteger(0));
        int index = counter.getAndIncrement() % instances.size();
        
        return instances.get(index);
    }
    
    public ServiceInstance selectInstanceByWeight(String serviceName) {
        List<ServiceInstance> instances = serviceRegistry.getHealthyInstances(serviceName);
        
        if (instances.isEmpty()) {
            throw new ServiceUnavailableException("No healthy instances for service: " + serviceName);
        }
        
        // Weighted round-robin
        int totalWeight = instances.stream().mapToInt(ServiceInstance::getWeight).sum();
        int randomWeight = ThreadLocalRandom.current().nextInt(totalWeight);
        
        int currentWeight = 0;
        for (ServiceInstance instance : instances) {
            currentWeight += instance.getWeight();
            if (randomWeight < currentWeight) {
                return instance;
            }
        }
        
        // Fallback
        return instances.get(0);
    }
}
```

**2. Database per Service Pattern**

### **📖 Introdução Conceitual:**
O Database per Service é como cada departamento de uma empresa ter seu próprio arquivo - o RH tem seus dados, Financeiro os seus, e ninguém mexe nos arquivos do outro. Cada microserviço possui seu próprio banco de dados, garantindo autonomia, independência de deploy e escalabilidade isolada. É fundamental para verdadeira arquitetura de microserviços, mas traz desafios de consistência de dados.

**Benefícios do Isolamento:**
- **🔒 Autonomia:** Cada serviço evolui independentemente
- **⚡ Escalabilidade:** Escalar banco conforme necessidade do serviço
- **🛡️ Isolamento de falhas:** Problema em um banco não afeta outros
- **🔧 Tecnologia adequada:** Usar o banco ideal para cada caso (SQL/NoSQL)

```java
// Implementação do padrão Database per Service
@Configuration
public class MultiTenantDatabaseConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> dataSources = new HashMap<>();
        
        // Cada serviço tem seu próprio banco
        dataSources.put("customer-service", createDataSource("customer_db"));
        dataSources.put("order-service", createDataSource("order_db"));
        dataSources.put("payment-service", createDataSource("payment_db"));
        dataSources.put("inventory-service", createDataSource("inventory_db"));
        
        RoutingDataSource routingDataSource = new RoutingDataSource();
        routingDataSource.setTargetDataSources(dataSources);
        routingDataSource.setDefaultTargetDataSource(dataSources.get("customer-service"));
        
        return routingDataSource;
    }
    
    private DataSource createDataSource(String databaseName) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/" + databaseName);
        config.setUsername("app_user");
        config.setPassword("app_password");
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2);
        
        return new HikariDataSource(config);
    }
}

// Context holder para routing
public class DatabaseContextHolder {
    
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();
    
    public static void setDatabaseContext(String databaseKey) {
        contextHolder.set(databaseKey);
    }
    
    public static String getDatabaseContext() {
        return contextHolder.get();
    }
    
    public static void clearDatabaseContext() {
        contextHolder.remove();
    }
}

// Routing DataSource
public class RoutingDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        return DatabaseContextHolder.getDatabaseContext();
    }
}

// Service com contexto específico
@Service
public class CustomerService {
    
    private final CustomerRepository customerRepository;
    
    @Transactional
    public Customer createCustomer(CreateCustomerRequest request) {
        // Definir contexto do banco
        DatabaseContextHolder.setDatabaseContext("customer-service");
        
        try {
            Customer customer = new Customer();
            customer.setName(request.getName());
            customer.setEmail(request.getEmail());
            customer.setCreatedAt(Instant.now());
            
            return customerRepository.save(customer);
            
        } finally {
            DatabaseContextHolder.clearDatabaseContext();
        }
    }
    
    // Método para buscar dados cross-service
    public CustomerOrderSummary getCustomerWithOrders(String customerId) {
        // Buscar dados do cliente
        DatabaseContextHolder.setDatabaseContext("customer-service");
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
        
        // Buscar pedidos via API (não diretamente no banco)
        List<Order> orders = orderServiceClient.getOrdersByCustomer(customerId);
        
        DatabaseContextHolder.clearDatabaseContext();
        
        return new CustomerOrderSummary(customer, orders);
    }
}
```

**3. Event-Driven Architecture**
```java
// Implementação de Event-Driven Architecture
@Component
public class EventBus {
    
    private final Map<Class<?>, List<EventHandler<?>>> handlers = new ConcurrentHashMap<>();
    private final ExecutorService eventExecutor = Executors.newFixedThreadPool(10);
    
    public <T> void subscribe(Class<T> eventType, EventHandler<T> handler) {
        handlers.computeIfAbsent(eventType, k -> new ArrayList<>()).add(handler);
    }
    
    public <T> void publish(T event) {
        List<EventHandler<?>> eventHandlers = handlers.get(event.getClass());
        
        if (eventHandlers != null) {
            eventHandlers.forEach(handler -> {
                eventExecutor.submit(() -> {
                    try {
                        ((EventHandler<T>) handler).handle(event);
                    } catch (Exception e) {
                        log.error("Error handling event: {}", event.getClass().getSimpleName(), e);
                    }
                });
            });
        }
    }
}

// Event Store
@Component
public class EventStore {
    
    private final JdbcTemplate jdbcTemplate;
    private final ObjectMapper objectMapper;
    
    public void saveEvent(DomainEvent event) {
        try {
            String eventData = objectMapper.writeValueAsString(event);
            
            jdbcTemplate.update(
                "INSERT INTO event_store (id, aggregate_id, event_type, event_data, version, created_at) " +
                "VALUES (?, ?, ?, ?, ?, ?)",
                UUID.randomUUID().toString(),
                event.getAggregateId(),
                event.getClass().getSimpleName(),
                eventData,
                event.getVersion(),
                event.getOccurredAt()
            );
            
        } catch (Exception e) {
            throw new EventStoreException("Failed to save event", e);
        }
    }
    
    public List<DomainEvent> getEvents(String aggregateId, int fromVersion) {
        return jdbcTemplate.query(
            "SELECT * FROM event_store WHERE aggregate_id = ? AND version >= ? ORDER BY version",
            (rs, rowNum) -> {
                try {
                    String eventType = rs.getString("event_type");
                    String eventData = rs.getString("event_data");
                    
                    Class<?> eventClass = Class.forName("com.bank.events." + eventType);
                    return (DomainEvent) objectMapper.readValue(eventData, eventClass);
                    
                } catch (Exception e) {
                    throw new EventStoreException("Failed to deserialize event", e);
                }
            },
            aggregateId, fromVersion
        );
    }
}

// Saga Pattern para transações distribuídas
@Component
public class OrderSaga {
    
    private final EventBus eventBus;
    private final SagaRepository sagaRepository;
    
    @EventHandler
    public void handle(OrderCreatedEvent event) {
        SagaInstance saga = new SagaInstance(
            UUID.randomUUID().toString(),
            event.getOrderId(),
            SagaStatus.STARTED
        );
        
        sagaRepository.save(saga);
        
        // Iniciar primeira etapa
        ReserveInventoryCommand command = new ReserveInventoryCommand(
            event.getOrderId(),
            event.getItems()
        );
        
        eventBus.publish(command);
    }
    
    @EventHandler
    public void handle(InventoryReservedEvent event) {
        SagaInstance saga = sagaRepository.findByOrderId(event.getOrderId());
        
        if (saga != null && saga.getStatus() == SagaStatus.STARTED) {
            // Próxima etapa: processar pagamento
            ProcessPaymentCommand command = new ProcessPaymentCommand(
                event.getOrderId(),
                event.getTotalAmount()
            );
            
            eventBus.publish(command);
        }
    }
    
    @EventHandler
    public void handle(PaymentProcessedEvent event) {
        SagaInstance saga = sagaRepository.findByOrderId(event.getOrderId());
        
        if (saga != null) {
            // Finalizar saga
            saga.setStatus(SagaStatus.COMPLETED);
            sagaRepository.save(saga);
            
            // Publicar evento de conclusão
            OrderCompletedEvent completedEvent = new OrderCompletedEvent(event.getOrderId());
            eventBus.publish(completedEvent);
        }
    }
    
    @EventHandler
    public void handle(PaymentFailedEvent event) {
        SagaInstance saga = sagaRepository.findByOrderId(event.getOrderId());
        
        if (saga != null) {
            // Compensar transação
            saga.setStatus(SagaStatus.COMPENSATING);
            sagaRepository.save(saga);
            
            // Liberar estoque
            ReleaseInventoryCommand compensateCommand = new ReleaseInventoryCommand(
                event.getOrderId()
            );
            
            eventBus.publish(compensateCommand);
        }
    }
}
```

---

## 🎯 **Conceito 4: Padrões de Consistência**

### **🔍 O que são?**
Padrões de consistência são como regras de contabilidade: definem quando e como os dados devem estar "balanceados" entre diferentes partes do sistema.

### **🎓 Padrões Fundamentais:**

**1. Eventual Consistency Pattern**

### **📖 Introdução Conceitual:**
A Eventual Consistency é como um grupo de WhatsApp onde nem todo mundo vê a mensagem ao mesmo tempo - alguns veem imediatamente, outros em alguns segundos, mas eventualmente todos ficam sincronizados. Em sistemas distribuídos, aceita-se que os dados podem estar temporariamente inconsistentes entre diferentes nós, mas convergem para um estado consistente ao longo do tempo. É essencial para alta disponibilidade e performance.

**Características da Eventual Consistency:**
- **⏰ Convergência temporal:** Dados se tornam consistentes ao longo do tempo
- **🌐 Disponibilidade:** Sistema continua funcionando mesmo com partições de rede
- **📊 Performance:** Operações não precisam esperar sincronização síncrona
- **🔄 Reconciliação:** Mecanismos para detectar e corrigir inconsistências

```java
// Implementação de Eventual Consistency
@Component
public class EventualConsistencyManager {
    
    private final EventStore eventStore;
    private final Map<String, ProjectionHandler> projectionHandlers;
    
    public void processEvent(DomainEvent event) {
        // Salvar evento
        eventStore.saveEvent(event);
        
        // Processar projeções assincronamente
        projectionHandlers.values().forEach(handler -> {
            CompletableFuture.runAsync(() -> {
                try {
                    handler.handle(event);
                } catch (Exception e) {
                    log.error("Error processing projection for event: {}", event.getClass().getSimpleName(), e);
                    // Reagendar para retry
                    scheduleRetry(event, handler);
                }
            });
        });
    }
    
    private void scheduleRetry(DomainEvent event, ProjectionHandler handler) {
        // Implementar retry com backoff
        RetryPolicy retryPolicy = RetryPolicy.exponentialBackoff(3, 1000, 2.0);
        
        CompletableFuture.runAsync(() -> {
            try {
                Thread.sleep(retryPolicy.calculateDelay(0));
                handler.handle(event);
            } catch (Exception e) {
                log.error("Retry failed for event: {}", event.getClass().getSimpleName(), e);
            }
        });
    }
}

// Read Model Projections
@Component
public class CustomerProjectionHandler implements ProjectionHandler {
    
    private final CustomerViewRepository customerViewRepository;
    
    @Override
    public void handle(DomainEvent event) {
        switch (event.getClass().getSimpleName()) {
            case "CustomerCreatedEvent":
                handleCustomerCreated((CustomerCreatedEvent) event);
                break;
            case "CustomerUpdatedEvent":
                handleCustomerUpdated((CustomerUpdatedEvent) event);
                break;
            case "CustomerDeletedEvent":
                handleCustomerDeleted((CustomerDeletedEvent) event);
                break;
        }
    }
    
    private void handleCustomerCreated(CustomerCreatedEvent event) {
        CustomerView view = new CustomerView();
        view.setId(event.getCustomerId());
        view.setName(event.getName());
        view.setEmail(event.getEmail());
        view.setCreatedAt(event.getOccurredAt());
        view.setVersion(event.getVersion());
        
        customerViewRepository.save(view);
    }
    
    private void handleCustomerUpdated(CustomerUpdatedEvent event) {
        CustomerView view = customerViewRepository.findById(event.getCustomerId())
            .orElseThrow(() -> new CustomerViewNotFoundException(event.getCustomerId()));
        
        // Verificar versão para evitar updates fora de ordem
        if (event.getVersion() > view.getVersion()) {
            view.setName(event.getName());
            view.setEmail(event.getEmail());
            view.setUpdatedAt(event.getOccurredAt());
            view.setVersion(event.getVersion());
            
            customerViewRepository.save(view);
        }
    }
}
```

**2. Saga Pattern com Compensação**
```java
// Implementação avançada do padrão Saga
@Component
public class OrderProcessingSaga {
    
    private final Map<String, SagaStep> sagaSteps = new LinkedHashMap<>();
    private final SagaRepository sagaRepository;
    
    public OrderProcessingSaga() {
        // Definir steps da saga
        sagaSteps.put("RESERVE_INVENTORY", new ReserveInventoryStep());
        sagaSteps.put("PROCESS_PAYMENT", new ProcessPaymentStep());
        sagaSteps.put("SHIP_ORDER", new ShipOrderStep());
        sagaSteps.put("SEND_CONFIRMATION", new SendConfirmationStep());
    }
    
    public void startSaga(String orderId, OrderData orderData) {
        SagaInstance saga = new SagaInstance(
            UUID.randomUUID().toString(),
            orderId,
            new ArrayList<>(sagaSteps.keySet()),
            SagaStatus.STARTED
        );
        
        sagaRepository.save(saga);
        
        // Executar primeiro step
        executeNextStep(saga, orderData);
    }
    
    private void executeNextStep(SagaInstance saga, OrderData orderData) {
        if (saga.hasNextStep()) {
            String nextStepName = saga.getNextStep();
            SagaStep step = sagaSteps.get(nextStepName);
            
            try {
                StepResult result = step.execute(orderData);
                
                if (result.isSuccess()) {
                    saga.markStepCompleted(nextStepName);
                    sagaRepository.save(saga);
                    
                    if (saga.isCompleted()) {
                        completeSaga(saga);
                    } else {
                        executeNextStep(saga, orderData);
                    }
                } else {
                    failSaga(saga, result.getError());
                }
                
            } catch (Exception e) {
                failSaga(saga, e.getMessage());
            }
        }
    }
    
    private void failSaga(SagaInstance saga, String error) {
        saga.setStatus(SagaStatus.FAILED);
        saga.setErrorMessage(error);
        sagaRepository.save(saga);
        
        // Iniciar compensação
        compensateSaga(saga);
    }
    
    private void compensateSaga(SagaInstance saga) {
        List<String> completedSteps = saga.getCompletedSteps();
        
        // Executar compensação na ordem reversa
        Collections.reverse(completedSteps);
        
        for (String stepName : completedSteps) {
            SagaStep step = sagaSteps.get(stepName);
            
            try {
                step.compensate(saga.getOrderId());
                log.info("Compensated step: {} for saga: {}", stepName, saga.getId());
                
            } catch (Exception e) {
                log.error("Failed to compensate step: {} for saga: {}", stepName, saga.getId(), e);
                // Continuar com outros steps de compensação
            }
        }
        
        saga.setStatus(SagaStatus.COMPENSATED);
        sagaRepository.save(saga);
    }
}

// Saga Steps
public abstract class SagaStep {
    
    public abstract StepResult execute(OrderData orderData);
    public abstract void compensate(String orderId);
}

public class ReserveInventoryStep extends SagaStep {
    
    @Autowired
    private InventoryService inventoryService;
    
    @Override
    public StepResult execute(OrderData orderData) {
        try {
            ReservationResult result = inventoryService.reserveItems(
                orderData.getOrderId(),
                orderData.getItems()
            );
            
            if (result.isSuccess()) {
                return StepResult.success();
            } else {
                return StepResult.failure("Inventory reservation failed: " + result.getErrorMessage());
            }
            
        } catch (Exception e) {
            return StepResult.failure("Inventory service error: " + e.getMessage());
        }
    }
    
    @Override
    public void compensate(String orderId) {
        try {
            inventoryService.releaseReservation(orderId);
            log.info("Released inventory reservation for order: {}", orderId);
            
        } catch (Exception e) {
            log.error("Failed to release inventory reservation for order: {}", orderId, e);
        }
    }
}

public class ProcessPaymentStep extends SagaStep {
    
    @Autowired
    private PaymentService paymentService;
    
    @Override
    public StepResult execute(OrderData orderData) {
        try {
            PaymentResult result = paymentService.processPayment(
                orderData.getOrderId(),
                orderData.getTotalAmount(),
                orderData.getPaymentMethod()
            );
            
            if (result.isSuccess()) {
                return StepResult.success();
            } else {
                return StepResult.failure("Payment processing failed: " + result.getErrorMessage());
            }
            
        } catch (Exception e) {
            return StepResult.failure("Payment service error: " + e.getMessage());
        }
    }
    
    @Override
    public void compensate(String orderId) {
        try {
            paymentService.refundPayment(orderId);
            log.info("Refunded payment for order: {}", orderId);
            
        } catch (Exception e) {
            log.error("Failed to refund payment for order: {}", orderId, e);
        }
    }
}
```

---

## 🎯 **Quadro Comparativo de Padrões**

### **📊 Guia de Decisão Rápida**

| Padrão | Problema que Resolve | Quando Usar | Complexidade | Exemplo Real |
|--------|---------------------|-------------|-------------|-------------|
| **Circuit Breaker** | Falhas em cascata | Serviços externos instáveis | 🟡 Média | Gateway de pagamento |
| **Bulkhead** | Recursos compartilhados | Operações com prioridades diferentes | 🟡 Média | Banking (transações vs relatórios) |
| **Retry** | Falhas transientes | Operações idempotentes | 🟢 Baixa | Consultas a APIs externas |
| **Rate Limiting** | Abuso de recursos | Proteção contra spam/DDoS | 🟡 Média | API pública |
| **Database Sharding** | Escalabilidade de dados | Bancos muito grandes | 🔴 Alta | Redes sociais |
| **CQRS** | Leitura vs Escrita | Diferentes padrões de acesso | 🔴 Alta | E-commerce |
| **Cache-Aside** | Latência de dados | Dados frequentemente acessados | 🟢 Baixa | Catálogo de produtos |
| **API Gateway** | Múltiplos serviços | Arquitetura microserviços | 🟡 Média | Agregador de APIs |
| **Database per Service** | Acoplamento de dados | Isolamento entre serviços | 🔴 Alta | Microserviços |
| **Event-Driven** | Comunicação assíncrona | Sistemas distribuídos | 🔴 Alta | Processamento de pedidos |
| **Eventual Consistency** | Consistência imediata | Sistemas distribuídos | 🔴 Alta | Sincronização de dados |
| **Saga Pattern** | Transações distribuídas | Operações cross-service | 🔴 Alta | Processamento de compras |

### **🎯 Matriz de Aplicação por Domínio**

| Domínio | Padrões Mais Usados | Padrões Ocasionais | Padrões Raramente Usados |
|---------|-------------------|-------------------|-------------------------|
| **🏦 Banking** | Circuit Breaker, Bulkhead, Retry, Rate Limiting | CQRS, Saga Pattern | Database Sharding |
| **🛒 E-commerce** | Cache-Aside, CQRS, Event-Driven, Saga Pattern | Database Sharding, API Gateway | Bulkhead |
| **📱 Social Media** | Database Sharding, Cache-Aside, Rate Limiting | Event-Driven, Eventual Consistency | Saga Pattern |
| **🎮 Gaming** | Rate Limiting, Cache-Aside, Event-Driven | Circuit Breaker, Bulkhead | Database Sharding |
| **🏥 Healthcare** | Circuit Breaker, Bulkhead, Retry, Saga Pattern | CQRS, Rate Limiting | Database Sharding |
| **📺 Streaming** | Cache-Aside, CDN, Rate Limiting | Database Sharding, Event-Driven | Saga Pattern |

### **⚡ Padrões por Escala do Sistema**

#### **🔰 Startup (< 1M usuários)**
- ✅ **Essenciais:** Circuit Breaker, Retry, Cache-Aside
- ⚠️ **Considerar:** Rate Limiting, CQRS
- ❌ **Evitar:** Database Sharding, Saga Pattern

#### **🚀 Crescimento (1M - 10M usuários)**
- ✅ **Essenciais:** Todos os padrões de resiliência, CQRS, Event-Driven
- ⚠️ **Considerar:** Database Sharding, Eventual Consistency
- ❌ **Evitar:** Over-engineering

#### **🌟 Enterprise (> 10M usuários)**
- ✅ **Essenciais:** Todos os padrões são relevantes
- ⚠️ **Considerar:** Múltiplos padrões combinados
- ❌ **Evitar:** Soluções monolíticas

### **🔄 Padrões Complementares**

#### **Combinações Poderosas:**
1. **Circuit Breaker + Retry + Rate Limiting** = Resiliência completa
2. **CQRS + Event-Driven + Saga Pattern** = Arquitetura orientada a eventos
3. **Database Sharding + Cache-Aside + Eventual Consistency** = Escalabilidade massiva
4. **API Gateway + Bulkhead + Circuit Breaker** = Microserviços resilientes

#### **Antipadrões (Evitar):**
1. **Retry + Non-Idempotent Operations** = Duplicação de dados
2. **Database Sharding + Complex Joins** = Performance ruim
3. **Saga Pattern + Tight Coupling** = Complexidade desnecessária
4. **Rate Limiting + Critical Operations** = Degradação de serviço

### **📈 Métricas de Sucesso por Padrão**

| Padrão | Métricas Principais | Valores Objetivo | Alertas |
|--------|-------------------|------------------|---------|
| **Circuit Breaker** | Failure rate, Success rate | <5% failure | >10% failure |
| **Bulkhead** | Resource utilization | <80% per pool | >90% per pool |
| **Retry** | Retry rate, Success after retry | <20% retry | >50% retry |
| **Rate Limiting** | Requests blocked, False positives | <5% blocked | >10% blocked |
| **Cache-Aside** | Cache hit rate, Miss latency | >80% hit rate | <60% hit rate |
| **CQRS** | Read/Write latency, Sync lag | <100ms, <1s | >500ms, >5s |
| **Database Sharding** | Shard utilization, Cross-shard queries | Balanced, <10% | Unbalanced, >20% |
| **Event-Driven** | Event processing time, Dead letters | <1s, <1% | >5s, >5% |

---

## 🎯 **Exercícios Práticos**

### **Exercício 1: Sistema de E-commerce Resiliente**
**Cenário:** Projetar sistema de e-commerce que suporte 1 milhão de usuários simultâneos.

**Sua tarefa:**
- Implementar Circuit Breaker para todos os serviços externos
- Configurar Rate Limiting por usuário e endpoint
- Aplicar Bulkhead pattern para isolar operações críticas
- Implementar Retry pattern com backoff exponencial

### **Exercício 2: Sistema Bancário Distribuído**
**Cenário:** Criar sistema bancário com múltiplas regiões e eventual consistency.

**Sua tarefa:**
- Implementar CQRS com Event Sourcing
- Configurar Database Sharding por região
- Aplicar Saga pattern para transferências entre bancos
- Implementar API Gateway com service discovery

### **Exercício 3: Sistema de Streaming**
**Cenário:** Arquitetar plataforma de streaming de vídeo como Netflix.

**Sua tarefa:**
- Implementar Cache-Aside pattern para metadados
- Configurar CDN pattern para distribuição de conteúdo
- Aplicar Load Balancing geográfico
- Implementar Event-Driven architecture para recomendações

---

## 🚀 **Perguntas de Entrevista sobre System Design**

### **💼 Cenários de Entrevista Avançados**

#### **🏦 Cenário Banking: Sistema PIX Nacional**
**Pergunta:** "Projete um sistema PIX que processe 1 milhão de transações por minuto, com disponibilidade de 99.99% e latência < 500ms."

**Resposta estruturada:**
```
1. Padrões de Resiliência:
   - Circuit Breaker para comunicação com bancos
   - Bulkhead para separar PIX de outros serviços
   - Retry para falhas transientes de rede
   - Rate Limiting por banco/CPF

2. Padrões de Escalabilidade:
   - Database Sharding por região/banco
   - CQRS para separar consultas de transações
   - Cache-Aside para dados de contas frequentes

3. Padrões de Distribuição:
   - API Gateway para roteamento por banco
   - Event-Driven para notificações
   - Database per Service para isolamento

4. Padrões de Consistência:
   - Saga Pattern para transações cross-banco
   - Eventual Consistency para sincronização
   - Compensação para reversão de erros
```

#### **🛒 Cenário E-commerce: Black Friday**
**Pergunta:** "Como você garantiria que um e-commerce suporte 10x o tráfego normal durante a Black Friday?"

**Resposta estruturada:**
```
1. Preparação (Semanas antes):
   - Cache warming para produtos populares
   - Database Sharding para catálogo
   - CDN para assets estáticos
   - Load testing com padrões realistas

2. Proteção (Durante o evento):
   - Rate Limiting dinâmico por usuário
   - Circuit Breaker para fornecedores
   - Bulkhead para separar checkout de navegação
   - Queue-based processing para pedidos

3. Fallback (Em caso de problemas):
   - Página estática para produtos populares
   - Retry com backoff exponencial
   - Graceful degradation de features
   - Waitlist para produtos esgotados

4. Monitoramento:
   - Métricas em tempo real
   - Alertas automáticos
   - Dashboards por padrão
   - Rollback automático
```

#### **📱 Cenário Social Media: Feed Global**
**Pergunta:** "Projete o feed de uma rede social como Twitter com 500 milhões de usuários ativos."

**Resposta estruturada:**
```
1. Armazenamento:
   - Database Sharding por usuário
   - Cache-Aside para timeline
   - Event-Driven para posts
   - CDN para mídia

2. Processamento:
   - Push model para usuários com poucos seguidores
   - Pull model para celebrities
   - Híbrido para otimização
   - Batch processing para analytics

3. Resiliência:
   - Circuit Breaker para serviços externos
   - Eventual Consistency para feeds
   - Retry para falhas de entrega
   - Bulkhead para diferentes tipos de conteúdo

4. Escalabilidade:
   - Read replicas por região
   - Microserviços por funcionalidade
   - Auto-scaling baseado em métricas
   - Geographic distribution
```

### **🎯 Perguntas sobre Padrões Específicos**

#### **1. "Quando você NÃO usaria Circuit Breaker?"**
**Resposta esperada:**
- Operações internas com alta confiabilidade
- Sistemas com poucos requests (< 100/min)
- Quando não há estratégia de fallback
- Operações críticas que não podem falhar

#### **2. "Como você escolheria entre CQRS e Database Sharding?"**
**Resposta esperada:**
- **CQRS:** Padrões de leitura/escrita diferentes, necessidade de views especializadas
- **Database Sharding:** Volume de dados muito grande, scaling horizontal necessário
- **Ambos:** Sistemas muito grandes com necessidades específicas

#### **3. "Explique os trade-offs do Saga Pattern"**
**Resposta esperada:**
- **Prós:** Transações distribuídas, eventual consistency, resiliência
- **Contras:** Complexidade, debugging difícil, compensação manual
- **Quando usar:** Operações críticas cross-service

#### **4. "Como você implementaria Rate Limiting distribuído?"**
**Resposta esperada:**
- Redis com sliding window
- Consistent hashing para distribuição
- Aproximação com eventual consistency
- Métricas por nó + agregação

### **🔥 Perguntas Pegadinha**

#### **"Você usaria todos os padrões em um sistema?"**
**Resposta correta:** 
- Não! Cada padrão tem custos e complexidade
- Análise de trade-offs é essencial
- Começar simples e evoluir
- Métricas guiam decisões

#### **"Database Sharding resolve todos os problemas de escala?"**
**Resposta correta:**
- Não! Cria novos problemas (cross-shard queries, rebalancing)
- Só para problemas específicos de volume
- Outras soluções podem ser melhores (cache, read replicas)
- Última opção, não primeira

#### **"Rate Limiting sempre melhora a experiência do usuário?"**
**Resposta correta:**
- Não! Pode impactar usuários legítimos
- Balanceamento entre proteção e usabilidade
- Diferentes estratégias por contexto
- Monitoramento de false positives

### **📊 Matriz de Avaliação em Entrevistas**

| Critério | Iniciante | Pleno | Sênior | Especialista |
|----------|-----------|--------|--------|--------------|
| **Conhecimento** | Conhece poucos padrões | Conhece padrões principais | Conhece todos + aplicações | Conhece + criação própria |
| **Aplicação** | Aplica sem contexto | Aplica com contexto básico | Escolhe padrão correto | Combina múltiplos padrões |
| **Trade-offs** | Não considera | Considera básicos | Analisa profundamente | Quantifica impactos |
| **Escala** | Pensa pequeno | Pensa médio | Pensa grande | Pensa em evolução |
| **Experiência** | Teórico | Alguns projetos | Múltiplos projetos | Mentor/arquiteto |

---

## 📚 **Resumo Executivo**

### **🎯 Takeaways Principais**

1. **Não existe bala de prata** - Cada padrão resolve problemas específicos
2. **Trade-offs são inevitáveis** - Ganhos em uma área custam em outra
3. **Contexto é rei** - Tamanho, escala e domínio influenciam escolhas
4. **Evolução gradual** - Comece simples, adicione complexidade conforme necessário
5. **Métricas guiam decisões** - Meça antes de otimizar

### **⚡ Padrões por Prioridade**

#### **🥇 Prioridade 1 (Todo sistema precisa):**
- Circuit Breaker
- Retry Pattern
- Cache-Aside

#### **🥈 Prioridade 2 (Sistemas em crescimento):**
- Rate Limiting
- Bulkhead
- CQRS

#### **🥉 Prioridade 3 (Sistemas complexos):**
- Database Sharding
- Event-Driven Architecture
- Saga Pattern

### **🚀 Próximos Passos para Dominar**

1. **Implemente cada padrão** - Hands-on é essencial
2. **Combine padrões** - Veja como trabalham juntos
3. **Meça impactos** - Benchmarks e métricas
4. **Estude casos reais** - Netflix, Amazon, Google
5. **Pratique entrevistas** - Simule cenários complexos

### **💡 Dicas Finais**

- **Para entrevistas:** Sempre discuta trade-offs
- **Para implementação:** Comece simples, evolua gradualmente
- **Para manutenção:** Monitore métricas de cada padrão
- **Para equipe:** Documente decisões e contexto

**Lembre-se:** System Design é uma arte que combina ciência, experiência e intuição. A prática leva à perfeição! 🎯

---

### **1. "Como você lidaria com hot partitions em um sistema distribuído?"**

**Resposta estruturada:**
- **Identificação**: Monitoramento de métricas por partition
- **Mitigação**: Consistent hashing com virtual nodes
- **Balanceamento**: Redistribuição automática de carga
- **Prevenção**: Estratégias de particionamento inteligentes

### **2. "Explique como garantir idempotência em operações distribuídas"**

**Resposta estruturada:**
- **Idempotency keys**: Chaves únicas por operação
- **Database constraints**: Unique constraints
- **Conditional operations**: Compare-and-swap
- **Event deduplication**: Deduplicação de eventos

### **3. "Como você implementaria um sistema de chat em tempo real?"**

**Resposta estruturada:**
- **WebSocket connections**: Conexões persistentes
- **Message queues**: Filas para delivery confiável
- **Presence system**: Sistema de presença online
- **Horizontal scaling**: Scaling de conexões

---

## 🎯 **Checklist de System Design**

### **Resiliência**
- [ ] Circuit Breaker implementado
- [ ] Bulkhead pattern aplicado
- [ ] Retry pattern com backoff
- [ ] Timeout configuration
- [ ] Graceful degradation

### **Escalabilidade**
- [ ] Load balancing strategy
- [ ] Database sharding
- [ ] Cache strategy
- [ ] CDN configuration
- [ ] Auto-scaling policies

### **Consistência**
- [ ] ACID vs BASE trade-offs
- [ ] Eventual consistency model
- [ ] Saga pattern for distributed transactions
- [ ] Event sourcing implementation
- [ ] Conflict resolution strategy

### **Performance**
- [ ] Cache-aside pattern
- [ ] Read replicas
- [ ] Database indexing
- [ ] Connection pooling
- [ ] Async processing

### **Monitoring**
- [ ] Health checks
- [ ] Metrics collection
- [ ] Distributed tracing
- [ ] Alerting system
- [ ] Log aggregation

---

## 🚀 **Próximos Passos**

1. **Pratique os padrões** - Implemente cada padrão individualmente
2. **Combine padrões** - Veja como se complementam
3. **Teste cenários** - Simule falhas e sobrecarga
4. **Meça performance** - Benchmarking dos padrões
5. **Estude casos reais** - Analise arquiteturas conhecidas

**Lembre-se:** System Design é sobre trade-offs. Sempre considere o contexto específico do problema! 