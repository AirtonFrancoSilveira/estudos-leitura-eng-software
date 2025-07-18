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