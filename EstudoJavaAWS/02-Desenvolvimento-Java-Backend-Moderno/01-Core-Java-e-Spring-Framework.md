# ☕ Java & Spring Framework - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Dominar Spring Boot e suas principais anotações
- ✅ Implementar segurança robusta com Spring Security
- ✅ Configurar cache distribuído com Redis
- ✅ Criar APIs RESTful seguindo melhores práticas
- ✅ Implementar padrões avançados do Spring

---

## 🎯 **Conceito 1: Spring Boot Avançado**

### **🔍 O que é?**
Spring Boot é como ter um assistente pessoal que configura sua casa automaticamente. Ele "adivinha" o que você precisa e configura tudo para você.

### **🎓 Conceitos Avançados:**

**1. Custom Auto-Configuration**
```java
// Condições customizadas para auto-configuração
@Configuration
@ConditionalOnClass(RedisTemplate.class)
@ConditionalOnProperty(name = "app.cache.enabled", havingValue = "true")
@EnableConfigurationProperties(CacheProperties.class)
public class CustomCacheAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .transactionAware()
            .build();
    }
}

// Configuration Properties
@ConfigurationProperties(prefix = "app.cache")
@Data
public class CacheProperties {
    private boolean enabled = true;
    private Duration ttl = Duration.ofMinutes(10);
    private int maxSize = 1000;
    private String keyPrefix = "app:cache:";
}
```

**2. Advanced Spring Boot Actuator**
```java
// Custom Health Indicator
@Component
public class BankingSystemHealthIndicator implements HealthIndicator {
    
    private final AccountService accountService;
    private final PaymentGatewayService paymentGatewayService;
    
    @Override
    public Health health() {
        Health.Builder builder = new Health.Builder();
        
        try {
            // Verificar serviços críticos
            boolean accountServiceUp = accountService.isHealthy();
            boolean paymentGatewayUp = paymentGatewayService.isHealthy();
            
            if (accountServiceUp && paymentGatewayUp) {
                return builder.up()
                    .withDetail("account-service", "UP")
                    .withDetail("payment-gateway", "UP")
                    .withDetail("timestamp", Instant.now())
                    .build();
            } else {
                return builder.down()
                    .withDetail("account-service", accountServiceUp ? "UP" : "DOWN")
                    .withDetail("payment-gateway", paymentGatewayUp ? "UP" : "DOWN")
                    .build();
            }
        } catch (Exception e) {
            return builder.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

// Custom Metrics
@Component
public class BankingMetrics {
    
    private final Counter transactionCounter;
    private final Timer transactionTimer;
    private final Gauge pendingTransactions;
    
    public BankingMetrics(MeterRegistry meterRegistry) {
        this.transactionCounter = Counter.builder("banking.transactions.total")
            .description("Total number of transactions")
            .register(meterRegistry);
            
        this.transactionTimer = Timer.builder("banking.transactions.duration")
            .description("Transaction processing time")
            .register(meterRegistry);
            
        this.pendingTransactions = Gauge.builder("banking.transactions.pending")
            .description("Number of pending transactions")
            .register(meterRegistry, this, BankingMetrics::getPendingTransactionsCount);
    }
    
    public void recordTransaction(String type, String status) {
        transactionCounter.increment(
            Tags.of("type", type, "status", status)
        );
    }
    
    public void recordTransactionTime(Duration duration) {
        transactionTimer.record(duration);
    }
    
    private double getPendingTransactionsCount() {
        // Implementar lógica para contar transações pendentes
        return 0.0;
    }
}
```

### **💡 Caso de Uso Prático: Sistema Bancário com Monitoramento Avançado**

