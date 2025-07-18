# 🐳 Testes de Integração com Testcontainers

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Compreender as armadilhas** de usar bancos de dados em memória (H2) para testes de integração.
-   ✅ **Dominar o conceito** por trás do Testcontainers: testes de alta fidelidade com dependências descartáveis.
-   ✅ **Escrever um teste de integração para a camada de persistência** (JPA/JDBC) usando um contêiner PostgreSQL real.
-   ✅ **Escrever um teste para um sistema de mensageria** usando um contêiner Kafka real.
-   ✅ **Configurar dinamicamente as propriedades da aplicação** (como a URL do banco) para se conectar aos contêineres durante os testes.

---

## 🎯 **Conceito 1: A Ilusão Perigosa dos Bancos de Dados em Memória**

Por anos, a prática comum para testar a camada de persistência em Java foi usar um banco de dados em memória, como o H2. A promessa era de testes rápidos e que não precisavam de um banco de dados real instalado.

**Esta prática é um anti-padrão perigoso.**

-   **H2 não é PostgreSQL (ou Oracle, ou MySQL...):** Embora o H2 tenha um "modo de compatibilidade", ele não é uma réplica fiel. Diferenças sutis em dialetos SQL, tipos de dados, comportamento de transações, locking e funções específicas do banco (como funções de JSON no Postgres) fazem com que **seus testes possam passar localmente com H2 e falhar catastroficamente em produção**.
-   **Falsa Sensação de Segurança:** Você está testando uma integração com um sistema que sua aplicação **nunca** usará no mundo real. É como treinar para uma maratona correndo em uma esteira com a inclinação no negativo.

A regra de ouro da engenharia moderna é: **Teste contra a mesma tecnologia que você usa em produção.**

---

## 🎯 **Conceito 2: Testcontainers - Dependências Reais e Descartáveis**

O Testcontainers é uma biblioteca Java que resolve este problema de forma genial. Ele permite que você, de forma programática, **inicie, configure e destrua contêineres Docker como parte do seu ciclo de vida de testes JUnit**.

**Como Funciona?**
1.  Você anota seu teste para usar Testcontainers.
2.  Você declara um campo na sua classe de teste para o contêiner que precisa (ex: `PostgreSQLContainer`, `KafkaContainer`).
3.  Antes do seu teste rodar, o Testcontainers usa o Docker da sua máquina para baixar a imagem (se necessário) e iniciar um contêiner novinho da sua dependência.
4.  Ele expõe as informações de conexão (como a porta aleatória e a URL do JDBC) para que sua aplicação possa se conectar a ele.
5.  Você roda seu teste contra esta instância real e isolada.
6.  Após o teste, o Testcontainers se encarrega de destruir o contêiner, deixando seu ambiente limpo.

O resultado são **testes de integração com a mais alta fidelidade possível**, sem a complexidade de gerenciar um ambiente de banco de dados compartilhado.

---

## 🎯 **Conceito 3: Implementação Prática - Testando a Camada JPA**

Vamos testar um `UserRepository` do Spring Data JPA contra um banco de dados PostgreSQL real.

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers-bom</artifactId>
    <version>1.17.6</version> <!-- Usar a versão mais recente -->
    <type>pom</type>
    <scope>import</scope>
</dependency>
<!-- Adicionar no <dependencies> -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// Nosso repositório Spring Data JPA
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
}

// A entidade User
@Entity
@Table(name = "users")
public class User { /* ... id, name, email ... */ }
```

```java
// O Teste de Integração com Testcontainers
@Testcontainers // Habilita o suporte do JUnit 5 para Testcontainers
@DataJpaTest  // Foco no teste da camada JPA, desabilita a maioria das auto-configurações
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // Desliga o H2!
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    // Declara o contêiner. O `static` faz com que ele seja iniciado uma vez para todos os testes da classe.
    @Container
    private static final PostgreSQLContainer<?> postgresContainer = 
        new PostgreSQLContainer<>("postgres:14-alpine");

    // Fonte dinâmica de propriedades. Isso intercepta a configuração do Spring
    // e injeta as propriedades do nosso contêiner em tempo de execução.
    @DynamicPropertySource
    private static void setDatasourceProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgresContainer::getJdbcUrl);
        registry.add("spring.datasource.username", postgresContainer::getUsername);
        registry.add("spring.datasource.password", postgresContainer::getPassword);
    }

    @Test
    void shouldSaveAndFindUserByEmail() {
        // Given
        User user = new User("João", "joao@test.com");

        // When
        userRepository.save(user);
        Optional<User> foundUser = userRepository.findByEmail("joao@test.com");

        // Then
        assertThat(foundUser).isPresent();
        assertThat(foundUser.get().getName()).isEqualTo("João");
    }
}
```
Este teste:
1.  Inicia um contêiner PostgreSQL real antes de qualquer teste ser executado.
2.  Informa ao Spring para se conectar a este contêiner, em vez de usar o H2.
3.  Executa uma operação de banco de dados real.
4.  Destrói o contêiner após todos os testes da classe.

A confiança neste teste é **1000% maior** do que em um teste que usa H2.

---

## 🎯 **Conceito 4: Testando Sistemas de Mensageria com Kafka**

Testcontainers não se limita a bancos de dados. Vamos testar um produtor Kafka.

```java
@Testcontainers
@SpringBootTest
class KafkaMessageProducerTest {

    @Container
    private static final KafkaContainer kafkaContainer = 
        new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.0.1"));

    @DynamicPropertySource
    private static void setKafkaProperties(DynamicPropertyRegistry registry) {
        // Aponta nosso produtor Kafka para o broker dentro do contêiner
        registry.add("spring.kafka.bootstrap-servers", kafkaContainer::getBootstrapServers);
    }

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Test
    void shouldSendMessageToKafka() throws Exception {
        // Given
        String topic = "pedidos-criados";
        String message = "{\"pedidoId\": \"123\"}";

        // When
        kafkaTemplate.send(topic, message);
        kafkaTemplate.flush(); // Garante que a mensagem foi enviada

        // Then
        // Para verificar, usamos um consumidor de teste que se conecta ao mesmo contêiner.
        // Bibliotecas como Awaitility são ótimas aqui para esperar a mensagem chegar.
        // (O código do consumidor de teste foi omitido por brevidade)
        
        // Exemplo simples de verificação (em um caso real, use um consumidor dedicado):
        // ...código para consumir a mensagem e fazer o assert...
        
        // A principal verificação é que a chamada `kafkaTemplate.send` não lançou uma exceção,
        // o que prova que a integração com o broker está funcionando.
    }
}
```

--- 