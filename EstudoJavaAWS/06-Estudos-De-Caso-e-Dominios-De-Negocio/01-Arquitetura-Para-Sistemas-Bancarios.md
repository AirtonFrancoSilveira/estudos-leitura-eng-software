# 🏦 Sistemas Bancários - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Projetar arquiteturas de core banking
- ✅ Implementar sistemas de segurança bancária
- ✅ Aplicar compliance e auditoria
- ✅ Desenvolver sistemas PIX, cartões e pagamentos
- ✅ Implementar detecção de fraudes

---

## 🎯 **Conceito 1: Arquitetura Core Banking**

### **🔍 O que é?**
Core Banking é como o coração de um banco: um sistema central que gerencia todas as operações bancárias, contas, transações e relacionamentos com clientes.

### **🎓 Conceitos Avançados:**

**1. Arquitetura Hexagonal para Banking**
```java
// Domain Layer - Regras de negócio puras
@Entity
public class Account {
    @Id
    private AccountId id;
    
    @Embedded
    private CustomerId customerId;
    
    @Embedded
    private Money balance;
    
    @Embedded
    private AccountType type;
    
    private AccountStatus status;
    
    private LocalDateTime createdAt;
    private LocalDateTime lastTransactionAt;
    
    // Regras de negócio
    public void debit(Money amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new InactiveAccountException("Conta inativa: " + id);
        }
        
        if (balance.isLessThan(amount)) {
            throw new InsufficientFundsException("Saldo insuficiente");
        }
        
        if (amount.isNegativeOrZero()) {
            throw new InvalidAmountException("Valor deve ser positivo");
        }
        
        this.balance = balance.subtract(amount);
        this.lastTransactionAt = LocalDateTime.now();
        
        // Publicar evento de domínio
        DomainEvents.publish(new MoneyDebitedEvent(id, amount));
    }
    
    public void credit(Money amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new InactiveAccountException("Conta inativa: " + id);
        }
        
        if (amount.isNegativeOrZero()) {
            throw new InvalidAmountException("Valor deve ser positivo");
        }
        
        this.balance = balance.add(amount);
        this.lastTransactionAt = LocalDateTime.now();
        
        // Publicar evento de domínio
        DomainEvents.publish(new MoneyCreditedEvent(id, amount));
    }
    
    public boolean canWithdraw(Money amount) {
        return status == AccountStatus.ACTIVE && 
               balance.isGreaterThanOrEqualTo(amount);
    }
}

// Value Objects
@Embeddable
public class Money {
    private BigDecimal amount;
    private Currency currency;
    
    public Money(BigDecimal amount, Currency currency) {
        if (amount == null) {
            throw new IllegalArgumentException("Amount cannot be null");
        }
        if (currency == null) {
            throw new IllegalArgumentException("Currency cannot be null");
        }
        
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }
    
    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    public Money subtract(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount.subtract(other.amount), this.currency);
    }
    
    public boolean isLessThan(Money other) {
        validateSameCurrency(other);
        return this.amount.compareTo(other.amount) < 0;
    }
    
    public boolean isGreaterThanOrEqualTo(Money other) {
        validateSameCurrency(other);
        return this.amount.compareTo(other.amount) >= 0;
    }
    
    public boolean isNegativeOrZero() {
        return amount.compareTo(BigDecimal.ZERO) <= 0;
    }
    
    private void validateSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException("Currencies must match");
        }
    }
}

// Aggregate Root
@Entity
public class Customer {
    @Id
    private CustomerId id;
    
    @Embedded
    private PersonalInfo personalInfo;
    
    @Embedded
    private ContactInfo contactInfo;
    
    @OneToMany(mappedBy = "customerId")
    private List<Account> accounts = new ArrayList<>();
    
    private CustomerStatus status;
    private CustomerType type;
    
    public Account openAccount(AccountType accountType, Money initialDeposit) {
        if (status != CustomerStatus.ACTIVE) {
            throw new InactiveCustomerException("Cliente inativo: " + id);
        }
        
        if (hasAccountOfType(accountType)) {
            throw new DuplicateAccountException("Cliente já possui conta do tipo: " + accountType);
        }
        
        Account account = new Account(
            AccountId.generate(),
            this.id,
            accountType,
            initialDeposit
        );
        
        accounts.add(account);
        
        // Publicar evento de domínio
        DomainEvents.publish(new AccountOpenedEvent(account.getId(), this.id));
        
        return account;
    }
    
    private boolean hasAccountOfType(AccountType type) {
        return accounts.stream()
            .anyMatch(account -> account.getType().equals(type));
    }
}
```

