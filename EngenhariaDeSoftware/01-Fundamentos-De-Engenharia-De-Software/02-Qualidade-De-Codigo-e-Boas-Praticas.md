# 🔍 Qualidade de Código - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Aplicar princípios de Clean Code
- ✅ Dominar os princípios SOLID
- ✅ Configurar ferramentas de análise de código
- ✅ Implementar refatoração sistemática
- ✅ Criar código testável e maintível

---

## 🎯 **Conceito 1: Clean Code - Código Limpo**

### **🔍 O que é?**
Clean Code é como escrever um livro que qualquer pessoa pode ler, entender e modificar facilmente. É código que conta uma história clara.

### **🎓 Conceitos Avançados:**

**1. Nomes Significativos**
```java
// ❌ Nomes ruins
public class BankingService {
    private List<Object> data; // Que dados?
    private String s; // String do quê?
    private int n; // Número de quê?
    
    public void process(String x, int y) { // Processar o quê?
        // lógica confusa
    }
}

// ✅ Nomes significativos
public class BankAccountService {
    private List<Account> customerAccounts;
    private String currentTransactionId;
    private int maxRetryAttempts;
    
    public TransactionResult processMoneyTransfer(
            String sourceAccountId, 
            BigDecimal transferAmount) {
        
        // Validar contas
        Account sourceAccount = validateAndGetAccount(sourceAccountId);
        
        // Verificar saldo disponível
        if (hasInsufficientFunds(sourceAccount, transferAmount)) {
            return TransactionResult.failure("Saldo insuficiente");
        }
        
        // Processar transferência
        return executeTransfer(sourceAccount, transferAmount);
    }
    
    private boolean hasInsufficientFunds(Account account, BigDecimal amount) {
        return account.getBalance().compareTo(amount) < 0;
    }
    
    private Account validateAndGetAccount(String accountId) {
        return accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(
                "Conta não encontrada: " + accountId));
    }
}
```

**2. Funções Pequenas e Focadas**
```java
// ❌ Função muito grande
public class OrderProcessor {
    public void processOrder(Order order) {
        // Validar dados do pedido
        if (order == null || order.getItems() == null || order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Pedido inválido");
        }
        
        // Calcular total
        BigDecimal total = BigDecimal.ZERO;
        for (OrderItem item : order.getItems()) {
            if (item.getQuantity() <= 0) {
                throw new IllegalArgumentException("Quantidade inválida");
            }
            BigDecimal itemTotal = item.getPrice().multiply(new BigDecimal(item.getQuantity()));
            total = total.add(itemTotal);
        }
        
        // Aplicar desconto
        if (order.getCustomer().isPremium()) {
            total = total.multiply(new BigDecimal("0.9"));
        }
        
        // Processar pagamento
        PaymentRequest paymentRequest = new PaymentRequest();
        paymentRequest.setAmount(total);
        paymentRequest.setCustomerId(order.getCustomer().getId());
        
        PaymentResult paymentResult = paymentService.processPayment(paymentRequest);
        
        if (!paymentResult.isSuccess()) {
            throw new PaymentFailedException("Pagamento falhou: " + paymentResult.getError());
        }
        
        // Atualizar estoque
        for (OrderItem item : order.getItems()) {
            inventoryService.updateStock(item.getProductId(), -item.getQuantity());
        }
        
        // Enviar confirmação
        emailService.sendOrderConfirmation(order.getCustomer().getEmail(), order);
        
        // Atualizar status
        order.setStatus(OrderStatus.CONFIRMED);
        orderRepository.save(order);
    }
}

// ✅ Funções pequenas e focadas
public class OrderProcessor {
    
    public void processOrder(Order order) {
        validateOrder(order);
        
        BigDecimal totalAmount = calculateTotalAmount(order);
        BigDecimal finalAmount = applyDiscounts(order, totalAmount);
        
        processPayment(order, finalAmount);
        updateInventory(order);
        sendConfirmation(order);
        updateOrderStatus(order);
    }
    
    private void validateOrder(Order order) {
        if (order == null) {
            throw new IllegalArgumentException("Pedido não pode ser nulo");
        }
        
        if (order.getItems() == null || order.getItems().isEmpty()) {
            throw new IllegalArgumentException("Pedido deve ter pelo menos um item");
        }
        
        validateOrderItems(order.getItems());
    }
    
    private void validateOrderItems(List<OrderItem> items) {
        for (OrderItem item : items) {
            if (item.getQuantity() <= 0) {
                throw new IllegalArgumentException(
                    "Quantidade deve ser positiva para produto: " + item.getProductId());
            }
            
            if (item.getPrice() == null || item.getPrice().compareTo(BigDecimal.ZERO) <= 0) {
                throw new IllegalArgumentException(
                    "Preço deve ser positivo para produto: " + item.getProductId());
            }
        }
    }
    
    private BigDecimal calculateTotalAmount(Order order) {
        return order.getItems().stream()
            .map(this::calculateItemTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
    
    private BigDecimal calculateItemTotal(OrderItem item) {
        return item.getPrice().multiply(new BigDecimal(item.getQuantity()));
    }
    
    private BigDecimal applyDiscounts(Order order, BigDecimal totalAmount) {
        if (order.getCustomer().isPremium()) {
            return applyPremiumDiscount(totalAmount);
        }
        
        return totalAmount;
    }
    
    private BigDecimal applyPremiumDiscount(BigDecimal amount) {
        BigDecimal discountRate = new BigDecimal("0.10"); // 10% de desconto
        BigDecimal discount = amount.multiply(discountRate);
        return amount.subtract(discount);
    }
    
    private void processPayment(Order order, BigDecimal amount) {
        PaymentRequest request = PaymentRequest.builder()
            .amount(amount)
            .customerId(order.getCustomer().getId())
            .orderId(order.getId())
            .build();
        
        PaymentResult result = paymentService.processPayment(request);
        
        if (!result.isSuccess()) {
            throw new PaymentFailedException(
                "Falha no pagamento: " + result.getErrorMessage());
        }
    }
    
    private void updateInventory(Order order) {
        for (OrderItem item : order.getItems()) {
            inventoryService.reserveStock(item.getProductId(), item.getQuantity());
        }
    }
    
    private void sendConfirmation(Order order) {
        String customerEmail = order.getCustomer().getEmail();
        emailService.sendOrderConfirmation(customerEmail, order);
    }
    
    private void updateOrderStatus(Order order) {
        order.setStatus(OrderStatus.CONFIRMED);
        order.setProcessedAt(LocalDateTime.now());
        orderRepository.save(order);
    }
}
```

