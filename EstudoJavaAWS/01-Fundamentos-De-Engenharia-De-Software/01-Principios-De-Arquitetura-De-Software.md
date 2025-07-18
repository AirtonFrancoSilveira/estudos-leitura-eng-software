# 🏗️ Arquitetura de Sistemas - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Explicar os trade-offs entre diferentes arquiteturas
- ✅ Projetar sistemas usando Domain-Driven Design
- ✅ Implementar padrões arquiteturais avançados
- ✅ Resolver casos de uso complexos com arquitetura apropriada

---

## 🎯 **Conceito 1: Arquitetura Monolítica**

### **🔍 O que é?**
Uma arquitetura monolítica é como uma casa onde todos os cômodos estão conectados por uma estrutura única. Toda a aplicação roda em um único processo, compartilhando memória, CPU e recursos.

### **🎓 Conceitos Avançados:**

**1. Modular Monolith Pattern**
```java
// Organização em módulos bem definidos
@SpringBootApplication
public class ECommerceApplication {
    public static void main(String[] args) {
        SpringApplication.run(ECommerceApplication.class, args);
    }
}

// Módulo de Pedidos
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderService orderService;
    
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        Order order = orderService.createOrder(request);
        return ResponseEntity.ok(order);
    }
}

// Módulo de Pagamentos
@RestController
@RequestMapping("/payments")
public class PaymentController {
    private final PaymentService paymentService;
    
    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
    
    @PostMapping
    public ResponseEntity<Payment> processPayment(@RequestBody PaymentRequest request) {
        Payment payment = paymentService.processPayment(request);
        return ResponseEntity.ok(payment);
    }
}
```

**2. Shared Database Anti-Pattern**
```java
// ❌ PROBLEMA: Todos os módulos acessam todas as tabelas
@Repository
public class OrderRepository {
    // Acesso direto a tabelas de outros módulos
    @Query("SELECT o FROM Order o JOIN Customer c ON o.customerId = c.id")
    List<Order> findOrdersWithCustomerData();
}

// ✅ SOLUÇÃO: Encapsulamento com interfaces
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final CustomerService customerService; // Interface bem definida
    
    public OrderDetails getOrderDetails(Long orderId) {
        Order order = orderRepository.findById(orderId);
        Customer customer = customerService.findById(order.getCustomerId());
        return new OrderDetails(order, customer);
    }
}
```

### **💡 Caso de Uso Prático: Sistema de E-commerce**

**Problema:** Você precisa criar um sistema de e-commerce para uma startup. A equipe tem 5 desenvolvedores e precisa entregar um MVP em 3 meses.

**Solução Monolítica:**
```java
@SpringBootApplication
public class ECommerceMonolithApplication {
    // Configuração única para toda aplicação
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create()
            .url("jdbc:postgresql://localhost:5432/ecommerce")
            .username("app")
            .password("secret")
            .build();
    }
    
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(jedisConnectionFactory());
        return template;
    }
}

// Todos os domínios em um único projeto
@Service
@Transactional
public class OrderService {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final CustomerRepository customerRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final EmailService emailService;
    
    public Order createOrder(CreateOrderRequest request) {
        // Validações
        Customer customer = customerRepository.findById(request.getCustomerId());
        Product product = productRepository.findById(request.getProductId());
        
        // Lógica de negócio
        if (inventoryService.isAvailable(product.getId(), request.getQuantity())) {
            Order order = new Order(customer, product, request.getQuantity());
            
            // Transação local - ACID garantido
            orderRepository.save(order);
            inventoryService.reserve(product.getId(), request.getQuantity());
            
            // Processar pagamento
            Payment payment = paymentService.processPayment(order.getTotal());
            
            if (payment.isApproved()) {
                order.setStatus(OrderStatus.CONFIRMED);
                emailService.sendConfirmation(customer.getEmail(), order);
            }
            
            return order;
        }
        
        throw new OutOfStockException("Product not available");
    }
}
```

**Vantagens desta abordagem:**
- 🚀 **Time to Market**: Deploy único, desenvolvimento rápido
- 🔒 **Consistency**: Transações ACID garantidas
- 🐛 **Debugging**: Stack trace completo em um lugar
- 📊 **Performance**: Sem latência de rede interna

**Quando usar:** Startups, MVPs, equipes pequenas (< 10 pessoas), domínios simples.