**2. Application Services (Use Cases)**
```java
// Port (Interface)
public interface TransferMoneyUseCase {
    TransferResult execute(TransferMoneyCommand command);
}

// Adapter (Implementation)
@Service
@Transactional
public class TransferMoneyService implements TransferMoneyUseCase {
    
    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;
    private final FraudDetectionService fraudDetectionService;
    private final NotificationService notificationService;
    private final AuditService auditService;
    
    public TransferMoneyService(AccountRepository accountRepository,
                               TransactionRepository transactionRepository,
                               FraudDetectionService fraudDetectionService,
                               NotificationService notificationService,
                               AuditService auditService) {
        this.accountRepository = accountRepository;
        this.transactionRepository = transactionRepository;
        this.fraudDetectionService = fraudDetectionService;
        this.notificationService = notificationService;
        this.auditService = auditService;
    }
    
    @Override
    public TransferResult execute(TransferMoneyCommand command) {
        // Criar ID único para a transação
        TransactionId transactionId = TransactionId.generate();
        
        try {
            // Auditoria - início da operação
            auditService.logTransferAttempt(transactionId, command);
            
            // Buscar contas
            Account sourceAccount = accountRepository.findById(command.getSourceAccountId())
                .orElseThrow(() -> new AccountNotFoundException("Conta origem não encontrada"));
            
            Account targetAccount = accountRepository.findById(command.getTargetAccountId())
                .orElseThrow(() -> new AccountNotFoundException("Conta destino não encontrada"));
            
            // Validações de negócio
            validateTransfer(sourceAccount, targetAccount, command.getAmount());
            
            // Detecção de fraude
            FraudAnalysisResult fraudResult = fraudDetectionService.analyzeTransfer(
                sourceAccount, targetAccount, command.getAmount());
            
            if (fraudResult.isHighRisk()) {
                auditService.logFraudDetection(transactionId, fraudResult);
                return TransferResult.failure(
                    transactionId, 
                    "Transação bloqueada por suspeita de fraude"
                );
            }
            
            // Executar transferência
            Money amount = new Money(command.getAmount(), command.getCurrency());
            
            sourceAccount.debit(amount);
            targetAccount.credit(amount);
            
            // Persistir mudanças
            accountRepository.save(sourceAccount);
            accountRepository.save(targetAccount);
            
            // Registrar transação
            Transaction transaction = new Transaction(
                transactionId,
                sourceAccount.getId(),
                targetAccount.getId(),
                amount,
                TransactionType.TRANSFER
            );
            
            transactionRepository.save(transaction);
            
            // Auditoria - sucesso
            auditService.logTransferSuccess(transactionId, transaction);
            
            // Notificações assíncronas
            notificationService.notifyTransferCompleted(sourceAccount, targetAccount, amount);
            
            return TransferResult.success(transactionId, "Transferência realizada com sucesso");
            
        } catch (Exception e) {
            // Auditoria - erro
            auditService.logTransferError(transactionId, command, e);
            
            // Log para monitoramento
            log.error("Erro na transferência: transactionId={}, error={}", 
                transactionId, e.getMessage(), e);
            
            return TransferResult.failure(transactionId, "Erro interno do sistema");
        }
    }
    
    private void validateTransfer(Account source, Account target, BigDecimal amount) {
        if (source.getId().equals(target.getId())) {
            throw new SameAccountTransferException("Não é possível transferir para a mesma conta");
        }
        
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidAmountException("Valor deve ser positivo");
        }
        
        if (amount.compareTo(new BigDecimal("100000")) > 0) {
            throw new TransferLimitExceededException("Valor excede limite de transferência");
        }
    }
}
```

**3. Infrastructure Layer**
```java
// Repository Implementation
@Repository
public class JpaAccountRepository implements AccountRepository {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public Optional<Account> findById(AccountId id) {
        return Optional.ofNullable(entityManager.find(Account.class, id));
    }
    
    @Override
    public Account save(Account account) {
        if (account.getId() == null) {
            entityManager.persist(account);
        } else {
            entityManager.merge(account);
        }
        return account;
    }
    
    @Override
    public List<Account> findByCustomerId(CustomerId customerId) {
        return entityManager.createQuery(
            "SELECT a FROM Account a WHERE a.customerId = :customerId", 
            Account.class)
            .setParameter("customerId", customerId)
            .getResultList();
    }
    
    @Override
    public List<Account> findAccountsWithBalanceGreaterThan(Money amount) {
        return entityManager.createQuery(
            "SELECT a FROM Account a WHERE a.balance.amount > :amount AND a.balance.currency = :currency",
            Account.class)
            .setParameter("amount", amount.getAmount())
            .setParameter("currency", amount.getCurrency())
            .getResultList();
    }
}

// Event Publisher
@Component
public class DomainEventPublisher {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public DomainEventPublisher(ApplicationEventPublisher eventPublisher) {
        this.eventPublisher = eventPublisher;
    }
    
    @EventListener
    public void handleMoneyDebitedEvent(MoneyDebitedEvent event) {
        // Publicar para sistemas externos
        eventPublisher.publishEvent(new AccountDebitedExternalEvent(
            event.getAccountId(),
            event.getAmount(),
            event.getOccurredAt()
        ));
    }
    
    @EventListener
    public void handleMoneyCreditedEvent(MoneyCreditedEvent event) {
        // Publicar para sistemas externos
        eventPublisher.publishEvent(new AccountCreditedExternalEvent(
            event.getAccountId(),
            event.getAmount(),
            event.getOccurredAt()
        ));
    }
}
```

### **💡 Caso de Uso Prático: Sistema de Conta Corrente**