**3. Tratamento de Erros Elegante**
```java
// ❌ Tratamento de erros ruim
public class BankingService {
    public void transfer(String fromAccount, String toAccount, BigDecimal amount) {
        try {
            Account from = accountRepository.findById(fromAccount);
            Account to = accountRepository.findById(toAccount);
            
            if (from.getBalance().compareTo(amount) < 0) {
                throw new Exception("Saldo insuficiente");
            }
            
            from.setBalance(from.getBalance().subtract(amount));
            to.setBalance(to.getBalance().add(amount));
            
            accountRepository.save(from);
            accountRepository.save(to);
            
        } catch (Exception e) {
            // Log genérico
            logger.error("Erro: " + e.getMessage());
            throw new RuntimeException("Erro na transferência");
        }
    }
}

// ✅ Tratamento de erros elegante
public class BankingService {
    
    public TransferResult transfer(String fromAccountId, String toAccountId, BigDecimal amount) {
        try {
            validateTransferParameters(fromAccountId, toAccountId, amount);
            
            Account sourceAccount = getAccountById(fromAccountId);
            Account targetAccount = getAccountById(toAccountId);
            
            validateSufficientFunds(sourceAccount, amount);
            
            executeTransfer(sourceAccount, targetAccount, amount);
            
            return TransferResult.success("Transferência realizada com sucesso");
            
        } catch (AccountNotFoundException e) {
            logger.warn("Conta não encontrada na transferência: {}", e.getMessage());
            return TransferResult.failure("Conta não encontrada", ErrorCode.ACCOUNT_NOT_FOUND);
            
        } catch (InsufficientFundsException e) {
            logger.warn("Saldo insuficiente para transferência: fromAccount={}, amount={}", 
                fromAccountId, amount);
            return TransferResult.failure("Saldo insuficiente", ErrorCode.INSUFFICIENT_FUNDS);
            
        } catch (ValidationException e) {
            logger.warn("Dados inválidos na transferência: {}", e.getMessage());
            return TransferResult.failure("Dados inválidos", ErrorCode.INVALID_DATA);
            
        } catch (Exception e) {
            logger.error("Erro inesperado na transferência: fromAccount={}, toAccount={}, amount={}", 
                fromAccountId, toAccountId, amount, e);
            return TransferResult.failure("Erro interno do sistema", ErrorCode.INTERNAL_ERROR);
        }
    }
    
    private void validateTransferParameters(String fromAccountId, String toAccountId, BigDecimal amount) {
        if (StringUtils.isBlank(fromAccountId) || StringUtils.isBlank(toAccountId)) {
            throw new ValidationException("IDs das contas são obrigatórios");
        }
        
        if (fromAccountId.equals(toAccountId)) {
            throw new ValidationException("Não é possível transferir para a mesma conta");
        }
        
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new ValidationException("Valor deve ser positivo");
        }
        
        if (amount.compareTo(MAX_TRANSFER_AMOUNT) > 0) {
            throw new ValidationException("Valor excede o limite máximo");
        }
    }
    
    private Account getAccountById(String accountId) {
        return accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException("Conta não encontrada: " + accountId));
    }
    
    private void validateSufficientFunds(Account account, BigDecimal amount) {
        if (account.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException(
                "Saldo insuficiente. Saldo atual: " + account.getBalance() + 
                ", Valor solicitado: " + amount);
        }
    }
    
    @Transactional
    private void executeTransfer(Account sourceAccount, Account targetAccount, BigDecimal amount) {
        sourceAccount.debit(amount);
        targetAccount.credit(amount);
        
        accountRepository.save(sourceAccount);
        accountRepository.save(targetAccount);
        
        // Registrar transação para auditoria
        auditService.recordTransfer(sourceAccount.getId(), targetAccount.getId(), amount);
    }
}
```