```java
@RestController
@RequestMapping("/api/transactions")
@Validated
@Slf4j
public class TransactionController {
    
    private final TransactionService transactionService;
    private final BankingMetrics metrics;
    
    @PostMapping("/transfer")
    @Timed(value = "banking.transfer.duration", description = "Transfer processing time")
    public ResponseEntity<TransactionResponse> transfer(
            @Valid @RequestBody TransferRequest request,
            Authentication authentication) {
        
        long startTime = System.currentTimeMillis();
        
        try {
            // Validações
            validateTransferRequest(request);
            
            // Processar transferência
            TransactionResult result = transactionService.processTransfer(request, authentication.getName());
            
            // Métricas
            metrics.recordTransaction("transfer", result.getStatus());
            metrics.recordTransactionTime(Duration.ofMillis(System.currentTimeMillis() - startTime));
            
            // Log estruturado
            log.info("Transfer completed successfully: orderId={}, amount={}, status={}", 
                result.getTransactionId(), request.getAmount(), result.getStatus());
            
            return ResponseEntity.ok(new TransactionResponse(result));
            
        } catch (InsufficientFundsException e) {
            metrics.recordTransaction("transfer", "insufficient_funds");
            log.warn("Transfer failed - insufficient funds: userId={}, amount={}", 
                authentication.getName(), request.getAmount());
            
            return ResponseEntity.badRequest()
                .body(new TransactionResponse("FAILED", e.getMessage()));
                
        } catch (Exception e) {
            metrics.recordTransaction("transfer", "error");
            log.error("Transfer processing error: userId={}, amount={}", 
                authentication.getName(), request.getAmount(), e);
            
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new TransactionResponse("ERROR", "Internal server error"));
        }
    }
    
    private void validateTransferRequest(TransferRequest request) {
        if (request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        
        if (request.getFromAccountId().equals(request.getToAccountId())) {
            throw new IllegalArgumentException("Cannot transfer to same account");
        }
    }
}
```

---

## 🎯 **Conceito 2: Spring Security Avançado**

### **🔍 O que é?**
Spring Security é como um sistema de segurança de um prédio corporativo: tem portaria (authentication), controle de acesso (authorization), câmeras (audit) e sistemas de alarme (threat detection).

### **🎓 Conceitos Avançados:**

**1. JWT com Refresh Token**
```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/banking/**").hasAnyRole("CUSTOMER", "TELLER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .headers(headers -> headers
                .frameOptions().deny()
                .contentTypeOptions().and()
                .httpStrictTransportSecurity(hstsConfig -> hstsConfig
                    .maxAgeInSeconds(31536000)
                    .includeSubdomains(true)
                )
            )
            .addFilterBefore(new JwtAuthenticationFilter(jwtDecoder()), UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        NimbusJwtDecoder decoder = NimbusJwtDecoder.withJwkSetUri(
            "https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX/.well-known/jwks.json"
        ).build();
        
        decoder.setJwtValidator(jwtValidator());
        return decoder;
    }
    
    @Bean
    public Converter<Jwt, AbstractAuthenticationToken> jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwtGrantedAuthoritiesConverter());
        return converter;
    }
    
    @Bean
    public Converter<Jwt, Collection<GrantedAuthority>> jwtGrantedAuthoritiesConverter() {
        return jwt -> {
            Collection<GrantedAuthority> authorities = new ArrayList<>();
            
            // Extrair roles do JWT
            List<String> roles = jwt.getClaimAsStringList("cognito:groups");
            if (roles != null) {
                authorities.addAll(roles.stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    .collect(Collectors.toList()));
            }
            
            // Extrair scopes
            List<String> scopes = jwt.getClaimAsStringList("scope");
            if (scopes != null) {
                authorities.addAll(scopes.stream()
                    .map(scope -> new SimpleGrantedAuthority("SCOPE_" + scope))
                    .collect(Collectors.toList()));
            }
            
            return authorities;
        };
    }
    
    @Bean
    public OAuth2TokenValidator<Jwt> jwtValidator() {
        List<OAuth2TokenValidator<Jwt>> validators = new ArrayList<>();
        validators.add(new JwtTimestampValidator());
        validators.add(new JwtIssuerValidator("https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX"));
        validators.add(new JwtAudienceValidator("your-app-client-id"));
        
        return new DelegatingOAuth2TokenValidator<>(validators);
    }
}
```

**2. Method-Level Security**
```java
@Service
@PreAuthorize("hasRole('BANKING_OFFICER')")
public class BankingService {
    
    @PreAuthorize("hasPermission(#accountId, 'ACCOUNT', 'READ')")
    public Account getAccount(String accountId) {
        return accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
    
    @PreAuthorize("hasPermission(#fromAccountId, 'ACCOUNT', 'WRITE') and hasPermission(#toAccountId, 'ACCOUNT', 'WRITE')")
    @PostAuthorize("returnObject.amount <= 10000 or hasRole('SENIOR_OFFICER')")
    public TransactionResult transfer(String fromAccountId, String toAccountId, BigDecimal amount) {
        // Lógica de transferência
        return transactionService.processTransfer(fromAccountId, toAccountId, amount);
    }
    
    @PreAuthorize("@securityService.canAccessCustomerData(authentication.name, #customerId)")
    public CustomerData getCustomerData(String customerId) {
        return customerRepository.findById(customerId)
            .map(CustomerData::from)
            .orElseThrow(() -> new CustomerNotFoundException(customerId));
    }
}

// Security Service para validações customizadas
@Service
public class SecurityService {
    
    private final AccountRepository accountRepository;
    private final UserRepository userRepository;
    
    public boolean canAccessCustomerData(String username, String customerId) {
        User user = userRepository.findByUsername(username);
        
        // Administradores podem ver qualquer cliente
        if (user.hasRole("ADMIN")) {
            return true;
        }
        
        // Usuários só podem ver seus próprios dados
        if (user.hasRole("CUSTOMER")) {
            return user.getCustomerId().equals(customerId);
        }
        
        // Funcionários podem ver clientes de sua agência
        if (user.hasRole("TELLER")) {
            Customer customer = customerRepository.findById(customerId);
            return customer.getBranchId().equals(user.getBranchId());
        }
        
        return false;
    }
    
    public boolean isAccountOwner(String accountId, String username) {
        Account account = accountRepository.findById(accountId);
        User user = userRepository.findByUsername(username);
        
        return account.getCustomerId().equals(user.getCustomerId());
    }
}
```