```java
// Use Case completo para conta corrente
@Service
@Transactional
public class CheckingAccountService {
    
    private final AccountRepository accountRepository;
    private final TransactionRepository transactionRepository;
    private final OverdraftPolicyService overdraftPolicyService;
    private final InterestCalculationService interestCalculationService;
    
    public CheckingAccountResult processCheckingTransaction(CheckingTransactionCommand command) {
        Account account = accountRepository.findById(command.getAccountId())
            .orElseThrow(() -> new AccountNotFoundException("Conta não encontrada"));
        
        // Verificar se é conta corrente
        if (!account.getType().equals(AccountType.CHECKING)) {
            throw new InvalidAccountTypeException("Operação válida apenas para conta corrente");
        }
        
        Money transactionAmount = new Money(command.getAmount(), command.getCurrency());
        
        switch (command.getTransactionType()) {
            case DEBIT:
                return processDebit(account, transactionAmount);
            case CREDIT:
                return processCredit(account, transactionAmount);
            case OVERDRAFT_PAYMENT:
                return processOverdraftPayment(account, transactionAmount);
            default:
                throw new UnsupportedTransactionTypeException("Tipo de transação não suportado");
        }
    }
    
    private CheckingAccountResult processDebit(Account account, Money amount) {
        // Verificar se tem saldo suficiente
        if (account.getBalance().isGreaterThanOrEqualTo(amount)) {
            account.debit(amount);
            recordTransaction(account, amount, TransactionType.DEBIT);
            return CheckingAccountResult.success("Débito realizado com sucesso");
        }
        
        // Verificar política de cheque especial
        Money overdraftLimit = overdraftPolicyService.getOverdraftLimit(account);
        Money totalAvailable = account.getBalance().add(overdraftLimit);
        
        if (totalAvailable.isGreaterThanOrEqualTo(amount)) {
            // Usar cheque especial
            account.debit(amount);
            
            // Calcular juros se entrou no cheque especial
            if (account.getBalance().isNegative()) {
                Money overdraftUsed = account.getBalance().abs();
                Money interest = interestCalculationService.calculateOverdraftInterest(
                    overdraftUsed, 
                    overdraftPolicyService.getOverdraftRate(account)
                );
                
                // Registrar juros
                recordTransaction(account, interest, TransactionType.OVERDRAFT_INTEREST);
            }
            
            recordTransaction(account, amount, TransactionType.DEBIT);
            return CheckingAccountResult.success("Débito realizado com cheque especial");
        }
        
        throw new InsufficientFundsException("Saldo insuficiente incluindo cheque especial");
    }
    
    private CheckingAccountResult processCredit(Account account, Money amount) {
        boolean wasInOverdraft = account.getBalance().isNegative();
        
        account.credit(amount);
        recordTransaction(account, amount, TransactionType.CREDIT);
        
        // Se estava negativo e agora está positivo, notificar saída do cheque especial
        if (wasInOverdraft && account.getBalance().isPositive()) {
            DomainEvents.publish(new OverdraftExitedEvent(account.getId()));
        }
        
        return CheckingAccountResult.success("Crédito realizado com sucesso");
    }
    
    private CheckingAccountResult processOverdraftPayment(Account account, Money amount) {
        if (account.getBalance().isPositive()) {
            throw new InvalidOperationException("Conta não está no cheque especial");
        }
        
        Money overdraftBalance = account.getBalance().abs();
        
        if (amount.isGreaterThan(overdraftBalance)) {
            // Pagar todo o cheque especial e creditar o restante
            account.credit(overdraftBalance);
            Money remaining = amount.subtract(overdraftBalance);
            account.credit(remaining);
            
            recordTransaction(account, overdraftBalance, TransactionType.OVERDRAFT_PAYMENT);
            recordTransaction(account, remaining, TransactionType.CREDIT);
            
            return CheckingAccountResult.success("Cheque especial quitado e saldo creditado");
        } else {
            account.credit(amount);
            recordTransaction(account, amount, TransactionType.OVERDRAFT_PAYMENT);
            
            return CheckingAccountResult.success("Pagamento de cheque especial realizado");
        }
    }
    
    private void recordTransaction(Account account, Money amount, TransactionType type) {
        Transaction transaction = new Transaction(
            TransactionId.generate(),
            account.getId(),
            amount,
            type,
            LocalDateTime.now()
        );
        
        transactionRepository.save(transaction);
    }
}
```

---

## 🎯 **Conceito 2: Sistema PIX**

### **🔍 O que é?**
PIX é o sistema de pagamentos instantâneos brasileiro, funcionando 24h por dia, 7 dias por semana, com liquidação em tempo real.

### **🎓 Implementação Completa:**