### **💡 Caso de Uso Prático: Refatoração de Sistema Bancário**

```java
// Sistema antes da refatoração
public class BankingSystemOld {
    public String process(String type, String acc1, String acc2, double amt, String ccy) {
        if (type.equals("TXF")) {
            // Transferência
            // ... 200 linhas de código
        } else if (type.equals("DEP")) {
            // Depósito
            // ... 150 linhas de código
        } else if (type.equals("WTH")) {
            // Saque
            // ... 100 linhas de código
        }
        return "OK";
    }
}

// Sistema após refatoração
public class BankingSystem {
    
    private final Map<TransactionType, TransactionProcessor> processors;
    
    public BankingSystem() {
        this.processors = Map.of(
            TransactionType.TRANSFER, new TransferProcessor(),
            TransactionType.DEPOSIT, new DepositProcessor(),
            TransactionType.WITHDRAWAL, new WithdrawalProcessor()
        );
    }
    
    public TransactionResult processTransaction(TransactionRequest request) {
        validateTransactionRequest(request);
        
        TransactionProcessor processor = getProcessorForType(request.getType());
        
        return processor.process(request);
    }
    
    private void validateTransactionRequest(TransactionRequest request) {
        if (request == null) {
            throw new ValidationException("Requisição não pode ser nula");
        }
        
        if (request.getType() == null) {
            throw new ValidationException("Tipo de transação é obrigatório");
        }
        
        if (request.getAmount() == null || request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new ValidationException("Valor deve ser positivo");
        }
    }
    
    private TransactionProcessor getProcessorForType(TransactionType type) {
        TransactionProcessor processor = processors.get(type);
        
        if (processor == null) {
            throw new UnsupportedTransactionTypeException(
                "Tipo de transação não suportado: " + type);
        }
        
        return processor;
    }
}

// Interface para processadores
public interface TransactionProcessor {
    TransactionResult process(TransactionRequest request);
}

// Implementação específica para transferência
public class TransferProcessor implements TransactionProcessor {
    
    private final AccountService accountService;
    private final AuditService auditService;
    
    @Override
    public TransactionResult process(TransactionRequest request) {
        validateTransferRequest(request);
        
        Account sourceAccount = accountService.getAccount(request.getSourceAccountId());
        Account targetAccount = accountService.getAccount(request.getTargetAccountId());
        
        validateTransferEligibility(sourceAccount, targetAccount, request.getAmount());
        
        return executeTransfer(sourceAccount, targetAccount, request.getAmount());
    }
    
    private void validateTransferRequest(TransactionRequest request) {
        if (StringUtils.isBlank(request.getSourceAccountId())) {
            throw new ValidationException("Conta de origem é obrigatória");
        }
        
        if (StringUtils.isBlank(request.getTargetAccountId())) {
            throw new ValidationException("Conta de destino é obrigatória");
        }
        
        if (request.getSourceAccountId().equals(request.getTargetAccountId())) {
            throw new ValidationException("Contas de origem e destino devem ser diferentes");
        }
    }
    
    private void validateTransferEligibility(Account source, Account target, BigDecimal amount) {
        if (source.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Saldo insuficiente");
        }
        
        if (source.isBlocked() || target.isBlocked()) {
            throw new AccountBlockedException("Uma das contas está bloqueada");
        }
    }
    
    @Transactional
    private TransactionResult executeTransfer(Account source, Account target, BigDecimal amount) {
        String transactionId = generateTransactionId();
        
        try {
            source.debit(amount);
            target.credit(amount);
            
            accountService.saveAccount(source);
            accountService.saveAccount(target);
            
            auditService.recordTransfer(transactionId, source.getId(), target.getId(), amount);
            
            return TransactionResult.success(transactionId);
            
        } catch (Exception e) {
            auditService.recordTransferFailure(transactionId, source.getId(), target.getId(), amount, e);
            throw new TransactionProcessingException("Falha ao processar transferência", e);
        }
    }
    
    private String generateTransactionId() {
        return "TXF" + System.currentTimeMillis() + UUID.randomUUID().toString().substring(0, 8);
    }
}
```