### **💡 Caso de Uso Prático: Sistema de Internet Banking**

```java
@RestController
@RequestMapping("/api/banking")
@PreAuthorize("hasRole('CUSTOMER')")
public class InternetBankingController {
    
    private final BankingService bankingService;
    private final SecurityService securityService;
    
    @GetMapping("/accounts")
    public ResponseEntity<List<AccountSummary>> getMyAccounts(Authentication authentication) {
        String customerId = extractCustomerId(authentication);
        List<AccountSummary> accounts = bankingService.getAccountsByCustomer(customerId);
        return ResponseEntity.ok(accounts);
    }
    
    @GetMapping("/accounts/{accountId}/transactions")
    @PreAuthorize("@securityService.isAccountOwner(#accountId, authentication.name)")
    public ResponseEntity<List<Transaction>> getTransactions(
            @PathVariable String accountId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            Authentication authentication) {
        
        Pageable pageable = PageRequest.of(page, size);
        List<Transaction> transactions = bankingService.getTransactions(accountId, pageable);
        
        // Filtrar informações sensíveis baseado no perfil do usuário
        List<Transaction> filteredTransactions = transactions.stream()
            .map(this::filterSensitiveData)
            .collect(Collectors.toList());
        
        return ResponseEntity.ok(filteredTransactions);
    }
    
    @PostMapping("/transfer")
    @PreAuthorize("@securityService.isAccountOwner(#request.fromAccountId, authentication.name)")
    public ResponseEntity<TransactionResult> transfer(
            @Valid @RequestBody TransferRequest request,
            Authentication authentication) {
        
        // Validações de segurança adicionais
        validateTransferLimits(request, authentication);
        
        // Log de auditoria
        auditService.logTransferAttempt(authentication.getName(), request);
        
        try {
            TransactionResult result = bankingService.processTransfer(request);
            
            // Log de sucesso
            auditService.logTransferSuccess(authentication.getName(), result);
            
            return ResponseEntity.ok(result);
            
        } catch (Exception e) {
            // Log de erro
            auditService.logTransferError(authentication.getName(), request, e);
            throw e;
        }
    }
    
    private void validateTransferLimits(TransferRequest request, Authentication authentication) {
        User user = userRepository.findByUsername(authentication.getName());
        
        // Verificar limite diário
        BigDecimal dailyLimit = user.getDailyTransferLimit();
        BigDecimal todayTotal = transactionService.getTodayTotal(user.getCustomerId());
        
        if (todayTotal.add(request.getAmount()).compareTo(dailyLimit) > 0) {
            throw new TransferLimitExceededException("Daily transfer limit exceeded");
        }
        
        // Verificar limite por transação
        BigDecimal transactionLimit = user.getTransactionLimit();
        if (request.getAmount().compareTo(transactionLimit) > 0) {
            throw new TransferLimitExceededException("Transaction limit exceeded");
        }
    }
    
    private String extractCustomerId(Authentication authentication) {
        Jwt jwt = (Jwt) authentication.getPrincipal();
        return jwt.getClaimAsString("custom:customer_id");
    }
    
    private Transaction filterSensitiveData(Transaction transaction) {
        // Mascarar dados sensíveis baseado no contexto de segurança
        if (!SecurityContextHolder.getContext().getAuthentication().getAuthorities()
                .contains(new SimpleGrantedAuthority("ROLE_ADMIN"))) {
            // Mascarar número de conta de destino
            transaction.setToAccountId(maskAccountNumber(transaction.getToAccountId()));
        }
        
        return transaction;
    }
    
    private String maskAccountNumber(String accountNumber) {
        if (accountNumber.length() <= 4) {
            return accountNumber;
        }
        return "****" + accountNumber.substring(accountNumber.length() - 4);
    }
}
```