**1. Domínio PIX**
```java
// Entidade PIX Key
@Entity
@Table(name = "pix_keys")
public class PixKey {
    @Id
    private String id;
    
    @Column(name = "key_value", unique = true)
    private String keyValue;
    
    @Enumerated(EnumType.STRING)
    private PixKeyType type;
    
    @Column(name = "account_id")
    private String accountId;
    
    @Column(name = "customer_id")
    private String customerId;
    
    @Enumerated(EnumType.STRING)
    private PixKeyStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "expires_at")
    private LocalDateTime expiresAt;
    
    public void activate() {
        if (status == PixKeyStatus.PENDING) {
            this.status = PixKeyStatus.ACTIVE;
        } else {
            throw new InvalidPixKeyStatusException("Chave PIX não pode ser ativada");
        }
    }
    
    public void deactivate() {
        if (status == PixKeyStatus.ACTIVE) {
            this.status = PixKeyStatus.INACTIVE;
        } else {
            throw new InvalidPixKeyStatusException("Chave PIX não pode ser desativada");
        }
    }
    
    public boolean isExpired() {
        return expiresAt != null && expiresAt.isBefore(LocalDateTime.now());
    }
}

// Entidade PIX Transaction
@Entity
@Table(name = "pix_transactions")
public class PixTransaction {
    @Id
    private String id;
    
    @Column(name = "end_to_end_id", unique = true)
    private String endToEndId;
    
    @Column(name = "pix_key")
    private String pixKey;
    
    @Column(name = "source_account_id")
    private String sourceAccountId;
    
    @Column(name = "target_account_id")
    private String targetAccountId;
    
    @Column(name = "amount")
    private BigDecimal amount;
    
    @Column(name = "description")
    private String description;
    
    @Enumerated(EnumType.STRING)
    private PixTransactionStatus status;
    
    @Column(name = "created_at")
    private LocalDateTime createdAt;
    
    @Column(name = "processed_at")
    private LocalDateTime processedAt;
    
    @Column(name = "bacen_response")
    private String bacenResponse;
    
    public void markAsProcessed() {
        this.status = PixTransactionStatus.COMPLETED;
        this.processedAt = LocalDateTime.now();
    }
    
    public void markAsFailed(String reason) {
        this.status = PixTransactionStatus.FAILED;
        this.bacenResponse = reason;
    }
}

// Service para PIX
@Service
@Transactional
public class PixService {
    
    private final PixKeyRepository pixKeyRepository;
    private final PixTransactionRepository pixTransactionRepository;
    private final AccountRepository accountRepository;
    private final BacenIntegrationService bacenIntegrationService;
    private final FraudDetectionService fraudDetectionService;
    private final NotificationService notificationService;
    
    public PixTransferResult processPixTransfer(PixTransferRequest request) {
        // Validar chave PIX
        PixKey pixKey = pixKeyRepository.findByKeyValue(request.getPixKey())
            .orElseThrow(() -> new PixKeyNotFoundException("Chave PIX não encontrada"));
        
        if (pixKey.getStatus() != PixKeyStatus.ACTIVE) {
            throw new InactivePixKeyException("Chave PIX inativa");
        }
        
        if (pixKey.isExpired()) {
            throw new ExpiredPixKeyException("Chave PIX expirada");
        }
        
        // Buscar contas
        Account sourceAccount = accountRepository.findById(request.getSourceAccountId())
            .orElseThrow(() -> new AccountNotFoundException("Conta origem não encontrada"));
        
        Account targetAccount = accountRepository.findById(pixKey.getAccountId())
            .orElseThrow(() -> new AccountNotFoundException("Conta destino não encontrada"));
        
        // Validações de negócio
        validatePixTransfer(sourceAccount, targetAccount, request.getAmount());
        
        // Análise de fraude
        FraudAnalysisResult fraudResult = fraudDetectionService.analyzePixTransaction(
            sourceAccount, targetAccount, request.getAmount());
        
        if (fraudResult.isHighRisk()) {
            throw new FraudDetectedException("Transação bloqueada por suspeita de fraude");
        }
        
        // Criar transação PIX
        PixTransaction pixTransaction = new PixTransaction(
            generatePixTransactionId(),
            generateEndToEndId(),
            request.getPixKey(),
            sourceAccount.getId(),
            targetAccount.getId(),
            request.getAmount(),
            request.getDescription()
        );
        
        try {
            // Integração com BACEN
            BacenPixResponse bacenResponse = bacenIntegrationService.processPixTransfer(
                pixTransaction);
            
            if (bacenResponse.isSuccess()) {
                // Processar transferência
                Money amount = new Money(request.getAmount(), Currency.BRL);
                
                sourceAccount.debit(amount);
                targetAccount.credit(amount);
                
                // Salvar mudanças
                accountRepository.save(sourceAccount);
                accountRepository.save(targetAccount);
                
                // Atualizar transação
                pixTransaction.markAsProcessed();
                pixTransaction.setBacenResponse(bacenResponse.getTransactionId());
                
                // Notificações
                notificationService.notifyPixTransferCompleted(sourceAccount, targetAccount, amount);
                
                return PixTransferResult.success(pixTransaction.getId(), 
                    "Transferência PIX realizada com sucesso");
                
            } else {
                pixTransaction.markAsFailed(bacenResponse.getErrorMessage());
                return PixTransferResult.failure(pixTransaction.getId(), 
                    bacenResponse.getErrorMessage());
            }
            
        } catch (Exception e) {
            pixTransaction.markAsFailed("Erro na comunicação com BACEN: " + e.getMessage());
            throw new PixProcessingException("Erro ao processar PIX", e);
            
        } finally {
            pixTransactionRepository.save(pixTransaction);
        }
    }
    
    public PixKeyRegistrationResult registerPixKey(PixKeyRegistrationRequest request) {
        // Validar tipo de chave
        if (!isValidPixKeyFormat(request.getKeyValue(), request.getKeyType())) {
            throw new InvalidPixKeyFormatException("Formato de chave PIX inválido");
        }
        
        // Verificar se já existe
        if (pixKeyRepository.existsByKeyValue(request.getKeyValue())) {
            throw new DuplicatePixKeyException("Chave PIX já cadastrada");
        }
        
        // Verificar limite de chaves por cliente
        long existingKeys = pixKeyRepository.countByCustomerId(request.getCustomerId());
        if (existingKeys >= getMaxKeysPerCustomer(request.getCustomerType())) {
            throw new PixKeyLimitExceededException("Limite de chaves PIX excedido");
        }
        
        // Criar chave PIX
        PixKey pixKey = new PixKey(
            generatePixKeyId(),
            request.getKeyValue(),
            request.getKeyType(),
            request.getAccountId(),
            request.getCustomerId()
        );
        
        // Registrar no BACEN
        BacenPixKeyResponse bacenResponse = bacenIntegrationService.registerPixKey(pixKey);
        
        if (bacenResponse.isSuccess()) {
            pixKey.activate();
            pixKeyRepository.save(pixKey);
            
            return PixKeyRegistrationResult.success(pixKey.getId(), 
                "Chave PIX registrada com sucesso");
        } else {
            throw new PixKeyRegistrationException("Erro ao registrar chave PIX no BACEN: " + 
                bacenResponse.getErrorMessage());
        }
    }
    
    private boolean isValidPixKeyFormat(String keyValue, PixKeyType keyType) {
        switch (keyType) {
            case CPF:
                return keyValue.matches("\\d{11}");
            case CNPJ:
                return keyValue.matches("\\d{14}");
            case EMAIL:
                return keyValue.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");
            case PHONE:
                return keyValue.matches("\\+55\\d{10,11}");
            case RANDOM:
                return keyValue.matches("[a-f0-9]{8}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{4}-[a-f0-9]{12}");
            default:
                return false;
        }
    }
    
    private int getMaxKeysPerCustomer(CustomerType customerType) {
        return customerType == CustomerType.INDIVIDUAL ? 5 : 20;
    }
    
    private void validatePixTransfer(Account source, Account target, BigDecimal amount) {
        if (source.getId().equals(target.getId())) {
            throw new SameAccountTransferException("Não é possível transferir para a mesma conta");
        }
        
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidAmountException("Valor deve ser positivo");
        }
        
        // Limite PIX durante o dia
        if (amount.compareTo(new BigDecimal("20000")) > 0) {
            throw new PixLimitExceededException("Valor excede limite diário PIX");
        }
        
        // Limite PIX durante a noite (20h às 6h)
        LocalTime now = LocalTime.now();
        if ((now.isAfter(LocalTime.of(20, 0)) || now.isBefore(LocalTime.of(6, 0))) &&
            amount.compareTo(new BigDecimal("1000")) > 0) {
            throw new PixNightLimitExceededException("Valor excede limite noturno PIX");
        }
    }
}
```

