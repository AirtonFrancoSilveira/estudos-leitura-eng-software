# 🧪 Testes para Sistemas Distribuídos

## 📚 **Objetivos de Aprendizagem**

Ao final deste módulo, você será capaz de:

-   ✅ **Compreender as limitações** das estratégias de teste tradicionais em arquiteturas de microserviços.
-   ✅ **Aplicar a Pirâmide de Testes Moderna** para alocar esforços de teste de forma eficaz.
-   ✅ **Garantir a compatibilidade entre serviços** usando Testes de Contrato com a biblioteca Pact.
-   ✅ **Criar ambientes de teste de integração realistas e descartáveis** com Testcontainers.
-   ✅ **Validar a resiliência do sistema** com conceitos de Chaos Engineering.
-   ✅ **Escolher a estratégia de teste correta** para cada cenário em um ambiente distribuído.

---

## 🎯 **Conceito 1: A Crise da Pirâmide de Testes Tradicional**

A clássica Pirâmide de Testes, com uma base larga de testes unitários, um meio de testes de integração e um topo fino de testes End-to-End (E2E), funcionou bem para sistemas monolíticos.

![Pirâmide de Testes Tradicional](https://martinfowler.com/bliki/images/testPyramid/test-pyramid.png)

No entanto, em um ecossistema de **microserviços**, este modelo entra em colapso:

-   **Testes Unitários são Insuficientes:** Um serviço pode ter 100% de cobertura de testes unitários e ainda assim falhar miseravelmente em produção, porque o problema não está no código *dentro* do serviço, mas na **comunicação entre os serviços**.
-   **Testes de Integração são Lentos e Instáveis:** Testar a integração real entre 10, 20 ou 50 serviços requer um ambiente de teste complexo, caro, lento para iniciar e que vive quebrado por motivos alheios ao seu teste (um serviço dependente está fora do ar, dados de teste foram corrompidos, etc.).
-   **Testes E2E são um Pesadelo:** Em uma arquitetura distribuída, um teste E2E que simula uma jornada completa do usuário pode atravessar dezenas de serviços. Eles são extremamente frágeis, lentos, difíceis de depurar e quase impossíveis de manter.

**O resultado:** os times gastam mais tempo consertando testes quebrados do que entregando valor, e a confiança na suíte de testes despenca.

---

## 🎯 **Conceito 2: A Pirâmide de Testes Moderna para Microserviços**

Para resolver essa crise, a comunidade de engenharia evoluiu para uma nova abordagem, que valoriza testes que verificam a **interação entre os componentes** de forma rápida e isolada.

![Pirâmide de Testes Moderna](https://testing.google.com/images/ice-cream-cone.png) 
*(A imagem representa um anti-padrão a ser evitado, onde a pirâmide está invertida. A abordagem moderna mantém a forma de pirâmide, mas com novas camadas)*

A pirâmide moderna se parece mais com isso:

1.  **Base Larga - Testes Unitários (Unit Tests):**
    -   **O que testam:** A lógica de negócio *dentro* de uma única classe ou componente, em total isolamento.
    -   **Velocidade:** Extremamente rápidos.
    -   **Foco:** Continuam sendo a base para garantir a correção do código.

2.  **Meio - Testes de Integração (Integration Tests) com Foco em Infraestrutura:**
    -   **O que testam:** A interação do seu serviço com componentes de infraestrutura, como bancos de dados, brokers de mensagens ou APIs externas.
    -   **Ferramenta Chave:** **Testcontainers**. Ele permite "subir" instâncias reais (um banco Postgres, um Kafka, etc.) dentro de contêineres Docker para o seu teste, garantindo um ambiente realista e totalmente descartável.
    -   **Velocidade:** Mais lentos que unitários, mas ainda rápidos o suficiente para rodar no pipeline de CI.

3.  **Meio-Topo - Testes de Contrato (Contract Tests):**
    -   **O que testam:** A "conversa" entre dois serviços. Eles garantem que um serviço consumidor (ex: `Frontend`) está fazendo as requisições que um serviço provedor (ex: `API de Pedidos`) espera, e que o provedor está retornando as respostas que o consumidor consegue entender.
    -   **Ferramenta Chave:** **Pact**.
    -   **Como funciona:** O consumidor define um "contrato" (um arquivo JSON) com suas expectativas. O provedor então verifica se consegue satisfazer esse contrato, tudo de forma assíncrona, sem que os dois serviços precisem estar rodando ao mesmo tempo.
    -   **Valor:** É a camada que substitui a maior parte dos frágeis testes E2E.

4.  **Topo Fino - Testes End-to-End (E2E) Mínimos:**
    -   **O que testam:** Apenas os **fluxos mais críticos e felizes** do sistema (ex: um usuário consegue se cadastrar, fazer login e completar uma compra).
    -   **Foco:** Não é encontrar bugs, mas sim garantir que o sistema está "ligado na tomada" e que as integrações mais fundamentais estão funcionando.
    -   **Quantidade:** Um número muito pequeno (5 a 10 testes no máximo) para evitar a fragilidade.

5.  **Fora da Pirâmide - Testes em Produção (Chaos Engineering):**
    -   **O que testam:** A resiliência do sistema a falhas do mundo real.
    -   **Ferramenta Chave:** **AWS Fault Injection Simulator (FIS)**, Gremlin.
    -   **Como funciona:** Injeta falhas deliberadamente em produção (ex: derruba um contêiner, adiciona latência na rede) para verificar se os mecanismos de resiliência (como Circuit Breakers e retries) funcionam como esperado.

Neste módulo, vamos nos aprofundar nas camadas de **Integração com Testcontainers** e **Contrato com Pact**, que são as maiores alavancas de qualidade em arquiteturas modernas.

--- 