---

## 🎯 **Conceito 2: Princípios SOLID**

### **🔍 O que são?**
SOLID são cinco princípios que tornam o código mais flexível, maintível e testável. É como ter regras de arquitetura para construir um prédio sólido.

### **🎓 Conceitos Avançados:**

**1. Single Responsibility Principle (SRP)**
```java
// ❌ Violação do SRP - múltiplas responsabilidades
public class User {
    private String id;
    private String name;
    private String email;
    
    // Responsabilidade 1: Representar dados do usuário
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    
    // Responsabilidade 2: Validar dados
    public boolean isValidEmail() {
        return email != null && email.contains("@");
    }
    
    // Responsabilidade 3: Persistir dados
    public void save() {
        // Código para salvar no banco
    }
    
    // Responsabilidade 4: Enviar notificações
    public void sendWelcomeEmail() {
        // Código para enviar email
    }
    
    // Responsabilidade 5: Formatação
    public String getFormattedName() {
        return name.toUpperCase();
    }
}

// ✅ Seguindo SRP - cada classe tem uma responsabilidade
public class User {
    private String id;
    private String name;
    private String email;
    
    public User(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }
    
    // Apenas representar dados do usuário
    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
}

@Component
public class UserValidator {
    
    public ValidationResult validate(User user) {
        ValidationResult result = new ValidationResult();
        
        if (StringUtils.isBlank(user.getName())) {
            result.addError("Nome é obrigatório");
        }
        
        if (!isValidEmail(user.getEmail())) {
            result.addError("Email inválido");
        }
        
        return result;
    }
    
    private boolean isValidEmail(String email) {
        return email != null && 
               email.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$");
    }
}

@Repository
public class UserRepository {
    
    public User save(User user) {
        // Lógica para salvar no banco
        return user;
    }
    
    public Optional<User> findById(String id) {
        // Lógica para buscar no banco
        return Optional.empty();
    }
}

@Service
public class UserNotificationService {
    
    private final EmailService emailService;
    
    public void sendWelcomeEmail(User user) {
        EmailMessage message = EmailMessage.builder()
            .to(user.getEmail())
            .subject("Bem-vindo!")
            .body("Olá " + user.getName() + ", bem-vindo ao sistema!")
            .build();
        
        emailService.send(message);
    }
}

@Component
public class UserFormatter {
    
    public String formatName(User user) {
        return user.getName().toUpperCase();
    }
    
    public String formatEmail(User user) {
        return user.getEmail().toLowerCase();
    }
}
```