---

## 🎯 **Conceito 3: Spring Data Avançado**

### **🔍 O que é?**
Spring Data é como ter um bibliotecário especializado que sabe exatamente onde encontrar qualquer informação, seja em livros físicos (SQL) ou digitais (NoSQL).

### **🎓 Conceitos Avançados:**

**1. Custom Repository Implementation**
```java
// Interface customizada
public interface CustomAccountRepository {
    List<Account> findAccountsWithHighBalance(BigDecimal threshold);
    List<Account> findAccountsByComplexCriteria(AccountSearchCriteria criteria);
    void updateBalanceOptimized(String accountId, BigDecimal amount);
}

// Implementação customizada
@Repository
public class CustomAccountRepositoryImpl implements CustomAccountRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public List<Account> findAccountsWithHighBalance(BigDecimal threshold) {
        String jpql = """
            SELECT a FROM Account a 
            WHERE a.balance > :threshold 
            AND a.status = 'ACTIVE' 
            AND a.type IN ('CHECKING', 'SAVINGS')
            ORDER BY a.balance DESC
            """;
        
        return entityManager.createQuery(jpql, Account.class)
            .setParameter("threshold", threshold)
            .getResultList();
    }
    
    @Override
    public List<Account> findAccountsByComplexCriteria(AccountSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Account> query = cb.createQuery(Account.class);
        Root<Account> root = query.from(Account.class);
        
        List<Predicate> predicates = new ArrayList<>();
        
        if (criteria.getMinBalance() != null) {
            predicates.add(cb.greaterThanOrEqualTo(root.get("balance"), criteria.getMinBalance()));
        }
        
        if (criteria.getMaxBalance() != null) {
            predicates.add(cb.lessThanOrEqualTo(root.get("balance"), criteria.getMaxBalance()));
        }
        
        if (criteria.getAccountType() != null) {
            predicates.add(cb.equal(root.get("type"), criteria.getAccountType()));
        }
        
        if (criteria.getCustomerName() != null) {
            Join<Account, Customer> customerJoin = root.join("customer");
            predicates.add(cb.like(cb.lower(customerJoin.get("name")), 
                "%" + criteria.getCustomerName().toLowerCase() + "%"));
        }
        
        query.select(root).where(cb.and(predicates.toArray(new Predicate[0])));
        
        return entityManager.createQuery(query).getResultList();
    }
    
    @Override
    @Modifying
    @Transactional
    public void updateBalanceOptimized(String accountId, BigDecimal amount) {
        String jpql = """
            UPDATE Account a 
            SET a.balance = a.balance + :amount, 
                a.lastModified = CURRENT_TIMESTAMP 
            WHERE a.id = :accountId
            """;
        
        int updatedRows = entityManager.createQuery(jpql)
            .setParameter("amount", amount)
            .setParameter("accountId", accountId)
            .executeUpdate();
        
        if (updatedRows == 0) {
            throw new AccountNotFoundException("Account not found: " + accountId);
        }
    }
}

// Repository principal
@Repository
public interface AccountRepository extends JpaRepository<Account, String>, CustomAccountRepository {
    
    @Query("SELECT a FROM Account a WHERE a.customer.id = :customerId AND a.status = 'ACTIVE'")
    List<Account> findActiveAccountsByCustomer(@Param("customerId") String customerId);
    
    @Query(value = "SELECT * FROM accounts WHERE balance > :threshold ORDER BY balance DESC LIMIT :limit", 
           nativeQuery = true)
    List<Account> findTopAccountsByBalance(@Param("threshold") BigDecimal threshold, 
                                         @Param("limit") int limit);
    
    @Modifying
    @Query("UPDATE Account a SET a.status = 'INACTIVE' WHERE a.lastTransaction < :cutoffDate")
    int markInactiveAccounts(@Param("cutoffDate") LocalDateTime cutoffDate);
}
```

**2. Redis Cache Integration**
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        LettuceConnectionFactory factory = new LettuceConnectionFactory("localhost", 6379);
        factory.setValidateConnection(true);
        return factory;
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
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
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .transactionAware()
            .build();
    }
}

// Service com cache
@Service
public class AccountService {
    
