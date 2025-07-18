# ✒️ Testes de Contrato com Pact

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Entender o problema** que os testes de contrato resolvem: a quebra de comunicação entre serviços.
-   ✅ **Dominar o fluxo do Teste de Contrato Orientado ao Consumidor** (Consumer-Driven Contract Testing).
-   ✅ **Escrever um teste de consumidor** em Java usando Pact-JVM que gera um "contrato" (Pact file).
-   ✅ **Verificar o contrato** no lado do provedor em Java, garantindo que o provedor satisfaz as expectativas do consumidor.
-   ✅ **Compreender o papel do Pact Broker** na automação e escalabilidade dos testes de contrato.

---

## 🎯 **Conceito 1: O Problema - A Falha Silenciosa da Integração**

Imagine um cenário comum:

-   **Time A** desenvolve o `servico-de-usuarios`.
-   **Time B** desenvolve o `servico-de-pedidos`, que consome a API do `servico-de-usuarios` para obter dados do cliente.

O Time A, em uma refatoração, decide renomear o campo `emailAddress` para `email` na resposta da API. Eles rodam todos os seus testes unitários e de integração, e tudo passa. Eles fazem o deploy.

**O resultado? O `servico-de-pedidos` quebra silenciosamente em produção.** Ele esperava `emailAddress` e agora recebe `email`, causando um erro de desserialização.

Este é o problema central que os Testes de Contrato resolvem. Eles criam um "acordo" formal e testável entre um consumidor e um provedor, garantindo que qualquer alteração que quebre esse acordo seja detectada **antes do deploy**.

---

## 🎯 **Conceito 2: O Fluxo do Consumer-Driven Contract Testing com Pact**

O Pact implementa uma abordagem chamada **Consumer-Driven Contract Testing**, o que significa que **o consumidor dita as regras**. O fluxo funciona em três etapas principais:

**Etapa 1: Lado do Consumidor (Consumer)**
1.  O consumidor escreve um teste unitário que simula uma requisição para a API do provedor.
2.  Nesse teste, ele define **exatamente** como a requisição deve ser (método, path, headers) e como a resposta esperada deve ser (status code, headers, corpo JSON).
3.  Quando o teste roda, em vez de fazer uma chamada HTTP real, o Pact cria um **servidor mock (falso)** que retorna a resposta esperada.
4.  Se o código do consumidor processar essa resposta mock corretamente, o teste passa e o Pact gera um arquivo JSON chamado **Pact File**. Este arquivo é o **contrato**.

**Etapa 2: Compartilhamento do Contrato**
5.  O Pact File gerado pelo consumidor é compartilhado com o provedor. A melhor forma de fazer isso é usando uma ferramenta central chamada **Pact Broker**.

**Etapa 3: Lado do Provedor (Provider)**
6.  O provedor pega o contrato (o Pact File) do consumidor.
7.  O Pact então executa uma tarefa de verificação:
    a. Ele inicia a aplicação real do provedor.
    b. Ele **"repete" (replays)** a requisição definida no contrato contra a aplicação real.
    c. Ele compara a resposta **real** da aplicação com a resposta esperada definida no contrato.
8.  Se a resposta real corresponder à resposta esperada no contrato, a verificação passa. Isso prova que o provedor não quebrou as expectativas do seu consumidor.

Se o Time A tivesse renomeado o campo para `email`, a verificação do provedor falharia, bloqueando o deploy e prevenindo a quebra em produção.

---

## 🎯 **Conceito 3: Implementação Prática com Java e Pact-JVM**

Vamos implementar o cenário do `servico-de-pedidos` (Consumer) e `servico-de-usuarios` (Provider).

### **1. Teste do Lado do Consumidor (`servico-de-pedidos`)**

```xml
<!-- pom.xml do Consumidor -->
<dependency>
    <groupId>au.com.dius.pact.consumer</groupId>
    <artifactId>junit5</artifactId>
    <version>4.3.5</version> <!-- Usar a versão mais recente -->
    <scope>test</scope>
</dependency>
```

```java
// Código do nosso cliente HTTP no serviço de pedidos
@Component
public class UserApiClient {
    private final RestTemplate restTemplate;

    public UserApiClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    // O endpoint base do serviço de usuários será injetado pelo Pact no teste
    public User getUserById(String id, String baseUrl) {
        return restTemplate.getForObject(baseUrl + "/users/" + id, User.class);
    }
}

// Classe de modelo
public class User {
    private String id;
    private String name;
    private String emailAddress; // O consumidor espera este campo
    // getters e setters
}
```