**2. Open/Closed Principle (OCP)**
```java
// ❌ Violação do OCP - precisa modificar para adicionar novos tipos
public class PaymentProcessor {
    
    public void processPayment(String paymentType, BigDecimal amount) {
        if (paymentType.equals("CREDIT_CARD")) {
            // Processar cartão de crédito
            processCreditCard(amount);
        } else if (paymentType.equals("PAYPAL")) {
            // Processar PayPal
            processPayPal(amount);
        } else if (paymentType.equals("BANK_TRANSFER")) {
            // Processar transferência bancária
            processBankTransfer(amount);
        }
        // Para adicionar PIX, seria necessário modificar este método
    }
    
    private void processCreditCard(BigDecimal amount) { /* ... */ }
    private void processPayPal(BigDecimal amount) { /* ... */ }
    private void processBankTransfer(BigDecimal amount) { /* ... */ }
}

// ✅ Seguindo OCP - extensível sem modificação
public interface PaymentProcessor {
    PaymentResult process(PaymentRequest request);
    boolean supports(PaymentType type);
}

@Component
public class CreditCardProcessor implements PaymentProcessor {
    
    @Override
    public PaymentResult process(PaymentRequest request) {
        // Lógica específica para cartão de crédito
        return processWithCreditCard(request);
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.CREDIT_CARD;
    }
    
    private PaymentResult processWithCreditCard(PaymentRequest request) {
        // Implementação específica
        return PaymentResult.success("Pagamento processado com cartão");
    }
}

@Component
public class PayPalProcessor implements PaymentProcessor {
    
    @Override
    public PaymentResult process(PaymentRequest request) {
        return processWithPayPal(request);
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.PAYPAL;
    }
    
    private PaymentResult processWithPayPal(PaymentRequest request) {
        return PaymentResult.success("Pagamento processado com PayPal");
    }
}

// Novo processador PIX - não precisa modificar código existente
@Component
public class PixProcessor implements PaymentProcessor {
    
    @Override
    public PaymentResult process(PaymentRequest request) {
        return processWithPix(request);
    }
    
    @Override
    public boolean supports(PaymentType type) {
        return type == PaymentType.PIX;
    }
    
    private PaymentResult processWithPix(PaymentRequest request) {
        return PaymentResult.success("Pagamento processado com PIX");
    }
}

@Service
public class PaymentService {
    
    private final List<PaymentProcessor> processors;
    
    public PaymentService(List<PaymentProcessor> processors) {
        this.processors = processors;
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        PaymentProcessor processor = findProcessor(request.getType());
        
        if (processor == null) {
            return PaymentResult.failure("Tipo de pagamento não suportado");
        }
        
        return processor.process(request);
    }
    
    private PaymentProcessor findProcessor(PaymentType type) {
        return processors.stream()
            .filter(p -> p.supports(type))
            .findFirst()
            .orElse(null);
    }
}
```

**3. Liskov Substitution Principle (LSP)**
```java
// ❌ Violação do LSP - subclasse não pode substituir a superclasse
public class Rectangle {
    protected double width;
    protected double height;
    
    public void setWidth(double width) {
        this.width = width;
    }
    
    public void setHeight(double height) {
        this.height = height;
    }
    
    public double getArea() {
        return width * height;
    }
}

public class Square extends Rectangle {
    
    @Override
    public void setWidth(double width) {
        this.width = width;
        this.height = width; // Quebra o comportamento esperado
    }
    
    @Override
    public void setHeight(double height) {
        this.width = height;
        this.height = height; // Quebra o comportamento esperado
    }
}

// Teste que falha com Square
public void testRectangle() {
    Rectangle rect = new Square(); // Violação do LSP
    rect.setWidth(5);
    rect.setHeight(4);
    
    // Espera-se 20, mas com Square será 16
    assertEquals(20, rect.getArea());
}

// ✅ Seguindo LSP - hierarquia correta
public abstract class Shape {
    public abstract double getArea();
    public abstract double getPerimeter();
}

public class Rectangle extends Shape {
    private final double width;
    private final double height;
    
    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }
    
    @Override
    public double getArea() {
        return width * height;
    }
    
    @Override
    public double getPerimeter() {
        return 2 * (width + height);
    }
    
    public double getWidth() { return width; }
    public double getHeight() { return height; }
}

public class Square extends Shape {
    private final double side;
    
    public Square(double side) {
        this.side = side;
    }
    
    @Override
    public double getArea() {
        return side * side;
    }
    
    @Override
    public double getPerimeter() {
        return 4 * side;
    }
    
    public double getSide() { return side; }
}

// Teste que funciona com qualquer Shape
public void testShapes() {
    Shape rectangle = new Rectangle(5, 4);
    Shape square = new Square(4);
    
    // Ambas podem ser tratadas como Shape
    calculateShapeArea(rectangle);
    calculateShapeArea(square);
}

private void calculateShapeArea(Shape shape) {
    System.out.println("Área: " + shape.getArea());
}
```