**2. Integração com BACEN**
```java
// Service para integração com BACEN
@Service
public class BacenIntegrationService {
    
    private final RestTemplate restTemplate;
    private final BacenAuthenticationService authService;
    private final BacenConfigurationProperties config;
    
    public BacenPixResponse processPixTransfer(PixTransaction transaction) {
        try {
            // Autenticação
            String accessToken = authService.getAccessToken();
            
            // Preparar request
            BacenPixTransferRequest request = BacenPixTransferRequest.builder()
                .endToEndId(transaction.getEndToEndId())
                .amount(transaction.getAmount())
                .sourceAccount(transaction.getSourceAccountId())
                .targetAccount(transaction.getTargetAccountId())
                .description(transaction.getDescription())
                .timestamp(transaction.getCreatedAt())
                .build();
            
            // Headers
            HttpHeaders headers = new HttpHeaders();
            headers.setBearerAuth(accessToken);
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.set("X-Request-ID", UUID.randomUUID().toString());
            
            HttpEntity<BacenPixTransferRequest> entity = new HttpEntity<>(request, headers);
            
            // Enviar para BACEN
            ResponseEntity<BacenPixTransferResponse> response = restTemplate.exchange(
                config.getPixTransferUrl(),
                HttpMethod.POST,
                entity,
                BacenPixTransferResponse.class
            );
            
            if (response.getStatusCode().is2xxSuccessful()) {
                BacenPixTransferResponse responseBody = response.getBody();
                return BacenPixResponse.success(responseBody.getTransactionId());
            } else {
                return BacenPixResponse.failure("Erro na comunicação com BACEN");
            }
            
        } catch (HttpClientErrorException e) {
            log.error("Erro do cliente na comunicação com BACEN: {}", e.getResponseBodyAsString());
            return BacenPixResponse.failure("Erro de validação: " + e.getMessage());
            
        } catch (HttpServerErrorException e) {
            log.error("Erro do servidor BACEN: {}", e.getResponseBodyAsString());
            return BacenPixResponse.failure("Erro interno do BACEN");
            
        } catch (Exception e) {
            log.error("Erro inesperado na comunicação com BACEN", e);
            return BacenPixResponse.failure("Erro de comunicação");
        }
    }
    
    public BacenPixKeyResponse registerPixKey(PixKey pixKey) {
        try {
            String accessToken = authService.getAccessToken();
            
            BacenPixKeyRegistrationRequest request = BacenPixKeyRegistrationRequest.builder()
                .keyValue(pixKey.getKeyValue())
                .keyType(pixKey.getType().toString())
                .accountId(pixKey.getAccountId())
                .customerId(pixKey.getCustomerId())
                .build();
            
            HttpHeaders headers = new HttpHeaders();
            headers.setBearerAuth(accessToken);
            headers.setContentType(MediaType.APPLICATION_JSON);
            
            HttpEntity<BacenPixKeyRegistrationRequest> entity = new HttpEntity<>(request, headers);
            
            ResponseEntity<BacenPixKeyRegistrationResponse> response = restTemplate.exchange(
                config.getPixKeyRegistrationUrl(),
                HttpMethod.POST,
                entity,
                BacenPixKeyRegistrationResponse.class
            );
            
            if (response.getStatusCode().is2xxSuccessful()) {
                return BacenPixKeyResponse.success("Chave registrada com sucesso");
            } else {
                return BacenPixKeyResponse.failure("Erro ao registrar chave");
            }
            
        } catch (Exception e) {
            log.error("Erro ao registrar chave PIX no BACEN", e);
            return BacenPixKeyResponse.failure("Erro de comunicação");
        }
    }
}
```

### **💡 Caso de Uso Prático: QR Code PIX**

