# ⚡ Otimização de Performance em Aplicações Java

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Entender os pilares da performance** de uma aplicação Java: Latência, Throughput e Utilização de Recursos.
-   ✅ **Dominar os conceitos essenciais da JVM:** Heap, Stack, Garbage Collection (GC) e o JIT Compiler.
-   ✅ **Realizar profiling de CPU e Memória** em uma aplicação para encontrar gargalos usando ferramentas como VisualVM e o profiler do IntelliJ.
-   ✅ **Aplicar técnicas de otimização de código** para evitar problemas comuns como N+1 selects, uso excessivo de memória e concorrência ineficiente.
-   ✅ **Aprender a fazer tuning básico da JVM** através de flags para otimizar o Garbage Collector e o tamanho do Heap.

---

## 🎯 **Conceito 1: Os 3 Pilares da Performance**

Antes de otimizar, precisamos saber o que medir. A performance de uma aplicação se resume a um balanço entre três fatores:

1.  **Latência (Latency):**
    -   **O que é:** O tempo que uma única operação leva para ser concluída (ex: o tempo para uma requisição API retornar).
    -   **Métrica:** Geralmente medida em milissegundos (ms).
    -   **Foco:** Crucial para a experiência do usuário em sistemas interativos.

2.  **Throughput (Vazão):**
    -   **O que é:** O número de operações que o sistema consegue executar em um determinado período (ex: requisições por segundo).
    -   **Métrica:** Operações/segundo, transações/minuto.
    -   **Foco:** Essencial para sistemas de processamento em lote (batch) e de alto volume.

3.  **Utilização de Recursos (Resource Utilization):**
    -   **O que é:** O quanto de CPU, memória (RAM), I/O de disco e rede sua aplicação consome para atingir uma determinada latência e throughput.
    -   **Métrica:** %, MB/s, IOPS.
    -   **Foco:** Diretamente ligado ao **custo** da infraestrutura. Otimizar a utilização de recursos significa gastar menos.

**O ETERNO TRADE-OFF:** Geralmente, otimizar um pilar tem um custo nos outros. Aumentar o throughput pode aumentar a latência. Reduzir a latência pode exigir mais CPU e memória. A meta de um especialista é encontrar o **ponto de equilíbrio ideal** para os requisitos do negócio.

---

## 🎯 **Conceito 2: Desmistificando a JVM - Onde a Magia Acontece**

Para otimizar uma aplicação Java, você precisa entender minimamente o que acontece por baixo dos panos.

-   **Heap:** A principal área de memória da JVM, onde todos os **objetos** são alocados. É o foco da maioria das otimizações de memória. O Heap é gerenciado pelo **Garbage Collector**.
-   **Stack:** Cada thread de execução tem sua própria stack. Ela armazena **variáveis locais e chamadas de método**. É muito mais rápida que o Heap, mas limitada em tamanho.
-   **Garbage Collector (GC):** O processo automático que "limpa" o Heap, removendo objetos que não são mais referenciados por nenhuma parte da aplicação, liberando memória. Um GC ineficiente pode causar **pausas na aplicação ("Stop-the-World")**, impactando diretamente a latência.
-   **Just-In-Time (JIT) Compiler:** A JVM não interpreta o bytecode Java o tempo todo. O JIT Compiler identifica o "código quente" (métodos que são executados com frequência) e os compila para código de máquina nativo em tempo de execução, tornando-os muito mais rápidos.

**A Regra 80/20 da Performance Java:** A maioria dos problemas de performance em aplicações de negócio está relacionada a duas áreas:
1.  **Alocação excessiva de memória**, que sobrecarrega o Garbage Collector.
2.  **Operações de I/O (Entrada/Saída) lentas**, principalmente chamadas a banco de dados e APIs externas.

---

## 🎯 **Conceito 3: Profiling - Encontrando o Código Lento**

**Primeira regra da otimização: Não adivinhe. Meça!**

Um **profiler** é uma ferramenta que se "acopla" à sua aplicação em execução e coleta dados detalhados sobre onde o tempo de CPU está sendo gasto e onde a memória está sendo alocada.

**Ferramentas Comuns:**
-   **VisualVM:** Ferramenta gratuita e poderosa que já vem com o JDK.
-   **Profilers de IDEs:** IntelliJ Ultimate e Eclipse têm profilers excelentes e integrados.
-   **Profilers Comerciais/APM:** Datadog, New Relic, Dynatrace (oferecem profiling contínuo em produção).