**4. Interface Segregation Principle (ISP)**
```java
// ❌ Violação do ISP - interface muito grande
public interface Worker {
    void work();
    void eat();
    void sleep();
    void attendMeeting();
    void writeCode();
    void manageTeam();
}

public class Developer implements Worker {
    
    @Override
    public void work() {
        // Desenvolvedor trabalha
    }
    
    @Override
    public void eat() {
        // Desenvolvedor come
    }
    
    @Override
    public void sleep() {
        // Desenvolvedor dorme
    }
    
    @Override
    public void attendMeeting() {
        // Desenvolvedor participa de reuniões
    }
    
    @Override
    public void writeCode() {
        // Desenvolvedor escreve código
    }
    
    @Override
    public void manageTeam() {
        // Desenvolvedor não gerencia equipe - método desnecessário
        throw new UnsupportedOperationException("Developer doesn't manage team");
    }
}

// ✅ Seguindo ISP - interfaces específicas
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public interface MeetingAttendable {
    void attendMeeting();
}

public interface Codeable {
    void writeCode();
}

public interface Manageable {
    void manageTeam();
}

public class Developer implements Workable, Eatable, Sleepable, MeetingAttendable, Codeable {
    
    @Override
    public void work() {
        System.out.println("Desenvolvedor trabalhando");
    }
    
    @Override
    public void eat() {
        System.out.println("Desenvolvedor comendo");
    }
    
    @Override
    public void sleep() {
        System.out.println("Desenvolvedor dormindo");
    }
    
    @Override
    public void attendMeeting() {
        System.out.println("Desenvolvedor em reunião");
    }
    
    @Override
    public void writeCode() {
        System.out.println("Desenvolvedor escrevendo código");
    }
}

public class Manager implements Workable, Eatable, Sleepable, MeetingAttendable, Manageable {
    
    @Override
    public void work() {
        System.out.println("Gerente trabalhando");
    }
    
    @Override
    public void eat() {
        System.out.println("Gerente comendo");
    }
    
    @Override
    public void sleep() {
        System.out.println("Gerente dormindo");
    }
    
    @Override
    public void attendMeeting() {
        System.out.println("Gerente em reunião");
    }
    
    @Override
    public void manageTeam() {
        System.out.println("Gerente gerenciando equipe");
    }
}
```

**5. Dependency Inversion Principle (DIP)**
```java
// ❌ Violação do DIP - dependência de concretização
public class OrderService {
    
    private EmailService emailService; // Dependência concreta
    private MySQLOrderRepository orderRepository; // Dependência concreta
    
    public OrderService() {
        this.emailService = new EmailService(); // Acoplamento forte
        this.orderRepository = new MySQLOrderRepository(); // Acoplamento forte
    }
    
    public void processOrder(Order order) {
        orderRepository.save(order);
        emailService.sendConfirmation(order.getCustomerEmail());
    }
}

// ✅ Seguindo DIP - dependência de abstrações
public interface NotificationService {
    void sendNotification(String recipient, String message);
}

public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String orderId);
}

@Service
public class OrderService {
    
    private final NotificationService notificationService;
    private final OrderRepository orderRepository;
    
    // Injeção de dependência via construtor
    public OrderService(NotificationService notificationService, 
                       OrderRepository orderRepository) {
        this.notificationService = notificationService;
        this.orderRepository = orderRepository;
    }
    
    public void processOrder(Order order) {
        orderRepository.save(order);
        
        String message = "Pedido " + order.getId() + " processado com sucesso";
        notificationService.sendNotification(order.getCustomerEmail(), message);
    }
}

// Implementações concretas
@Component
public class EmailNotificationService implements NotificationService {
    
    @Override
    public void sendNotification(String recipient, String message) {
        // Implementação específica para email
        System.out.println("Enviando email para " + recipient + ": " + message);
    }
}

@Component
public class SMSNotificationService implements NotificationService {
    
    @Override
    public void sendNotification(String recipient, String message) {
        // Implementação específica para SMS
        System.out.println("Enviando SMS para " + recipient + ": " + message);
    }
}

@Repository
public class JpaOrderRepository implements OrderRepository {
    
    @Override
    public void save(Order order) {
        // Implementação específica para JPA
    }
    
    @Override
    public Optional<Order> findById(String orderId) {
        // Implementação específica para JPA
        return Optional.empty();
    }
}
```

### **💡 Caso de Uso Prático: Sistema de Fidelidade**