```java
// Service para QR Code PIX
@Service
public class PixQRCodeService {
    
    private final PixKeyRepository pixKeyRepository;
    private final QRCodeGenerator qrCodeGenerator;
    private final PixTransactionRepository pixTransactionRepository;
    
    public PixQRCodeResult generateQRCode(PixQRCodeRequest request) {
        // Validar chave PIX
        PixKey pixKey = pixKeyRepository.findByKeyValue(request.getPixKey())
            .orElseThrow(() -> new PixKeyNotFoundException("Chave PIX não encontrada"));
        
        if (pixKey.getStatus() != PixKeyStatus.ACTIVE) {
            throw new InactivePixKeyException("Chave PIX inativa");
        }
        
        // Gerar dados do QR Code
        PixQRCodeData qrData = PixQRCodeData.builder()
            .pixKey(request.getPixKey())
            .recipientName(request.getRecipientName())
            .recipientCity(request.getRecipientCity())
            .amount(request.getAmount())
            .description(request.getDescription())
            .txId(generateTxId())
            .build();
        
        // Gerar string do QR Code (formato PIX)
        String qrCodeString = generatePixQRCodeString(qrData);
        
        // Gerar imagem do QR Code
        byte[] qrCodeImage = qrCodeGenerator.generateQRCode(qrCodeString, 300, 300);
        
        // Salvar QR Code para controle
        PixQRCode qrCode = new PixQRCode(
            qrData.getTxId(),
            qrData.getPixKey(),
            qrCodeString,
            qrData.getAmount(),
            LocalDateTime.now(),
            request.getExpirationMinutes() != null ? 
                LocalDateTime.now().plusMinutes(request.getExpirationMinutes()) : null
        );
        
        pixQRCodeRepository.save(qrCode);
        
        return PixQRCodeResult.success(qrData.getTxId(), qrCodeString, qrCodeImage);
    }
    
    private String generatePixQRCodeString(PixQRCodeData data) {
        StringBuilder qrCode = new StringBuilder();
        
        // Payload Format Indicator
        qrCode.append("00020126");
        
        // Merchant Account Information
        String merchantInfo = buildMerchantAccountInfo(data.getPixKey());
        qrCode.append(String.format("26%02d%s", merchantInfo.length(), merchantInfo));
        
        // Merchant Category Code
        qrCode.append("52040000");
        
        // Transaction Currency (BRL)
        qrCode.append("5303986");
        
        // Transaction Amount
        if (data.getAmount() != null) {
            String amount = data.getAmount().setScale(2, RoundingMode.HALF_UP).toString();
            qrCode.append(String.format("54%02d%s", amount.length(), amount));
        }
        
        // Country Code
        qrCode.append("5802BR");
        
        // Merchant Name
        qrCode.append(String.format("59%02d%s", 
            data.getRecipientName().length(), data.getRecipientName()));
        
        // Merchant City
        qrCode.append(String.format("60%02d%s", 
            data.getRecipientCity().length(), data.getRecipientCity()));
        
        // Additional Data Field
        if (data.getDescription() != null || data.getTxId() != null) {
            String additionalData = buildAdditionalDataField(data);
            qrCode.append(String.format("62%02d%s", additionalData.length(), additionalData));
        }
        
        // CRC16
        qrCode.append("6304");
        String crc = calculateCRC16(qrCode.toString());
        qrCode.append(crc);
        
        return qrCode.toString();
    }
    
    private String buildMerchantAccountInfo(String pixKey) {
        StringBuilder info = new StringBuilder();
        info.append("0014BR.GOV.BCB.PIX");
        info.append(String.format("01%02d%s", pixKey.length(), pixKey));
        return info.toString();
    }
    
    private String buildAdditionalDataField(PixQRCodeData data) {
        StringBuilder additional = new StringBuilder();
        
        if (data.getTxId() != null) {
            additional.append(String.format("05%02d%s", data.getTxId().length(), data.getTxId()));
        }
        
        if (data.getDescription() != null) {
            additional.append(String.format("08%02d%s", 
                data.getDescription().length(), data.getDescription()));
        }
        
        return additional.toString();
    }
    
    private String calculateCRC16(String data) {
        // Implementação do algoritmo CRC16 para PIX
        int crc = 0xFFFF;
        byte[] bytes = data.getBytes(StandardCharsets.UTF_8);
        
        for (byte b : bytes) {
            crc ^= (b & 0xFF) << 8;
            for (int i = 0; i < 8; i++) {
                if ((crc & 0x8000) != 0) {
                    crc = (crc << 1) ^ 0x1021;
                } else {
                    crc = crc << 1;
                }
                crc &= 0xFFFF;
            }
        }
        
        return String.format("%04X", crc);
    }
}
```

---

## 🎯 **Conceito 3: Detecção de Fraudes**

### **🔍 O que é?**
Sistema de detecção de fraudes é como um sistema de segurança inteligente que analisa padrões de comportamento e identifica atividades suspeitas em tempo real.

### **🎓 Implementação Avançada:**