**Fluxo de Trabalho de Profiling de CPU:**
1.  **Inicie a aplicação localmente.**
2.  **Conecte o profiler** (ex: VisualVM) ao processo Java.
3.  **Execute a carga de trabalho** que você quer analisar (ex: dispare requisições para o endpoint lento com uma ferramenta como `k6` ou `JMeter`).
4.  **Capture um "snapshot" de CPU.**
5.  **Analise os resultados:** O profiler mostrará uma lista de métodos ordenada pelo tempo gasto. Procure pelos seus próprios métodos no topo da lista. Isso revelará os "hotspots" do seu código.

**Exemplo de Análise (VisualVM):**
Imagine que o profiler mostra que 70% do tempo é gasto no método `formatarRelatorio()`. Ao inspecioná-lo, você descobre um loop que está concatenando Strings de forma ineficiente (`String a = a + b;`). A correção é usar um `StringBuilder`. O profiling te levou diretamente à causa raiz.

**Fluxo de Trabalho de Profiling de Memória:**
1.  Siga os mesmos passos, mas capture um **"heap dump"**.
2.  **Analise o heap dump:** A ferramenta mostrará quais tipos de objetos estão ocupando mais espaço na memória. Isso pode revelar memory leaks (objetos que nunca são liberados pelo GC) ou estruturas de dados ineficientes que consomem memória desnecessariamente.

---

## 🎯 **Conceito 4: Padrões de Otimização e Anti-Padrões Comuns**

### **1. O Problema N+1 Selects (I/O)**
O anti-padrão mais comum em aplicações com ORM (JPA/Hibernate).

**Cenário:** Você busca 100 pedidos, e para cada pedido, o sistema faz uma nova query para buscar os dados do cliente. Total: 1 (pedidos) + 100 (clientes) = **101 queries**.

```java
// Código problemático
List<Pedido> pedidos = pedidoRepository.findAll();
for (Pedido pedido : pedidos) {
    // Causa uma nova query para cada pedido!
    Cliente cliente = pedido.getCliente(); 
    System.out.println(cliente.getNome());
}
```
**Solução: Eager Fetching com JOIN FETCH ou Entity Graphs.**
Instrua o ORM a trazer todos os dados necessários em uma única query.

```java
// Solução com JPQL
@Query("SELECT p FROM Pedido p JOIN FETCH p.cliente")
List<Pedido> findAllComClientes();
```
Agora, apenas **1 query** é executada. A diferença de performance é brutal.

### **2. Alocação Excessiva de Memória (GC)**
Evite criar objetos desnecessários, especialmente dentro de loops.

**Cenário:** Processar um arquivo grande, lendo-o inteiro para a memória.
```java
// Código problemático
byte[] bytes = Files.readAllBytes(Paths.get("arquivo_grande.csv"));
// Se o arquivo tiver 2GB, seu heap precisa ter 2GB livres.
```
**Solução: Streaming.**
Processe os dados em pedaços, sem carregar tudo de uma vez.
```java
// Solução
try (Stream<String> lines = Files.lines(Paths.get("arquivo_grande.csv"))) {
    lines.forEach(line -> {
        // Processa uma linha de cada vez, com uso mínimo de memória.
    });
}
```

### **3. Tuning Básico da JVM**
Você pode passar flags para a JVM na inicialização para ajustar seu comportamento.

**Flags Essenciais:**
-   `-Xms<tamanho>` e `-Xmx<tamanho>`: Definem o tamanho inicial e máximo do Heap. **A melhor prática em servidores é definir os dois com o mesmo valor** (ex: `-Xms2g -Xmx2g`). Isso evita que a JVM perca tempo redimensionando o Heap.
-   `-XX:+UseG1GC`: A partir do Java 9+, o G1 (Garbage-First) é o GC padrão. Ele oferece um bom balanço entre latência e throughput e raramente precisa de tuning. Para Java 8, é uma ótima escolha para substituir o ParallelGC.
-   `-XX:+HeapDumpOnOutOfMemoryError`: **ESSENCIAL EM PRODUÇÃO.** Gera um heap dump automaticamente se a aplicação quebrar por falta de memória, permitindo a análise post-mortem.

**Exemplo de linha de comando para um serviço em produção:**
```bash
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -jar minha-app.jar
```
Este é um ponto de partida sólido e conservador para a maioria das aplicações web.

--- 