```java
// Sistema de fidelidade seguindo todos os princípios SOLID
public interface LoyaltyCalculator {
    int calculatePoints(Purchase purchase);
}

public interface LoyaltyStorage {
    void savePoints(String customerId, int points);
    int getPoints(String customerId);
}

public interface LoyaltyNotification {
    void notifyPointsEarned(String customerId, int points);
}

@Component
public class StandardLoyaltyCalculator implements LoyaltyCalculator {
    
    @Override
    public int calculatePoints(Purchase purchase) {
        // 1 ponto para cada R$ 1,00
        return purchase.getAmount().intValue();
    }
}

@Component
public class PremiumLoyaltyCalculator implements LoyaltyCalculator {
    
    @Override
    public int calculatePoints(Purchase purchase) {
        // 2 pontos para cada R$ 1,00 para clientes premium
        return purchase.getAmount().intValue() * 2;
    }
}

@Service
public class LoyaltyService {
    
    private final LoyaltyStorage storage;
    private final LoyaltyNotification notification;
    private final Map<CustomerType, LoyaltyCalculator> calculators;
    
    public LoyaltyService(LoyaltyStorage storage, 
                         LoyaltyNotification notification,
                         List<LoyaltyCalculator> calculators) {
        this.storage = storage;
        this.notification = notification;
        this.calculators = Map.of(
            CustomerType.STANDARD, new StandardLoyaltyCalculator(),
            CustomerType.PREMIUM, new PremiumLoyaltyCalculator()
        );
    }
    
    public void processLoyaltyPoints(String customerId, Purchase purchase, CustomerType type) {
        LoyaltyCalculator calculator = calculators.get(type);
        
        int points = calculator.calculatePoints(purchase);
        
        storage.savePoints(customerId, points);
        notification.notifyPointsEarned(customerId, points);
    }
}
```

---

## 🎯 **Conceito 3: Ferramentas de Análise de Código**

### **🔍 O que são?**
Ferramentas de análise são como revisores automáticos que encontram problemas no código antes que se tornem bugs em produção.

### **🎓 Configuração Avançada:**

**1. SonarQube Configuration**
```xml
<!-- pom.xml -->
<properties>
    <sonar.organization>banking-org</sonar.organization>
    <sonar.host.url>https://sonarcloud.io</sonar.host.url>
    <sonar.login>${env.SONAR_TOKEN}</sonar.login>
    <sonar.coverage.jacoco.xmlReportPaths>target/site/jacoco/jacoco.xml</sonar.coverage.jacoco.xmlReportPaths>
    <sonar.junit.reportPaths>target/surefire-reports</sonar.junit.reportPaths>
</properties>

<build>
    <plugins>
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
                <execution>
                    <id>check</id>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <rule>
                                <element>PACKAGE</element>
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum>
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

**2. Checkstyle Configuration**
```xml
<!-- checkstyle.xml -->
<?xml version="1.0"?>
<!DOCTYPE module PUBLIC
          "-//Puppy Crawl//DTD Check Configuration 1.3//EN"
          "https://checkstyle.org/dtds/configuration_1_3.dtd">