**1. Engine de Detecção de Fraudes**
```java
// Interface para regras de fraude
public interface FraudRule {
    FraudRuleResult evaluate(TransactionContext context);
    String getRuleName();
    int getPriority();
}

// Implementação de regras específicas
@Component
public class AmountThresholdRule implements FraudRule {
    
    private final FraudConfigurationService configService;
    
    @Override
    public FraudRuleResult evaluate(TransactionContext context) {
        BigDecimal threshold = configService.getAmountThreshold(context.getCustomerType());
        
        if (context.getAmount().compareTo(threshold) > 0) {
            return FraudRuleResult.suspicious(
                getRuleName(),
                "Valor acima do limite: " + context.getAmount(),
                FraudRiskLevel.HIGH
            );
        }
        
        return FraudRuleResult.clean();
    }
    
    @Override
    public String getRuleName() {
        return "AMOUNT_THRESHOLD";
    }
    
    @Override
    public int getPriority() {
        return 1;
    }
}

@Component
public class VelocityRule implements FraudRule {
    
    private final TransactionRepository transactionRepository;
    
    @Override
    public FraudRuleResult evaluate(TransactionContext context) {
        LocalDateTime oneHourAgo = LocalDateTime.now().minusHours(1);
        
        List<Transaction> recentTransactions = transactionRepository
            .findByCustomerIdAndCreatedAtAfter(context.getCustomerId(), oneHourAgo);
        
        if (recentTransactions.size() > 10) {
            return FraudRuleResult.suspicious(
                getRuleName(),
                "Muitas transações em pouco tempo: " + recentTransactions.size(),
                FraudRiskLevel.MEDIUM
            );
        }
        
        return FraudRuleResult.clean();
    }
    
    @Override
    public String getRuleName() {
        return "VELOCITY_CHECK";
    }
    
    @Override
    public int getPriority() {
        return 2;
    }
}

@Component
public class GeolocationRule implements FraudRule {
    
    private final CustomerLocationService locationService;
    
    @Override
    public FraudRuleResult evaluate(TransactionContext context) {
        if (context.getLocation() == null) {
            return FraudRuleResult.clean();
        }
        
        List<Location> recentLocations = locationService.getRecentLocations(
            context.getCustomerId(), Duration.ofHours(24));
        
        if (recentLocations.isEmpty()) {
            return FraudRuleResult.clean();
        }
        
        // Verificar se a localização atual está muito distante das recentes
        Location lastKnownLocation = recentLocations.get(0);
        double distance = calculateDistance(context.getLocation(), lastKnownLocation);
        
        if (distance > 500) { // 500 km
            return FraudRuleResult.suspicious(
                getRuleName(),
                "Localização suspeita: " + distance + " km da última transação",
                FraudRiskLevel.HIGH
            );
        }
        
        return FraudRuleResult.clean();
    }
    
    @Override
    public String getRuleName() {
        return "GEOLOCATION_CHECK";
    }
    
    @Override
    public int getPriority() {
        return 3;
    }
    
    private double calculateDistance(Location loc1, Location loc2) {
        // Implementar cálculo de distância usando fórmula de Haversine
        final int R = 6371; // Raio da Terra em km
        
        double latDistance = Math.toRadians(loc2.getLatitude() - loc1.getLatitude());
        double lonDistance = Math.toRadians(loc2.getLongitude() - loc1.getLongitude());
        
        double a = Math.sin(latDistance / 2) * Math.sin(latDistance / 2)
                + Math.cos(Math.toRadians(loc1.getLatitude())) * Math.cos(Math.toRadians(loc2.getLatitude()))
                * Math.sin(lonDistance / 2) * Math.sin(lonDistance / 2);
        
        double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
        
        return R * c;
    }
}

// Engine principal de detecção
@Service
public class FraudDetectionEngine {
    
    private final List<FraudRule> fraudRules;
    private final FraudAnalysisRepository fraudAnalysisRepository;
    private final NotificationService notificationService;
    
    public FraudDetectionEngine(List<FraudRule> fraudRules,
                               FraudAnalysisRepository fraudAnalysisRepository,
                               NotificationService notificationService) {
        this.fraudRules = fraudRules.stream()
            .sorted(Comparator.comparingInt(FraudRule::getPriority))
            .collect(Collectors.toList());
        this.fraudAnalysisRepository = fraudAnalysisRepository;
        this.notificationService = notificationService;
    }
    
    public FraudAnalysisResult analyzeTransaction(TransactionContext context) {
        FraudAnalysisResult result = new FraudAnalysisResult(context.getTransactionId());
        
        List<FraudRuleResult> ruleResults = new ArrayList<>();
        
        // Executar todas as regras
        for (FraudRule rule : fraudRules) {
            try {
                FraudRuleResult ruleResult = rule.evaluate(context);
                ruleResults.add(ruleResult);
                
                if (ruleResult.isSuspicious()) {
                    result.addSuspiciousRule(rule.getRuleName(), ruleResult);
                    
                    // Se for risco alto, parar a análise
                    if (ruleResult.getRiskLevel() == FraudRiskLevel.HIGH) {
                        result.setFinalDecision(FraudDecision.BLOCK);
                        break;
                    }
                }
                
            } catch (Exception e) {
                log.error("Erro ao executar regra de fraude {}: {}", rule.getRuleName(), e.getMessage());
                // Continuar com outras regras
            }
        }
        
        // Determinar decisão final
        if (result.getFinalDecision() == null) {
            result.setFinalDecision(calculateFinalDecision(ruleResults));
        }
        
        // Salvar análise
        fraudAnalysisRepository.save(result);
        
        // Notificar se necessário
        if (result.getFinalDecision() == FraudDecision.BLOCK) {
            notificationService.notifyFraudDetected(context, result);
        }
        
        return result;
    }
    
    private FraudDecision calculateFinalDecision(List<FraudRuleResult> ruleResults) {
        int highRiskCount = 0;
        int mediumRiskCount = 0;
        
        for (FraudRuleResult result : ruleResults) {
            if (result.isSuspicious()) {
                switch (result.getRiskLevel()) {
                    case HIGH:
                        highRiskCount++;
                        break;
                    case MEDIUM:
                        mediumRiskCount++;
                        break;
                }
            }
        }
        
        if (highRiskCount >= 1) {
            return FraudDecision.BLOCK;
        } else if (mediumRiskCount >= 2) {
            return FraudDecision.REVIEW;
        } else {
            return FraudDecision.APPROVE;
        }
    }
}
```

**2. Machine Learning para Fraudes**
```java
// Service para análise usando ML
@Service
public class MLFraudDetectionService {
    
    private final MLModelService mlModelService;
    private final FeatureExtractor featureExtractor;
    private final FraudModelRepository fraudModelRepository;
    
    public MLFraudPrediction predictFraud(TransactionContext context) {
        try {
            // Extrair features da transação
            Map<String, Double> features = featureExtractor.extractFeatures(context);
            
            // Obter modelo atual
            FraudModel currentModel = fraudModelRepository.getCurrentModel();
            
            // Fazer predição
            MLPredictionResult prediction = mlModelService.predict(currentModel, features);
            
            return new MLFraudPrediction(
                prediction.getFraudProbability(),
                prediction.getConfidenceScore(),
                prediction.getFeatureImportance()
            );
            
        } catch (Exception e) {
            log.error("Erro na predição ML de fraude", e);
            return MLFraudPrediction.unknown();
        }
    }
}

@Component
public class FeatureExtractor {
    
    private final CustomerBehaviorService behaviorService;
    private final TransactionHistoryService historyService;
    
    public Map<String, Double> extractFeatures(TransactionContext context) {
        Map<String, Double> features = new HashMap<>();
        
        // Features básicas da transação
        features.put("amount", context.getAmount().doubleValue());
        features.put("hour_of_day", (double) context.getTransactionTime().getHour());
        features.put("day_of_week", (double) context.getTransactionTime().getDayOfWeek().getValue());
        
        // Features comportamentais
        CustomerBehavior behavior = behaviorService.getCustomerBehavior(context.getCustomerId());
        features.put("avg_transaction_amount", behavior.getAverageTransactionAmount());
        features.put("transaction_frequency", behavior.getTransactionFrequency());
        features.put("preferred_time_range", behavior.getPreferredTimeRange());
        
        // Features históricas
        TransactionHistory history = historyService.getTransactionHistory(
            context.getCustomerId(), Duration.ofDays(30));
        
        features.put("transactions_last_30_days", (double) history.getTransactionCount());
        features.put("max_amount_last_30_days", history.getMaxAmount());
        features.put("unique_merchants_last_30_days", (double) history.getUniqueMerchantCount());
        
        // Features de localização
        if (context.getLocation() != null) {
            features.put("is_new_location", context.getLocation().isNew() ? 1.0 : 0.0);
            features.put("distance_from_home", context.getLocation().getDistanceFromHome());
        }
        
        // Features de dispositivo
        if (context.getDeviceInfo() != null) {
            features.put("is_new_device", context.getDeviceInfo().isNew() ? 1.0 : 0.0);
            features.put("device_risk_score", context.getDeviceInfo().getRiskScore());
        }
        
        return features;
    }
}
```

