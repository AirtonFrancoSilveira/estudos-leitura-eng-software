# 🏗️ Padrões de Migração para a Nuvem

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Compreender os "6 R's" da migração para a nuvem** e saber quando aplicar cada estratégia.
-   ✅ **Dominar o Padrão Strangler Fig (Asfixiador)** como a principal abordagem para modernizar sistemas legados de forma incremental e segura.
-   ✅ **Projetar uma arquitetura de transição** que permita a coexistência do sistema legado e dos novos microserviso.
-   ✅ **Entender estratégias para a migração de bancos de dados monolíticos**, um dos maiores desafios da modernização.
-   ✅ **Utilizar serviços da AWS** como o Application Load Balancer (ALB) e o API Gateway para implementar o padrão Strangler Fig.

---

## 🎯 **Conceito 1: Os 6 R's - As Estratégias Fundamentais de Migração**

Quando confrontado com a tarefa de mover uma aplicação para a nuvem, a primeira decisão é **COMO** movê-la. O Gartner popularizou um framework conhecido como os "6 R's da Migração":

1.  **Rehosting (Lift and Shift):**
    -   **O que é:** Mover a aplicação do servidor on-premise para uma instância EC2 na AWS, sem nenhuma modificação.
    -   **Quando usar:** Para ganhos rápidos de migração, aplicações "caixa-preta" que não podem ser modificadas, ou como um primeiro passo antes de uma modernização futura.
    -   **Vantagem:** Rápido e de baixo esforço.
    -   **Desvantagem:** Não aproveita os benefícios da nuvem (elasticidade, serviços gerenciados) e pode ser caro.

2.  **Replatforming (Lift and Reshape):**
    -   **O que é:** Um "lift and shift" com algumas otimizações. Por exemplo, mover a aplicação para o EC2, mas migrar o banco de dados de um Oracle auto-gerenciado para o Amazon RDS.
    -   **Quando usar:** Quando você quer obter alguns benefícios da nuvem (como um DB gerenciado) sem reescrever a aplicação.
    -   **Vantagem:** Balanço entre esforço e benefício.
    -   **Desvantagem:** Ainda mantém a arquitetura monolítica original.

3.  **Repurchasing (Drop and Shop):**
    -   **O que é:** Descartar a aplicação legada e mover para um produto SaaS (Software as a Service). Ex: substituir um CRM interno antigo pelo Salesforce.
    -   **Quando usar:** Quando uma solução comercial atende às suas necessidades e o custo de manutenção do sistema legado é alto.

4.  **Refactoring / Rearchitecting (Re-arquitetar):**
    -   **O que é:** Repensar e reescrever completamente (ou em grande parte) a aplicação para ser **Cloud-Native**. É aqui que entram os microserviços, serverless e os padrões que temos estudado.
    -   **Quando usar:** Quando há uma forte necessidade de negócio para novas features, escalabilidade ou performance que o legado não consegue atender.
    -   **Vantagem:** Máximo aproveitamento dos benefícios da nuvem.
    -   **Desvantagem:** O mais caro, demorado e arriscado. **O Padrão Strangler Fig é a principal técnica para mitigar esse risco.**

5.  **Retiring (Aposentar):**
    -   **O que é:** Simplesmente desligar a aplicação, pois ela não é mais necessária.

6.  **Retaining (Reter):**
    -   **O que é:** Manter a aplicação on-premise. Isso pode ser necessário por motivos de compliance, latência ou porque o custo/benefício da migração não compensa.

---

## 🎯 **Conceito 2: O Padrão Strangler Fig (Asfixiador)**

Nomeado por Martin Fowler, este padrão se inspira em figueiras que "estrangulam" árvores hospedeiras. A ideia é construir seu novo sistema **ao redor** do sistema legado, redirecionando gradualmente a funcionalidade para os novos serviços até que o legado seja "estrangulado" e possa ser aposentado com segurança.

É a abordagem mais segura e recomendada para a modernização de sistemas críticos.

**O Fluxo do Strangler Fig:**