---

## 🎯 **Conceito 2: Arquitetura de Microserviços**

### **🔍 O que é?**
Microserviços são como um condomínio onde cada apartamento (serviço) tem sua própria infraestrutura, mas todos compartilham alguns recursos comuns (portaria, elevadores).

### **🎓 Conceitos Avançados:**

**1. Service Mesh Pattern**
```java
// Configuração Istio para Service Mesh
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: order-service
spec:
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: order-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: order-service
        subset: v1
      weight: 90
    - destination:
        host: order-service
        subset: v2
      weight: 10
```

**2. Circuit Breaker Pattern**
```java
@Component
public class PaymentServiceClient {
    
    private final RestTemplate restTemplate;
    private final CircuitBreaker circuitBreaker;
    
    public PaymentServiceClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
        this.circuitBreaker = CircuitBreaker.ofDefaults("payment-service");
        
        // Configuração do Circuit Breaker
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                log.info("Circuit breaker state transition: {}", event));
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        return circuitBreaker.executeSupplier(() -> {
            ResponseEntity<PaymentResult> response = restTemplate.postForEntity(
                "http://payment-service/payments",
                request,
                PaymentResult.class
            );
            return response.getBody();
        });
    }
}
```

**3. Event-Driven Communication**
```java
// Order Service - Publisher
@Service
public class OrderService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Publicar evento
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(),
            order.getCustomerId(),
            order.getTotal(),
            order.getItems()
        );
        
        eventPublisher.publishEvent(event);
        return order;
    }
}

// Payment Service - Subscriber
@Service
public class PaymentEventHandler {
    
    @EventListener
    @Async
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            PaymentRequest request = PaymentRequest.builder()
                .orderId(event.getOrderId())
                .customerId(event.getCustomerId())
                .amount(event.getTotal())
                .build();
            
            PaymentResult result = paymentService.processPayment(request);
            
            // Publicar resultado
            if (result.isSuccess()) {
                eventPublisher.publishEvent(new PaymentCompletedEvent(event.getOrderId()));
            } else {
                eventPublisher.publishEvent(new PaymentFailedEvent(event.getOrderId()));
            }
            
        } catch (Exception e) {
            log.error("Error processing payment for order: {}", event.getOrderId(), e);
            eventPublisher.publishEvent(new PaymentFailedEvent(event.getOrderId()));
        }
    }
}
```

### **💡 Caso de Uso Prático: Sistema Bancário**

**Problema:** Banco com 50 milhões de clientes, 100 desenvolvedores, precisa de alta disponibilidade e compliance rigoroso.

**Solução Microserviços:**
```java
// Account Service
@RestController
@RequestMapping("/accounts")
public class AccountController {
    
    private final AccountService accountService;
    
    @GetMapping("/{accountId}/balance")
    @PreAuthorize("hasRole('CUSTOMER') and @accountService.isOwner(#accountId, authentication.name)")
    public ResponseEntity<BalanceResponse> getBalance(@PathVariable String accountId) {
        BigDecimal balance = accountService.getBalance(accountId);
        return ResponseEntity.ok(new BalanceResponse(balance));
    }
}

// Transaction Service
@RestController
@RequestMapping("/transactions")
public class TransactionController {
    
    private final TransactionService transactionService;
    
    @PostMapping("/transfer")
    public ResponseEntity<TransactionResult> transfer(@RequestBody TransferRequest request) {
        TransactionResult result = transactionService.processTransfer(request);
        return ResponseEntity.ok(result);
    }
}

// Notification Service
@Service
public class NotificationService {
    
    @EventListener
    public void handleTransactionCompleted(TransactionCompletedEvent event) {
        // Enviar notificação push
        pushNotificationService.send(event.getCustomerId(), 
            "Transferência realizada com sucesso: R$ " + event.getAmount());
        
        // Enviar email
        emailService.sendTransactionConfirmation(event);
        
        // Enviar SMS se valor alto
        if (event.getAmount().compareTo(BigDecimal.valueOf(1000)) > 0) {
            smsService.sendAlert(event.getCustomerId(), event);
        }
    }
}
```

**Vantagens desta abordagem:**
- 🏗️ **Scalability**: Escalar serviços independentemente
- 🔧 **Technology Diversity**: Diferentes tecnologias por serviço
- 🚀 **Independent Deployment**: Deploy sem afetar outros serviços
- 🏢 **Team Autonomy**: Equipes independentes