### **💡 Caso de Uso Prático: Sistema Anti-Fraude para Cartão**

```java
// Sistema específico para fraudes em cartão
@Service
public class CardFraudDetectionService {
    
    private final FraudDetectionEngine fraudEngine;
    private final CardTransactionRepository cardTransactionRepository;
    private final CardSecurityService cardSecurityService;
    
    public CardFraudAnalysisResult analyzeCardTransaction(CardTransactionRequest request) {
        // Criar contexto da transação
        CardTransactionContext context = CardTransactionContext.builder()
            .transactionId(request.getTransactionId())
            .cardNumber(request.getCardNumber())
            .amount(request.getAmount())
            .merchantId(request.getMerchantId())
            .merchantCategory(request.getMerchantCategory())
            .location(request.getLocation())
            .transactionTime(LocalDateTime.now())
            .build();
        
        // Análise específica para cartão
        CardFraudAnalysisResult result = new CardFraudAnalysisResult(context.getTransactionId());
        
        // Verificar se cartão está bloqueado
        if (cardSecurityService.isCardBlocked(context.getCardNumber())) {
            result.setDecision(FraudDecision.BLOCK);
            result.setReason("Cartão bloqueado");
            return result;
        }
        
        // Verificar tentativas de transação recentes
        List<CardTransaction> recentAttempts = cardTransactionRepository
            .findRecentAttemptsByCard(context.getCardNumber(), Duration.ofMinutes(5));
        
        if (recentAttempts.size() > 3) {
            result.setDecision(FraudDecision.BLOCK);
            result.setReason("Múltiplas tentativas em pouco tempo");
            return result;
        }
        
        // Verificar padrão de gastos
        if (isUnusualSpendingPattern(context)) {
            result.setDecision(FraudDecision.REVIEW);
            result.setReason("Padrão de gastos incomum");
            return result;
        }
        
        // Verificar localização
        if (isUnusualLocation(context)) {
            result.setDecision(FraudDecision.REVIEW);
            result.setReason("Localização incomum");
            return result;
        }
        
        // Análise geral de fraude
        FraudAnalysisResult generalAnalysis = fraudEngine.analyzeTransaction(context);
        
        result.setDecision(generalAnalysis.getFinalDecision());
        result.setReason(generalAnalysis.getReason());
        result.setRiskScore(generalAnalysis.getRiskScore());
        
        return result;
    }
    
    private boolean isUnusualSpendingPattern(CardTransactionContext context) {
        // Buscar histórico dos últimos 30 dias
        List<CardTransaction> history = cardTransactionRepository
            .findByCardNumberAndDateRange(
                context.getCardNumber(),
                LocalDateTime.now().minusDays(30),
                LocalDateTime.now()
            );
        
        if (history.isEmpty()) {
            return false;
        }
        
        // Calcular estatísticas
        double averageAmount = history.stream()
            .mapToDouble(t -> t.getAmount().doubleValue())
            .average()
            .orElse(0.0);
        
        double standardDeviation = calculateStandardDeviation(history);
        
        // Verificar se valor está muito acima da média
        double currentAmount = context.getAmount().doubleValue();
        return currentAmount > (averageAmount + 2 * standardDeviation);
    }
    
    private boolean isUnusualLocation(CardTransactionContext context) {
        if (context.getLocation() == null) {
            return false;
        }
        
        // Buscar localizações recentes
        List<CardTransaction> recentTransactions = cardTransactionRepository
            .findByCardNumberAndDateRange(
                context.getCardNumber(),
                LocalDateTime.now().minusDays(7),
                LocalDateTime.now()
            );
        
        List<Location> recentLocations = recentTransactions.stream()
            .map(CardTransaction::getLocation)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
        
        if (recentLocations.isEmpty()) {
            return false;
        }
        
        // Verificar se localização atual está muito distante
        Location currentLocation = context.getLocation();
        
        return recentLocations.stream()
            .noneMatch(loc -> calculateDistance(currentLocation, loc) < 100); // 100 km
    }
    
    private double calculateStandardDeviation(List<CardTransaction> transactions) {
        double mean = transactions.stream()
            .mapToDouble(t -> t.getAmount().doubleValue())
            .average()
            .orElse(0.0);
        
        double variance = transactions.stream()
            .mapToDouble(t -> t.getAmount().doubleValue())
            .map(amount -> Math.pow(amount - mean, 2))
            .average()
            .orElse(0.0);
        
        return Math.sqrt(variance);
    }
}
```

---

## 🎯 **Exercícios Práticos**

### **Exercício 1: Sistema de Conta Poupança**
**Cenário:** Implementar sistema de conta poupança com rendimento automático.

**Sua tarefa:**
- Criar domínio para conta poupança
- Implementar cálculo de rendimento
- Configurar job para aplicar rendimento
- Implementar auditoria de rendimentos

### **Exercício 2: Sistema de Empréstimo**
**Cenário:** Criar sistema de análise e concessão de empréstimos.

**Sua tarefa:**
- Implementar análise de crédito
- Criar sistema de score
- Implementar aprovação automática
- Configurar notificações

### **Exercício 3: Sistema de Compliance**
**Cenário:** Implementar sistema de compliance para transações.

**Sua tarefa:**
- Criar regras de compliance
- Implementar verificação automática
- Configurar alertas para auditoria
- Implementar relatórios regulatórios

---

## 🚀 **Próximos Passos**

1. **Estude regulamentações** - BACEN, LGPD, PCI-DSS
2. **Pratique arquitetura** - Event-driven, CQRS
3. **Implemente segurança** - Criptografia, tokenização
4. **Configure monitoramento** - SLA, métricas de negócio

**Lembre-se:** Sistemas bancários exigem alta disponibilidade, segurança e compliance! 