1.  **Identificar uma Funcionalidade:** Escolha um módulo ou funcionalidade do sistema legado para ser o primeiro a ser migrado. Idealmente, um que seja relativamente desacoplado e que traga valor rápido.

2.  **Introduzir uma Fachada (Façade):** Coloque um proxy reverso, como um **Application Load Balancer (ALB)** ou um **API Gateway**, na frente de **TODA** a aplicação legada. Inicialmente, ele apenas repassa 100% do tráfego para o legado.

3.  **Construir o Novo Serviço:** Desenvolva o novo microserviço que implementa a funcionalidade escolhida, usando tecnologias e arquiteturas modernas.

4.  **Redirecionar o Tráfego (O Passo Chave):** Modifique a configuração da Fachada (ALB/API Gateway) para que as chamadas para a funcionalidade específica sejam agora roteadas para o **novo microserviço**, enquanto todo o resto do tráfego continua indo para o legado.

5.  **Iterar:** Repita os passos 1 a 4 para a próxima funcionalidade. Com o tempo, mais e mais rotas são desviadas do legado para os novos serviços.

6.  **Aposentar:** Quando a última funcionalidade for migrada, o sistema legado não recebe mais nenhum tráfego. Ele foi completamente "estrangulado" e pode ser desligado com segurança.

![Strangler Fig Pattern](https://miro.medium.com/max/1400/1*dK-L-z9kY1Y_0q7Y_Q_yqg.png)

### **Implementando a Fachada com o AWS Application Load Balancer (ALB)**

O ALB é perfeito para este padrão, pois suporta **roteamento baseado em path e host**.

**Exemplo de Configuração de Listeners no ALB:**

-   **Listener Rule 1 (Prioridade 1):**
    -   **IF** Path is `/api/v2/usuarios*`
    -   **THEN** Forward to `target-group-novo-servico-usuarios`

-   **Listener Rule 2 (Prioridade 1):**
    -   **IF** Path is `/api/v2/pedidos*`
    -   **THEN** Forward to `target-group-novo-servico-pedidos`

-   **Listener Rule Default (Última prioridade):**
    -   **IF** (nenhuma das regras acima corresponder)
    -   **THEN** Forward to `target-group-sistema-legado`

À medida que você cria novos serviços, você simplesmente adiciona novas regras com maior prioridade, "roubando" o tráfego do legado de forma incremental e controlada.

---

## 🎯 **Conceito 3: O Desafio do Banco de Dados Monolítico**

Frequentemente, o maior obstáculo na migração é um grande banco de dados monolítico compartilhado por todas as partes do sistema legado. Estrangular as funcionalidades da aplicação é uma coisa; estrangular o banco de dados é outra, muito mais complexa.

**Estratégias Comuns:**

1.  **Banco de Dados Compartilhado (Temporariamente):**
    -   No início, permita que os novos microserviços acessem o banco de dados legado (idealmente através de uma camada de API ou views, não diretamente).
    -   **Pró:** Permite uma migração mais rápida da lógica de negócio.
    -   **Contra:** Mantém o acoplamento no nível dos dados. É uma solução de transição, não o objetivo final.

2.  **Sincronização de Dados (Data Synchronization):**
    -   Crie um novo banco de dados para o seu novo microserviço.
    -   Use ferramentas como **AWS Database Migration Service (DMS)** com Change Data Capture (CDC) para manter os dados sincronizados em tempo real entre o banco legado e o novo banco durante a transição.
    -   Quando a migração da funcionalidade estiver completa, a sincronização pode ser desativada.

3.  **Branch by Abstraction:**
    -   Uma técnica de refatoração onde você cria uma camada de abstração (uma interface) na frente do acesso aos dados.
    -   Inicialmente, a implementação dessa interface lê/escreve no banco legado.
    -   Depois, você cria uma nova implementação que lê/escreve em ambos os bancos (legado e novo) para manter a consistência.
    -   Finalmente, você troca para uma implementação que usa apenas o novo banco.

A migração de dados é um campo complexo, mas a combinação do padrão Strangler Fig com uma estratégia de sincronização de dados como o DMS é uma das abordagens mais robustas e seguras.

--- 