    private final AccountRepository accountRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    
    @Cacheable(value = "accounts", key = "#accountId")
    public Account getAccount(String accountId) {
        return accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
    
    @CacheEvict(value = "accounts", key = "#account.id")
    public Account updateAccount(Account account) {
        return accountRepository.save(account);
    }
    
    @CachePut(value = "accounts", key = "#result.id")
    public Account createAccount(Account account) {
        return accountRepository.save(account);
    }
    
    // Cache manual para lógica complexa
    public List<Account> getAccountsByCustomer(String customerId) {
        String cacheKey = "customer:accounts:" + customerId;
        
        // Tentar buscar no cache
        List<Account> cachedAccounts = (List<Account>) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedAccounts != null) {
            return cachedAccounts;
        }
        
        // Buscar no banco
        List<Account> accounts = accountRepository.findActiveAccountsByCustomer(customerId);
        
        // Cachear por 1 hora
        redisTemplate.opsForValue().set(cacheKey, accounts, Duration.ofHours(1));
        
        return accounts;
    }
    
    // Invalidar cache relacionado
    @CacheEvict(value = "accounts", allEntries = true)
    public void invalidateAccountCache() {
        // Método para invalidar todo o cache de accounts
    }
}
```

### **💡 Caso de Uso Prático: Sistema de Cache Inteligente**

```java
@Service
public class SmartCacheService {
    
    private final RedisTemplate<String, Object> redisTemplate;
    private final AccountRepository accountRepository;
    
    // Cache com TTL dinâmico baseado na frequência de acesso
    public Account getAccountWithSmartCaching(String accountId) {
        String cacheKey = "account:" + accountId;
        String accessCountKey = "account:access:" + accountId;
        
        // Buscar no cache
        Account cachedAccount = (Account) redisTemplate.opsForValue().get(cacheKey);
        
        if (cachedAccount != null) {
            // Incrementar contador de acesso
            redisTemplate.opsForValue().increment(accessCountKey);
            return cachedAccount;
        }
        
        // Buscar no banco
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        
        // Determinar TTL baseado na frequência de acesso
        Long accessCount = (Long) redisTemplate.opsForValue().get(accessCountKey);
        Duration ttl = calculateTTL(accessCount);
        
        // Cachear com TTL dinâmico
        redisTemplate.opsForValue().set(cacheKey, account, ttl);
        redisTemplate.opsForValue().increment(accessCountKey);
        
        return account;
    }
    
    private Duration calculateTTL(Long accessCount) {
        if (accessCount == null) {
            return Duration.ofMinutes(5); // Default TTL
        }
        
        if (accessCount > 100) {
            return Duration.ofHours(2); // Muito acessado
        } else if (accessCount > 50) {
            return Duration.ofHours(1); // Moderadamente acessado
        } else {
            return Duration.ofMinutes(15); // Pouco acessado
        }
    }
    
    // Cache warming para dados críticos
    @EventListener(ApplicationReadyEvent.class)
    public void warmUpCache() {
        log.info("Starting cache warm-up...");
        
        // Carregar contas mais acessadas
        List<Account> frequentAccounts = accountRepository.findTopAccountsByBalance(
            BigDecimal.valueOf(10000), 100);
        
        frequentAccounts.forEach(account -> {
            String cacheKey = "account:" + account.getId();
            redisTemplate.opsForValue().set(cacheKey, account, Duration.ofHours(1));
        });
        
        log.info("Cache warm-up completed for {} accounts", frequentAccounts.size());
    }
}
```

---

## 🎯 **Exercícios Práticos**

### **Exercício 1: Sistema de Notificações**
**Cenário:** Criar sistema de notificações com diferentes canais (email, SMS, push).

**Sua tarefa:**
- Implementar Strategy pattern com Spring
- Usar @Conditional para auto-configuração
- Criar métricas customizadas

### **Exercício 2: Sistema de Auditoria**
**Cenário:** Implementar auditoria completa com Spring AOP.

**Sua tarefa:**
- Criar aspecto para log de operações
- Implementar rastreamento de alterações
- Configurar diferentes níveis de log

### **Exercício 3: Cache Distribuído**
**Cenário:** Sistema de cache para dados de clientes com invalidação inteligente.

**Sua tarefa:**
- Implementar cache em múltiplas camadas
- Criar estratégias de invalidação
- Monitorar performance do cache

---

## 🚀 **Próximos Passos**

1. **Pratique os conceitos** - Implemente cada exemplo
2. **Estude AWS** - Próximo módulo: 03-AWS-Servicos.md
3. **Explore Spring Cloud** - Microservices patterns
4. **Implemente projetos reais** - Combine todos os conceitos

**Lembre-se:** Spring é um ecossistema. Domine os fundamentos primeiro! 