**Quando usar:** Empresas grandes, domínios complexos, equipes grandes (> 20 pessoas), alta escala.

---

## 🎯 **Conceito 3: Domain-Driven Design (DDD)**

### **🔍 O que é?**
DDD é como organizar uma biblioteca: você agrupa livros por assunto (domínio), cria seções (bounded contexts) e define regras de como cada seção funciona.

### **🎓 Conceitos Avançados:**

**1. Bounded Context**
```java
// Contexto de Vendas
@Entity
@Table(name = "sales_customer")
public class Customer {
    @Id
    private String id;
    private String name;
    private String email;
    private CustomerType type; // INDIVIDUAL, CORPORATE
    private BigDecimal creditLimit;
    
    // Comportamentos específicos do contexto de vendas
    public boolean canPurchase(BigDecimal amount) {
        return creditLimit.compareTo(amount) >= 0;
    }
}

// Contexto de Suporte
@Entity
@Table(name = "support_customer")
public class Customer {
    @Id
    private String id;
    private String name;
    private String email;
    private SupportLevel level; // BASIC, PREMIUM, ENTERPRISE
    private List<Ticket> tickets;
    
    // Comportamentos específicos do contexto de suporte
    public boolean canCreateTicket() {
        return level != SupportLevel.BASIC || tickets.size() < 3;
    }
}
```

**2. Aggregate Root**
```java
@Entity
public class Order {
    @Id
    private OrderId id;
    private CustomerId customerId;
    private OrderStatus status;
    private Money total;
    
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();
    
    // Aggregate Root - controla acesso aos items
    public void addItem(ProductId productId, int quantity, Money unitPrice) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        
        OrderItem item = new OrderItem(this, productId, quantity, unitPrice);
        items.add(item);
        recalculateTotal();
    }
    
    public void confirm() {
        if (items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }
        
        this.status = OrderStatus.CONFIRMED;
        
        // Publicar evento de domínio
        DomainEvents.publish(new OrderConfirmedEvent(this.id, this.customerId, this.total));
    }
    
    private void recalculateTotal() {
        this.total = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

**3. Domain Events**
```java
// Evento de Domínio
public class OrderConfirmedEvent {
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money total;
    private final Instant occurredAt;
    
    public OrderConfirmedEvent(OrderId orderId, CustomerId customerId, Money total) {
        this.orderId = orderId;
        this.customerId = customerId;
        this.total = total;
        this.occurredAt = Instant.now();
    }
    
    // Getters...
}

// Event Handler
@Component
public class OrderEventHandler {
    
    @EventHandler
    public void handleOrderConfirmed(OrderConfirmedEvent event) {
        // Lógica de negócio que deve acontecer quando pedido é confirmado
        inventoryService.reserveItems(event.getOrderId());
        emailService.sendOrderConfirmation(event.getCustomerId(), event.getOrderId());
        loyaltyService.addPoints(event.getCustomerId(), event.getTotal());
    }
}
```

### **💡 Caso de Uso Prático: Sistema de Reservas de Hotel**

**Problema:** Hotel chain com múltiplos domínios: reservas, housekeeping, billing, loyalty program.

**Solução DDD:**
```java
// Bounded Context: Reservations
@Entity
public class Reservation {
    @EmbeddedId
    private ReservationId id;
    
    @Embedded
    private Guest guest;
    
    @Embedded
    private Room room;
    
    @Embedded
    private DateRange stayPeriod;
    
    private ReservationStatus status;
    private Money totalAmount;
    
    // Regras de negócio específicas do domínio
    public void checkIn() {
        if (status != ReservationStatus.CONFIRMED) {
            throw new IllegalStateException("Only confirmed reservations can check in");
        }
        
        if (!stayPeriod.includesDate(LocalDate.now())) {
            throw new IllegalStateException("Check-in date is not within reservation period");
        }
        
        this.status = ReservationStatus.CHECKED_IN;
        
        // Evento de domínio
        DomainEvents.publish(new GuestCheckedInEvent(id, guest.getId(), room.getId()));
    }
    