```java
// O Teste de Contrato do Consumidor
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "servico-de-usuarios") // Nome do nosso Provedor
public class UserApiClientContractTest {

    @Pact(consumer = "servico-de-pedidos") // Nome do nosso Consumidor
    public RequestResponsePact createPact(PactDslWithProvider builder) {
        // Define as expectativas do consumidor
        return builder
            .given("um usuário com id 10 existe") // O estado que o provedor precisa ter
            .uponReceiving("uma requisição para o usuário 10")
                .method("GET")
                .path("/users/10")
            .willRespondWith()
                .status(200)
                .header("Content-Type", "application/json")
                .body(new PactDslJsonBody()
                    .stringType("id", "10")
                    .stringType("name", "João da Silva")
                    .stringType("emailAddress", "joao.silva@example.com") // Define o campo esperado
                )
            .toPact(); // Constrói o contrato
    }

    @Test
    void testGetUserById(MockServer mockServer) {
        // O Pact inicia um servidor mock na porta mockServer.getPort()
        // e o configura para responder conforme definido no contrato.

        UserApiClient userApiClient = new UserApiClient(new RestTemplate());
        User user = userApiClient.getUserById("10", mockServer.getUrl());

        // Verificamos se nosso cliente consegue processar a resposta do mock
        assertEquals("João da Silva", user.getName());
        assertEquals("joao.silva@example.com", user.getEmailAddress());
    }
}
```
Ao rodar este teste, um arquivo chamado `servico-de-pedidos-servico-de-usuarios.json` será gerado na pasta `target/pacts`. Este é o nosso contrato!

### **2. Verificação do Lado do Provedor (`servico-de-usuarios`)**

```xml
<!-- pom.xml do Provedor -->
<dependency>
    <groupId>au.com.dius.pact.provider</groupId>
    <artifactId>junit5spring</artifactId>
    <version>4.3.5</version>
    <scope>test</scope>
</dependency>
```

```java
// O Controller do nosso serviço de usuários (o Provedor)
@RestController
public class UserController {
    // ...
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUserById(@PathVariable String id) {
        // Digamos que aqui a implementação real retorne um campo "email"
        User user = new User(id, "João da Silva", "joao.silva@example.com"); 
        return ResponseEntity.ok(user);
    }
}

// Classe de modelo do Provedor
public class User {
    private String id;
    private String name;
    private String email; // O provedor mudou o nome do campo!
    // construtores, getters e setters
}
```

```java
// O Teste de Verificação do Provedor
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("servico-de-usuarios") // Nome do Provedor (deve ser igual ao do teste do consumidor)
@PactBroker // Ou @PactFolder("target/pacts") para ler de uma pasta local
public class UserApiProviderContractTest {

    @LocalServerPort
    private int port;

    @BeforeEach
    void setup(PactVerificationContext context) {
        // Informa ao Pact qual a URL base da nossa API real
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void pactVerificationTestTemplate(PactVerificationContext context) {
        // Este template executa a verificação
        context.verifyInteraction();
    }

    // O Pact precisa saber como colocar o provedor no estado correto ("given")
    @State("um usuário com id 10 existe")
    public void user10Exists() {
        // Como este é um exemplo simples, não precisamos fazer nada.
        // Em um caso real, você poderia usar este método para inserir
        // dados no seu banco de dados de teste antes da verificação.
        System.out.println("Setup do estado: usuário 10 existe.");
    }
}
```
Quando o teste de verificação do provedor for executado, ele vai **falhar**. O Pact vai gerar um relatório de erro claro, mostrando a diferença:
```
Failures:

1) Verifying a pact between servico-de-pedidos and servico-de-usuarios
  Given um usuário com id 10 existe
  uma requisição para o usuário 10
    returns a response which
      has a matching body
        $.body.emailAddress -> Expected 'joao.silva@example.com' but was missing
```
O erro aponta exatamente a quebra de contrato, forçando o Time A a corrigir a API ou a negociar uma nova versão do contrato com o Time B.

---

## 🎯 **Conceito 4: Escalando com o Pact Broker**

Publicar e compartilhar arquivos JSON manualmente não é escalável. O **Pact Broker** é uma aplicação web que atua como um repositório central para os contratos.

**Benefícios do Broker:**
-   **Repositório Central:** Publica e versiona todos os contratos.
-   **Descoberta Automática:** O provedor descobre automaticamente quais contratos precisa verificar.
-   **Tags e Versionamento:** Permite associar contratos a versões de código e ambientes (`main`, `feat-xyz`, `prod`).
-   **`can-i-deploy`:** A ferramenta mais poderosa. Antes de fazer deploy, você pode perguntar ao Broker: "Posso fazer o deploy da versão `1.5.0` do `servico-de-usuarios` em produção?". O Broker responderá "sim" apenas se essa versão tiver verificado com sucesso a versão mais recente de **todos** os seus consumidores que estão em produção. Isso é um **gateway de segurança para o seu pipeline de CI/CD**.

--- 