<module name="Checker">
    <property name="charset" value="UTF-8"/>
    <property name="severity" value="error"/>
    <property name="fileExtensions" value="java, properties, xml"/>

    <module name="FileTabCharacter">
        <property name="eachLine" value="true"/>
    </module>

    <module name="TreeWalker">
        <!-- Naming Conventions -->
        <module name="ConstantName"/>
        <module name="LocalFinalVariableName"/>
        <module name="LocalVariableName"/>
        <module name="MemberName"/>
        <module name="MethodName"/>
        <module name="PackageName"/>
        <module name="ParameterName"/>
        <module name="StaticVariableName"/>
        <module name="TypeName"/>

        <!-- Size Violations -->
        <module name="LineLength">
            <property name="max" value="120"/>
            <property name="ignorePattern" value="^package.*|^import.*|a href|href|http://|https://|ftp://"/>
        </module>
        <module name="MethodLength">
            <property name="max" value="50"/>
        </module>
        <module name="ParameterNumber">
            <property name="max" value="5"/>
        </module>

        <!-- Whitespace -->
        <module name="EmptyForIteratorPad"/>
        <module name="GenericWhitespace"/>
        <module name="MethodParamPad"/>
        <module name="NoWhitespaceAfter"/>
        <module name="NoWhitespaceBefore"/>
        <module name="OperatorWrap"/>
        <module name="ParenPad"/>
        <module name="TypecastParenPad"/>
        <module name="WhitespaceAfter"/>
        <module name="WhitespaceAround"/>

        <!-- Modifier Checks -->
        <module name="ModifierOrder"/>
        <module name="RedundantModifier"/>

        <!-- Blocks -->
        <module name="AvoidNestedBlocks"/>
        <module name="EmptyBlock"/>
        <module name="LeftCurly"/>
        <module name="NeedBraces"/>
        <module name="RightCurly"/>

        <!-- Common Coding -->
        <module name="AvoidInlineConditionals"/>
        <module name="EmptyStatement"/>
        <module name="EqualsHashCode"/>
        <module name="HiddenField"/>
        <module name="IllegalInstantiation"/>
        <module name="InnerAssignment"/>
        <module name="MagicNumber"/>
        <module name="MissingSwitchDefault"/>
        <module name="SimplifyBooleanExpression"/>
        <module name="SimplifyBooleanReturn"/>

        <!-- Class Design -->
        <module name="DesignForExtension"/>
        <module name="FinalClass"/>
        <module name="HideUtilityClassConstructor"/>
        <module name="InterfaceIsType"/>
        <module name="VisibilityModifier"/>

        <!-- Miscellaneous -->
        <module name="ArrayTypeStyle"/>
        <module name="FinalParameters"/>
        <module name="TodoComment"/>
        <module name="UpperEll"/>
    </module>
</module>
```

**3. PMD Configuration**
```xml
<!-- pmd-ruleset.xml -->
<?xml version="1.0"?>
<ruleset name="Banking App Rules"
         xmlns="http://pmd.sourceforge.net/ruleset/2.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://pmd.sourceforge.net/ruleset/2.0.0 https://pmd.sourceforge.net/ruleset_2_0_0.xsd">

    <description>PMD rules for Banking Application</description>

    <rule ref="category/java/bestpractices.xml">
        <exclude name="GuardLogStatement"/>
    </rule>

    <rule ref="category/java/codestyle.xml">
        <exclude name="OnlyOneReturn"/>
        <exclude name="AtLeastOneConstructor"/>
        <exclude name="CallSuperInConstructor"/>
    </rule>

    <rule ref="category/java/design.xml">
        <exclude name="LawOfDemeter"/>
        <exclude name="LoosePackageCoupling"/>
    </rule>

    <rule ref="category/java/documentation.xml">
        <exclude name="CommentRequired"/>
    </rule>

    <rule ref="category/java/errorprone.xml"/>

    <rule ref="category/java/multithreading.xml"/>

    <rule ref="category/java/performance.xml"/>

    <rule ref="category/java/security.xml"/>

    <!-- Custom rules -->
    <rule ref="category/java/design.xml/CyclomaticComplexity">
        <properties>
            <property name="classReportLevel" value="80"/>
            <property name="methodReportLevel" value="10"/>
        </properties>
    </rule>

    <rule ref="category/java/design.xml/TooManyMethods">
        <properties>
            <property name="maxmethods" value="15"/>
        </properties>
    </rule>

    <rule ref="category/java/design.xml/TooManyFields">
        <properties>
            <property name="maxfields" value="10"/>
        </properties>
    </rule>
</ruleset>
```

---

## 🎯 **Exercícios Práticos**

### **Exercício 1: Refatoração SOLID**
**Cenário:** Classe com múltiplas responsabilidades.

**Sua tarefa:**
- Identifique violações dos princípios SOLID
- Refatore aplicando cada princípio
- Justifique as mudanças

### **Exercício 2: Clean Code Challenge**
**Cenário:** Código legado com problemas de nomenclatura e estrutura.

**Sua tarefa:**
- Melhore nomes de variáveis e métodos
- Extraia métodos menores
- Implemente tratamento de erros adequado

### **Exercício 3: Análise de Código**
**Cenário:** Configurar pipeline de qualidade completa.

**Sua tarefa:**
- Configure SonarQube, Checkstyle e PMD
- Defina métricas de qualidade
- Implemente quality gates

---

## 🚀 **Próximos Passos**

1. **Pratique refatoração** - Melhore código existente
2. **Estude testes** - Próximo: 06-Piramide-Testes.md
3. **Configure ferramentas** - Análise automatizada
4. **Aplique princípios** - Em todos os projetos

**Lembre-se:** Código limpo é um investimento no futuro! 