    public void cancel() {
        if (status == ReservationStatus.CHECKED_IN) {
            throw new IllegalStateException("Cannot cancel active reservation");
        }
        
        LocalDate now = LocalDate.now();
        if (stayPeriod.getStartDate().minusDays(1).isBefore(now)) {
            // Política de cancelamento
            Money penalty = totalAmount.multiply(0.1); // 10% penalty
            this.totalAmount = this.totalAmount.subtract(penalty);
        }
        
        this.status = ReservationStatus.CANCELLED;
        
        DomainEvents.publish(new ReservationCancelledEvent(id, guest.getId(), room.getId()));
    }
}

// Value Object
@Embeddable
public class DateRange {
    private LocalDate startDate;
    private LocalDate endDate;
    
    public DateRange(LocalDate startDate, LocalDate endDate) {
        if (startDate.isAfter(endDate)) {
            throw new IllegalArgumentException("Start date must be before end date");
        }
        this.startDate = startDate;
        this.endDate = endDate;
    }
    
    public boolean includesDate(LocalDate date) {
        return !date.isBefore(startDate) && !date.isAfter(endDate);
    }
    
    public int getDurationInDays() {
        return (int) ChronoUnit.DAYS.between(startDate, endDate);
    }
}
```

---

## 🎯 **Conceito 4: Padrão CQRS (Command Query Responsibility Segregation)**

### **🔍 O que é?**
CQRS é como ter duas bibliotecas: uma para escrever novos livros (Command) e outra otimizada para leitura rápida (Query). Cada uma tem sua organização específica.

### **🎓 Conceitos Avançados:**

**1. Command Side (Escrita)**
```java
// Command
public class CreateOrderCommand {
    private final CustomerId customerId;
    private final List<OrderItemData> items;
    private final ShippingAddress shippingAddress;
    
    // Constructor, getters...
}

// Command Handler
@Component
public class CreateOrderCommandHandler {
    
    private final OrderRepository orderRepository;
    private final EventStore eventStore;
    
    public OrderId handle(CreateOrderCommand command) {
        // Validações
        validateCommand(command);
        
        // Criar aggregate
        Order order = new Order(
            OrderId.generate(),
            command.getCustomerId(),
            command.getItems(),
            command.getShippingAddress()
        );
        
        // Salvar eventos
        List<DomainEvent> events = order.getUncommittedEvents();
        eventStore.saveEvents(order.getId(), events);
        
        // Salvar snapshot para performance
        orderRepository.save(order);
        
        return order.getId();
    }
}
```

**2. Query Side (Leitura)**
```java
// Read Model
@Entity
@Table(name = "order_view")
public class OrderView {
    @Id
    private String id;
    private String customerId;
    private String customerName;
    private String customerEmail;
    private String status;
    private BigDecimal total;
    private LocalDateTime createdAt;
    private int itemCount;
    private String shippingAddress;
    
    // Dados desnormalizados para performance de leitura
}

// Query Handler
@Component
public class OrderQueryHandler {
    
    private final OrderViewRepository orderViewRepository;
    
    public List<OrderView> getOrdersByCustomer(CustomerId customerId) {
        return orderViewRepository.findByCustomerIdOrderByCreatedAtDesc(customerId.getValue());
    }
    
    public OrderView getOrderById(OrderId orderId) {
        return orderViewRepository.findById(orderId.getValue())
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}
```

**3. Event-Driven Synchronization**
```java
// Event Handler para sincronizar Read Model
@Component
public class OrderViewEventHandler {
    
    private final OrderViewRepository orderViewRepository;
    private final CustomerService customerService;
    
    @EventHandler
    public void handleOrderCreated(OrderCreatedEvent event) {
        Customer customer = customerService.getCustomer(event.getCustomerId());
        
        OrderView orderView = new OrderView();
        orderView.setId(event.getOrderId().getValue());
        orderView.setCustomerId(event.getCustomerId().getValue());
        orderView.setCustomerName(customer.getName());
        orderView.setCustomerEmail(customer.getEmail());
        orderView.setStatus(event.getStatus().name());
        orderView.setTotal(event.getTotal().getAmount());
        orderView.setCreatedAt(event.getOccurredAt());
        orderView.setItemCount(event.getItems().size());
        orderView.setShippingAddress(event.getShippingAddress().getFullAddress());
        
        orderViewRepository.save(orderView);
    }
    
