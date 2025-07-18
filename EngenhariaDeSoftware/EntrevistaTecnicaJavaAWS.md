# Entrevista Técnica Java e AWS - Guia Completo

## Índice
1. [Arquitetura de Sistemas](#arquitetura-de-sistemas)
2. [Java e Spring Framework](#java-e-spring-framework)
3. [AWS - Conceitos e Ferramentas](#aws---conceitos-e-ferramentas)
4. [CI/CD e DevOps](#cicd-e-devops)
5. [Qualidade de Código e Boas Práticas](#qualidade-de-código-e-boas-práticas)
6. [Clean Code e SOLID](#clean-code-e-solid)
7. [Estratégias de Deploy](#estratégias-de-deploy)
8. [Pirâmide de Testes](#pirâmide-de-testes)
9. [Mensageria e Filas](#mensageria-e-filas)
10. [Sistemas Bancários](#sistemas-bancários)
11. [Monitoramento e Observabilidade](#monitoramento-e-observabilidade)
12. [Casos de Uso Práticos](#casos-de-uso-práticos)

---

## Arquitetura de Sistemas

### 1. Quais são os principais tipos de arquitetura de sistemas?

**Resposta:**
- **Monolítica**: Toda aplicação em um único deployable
  - Vantagens: Simplicidade, desenvolvimento inicial rápido, debugging mais fácil
  - Desvantagens: Acoplamento, dificuldade de escalabilidade, tecnologia única

- **Microserviços**: Aplicação dividida em serviços pequenos e independentes
  - Vantagens: Escalabilidade independente, tecnologias diferentes, resiliência
  - Desvantagens: Complexidade de rede, distributed transactions, debugging complexo

- **SOA (Service-Oriented Architecture)**: Serviços empresariais através de contratos
  - Vantagens: Reusabilidade, interoperabilidade
  - Desvantagens: Overhead de comunicação, complexidade de governança

- **Event-Driven Architecture**: Comunicação através de eventos
  - Vantagens: Desacoplamento, escalabilidade, processamento assíncrono
  - Desvantagens: Complexidade de debugging, eventual consistency

### 2. Como implementar Domain-Driven Design (DDD)?

**Resposta:**
```java
// Exemplo de implementação DDD
// Value Object
public class Money {
    private final BigDecimal amount;
    private final Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        this.amount = Objects.requireNonNull(amount);
        this.currency = Objects.requireNonNull(currency);
    }
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

// Entity
@Entity
public class Account {
    @Id
    private AccountId id;
    private Money balance;
    private CustomerId customerId;
    
    public void withdraw(Money amount) {
        if (balance.isLessThan(amount)) {
            throw new InsufficientFundsException();
        }
        this.balance = balance.subtract(amount);
        // Publish domain event
        DomainEvents.publish(new MoneyWithdrawnEvent(id, amount));
    }
}

// Aggregate Root
@Entity
public class Customer {
    @Id
    private CustomerId id;
    @OneToMany(mappedBy = "customerId")
    private List<Account> accounts;
    
    public void openAccount(Money initialDeposit) {
        Account account = new Account(new AccountId(), initialDeposit, this.id);
        accounts.add(account);
        DomainEvents.publish(new AccountOpenedEvent(account.getId()));
    }
}
```

### 3. Explique o padrão Hexagonal (Ports and Adapters)

**Resposta:**
```java
// Port (Interface)
public interface PaymentGateway {
    PaymentResult processPayment(Payment payment);
}

// Adapter (Implementação)
@Component
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult processPayment(Payment payment) {
        // Implementação específica do Stripe
        return stripeClient.charge(payment.getAmount(), payment.getCard());
    }
}

// Domain Service
@Service
public class PaymentService {
    private final PaymentGateway paymentGateway;
    
    public PaymentService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }
    
    public PaymentResult processPayment(Payment payment) {
        // Lógica de domínio
        validatePayment(payment);
        return paymentGateway.processPayment(payment);
    }
}
```

### 4. O que é CQRS e quando usar?

**Resposta:**
CQRS (Command Query Responsibility Segregation) separa operações de leitura e escrita:

```java
// Command Side
@Component
public class AccountCommandHandler {
    @EventSourcing
    public void handle(CreateAccountCommand command) {
        Account account = new Account(command.getId(), command.getInitialBalance());
        accountRepository.save(account);
    }
}

// Query Side
@Component
public class AccountQueryHandler {
    @ReadOnly
    public AccountView getAccount(String accountId) {
        return accountViewRepository.findById(accountId);
    }
}
```

**Quando usar:**
- Sistemas com alta complexidade de leitura vs escrita
- Necessidade de otimização independente
- Auditoria e compliance rigorosos
- Event Sourcing

---

## Java e Spring Framework

### 1. Principais anotações Spring e seus usos

**Resposta:**
```java
// Configuração
@Configuration
@EnableJpaRepositories
@EnableTransactionManagement
public class DatabaseConfig {
    
    @Bean
    @Primary
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }
}

// Componentes
@RestController
@RequestMapping("/api/accounts")
@Validated
public class AccountController {
    
    @Autowired
    private AccountService accountService;
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ResponseEntity<Account> createAccount(@Valid @RequestBody CreateAccountRequest request) {
        Account account = accountService.createAccount(request);
        return ResponseEntity.created(URI.create("/api/accounts/" + account.getId())).body(account);
    }
    
    @GetMapping("/{id}")
    @Cacheable(value = "accounts", key = "#id")
    public Account getAccount(@PathVariable String id) {
        return accountService.getAccount(id);
    }
}

// Service
@Service
@Transactional
public class AccountService {
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Retryable(value = {Exception.class}, maxAttempts = 3)
    public Account createAccount(CreateAccountRequest request) {
        // Lógica de negócio
        Account account = new Account(request.getCustomerId(), request.getInitialBalance());
        return accountRepository.save(account);
    }
    
    @Recover
    public Account recover(Exception ex, CreateAccountRequest request) {
        // Fallback logic
        throw new AccountCreationFailedException("Failed to create account after retries");
    }
}

// Repository
@Repository
public interface AccountRepository extends JpaRepository<Account, String> {
    
    @Query("SELECT a FROM Account a WHERE a.balance > :minBalance")
    List<Account> findAccountsWithMinimumBalance(@Param("minBalance") BigDecimal minBalance);
    
    @Modifying
    @Query("UPDATE Account a SET a.balance = a.balance + :amount WHERE a.id = :accountId")
    int updateBalance(@Param("accountId") String accountId, @Param("amount") BigDecimal amount);
}
```

### 2. Spring Security Configuration

**Resposta:**
```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/accounts/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt())
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.configurationSource(corsConfigurationSource()));
        
        return http.build();
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withJwkSetUri("https://cognito-idp.us-east-1.amazonaws.com/.well-known/jwks.json").build();
    }
}
```

### 3. Como implementar Cache com Redis?

**Resposta:**
```java
@Configuration
@EnableCaching
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(config)
            .build();
    }
}

@Service
public class CustomerService {
    
    @Cacheable(value = "customers", key = "#customerId")
    public Customer getCustomer(String customerId) {
        return customerRepository.findById(customerId);
    }
    
    @CacheEvict(value = "customers", key = "#customer.id")
    public Customer updateCustomer(Customer customer) {
        return customerRepository.save(customer);
    }
    
    @CachePut(value = "customers", key = "#result.id")
    public Customer createCustomer(Customer customer) {
        return customerRepository.save(customer);
    }
}
```

---

## AWS - Conceitos e Ferramentas

### 1. Principais serviços AWS para aplicações Java

**Resposta:**

**Compute:**
- **EC2**: Máquinas virtuais escaláveis
- **ECS**: Container orchestration
- **EKS**: Kubernetes gerenciado
- **Lambda**: Serverless computing
- **Elastic Beanstalk**: Platform-as-a-Service

**Storage:**
- **S3**: Object storage
- **EBS**: Block storage para EC2
- **EFS**: File system compartilhado

**Database:**
- **RDS**: Bancos relacionais gerenciados
- **DynamoDB**: NoSQL managed
- **ElastiCache**: Cache em memória (Redis/Memcached)

**Networking:**
- **VPC**: Virtual Private Cloud
- **ELB**: Load Balancer
- **CloudFront**: CDN
- **API Gateway**: Gateway para APIs

**Security:**
- **IAM**: Identity and Access Management
- **Cognito**: User authentication
- **KMS**: Key Management Service
- **WAF**: Web Application Firewall

### 2. Como configurar um ambiente AWS para aplicação Java?

**Resposta:**
```yaml
# docker-compose.yml para desenvolvimento local
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - AWS_REGION=us-east-1
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - SPRING_PROFILES_ACTIVE=aws
    depends_on:
      - localstack
      
  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,sns,dynamodb
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - "/tmp/localstack:/tmp/localstack"
```

```java
// Configuração AWS SDK
@Configuration
public class AWSConfig {
    
    @Bean
    public AmazonS3 s3Client() {
        return AmazonS3ClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
    
    @Bean
    public AmazonSQS sqsClient() {
        return AmazonSQSClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
    
    @Bean
    public AmazonDynamoDB dynamoDBClient() {
        return AmazonDynamoDBClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .withCredentials(new DefaultAWSCredentialsProviderChain())
            .build();
    }
}
```

### 3. Lambda Functions com Spring Boot

**Resposta:**
```java
@Component
public class LambdaHandler implements RequestHandler<APIGatewayProxyRequestEvent, APIGatewayProxyResponseEvent> {
    
    @Autowired
    private PaymentService paymentService;
    
    @Override
    public APIGatewayProxyResponseEvent handleRequest(APIGatewayProxyRequestEvent input, Context context) {
        try {
            PaymentRequest request = objectMapper.readValue(input.getBody(), PaymentRequest.class);
            PaymentResult result = paymentService.processPayment(request);
            
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(200)
                .withBody(objectMapper.writeValueAsString(result))
                .withHeaders(Map.of("Content-Type", "application/json"))
                .build();
        } catch (Exception e) {
            context.getLogger().log("Error processing payment: " + e.getMessage());
            return APIGatewayProxyResponseEvent.builder()
                .withStatusCode(500)
                .withBody("{\"error\":\"Internal server error\"}")
                .build();
        }
    }
}
```

### 4. DynamoDB com Spring Data

**Resposta:**
```java
@DynamoDBTable(tableName = "Transactions")
public class Transaction {
    
    @DynamoDBHashKey
    private String transactionId;
    
    @DynamoDBRangeKey
    private String timestamp;
    
    @DynamoDBAttribute
    private String accountId;
    
    @DynamoDBAttribute
    private BigDecimal amount;
    
    @DynamoDBAttribute
    private String type;
    
    @DynamoDBAttribute
    private String status;
    
    // Getters e setters
}

@Repository
public interface TransactionRepository extends CrudRepository<Transaction, String> {
    
    List<Transaction> findByAccountIdAndTimestampBetween(
        String accountId, 
        String startTimestamp, 
        String endTimestamp
    );
    
    @Query("SELECT * FROM Transactions WHERE accountId = :accountId AND #status = :status")
    List<Transaction> findByAccountIdAndStatus(
        @Param("accountId") String accountId,
        @Param("status") String status
    );
}
```

---

## CI/CD e DevOps

### 1. Etapas essenciais de uma pipeline CI/CD

**Resposta:**

**Continuous Integration (CI):**
1. **Source Control**: Git hooks, branch protection
2. **Build**: Compilação, dependências
3. **Test**: Unit tests, integration tests
4. **Code Analysis**: SonarQube, SpotBugs
5. **Security Scan**: OWASP dependency check
6. **Package**: Docker image, JAR/WAR

**Continuous Delivery (CD):**
1. **Deploy to Staging**: Ambiente de homologação
2. **End-to-End Tests**: Testes automatizados
3. **Performance Tests**: Load testing
4. **Security Tests**: Penetration testing
5. **Approval Gates**: Manual approval para produção
6. **Deploy to Production**: Deploy automatizado

```yaml
# GitHub Actions exemplo
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up JDK 17
      uses: actions/setup-java@v2
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Cache Maven dependencies
      uses: actions/cache@v2
      with:
        path: ~/.m2
        key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
    
    - name: Run tests
      run: mvn clean test
    
    - name: Code Coverage
      run: mvn jacoco:report
    
    - name: SonarQube Scan
      run: mvn sonar:sonar
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
    
    - name: Build Docker image
      run: |
        docker build -t myapp:${{ github.sha }} .
        docker tag myapp:${{ github.sha }} myapp:latest
    
    - name: Push to ECR
      run: |
        aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_REGISTRY
        docker push $ECR_REGISTRY/myapp:${{ github.sha }}
        docker push $ECR_REGISTRY/myapp:latest
```

### 2. Dockerfile otimizado para Spring Boot

**Resposta:**
```dockerfile
# Multi-stage build
FROM maven:3.8.6-openjdk-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Runtime image
FROM openjdk:17-jre-slim
WORKDIR /app

# Criar usuário não-root
RUN addgroup --system spring && adduser --system spring --ingroup spring
USER spring:spring

# Copiar JAR
COPY --from=builder /app/target/myapp-*.jar app.jar

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Expor porta
EXPOSE 8080

# JVM options
ENV JAVA_OPTS="-Xmx512m -Xms256m -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# Entrypoint
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

---

## Qualidade de Código e Boas Práticas

### 1. Configuração de ferramentas de qualidade

**Resposta:**
```xml
<!-- pom.xml -->
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.9.1.2184</version>
</plugin>

<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.7</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>

<plugin>
    <groupId>com.github.spotbugs</groupId>
    <artifactId>spotbugs-maven-plugin</artifactId>
    <version>4.7.1.1</version>
</plugin>

<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>7.1.1</version>
</plugin>
```

### 2. Padrões de código e formatação

**Resposta:**
```java
// Exemplo de classe bem estruturada
@Service
@Slf4j
public class BankAccountService {
    
    private static final BigDecimal MINIMUM_BALANCE = BigDecimal.valueOf(100);
    private static final String TRANSACTION_PREFIX = "TXN";
    
    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;
    private final NotificationService notificationService;
    
    public BankAccountService(AccountRepository accountRepository,
                             TransactionRepository transactionRepository,
                             NotificationService notificationService) {
        this.accountRepository = accountRepository;
        this.transactionRepository = transactionRepository;
        this.notificationService = notificationService;
    }
    
    @Transactional
    public TransactionResult transfer(String fromAccountId, String toAccountId, BigDecimal amount) {
        validateTransferRequest(fromAccountId, toAccountId, amount);
        
        Account fromAccount = getAccountById(fromAccountId);
        Account toAccount = getAccountById(toAccountId);
        
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Insufficient funds in account: " + fromAccountId);
        }
        
        fromAccount.debit(amount);
        toAccount.credit(amount);
        
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
        
        Transaction transaction = createTransaction(fromAccountId, toAccountId, amount);
        transactionRepository.save(transaction);
        
        notificationService.notifyTransfer(fromAccount, toAccount, amount);
        
        log.info("Transfer completed successfully: {} -> {} amount: {}", 
                fromAccountId, toAccountId, amount);
        
        return new TransactionResult(transaction.getId(), TransactionStatus.SUCCESS);
    }
    
    private void validateTransferRequest(String fromAccountId, String toAccountId, BigDecimal amount) {
        if (StringUtils.isBlank(fromAccountId) || StringUtils.isBlank(toAccountId)) {
            throw new IllegalArgumentException("Account IDs cannot be blank");
        }
        
        if (fromAccountId.equals(toAccountId)) {
            throw new IllegalArgumentException("Cannot transfer to the same account");
        }
        
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }
    
    private Account getAccountById(String accountId) {
        return accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException("Account not found: " + accountId));
    }
    
    private Transaction createTransaction(String fromAccountId, String toAccountId, BigDecimal amount) {
        return Transaction.builder()
            .id(generateTransactionId())
            .fromAccountId(fromAccountId)
            .toAccountId(toAccountId)
            .amount(amount)
            .type(TransactionType.TRANSFER)
            .status(TransactionStatus.SUCCESS)
            .timestamp(Instant.now())
            .build();
    }
    
    private String generateTransactionId() {
        return TRANSACTION_PREFIX + System.currentTimeMillis() + UUID.randomUUID().toString().substring(0, 8);
    }
}
```

---

## Clean Code e SOLID

### 1. Princípios SOLID explicados com exemplos

**Resposta:**

**Single Responsibility Principle (SRP):**
```java
// ❌ Violação do SRP
public class User {
    private String name;
    private String email;
    
    public void save() {
        // Lógica de persistência
    }
    
    public void sendEmail() {
        // Lógica de envio de email
    }
}

// ✅ Seguindo SRP
public class User {
    private String name;
    private String email;
    
    // Apenas dados do usuário
}

public class UserRepository {
    public void save(User user) {
        // Lógica de persistência
    }
}

public class EmailService {
    public void sendEmail(User user, String message) {
        // Lógica de envio de email
    }
}
```

**Open/Closed Principle (OCP):**
```java
// ✅ Aberto para extensão, fechado para modificação
public abstract class PaymentProcessor {
    public abstract PaymentResult process(Payment payment);
}

public class CreditCardProcessor extends PaymentProcessor {
    @Override
    public PaymentResult process(Payment payment) {
        // Lógica específica para cartão de crédito
    }
}

public class PayPalProcessor extends PaymentProcessor {
    @Override
    public PaymentResult process(Payment payment) {
        // Lógica específica para PayPal
    }
}

public class PaymentService {
    public PaymentResult processPayment(Payment payment, PaymentProcessor processor) {
        return processor.process(payment);
    }
}
```

**Liskov Substitution Principle (LSP):**
```java
// ✅ Subclasses devem ser substituíveis por suas classes base
public abstract class Shape {
    public abstract double calculateArea();
}

public class Rectangle extends Shape {
    protected double width;
    protected double height;
    
    @Override
    public double calculateArea() {
        return width * height;
    }
}

public class Square extends Shape {
    private double side;
    
    @Override
    public double calculateArea() {
        return side * side;
    }
}
```

**Interface Segregation Principle (ISP):**
```java
// ❌ Interface muito gorda
public interface Worker {
    void work();
    void eat();
    void sleep();
}

// ✅ Interfaces segregadas
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class Human implements Workable, Eatable, Sleepable {
    // Implementação
}

public class Robot implements Workable {
    // Robô não precisa comer ou dormir
}
```

**Dependency Inversion Principle (DIP):**
```java
// ✅ Dependência de abstrações, não de concretização
public interface NotificationService {
    void sendNotification(String message);
}

public class EmailNotificationService implements NotificationService {
    @Override
    public void sendNotification(String message) {
        // Envio por email
    }
}

public class SMSNotificationService implements NotificationService {
    @Override
    public void sendNotification(String message) {
        // Envio por SMS
    }
}

public class OrderService {
    private final NotificationService notificationService;
    
    public OrderService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
    
    public void processOrder(Order order) {
        // Processar pedido
        notificationService.sendNotification("Order processed: " + order.getId());
    }
}
```

### 2. Clean Code - Boas práticas

**Resposta:**
```java
// ✅ Nomes significativos
public class BankAccountService {
    private static final int MINIMUM_BALANCE_THRESHOLD = 100;
    
    public TransferResult transferFunds(String sourceAccountId, String targetAccountId, BigDecimal amount) {
        validateTransferParameters(sourceAccountId, targetAccountId, amount);
        
        Account sourceAccount = findAccountById(sourceAccountId);
        Account toAccount = findAccountById(toAccountId);
        
        if (hasInsufficientFunds(sourceAccount, amount)) {
            return TransferResult.failure("Insufficient funds");
        }
        
        executeTransfer(sourceAccount, toAccount, amount);
        return TransferResult.success();
    }
    
    private void validateTransferParameters(String sourceAccountId, String toAccountId, BigDecimal amount) {
        if (isBlank(sourceAccountId) || isBlank(toAccountId)) {
            throw new IllegalArgumentException("Account IDs cannot be blank");
        }
        
        if (fromAccountId.equals(toAccountId)) {
            throw new IllegalArgumentException("Cannot transfer to the same account");
        }
        
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }
    
    private boolean hasInsufficientFunds(Account account, BigDecimal amount) {
        return account.getBalance().compareTo(amount) < 0;
    }
    
    private boolean isZeroOrNegative(BigDecimal amount) {
        return amount.compareTo(BigDecimal.ZERO) <= 0;
    }
    
    private boolean isBlank(String value) {
        return value == null || value.trim().isEmpty();
    }
}
```

---

## Estratégias de Deploy

### 1. Blue-Green Deployment

**Resposta:**
```yaml
# Blue-Green deployment com Docker Compose
version: '3.8'
services:
  # Blue Environment
  app-blue:
    image: myapp:blue
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=blue
      - DATABASE_URL=jdbc:postgresql://db:5432/myapp
    depends_on:
      - db
    networks:
      - app-network
  
  # Green Environment  
  app-green:
    image: myapp:green
    ports:
      - "8081:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=green
      - DATABASE_URL=jdbc:postgresql://db:5432/myapp
    depends_on:
      - db
    networks:
      - app-network
  
  # Load Balancer
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app-blue
      - app-green
    networks:
      - app-network
```

```bash
# Script de deploy Blue-Green
#!/bin/bash

CURRENT_ENV=$(curl -s http://localhost/health | jq -r '.environment')
NEW_ENV=$([ "$CURRENT_ENV" = "blue" ] && echo "green" || echo "blue")

echo "Current environment: $CURRENT_ENV"
echo "Deploying to: $NEW_ENV"

# Deploy new version
docker-compose up -d app-$NEW_ENV

# Health check
for i in {1..30}; do
    if curl -f http://localhost:808$([[ "$NEW_ENV" == "green" ]] && echo "1" || echo "0")/actuator/health; then
        echo "Health check passed"
        break
    fi
    sleep 10
done

# Switch traffic
sed -i "s/app-$CURRENT_ENV/app-$NEW_ENV/g" nginx.conf
docker-compose exec nginx nginx -s reload

# Stop old environment
docker-compose stop app-$CURRENT_ENV
```

### 2. Canary Deployment

**Resposta:**
```yaml
# Kubernetes Canary Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: myapp:v1.1
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

### 3. Rolling Deployment

**Resposta:**
```yaml
# Kubernetes Rolling Update
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 20
```

---

## Pirâmide de Testes

### 1. Estrutura da Pirâmide de Testes

**Resposta:**
```
        /\
       /  \
      / E2E \     <- Poucos testes (UI/API end-to-end)
     /______\
    /        \
   /Integration\ <- Testes de integração (componentes)
  /____________\
 /              \
/  Unit Tests    \  <- Muitos testes (métodos/classes)
/________________\
```

### 2. Testes Unitários

**Resposta:**
```java
@ExtendWith(MockitoExtension.class)
class BankAccountServiceTest {
    
    @Mock
    private AccountRepository accountRepository;
    
    @Mock
    private TransactionRepository transactionRepository;
    
    @Mock
    private NotificationService notificationService;
    
    @InjectMocks
    private BankAccountService bankAccountService;
    
    @Test
    void shouldTransferFundsSuccessfully() {
        // Given
        String fromAccountId = "ACC001";
        String toAccountId = "ACC002";
        BigDecimal amount = BigDecimal.valueOf(100);
        
        Account fromAccount = Account.builder()
            .id(fromAccountId)
            .balance(BigDecimal.valueOf(500))
            .build();
            
        Account toAccount = Account.builder()
            .id(toAccountId)
            .balance(BigDecimal.valueOf(200))
            .build();
        
        when(accountRepository.findById(fromAccountId)).thenReturn(Optional.of(fromAccount));
        when(accountRepository.findById(toAccountId)).thenReturn(Optional.of(toAccount));
        when(transactionRepository.save(any(Transaction.class))).thenReturn(new Transaction());
        
        // When
        TransactionResult result = bankAccountService.transfer(fromAccountId, toAccountId, amount);
        
        // Then
        assertThat(result.getStatus()).isEqualTo(TransactionStatus.SUCCESS);
        verify(accountRepository, times(2)).save(any(Account.class));
        verify(transactionRepository).save(any(Transaction.class));
        verify(notificationService).notifyTransfer(fromAccount, toAccount, amount);
    }
    
    @Test
    void shouldThrowExceptionWhenInsufficientFunds() {
        // Given
        String fromAccountId = "ACC001";
        String toAccountId = "ACC002";
        BigDecimal amount = BigDecimal.valueOf(600);
        
        Account fromAccount = Account.builder()
            .id(fromAccountId)
            .balance(BigDecimal.valueOf(500))
            .build();
            
        when(accountRepository.findById(fromAccountId)).thenReturn(Optional.of(fromAccount));
        
        // When & Then
        assertThatThrownBy(() -> bankAccountService.transfer(fromAccountId, toAccountId, amount))
            .isInstanceOf(InsufficientFundsException.class)
            .hasMessage("Insufficient funds in account: " + fromAccountId);
    }
}
```

### 3. Testes de Integração

**Resposta:**
```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
@Transactional
class BankAccountIntegrationTest {
    
    @Autowired
    private BankAccountService bankAccountService;
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Autowired
    private TransactionRepository transactionRepository;
    
    @Test
    void shouldTransferFundsAndPersistTransaction() {
        // Given
        Account fromAccount = Account.builder()
            .id("ACC001")
            .balance(BigDecimal.valueOf(1000))
            .build();
            
        Account toAccount = Account.builder()
            .id("ACC002")
            .balance(BigDecimal.valueOf(500))
            .build();
        
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
        
        BigDecimal transferAmount = BigDecimal.valueOf(200);
        
        // When
        TransactionResult result = bankAccountService.transfer("ACC001", "ACC002", transferAmount);
        
        // Then
        assertThat(result.getStatus()).isEqualTo(TransactionStatus.SUCCESS);
        
        // Verify balances were updated
        Account updatedFromAccount = accountRepository.findById("ACC001").get();
        Account updatedToAccount = accountRepository.findById("ACC002").get();
        
        assertThat(updatedFromAccount.getBalance()).isEqualTo(BigDecimal.valueOf(800));
        assertThat(updatedToAccount.getBalance()).isEqualTo(BigDecimal.valueOf(700));
        
        // Verify transaction was created
        List<Transaction> transactions = transactionRepository.findAll();
        assertThat(transactions).hasSize(1);
        
        Transaction transaction = transactions.get(0);
        assertThat(transaction.getFromAccountId()).isEqualTo("ACC001");
        assertThat(transaction.getToAccountId()).isEqualTo("ACC002");
        assertThat(transaction.getAmount()).isEqualTo(transferAmount);
    }
}
```

### 4. Testes End-to-End

**Resposta:**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class BankAccountE2ETest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private AccountRepository accountRepository;
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Test
    void shouldCompleteTransferEndToEnd() {
        // Given
        Account fromAccount = Account.builder()
            .id("ACC001")
            .balance(BigDecimal.valueOf(1000))
            .build();
            
        Account toAccount = Account.builder()
            .id("ACC002")
            .balance(BigDecimal.valueOf(500))
            .build();
        
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
        
        TransferRequest request = TransferRequest.builder()
            .fromAccountId("ACC001")
            .toAccountId("ACC002")
            .amount(BigDecimal.valueOf(200))
            .build();
        
        // When
        ResponseEntity<TransactionResult> response = restTemplate.postForEntity(
            "/api/accounts/transfer",
            request,
            TransactionResult.class
        );
        
        // Then
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody().getStatus()).isEqualTo(TransactionStatus.SUCCESS);
        
        // Verify through API
        ResponseEntity<Account> fromAccountResponse = restTemplate.getForEntity(
            "/api/accounts/ACC001",
            Account.class
        );
        
        ResponseEntity<Account> toAccountResponse = restTemplate.getForEntity(
            "/api/accounts/ACC002",
            Account.class
        );
        
        assertThat(fromAccountResponse.getBody().getBalance()).isEqualTo(BigDecimal.valueOf(800));
        assertThat(toAccountResponse.getBody().getBalance()).isEqualTo(BigDecimal.valueOf(700));
    }
}
```

### 5. Diferenças entre tipos de teste

**Resposta:**
- **Testes Unitários**: Testam uma única unidade de código em isolamento
  - Rápidos (< 1s)
  - Sem dependências externas
  - Alto coverage
  - Feedback imediato
  
- **Testes de Integração**: Testam interação entre componentes
  - Moderados (1-10s)
  - Banco de dados, APIs
  - Validam fluxos completos
  - Detectam problemas de integração
  
- **Testes End-to-End**: Testam sistema completo
  - Lentos (10s-minutes)
  - Interface de usuário
  - Validam experiência do usuário
  - Alta confiança, baixa manutenibilidade

---

## Mensageria e Filas

### 1. Conceitos fundamentais

**Resposta:**

**Filas (Queues):**
- Comunicação ponto-a-ponto
- Uma mensagem, um consumidor
- Garantia de entrega
- Processamento sequencial

**Tópicos (Topics):**
- Comunicação pub/sub
- Uma mensagem, múltiplos consumidores
- Broadcast de eventos
- Processamento paralelo

### 2. Implementação com RabbitMQ

**Resposta:**
```java
@Configuration
@EnableRabbit
public class RabbitConfig {
    
    public static final String TRANSACTION_QUEUE = "transaction.queue";
    public static final String NOTIFICATION_EXCHANGE = "notification.exchange";
    public static final String NOTIFICATION_QUEUE = "notification.queue";
    
    @Bean
    public Queue transactionQueue() {
        return QueueBuilder.durable(TRANSACTION_QUEUE)
            .withArgument("x-dead-letter-exchange", "dlx.exchange")
            .withArgument("x-dead-letter-routing-key", "dlx.transaction")
            .build();
    }
    
    @Bean
    public TopicExchange notificationExchange() {
        return new TopicExchange(NOTIFICATION_EXCHANGE);
    }
    
    @Bean
    public Queue notificationQueue() {
        return QueueBuilder.durable(NOTIFICATION_QUEUE).build();
    }
    
    @Bean
    public Binding notificationBinding() {
        return BindingBuilder
            .bind(notificationQueue())
            .to(notificationExchange())
            .with("notification.*");
    }
    
    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(new Jackson2JsonMessageConverter());
        template.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("Message delivered successfully");
            } else {
                log.error("Message delivery failed: {}", cause);
            }
        });
        return template;
    }
}

@Component
public class TransactionPublisher {
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    public void publishTransaction(TransactionEvent event) {
        rabbitTemplate.convertAndSend(
            RabbitConfig.TRANSACTION_QUEUE,
            event,
            message -> {
                message.getMessageProperties().setHeader("eventType", event.getType());
                message.getMessageProperties().setHeader("timestamp", System.currentTimeMillis());
                return message;
            }
        );
    }
}

@Component
public class TransactionConsumer {
    
    @RabbitListener(queues = RabbitConfig.TRANSACTION_QUEUE)
    public void handleTransaction(TransactionEvent event) {
        try {
            log.info("Processing transaction: {}", event);
            
            // Processar transação
            processTransaction(event);
            
            // Publicar evento de sucesso
            publishNotification(event);
            
        } catch (Exception e) {
            log.error("Error processing transaction: {}", e.getMessage());
            throw new AmqpRejectAndDontRequeueException("Transaction processing failed", e);
        }
    }
    
    private void processTransaction(TransactionEvent event) {
        // Lógica de processamento
    }
    
    private void publishNotification(TransactionEvent event) {
        NotificationEvent notification = NotificationEvent.builder()
            .transactionId(event.getTransactionId())
            .type(NotificationType.TRANSACTION_COMPLETED)
            .message("Transaction completed successfully")
            .build();
            
        rabbitTemplate.convertAndSend(
            RabbitConfig.NOTIFICATION_EXCHANGE,
            "notification.transaction",
            notification
        );
    }
}
```

### 3. Implementação com Amazon SQS

**Resposta:**
```java
@Configuration
public class SQSConfig {
    
    @Bean
    public AmazonSQS amazonSQS() {
        return AmazonSQSClientBuilder.standard()
            .withRegion(Regions.US_EAST_1)
            .build();
    }
    
    @Bean
    public QueueMessagingTemplate queueMessagingTemplate(AmazonSQS amazonSQS) {
        return new QueueMessagingTemplate(amazonSQS);
    }
}

@Component
public class SQSPublisher {
    
    @Autowired
    private QueueMessagingTemplate queueMessagingTemplate;
    
    @Value("${aws.sqs.transaction-queue}")
    private String transactionQueue;
    
    public void publishTransaction(TransactionEvent event) {
        try {
            Map<String, Object> headers = new HashMap<>();
            headers.put("eventType", event.getType());
            headers.put("timestamp", System.currentTimeMillis());
            
            queueMessagingTemplate.convertAndSend(transactionQueue, event, headers);
            
            log.info("Message published to SQS: {}", event);
            
        } catch (Exception e) {
            log.error("Error publishing message to SQS: {}", e.getMessage());
            throw new RuntimeException("Failed to publish message", e);
        }
    }
}

@Component
public class SQSConsumer {
    
    @SqsListener("${aws.sqs.transaction-queue}")
    public void handleTransaction(TransactionEvent event, @Header Map<String, Object> headers) {
        try {
            log.info("Received transaction from SQS: {}", event);
            
            // Processar transação
            processTransaction(event);
            
            // Mensagem será automaticamente deletada se não houver exception
            
        } catch (Exception e) {
            log.error("Error processing transaction: {}", e.getMessage());
            throw e; // Rejeitar mensagem para reprocessamento
        }
    }
}
```

### 4. Como mensageria melhora performance

**Resposta:**

**Benefícios:**
1. **Desacoplamento**: Serviços não dependem da disponibilidade uns dos outros
2. **Assincronia**: Processamento não-bloqueante
3. **Escalabilidade**: Múltiplos consumidores podem processar mensagens
4. **Reliability**: Mensagens persistem até serem processadas
5. **Load Balancing**: Distribuição automática de carga

**Exemplo prático:**
```java
// Sem mensageria - Síncrono
@Service
public class OrderService {
    
    @Autowired
    private PaymentService paymentService;
    
    @Autowired
    private InventoryService inventoryService;
    
    @Autowired
    private NotificationService notificationService;
    
    public OrderResult processOrder(Order order) {
        // Todas as operações são síncronas e bloqueantes
        PaymentResult payment = paymentService.processPayment(order.getPayment()); // 2s
        InventoryResult inventory = inventoryService.reserveItems(order.getItems()); // 1s
        notificationService.sendConfirmation(order.getCustomer()); // 3s
        
        return new OrderResult(order.getId(), "SUCCESS");
        // Total: 6 segundos
    }
}

// Com mensageria - Assíncrono
@Service
public class OrderService {
    
    @Autowired
    private OrderPublisher orderPublisher;
    
    public OrderResult processOrder(Order order) {
        // Apenas publica eventos assíncronos
        orderPublisher.publishOrderCreated(order); // 50ms
        
        return new OrderResult(order.getId(), "PROCESSING");
        // Total: 50ms - resposta imediata
    }
}
```

### 5. Padrões de mensageria

**Resposta:**

**Event Sourcing:**
```java
@Entity
public class EventStore {
    @Id
    private String id;
    private String aggregateId;
    private String eventType;
    private String eventData;
    private Instant timestamp;
    
    // Getters e setters
}

@Component
public class AccountEventHandler {
    
    @EventHandler
    public void handle(AccountCreatedEvent event) {
        // Salvar evento no event store
        eventStore.save(new EventStore(
            UUID.randomUUID().toString(),
            event.getAccountId(),
            "AccountCreated",
            objectMapper.writeValueAsString(event),
            Instant.now()
        ));
    }
}
```

**Saga Pattern:**
```java
@Component
public class OrderSaga {
    
    @SagaOrchestrationStart
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Iniciar saga
        sagaManager.choreography()
            .step("reserveInventory")
                .invoke(inventoryService::reserveItems)
                .onFailure(inventoryService::releaseItems)
            .step("processPayment")
                .invoke(paymentService::processPayment)
                .onFailure(paymentService::refundPayment)
            .step("shipOrder")
                .invoke(shippingService::shipOrder)
                .onFailure(shippingService::cancelShipment)
            .execute();
    }
}
```

---

## Sistemas Bancários

### 1. Arquitetura de sistema bancário

**Resposta:**
```java
// Core Banking System Architecture
@RestController
@RequestMapping("/api/banking")
public class BankingController {
    
    @Autowired
    private AccountService accountService;
    
    @Autowired
    private TransactionService transactionService;
    
    @Autowired
    private ComplianceService complianceService;
    
    @PostMapping("/transfer")
    @PreAuthorize("hasRole('CUSTOMER')")
    public ResponseEntity<TransactionResult> transfer(
            @Valid @RequestBody TransferRequest request,
            Authentication authentication) {
        
        // Compliance check
        complianceService.validateTransfer(request, authentication.getName());
        
        // Process transfer
        TransactionResult result = transactionService.processTransfer(request);
        
        return ResponseEntity.ok(result);
    }
    
    @GetMapping("/accounts/{accountId}/balance")
    @PreAuthorize("hasRole('CUSTOMER') and @accountService.isOwner(#accountId, authentication.name)")
    public ResponseEntity<BalanceResponse> getBalance(@PathVariable String accountId) {
        BigDecimal balance = accountService.getBalance(accountId);
        return ResponseEntity.ok(new BalanceResponse(balance));
    }
}
```

### 2. Controle de fraude

**Resposta:**
```java
@Service
public class FraudDetectionService {
    
    @Autowired
    private TransactionRepository transactionRepository;
    
    @Autowired
    private RiskAnalysisService riskAnalysisService;
    
    public FraudCheckResult analyzeTransaction(Transaction transaction) {
        List<FraudRule> rules = Arrays.asList(
            new AmountThresholdRule(BigDecimal.valueOf(10000)),
            new FrequencyRule(Duration.ofHours(1), 5),
            new UnusualLocationRule(),
            new TimePatternRule(),
            new VelocityRule()
        );
        
        FraudScore score = FraudScore.zero();
        
        for (FraudRule rule : rules) {
            FraudScore ruleScore = rule.evaluate(transaction);
            score = score.add(ruleScore);
        }
        
        if (score.isHighRisk()) {
            return FraudCheckResult.block("High fraud risk detected");
        } else if (score.isMediumRisk()) {
            return FraudCheckResult.review("Manual review required");
        } else {
            return FraudCheckResult.approve("Transaction approved");
        }
    }
}

public class AmountThresholdRule implements FraudRule {
    private final BigDecimal threshold;
    
    public AmountThresholdRule(BigDecimal threshold) {
        this.threshold = threshold;
    }
    
    @Override
    public FraudScore evaluate(Transaction transaction) {
        if (transaction.getAmount().compareTo(threshold) > 0) {
            return FraudScore.high(50, "Amount exceeds threshold");
        }
        return FraudScore.zero();
    }
}
```

### 3. Compliance e auditoria

**Resposta:**
```java
@Entity
@Table(name = "audit_trail")
public class AuditTrail {
    @Id
    private String id;
    private String userId;
    private String action;
    private String resource;
    private String details;
    private Instant timestamp;
    private String ipAddress;
    private String userAgent;
    
    // Getters e setters
}

@Component
@Aspect
public class AuditAspect {
    
    @Autowired
    private AuditService auditService;
    
    @Around("@annotation(Auditable)")
    public Object auditMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        // Capturar contexto
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();
        
        AuditRecord record = AuditRecord.builder()
            .userId(auth.getName())
            .action(methodName)
            .resource(extractResource(args))
            .ipAddress(request.getRemoteAddr())
            .userAgent(request.getHeader("User-Agent"))
            .timestamp(Instant.now())
            .build();
        
        try {
            Object result = joinPoint.proceed();
            record.setStatus("SUCCESS");
            record.setDetails(objectMapper.writeValueAsString(result));
            return result;
        } catch (Exception e) {
            record.setStatus("FAILED");
            record.setDetails(e.getMessage());
            throw e;
        } finally {
            auditService.saveAuditRecord(record);
        }
    }
}

@Service
public class ComplianceService {
    
    public void validateTransfer(TransferRequest request, String userId) {
        // KYC Check
        if (!kycService.isVerified(userId)) {
            throw new ComplianceException("KYC verification required");
        }
        
        // AML Check
        if (amlService.isBlacklisted(request.getToAccountId())) {
            throw new ComplianceException("Destination account is blacklisted");
        }
        
        // Daily limit check
        BigDecimal dailyTotal = transactionRepository.getDailyTotal(userId);
        if (dailyTotal.add(request.getAmount()).compareTo(DAILY_LIMIT) > 0) {
            throw new ComplianceException("Daily transaction limit exceeded");
        }
    }
}
```

### 4. Padrões de segurança bancária

**Resposta:**
```java
@Configuration
@EnableWebSecurity
public class BankingSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/banking/**").hasAnyRole("CUSTOMER", "TELLER")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .csrf(csrf -> csrf.disable())
            .headers(headers -> headers
                .frameOptions().deny()
                .contentTypeOptions().and()
                .httpStrictTransportSecurity(hstsConfig -> hstsConfig
                    .maxAgeInSeconds(31536000)
                    .includeSubdomains(true)
                )
            );
        
        return http.build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    @Bean
    public JwtDecoder jwtDecoder() {
        return NimbusJwtDecoder.withJwkSetUri("https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX/.well-known/jwks.json").build();
    }
}

@Component
public class EncryptionService {
    
    @Value("${encryption.key}")
    private String encryptionKey;
    
    public String encryptPII(String data) {
        try {
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            SecretKeySpec keySpec = new SecretKeySpec(encryptionKey.getBytes(), "AES");
            cipher.init(Cipher.ENCRYPT_MODE, keySpec);
            
            byte[] encryptedData = cipher.doFinal(data.getBytes());
            return Base64.getEncoder().encodeToString(encryptedData);
        } catch (Exception e) {
            throw new RuntimeException("Encryption failed", e);
        }
    }
    
    public String decryptPII(String encryptedData) {
        try {
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            SecretKeySpec keySpec = new SecretKeySpec(encryptionKey.getBytes(), "AES");
            cipher.init(Cipher.DECRYPT_MODE, keySpec);
            
            byte[] decryptedData = cipher.doFinal(Base64.getDecoder().decode(encryptedData));
            return new String(decryptedData);
        } catch (Exception e) {
            throw new RuntimeException("Decryption failed", e);
        }
    }
}
```

---

## Monitoramento e Observabilidade

### 1. Configuração de monitoramento

**Resposta:**
```java
// Spring Boot Actuator
@Configuration
public class MonitoringConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags("application", "banking-service")
            .commonTags("environment", "production");
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

@Service
public class TransactionService {
    
    private final Counter transactionCounter;
    private final Timer transactionTimer;
    private final Gauge pendingTransactions;
    
    public TransactionService(MeterRegistry meterRegistry) {
        this.transactionCounter = Counter.builder("transactions.total")
            .description("Total number of transactions")
            .tag("type", "transfer")
            .register(meterRegistry);
            
        this.transactionTimer = Timer.builder("transactions.duration")
            .description("Transaction processing time")
            .register(meterRegistry);
            
        this.pendingTransactions = Gauge.builder("transactions.pending")
            .description("Number of pending transactions")
            .register(meterRegistry, this, TransactionService::getPendingCount);
    }
    
    @Timed(value = "transactions.processing", description = "Time taken to process transaction")
    public TransactionResult processTransaction(TransferRequest request) {
        return transactionTimer.recordCallable(() -> {
            try {
                TransactionResult result = doProcessTransaction(request);
                transactionCounter.increment(Tags.of("status", "success"));
                return result;
            } catch (Exception e) {
                transactionCounter.increment(Tags.of("status", "error"));
                throw e;
            }
        });
    }
}
```

### 2. Logging estruturado

**Resposta:**
```java
@Component
@Slf4j
public class TransactionLogger {
    
    private final ObjectMapper objectMapper;
    
    public void logTransactionStart(String transactionId, TransferRequest request) {
        Map<String, Object> logData = Map.of(
            "event", "transaction_started",
            "transactionId", transactionId,
            "fromAccount", request.getFromAccountId(),
            "toAccount", request.getToAccountId(),
            "amount", request.getAmount(),
            "timestamp", Instant.now()
        );
        
        log.info("Transaction started: {}", toJsonString(logData));
    }
    
    public void logTransactionCompleted(String transactionId, TransactionResult result) {
        Map<String, Object> logData = Map.of(
            "event", "transaction_completed",
            "transactionId", transactionId,
            "status", result.getStatus(),
            "duration", result.getProcessingTime(),
            "timestamp", Instant.now()
        );
        
        log.info("Transaction completed: {}", toJsonString(logData));
    }
    
    public void logTransactionFailed(String transactionId, Exception error) {
        Map<String, Object> logData = Map.of(
            "event", "transaction_failed",
            "transactionId", transactionId,
            "error", error.getMessage(),
            "stackTrace", getStackTrace(error),
            "timestamp", Instant.now()
        );
        
        log.error("Transaction failed: {}", toJsonString(logData));
    }
    
    private String toJsonString(Map<String, Object> data) {
        try {
            return objectMapper.writeValueAsString(data);
        } catch (Exception e) {
            return data.toString();
        }
    }
}
```

### 3. Health Checks

**Resposta:**
```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection connection = dataSource.getConnection()) {
            if (connection.isValid(5)) {
                return Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connected")
                    .build();
            } else {
                return Health.down()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "Connection invalid")
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("database", "PostgreSQL")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}

@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {
    
    @Autowired
    private RestTemplate restTemplate;
    
    @Value("${external.service.url}")
    private String externalServiceUrl;
    
    @Override
    public Health health() {
        try {
            ResponseEntity<String> response = restTemplate.getForEntity(
                externalServiceUrl + "/health",
                String.class
            );
            
            if (response.getStatusCode().is2xxSuccessful()) {
                return Health.up()
                    .withDetail("service", "external-payment-service")
                    .withDetail("status", "Available")
                    .withDetail("responseTime", getResponseTime())
                    .build();
            } else {
                return Health.down()
                    .withDetail("service", "external-payment-service")
                    .withDetail("status", "Unavailable")
                    .withDetail("httpStatus", response.getStatusCode())
                    .build();
            }
        } catch (Exception e) {
            return Health.down()
                .withDetail("service", "external-payment-service")
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### 4. Distributed Tracing

**Resposta:**
```java
@Configuration
public class TracingConfig {
    
    @Bean
    public Sender sender() {
        return OkHttpSender.create("http://jaeger:14268/api/traces");
    }
    
    @Bean
    public AsyncReporter<Span> spanReporter() {
        return AsyncReporter.create(sender());
    }
    
    @Bean
    public Tracing tracing() {
        return Tracing.newBuilder()
            .localServiceName("banking-service")
            .spanReporter(spanReporter())
            .sampler(Sampler.create(1.0f))
            .build();
    }
}

@Service
public class TracedTransactionService {
    
    private final Tracing tracing;
    private final Tracer tracer;
    
    public TracedTransactionService(Tracing tracing) {
        this.tracing = tracing;
        this.tracer = tracing.tracer();
    }
    
    public TransactionResult processTransaction(TransferRequest request) {
        Span span = tracer.nextSpan()
            .name("process-transaction")
            .tag("transaction.type", "transfer")
            .tag("transaction.amount", request.getAmount().toString())
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            // Validate request
            Span validationSpan = tracer.nextSpan()
                .name("validate-request")
                .start();
            
            try (Tracer.SpanInScope validationScope = tracer.withSpanInScope(validationSpan)) {
                validateRequest(request);
            } finally {
                validationSpan.end();
            }
            
            // Process transaction
            Span processingSpan = tracer.nextSpan()
                .name("process-transfer")
                .start();
            
            try (Tracer.SpanInScope processingScope = tracer.withSpanInScope(processingSpan)) {
                return doProcessTransaction(request);
            } finally {
                processingSpan.end();
            }
            
        } catch (Exception e) {
            span.tag("error", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

---

## Casos de Uso Práticos

### 1. Sistema de PIX

**Resposta:**
```java
@Service
public class PixService {
    
    @Autowired
    private PixKeyRepository pixKeyRepository;
    
    @Autowired
    private BacenService bacenService;
    
    @Autowired
    private FraudDetectionService fraudDetectionService;
    
    @Transactional
    public PixTransferResult processPixTransfer(PixTransferRequest request) {
        // Validar chave PIX
        PixKey destinationKey = pixKeyRepository.findByKey(request.getPixKey())
            .orElseThrow(() -> new PixKeyNotFoundException("PIX key not found"));
        
        // Validar conta de origem
        Account sourceAccount = accountRepository.findById(request.getSourceAccountId())
            .orElseThrow(() -> new AccountNotFoundException("Source account not found"));
        
        // Verificar saldo
        if (sourceAccount.getBalance().compareTo(request.getAmount()) < 0) {
            throw new InsufficientFundsException("Insufficient funds");
        }
        
        // Análise de fraude
        FraudCheckResult fraudCheck = fraudDetectionService.analyzePixTransaction(request);
        if (fraudCheck.isBlocked()) {
            throw new FraudException("Transaction blocked by fraud detection");
        }
        
        // Processar transferência
        PixTransaction transaction = PixTransaction.builder()
            .id(generatePixTransactionId())
            .sourceAccountId(request.getSourceAccountId())
            .destinationKey(request.getPixKey())
            .amount(request.getAmount())
            .message(request.getMessage())
            .status(PixStatus.PROCESSING)
            .timestamp(Instant.now())
            .build();
        
        try {
            // Comunicar com BACEN
            BacenResponse bacenResponse = bacenService.processPixTransfer(transaction);
            
            if (bacenResponse.isSuccess()) {
                // Debitar conta origem
                sourceAccount.debit(request.getAmount());
                accountRepository.save(sourceAccount);
                
                transaction.setStatus(PixStatus.COMPLETED);
                transaction.setBacenTransactionId(bacenResponse.getTransactionId());
                
                // Notificar usuário
                notificationService.sendPixConfirmation(sourceAccount, transaction);
                
            } else {
                transaction.setStatus(PixStatus.FAILED);
                transaction.setErrorMessage(bacenResponse.getErrorMessage());
            }
            
        } catch (Exception e) {
            transaction.setStatus(PixStatus.FAILED);
            transaction.setErrorMessage(e.getMessage());
            throw new PixProcessingException("Failed to process PIX transfer", e);
        } finally {
            pixTransactionRepository.save(transaction);
        }
        
        return PixTransferResult.builder()
            .transactionId(transaction.getId())
            .status(transaction.getStatus())
            .build();
    }
}
```

### 2. Sistema de cartão de crédito

**Resposta:**
```java
@Service
public class CreditCardService {
    
    @Autowired
    private CreditCardRepository creditCardRepository;
    
    @Autowired
    private AuthorizationService authorizationService;
    
    @Autowired
    private RiskEngineService riskEngineService;
    
    public AuthorizationResult authorizeTransaction(AuthorizationRequest request) {
        // Buscar cartão
        CreditCard card = creditCardRepository.findByNumber(request.getCardNumber())
            .orElseThrow(() -> new CardNotFoundException("Card not found"));
        
        // Validações básicas
        if (card.isBlocked()) {
            return AuthorizationResult.denied("Card is blocked");
        }
        
        if (card.isExpired()) {
            return AuthorizationResult.denied("Card is expired");
        }
        
        if (!card.getCvv().equals(request.getCvv())) {
            card.incrementFailedAttempts();
            if (card.getFailedAttempts() >= 3) {
                card.setBlocked(true);
            }
            creditCardRepository.save(card);
            return AuthorizationResult.denied("Invalid CVV");
        }
        
        // Verificar limite
        BigDecimal availableLimit = card.getLimit().subtract(card.getUsedLimit());
        if (availableLimit.compareTo(request.getAmount()) < 0) {
            return AuthorizationResult.denied("Insufficient limit");
        }
        
        // Análise de risco
        RiskAnalysisResult riskResult = riskEngineService.analyzeTransaction(request, card);
        
        if (riskResult.isHighRisk()) {
            // Requerer autenticação adicional
            return AuthorizationResult.requiresAuthentication("3DS authentication required");
        }
        
        // Autorizar transação
        try {
            Transaction transaction = Transaction.builder()
                .id(generateTransactionId())
                .cardId(card.getId())
                .amount(request.getAmount())
                .merchantId(request.getMerchantId())
                .description(request.getDescription())
                .status(TransactionStatus.APPROVED)
                .timestamp(Instant.now())
                .build();
            
            // Atualizar limite usado
            card.setUsedLimit(card.getUsedLimit().add(request.getAmount()));
            card.resetFailedAttempts();
            
            creditCardRepository.save(card);
            transactionRepository.save(transaction);
            
            return AuthorizationResult.approved(transaction.getId());
            
        } catch (Exception e) {
            log.error("Error processing authorization: {}", e.getMessage());
            return AuthorizationResult.denied("Processing error");
        }
    }
}
```

### 3. Sistema de score de crédito

**Resposta:**
```java
@Service
public class CreditScoreService {
    
    @Autowired
    private CustomerRepository customerRepository;
    
    @Autowired
    private TransactionHistoryRepository transactionHistoryRepository;
    
    @Autowired
    private ExternalBureauService externalBureauService;
    
    public CreditScoreResult calculateCreditScore(String customerId) {
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new CustomerNotFoundException("Customer not found"));
        
        CreditScoreCalculator calculator = new CreditScoreCalculator();
        
        // Histórico de transações
        List<Transaction> transactions = transactionHistoryRepository
            .findByCustomerIdAndTimestampAfter(customerId, Instant.now().minus(Duration.ofDays(365)));
        
        calculator.addScore(analyzeTransactionHistory(transactions));
        
        // Dados externos (SPC, Serasa)
        ExternalBureauData bureauData = externalBureauService.getBureauData(customer.getCpf());
        calculator.addScore(analyzeExternalBureauData(bureauData));
        
        // Relacionamento com banco
        BankRelationshipData relationshipData = getBankRelationshipData(customerId);
        calculator.addScore(analyzeRelationshipData(relationshipData));
        
        // Renda e patrimônio
        calculator.addScore(analyzeIncomeAndAssets(customer));
        
        CreditScore finalScore = calculator.calculate();
        
        return CreditScoreResult.builder()
            .customerId(customerId)
            .score(finalScore.getValue())
            .rating(finalScore.getRating())
            .factors(finalScore.getFactors())
            .calculatedAt(Instant.now())
            .build();
    }
    
    private ScoreComponent analyzeTransactionHistory(List<Transaction> transactions) {
        if (transactions.isEmpty()) {
            return ScoreComponent.builder()
                .name("transaction_history")
                .score(0)
                .weight(0.3)
                .description("No transaction history")
                .build();
        }
        
        // Análise de padrões
        double averageBalance = transactions.stream()
            .mapToDouble(t -> t.getBalance().doubleValue())
            .average()
            .orElse(0.0);
        
        long overdrafts = transactions.stream()
            .filter(t -> t.getBalance().compareTo(BigDecimal.ZERO) < 0)
            .count();
        
        double regularityScore = calculateRegularity(transactions);
        
        int score = calculateTransactionScore(averageBalance, overdrafts, regularityScore);
        
        return ScoreComponent.builder()
            .name("transaction_history")
            .score(score)
            .weight(0.3)
            .description("Based on transaction patterns")
            .build();
    }
    
    private ScoreComponent analyzeExternalBureauData(ExternalBureauData bureauData) {
        int score = 0;
        
        // CPF status
        if (bureauData.getCpfStatus() == CpfStatus.CLEAN) {
            score += 200;
        } else if (bureauData.getCpfStatus() == CpfStatus.RESTRICTED) {
            score -= 100;
        }
        
        // Negative records
        score -= bureauData.getNegativeRecords().size() * 50;
        
        // Credit history length
        score += Math.min(bureauData.getCreditHistoryMonths() * 2, 100);
        
        return ScoreComponent.builder()
            .name("external_bureau")
            .score(Math.max(0, Math.min(300, score)))
            .weight(0.4)
            .description("External bureau data analysis")
            .build();
    }
}
```

---

## Perguntas Comportamentais e Conceituais

### 1. Como você abordaria a migração de um sistema monolítico para microserviços?

**Resposta:**
1. **Análise do sistema atual**: Mapear domínios, dependências e gargalos
2. **Estratégia Strangler Pattern**: Migração gradual substituindo partes do monolito
3. **Identificar bounded contexts**: Definir fronteiras dos microserviços
4. **Database decomposition**: Separar bancos de dados por domínio
5. **API Gateway**: Centralizar roteamento e cross-cutting concerns
6. **Service mesh**: Gerenciar comunicação entre serviços
7. **Monitoramento**: Observabilidade distribuída
8. **Rollback plan**: Estratégia de reversão para cada etapa

### 2. Quando NÃO usar microserviços?

**Resposta:**
- **Equipe pequena**: Overhead de manutenção maior que benefícios
- **Domínio simples**: Complexidade desnecessária
- **Performance crítica**: Latência de rede pode ser problemática
- **Transações ACID**: Distributed transactions são complexas
- **Startup/MVP**: Foco deve ser em time-to-market
- **Dados altamente acoplados**: Quando separação não faz sentido

### 3. Como garantir consistência de dados em sistemas distribuídos?

**Resposta:**
- **Eventual Consistency**: Aceitar inconsistência temporária
- **Saga Pattern**: Transações distribuídas com compensação
- **Event Sourcing**: Source of truth baseado em eventos
- **CQRS**: Separar modelos de leitura e escrita
- **Two-Phase Commit**: Para casos que exigem ACID
- **Distributed Locks**: Para sincronização crítica

### 4. Explique as diferenças entre tipos de Load Balancer

**Resposta:**
- **Layer 4 (Transport)**: Baseado em IP/Porto, mais rápido
- **Layer 7 (Application)**: Baseado em conteúdo HTTP, mais inteligente
- **Round Robin**: Distribuição sequencial
- **Least Connections**: Menor número de conexões ativas
- **IP Hash**: Baseado no hash do IP do cliente
- **Weighted**: Distribuição baseada em pesos configurados

## Conclusão

Este guia abrangente cobre os principais tópicos para entrevistas técnicas Java e AWS. Pratique implementando esses conceitos e esteja preparado para discussões aprofundadas sobre trade-offs, casos de uso e melhores práticas.

**Dicas finais:**
- Sempre considere trade-offs em suas respostas
- Relacione conceitos com experiências práticas
- Demonstre conhecimento de ferramentas e frameworks
- Seja específico sobre implementações e configurações
- Mostre preocupação com qualidade, segurança e performance
</rewritten_file> 