    @EventHandler
    public void handleOrderStatusChanged(OrderStatusChangedEvent event) {
        OrderView orderView = orderViewRepository.findById(event.getOrderId().getValue());
        if (orderView != null) {
            orderView.setStatus(event.getNewStatus().name());
            orderViewRepository.save(orderView);
        }
    }
}
```

### **💡 Caso de Uso Prático: Sistema de E-commerce com Alta Performance**

**Problema:** E-commerce com milhões de produtos, necessidade de buscas complexas e relatórios em tempo real.

**Solução CQRS:**
```java
// Command Side - Otimizado para escrita
@RestController
@RequestMapping("/commands")
public class ProductCommandController {
    
    private final ProductCommandHandler commandHandler;
    
    @PostMapping("/products")
    public ResponseEntity<ProductId> createProduct(@RequestBody CreateProductCommand command) {
        ProductId productId = commandHandler.handle(command);
        return ResponseEntity.ok(productId);
    }
    
    @PutMapping("/products/{id}/price")
    public ResponseEntity<Void> updatePrice(@PathVariable String id, @RequestBody UpdatePriceCommand command) {
        command.setProductId(new ProductId(id));
        commandHandler.handle(command);
        return ResponseEntity.ok().build();
    }
}

// Query Side - Otimizado para leitura
@RestController
@RequestMapping("/queries")
public class ProductQueryController {
    
    private final ProductQueryHandler queryHandler;
    
    @GetMapping("/products/search")
    public ResponseEntity<List<ProductView>> searchProducts(
            @RequestParam String query,
            @RequestParam(required = false) String category,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        ProductSearchQuery searchQuery = ProductSearchQuery.builder()
            .query(query)
            .category(category)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .page(page)
            .size(size)
            .build();
        
        List<ProductView> products = queryHandler.searchProducts(searchQuery);
        return ResponseEntity.ok(products);
    }
}
```

**Vantagens desta abordagem:**
- 🚀 **Performance**: Leitura e escrita otimizadas independentemente
- 📊 **Scalability**: Scaling diferente para reads vs writes
- 🔍 **Complex Queries**: Read models específicos para cada necessidade
- 📈 **Analytics**: Dados desnormalizados para relatórios

---

## 🎯 **Exercícios Práticos**

### **Exercício 1: Escolha de Arquitetura**
**Cenário:** Startup de delivery de comida, 3 desenvolvedores, 6 meses para MVP.

**Sua tarefa:** Justifique a escolha arquitetural e desenhe a solução.

**Pontos a considerar:**
- Tamanho da equipe
- Tempo para entrega
- Complexidade do domínio
- Escalabilidade futura

### **Exercício 2: Migration Strategy**
**Cenário:** Sistema monolítico de RH com 200k usuários precisa ser migrado para microserviços.

**Sua tarefa:** Defina uma estratégia de migração gradual.

**Pontos a considerar:**
- Identificação de bounded contexts
- Estratégia de dados
- Rollback plans
- Timeline de migração

### **Exercício 3: DDD Modeling**
**Cenário:** Sistema de gestão hospitalar (pacientes, médicos, consultas, internações).

**Sua tarefa:** Modele usando DDD com aggregates, value objects e domain events.

---

## 🎯 **Perguntas de Entrevista Esperadas**

### **1. "Explique quando você usaria microserviços vs monolito"**
**Resposta estruturada:**
- Contexto da empresa (tamanho, maturidade)
- Complexidade do domínio
- Requisitos de escalabilidade
- Capacidade da equipe
- Trade-offs de cada abordagem

### **2. "Como você garantiria consistência entre microserviços?"**
**Resposta estruturada:**
- Saga Pattern para transações distribuídas
- Event Sourcing para auditoria
- Eventual consistency vs Strong consistency
- Compensating actions

### **3. "Desenhe uma arquitetura para um sistema de pagamentos"**
**Resposta estruturada:**
- Bounded contexts (Account, Payment, Fraud, etc.)
- Segregação de responsabilidades
- Patterns de segurança
- Compliance e auditoria

---

## 🚀 **Próximos Passos**

1. **Pratique os exemplos** - Implemente cada padrão
2. **Estude o próximo módulo** - 02-Java-Spring.md
3. **Faça os exercícios** - Simule cenários reais
4. **Desenhe arquiteturas** - Pratique no papel

**Lembre-se:** Arquitetura é sobre trade-offs. Sempre considere o contexto! 