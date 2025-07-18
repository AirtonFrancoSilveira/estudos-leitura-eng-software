# 💰 Estratégias de Otimização de Custo na AWS

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Adotar a mentalidade de FinOps:** Entender que o custo é uma métrica de engenharia, não apenas um problema financeiro.
-   ✅ **Realizar o "Right Sizing"** de recursos, eliminando o desperdício por superprovisionamento.
-   ✅ **Dominar os diferentes modelos de compra da AWS** (Savings Plans, Spot, Reserved Instances) para obter descontos massivos.
-   ✅ **Implementar políticas de ciclo de vida no S3** para mover dados para camadas de armazenamento mais baratas automaticamente.
-   ✅ **Utilizar ferramentas da AWS como Cost Explorer e AWS Budgets** para monitorar, analisar e alertar sobre os gastos.
-   ✅ **Automatizar a limpeza** de recursos não utilizados (os "zumbis" da nuvem).

---

## 🎯 **Conceito 1: FinOps - A Engenharia Encontra as Finanças**

**FinOps** (Cloud Financial Operations) é uma mudança cultural. É a prática de trazer a responsabilidade financeira para o modelo operacional da nuvem, onde **times de engenharia são donos não apenas do serviço que constroem, mas também do custo que ele gera**.

Em um ambiente cloud, cada decisão de arquitetura tem um impacto direto e imediato na fatura. Escolher um tipo de instância, um modelo de banco de dados ou uma estratégia de logging são decisões tanto técnicas quanto financeiras.

**Os 3 Pilares do Controle de Custos na Nuvem:**

1.  **Visibilidade:** Saber exatamente onde o dinheiro está sendo gasto.
2.  **Otimização:** Tomar ações para reduzir os gastos.
3.  **Governança:** Criar políticas e automações para manter os custos sob controle a longo prazo.

---

## 🎯 **Conceito 2: O Pilar da Otimização - Right Sizing**

Este é o ponto de partida mais rápido e impactante. **Right Sizing** é o processo de analisar a utilização real dos seus recursos (CPU, memória) e ajustá-los para o tamanho mínimo necessário para atender à demanda com segurança.

O erro mais comum na nuvem é o **superprovisionamento (overprovisioning)**: alocar uma instância `t3.xlarge` por "precaução", quando os dados mostram que uma `t3.medium` daria conta do recado com 80% de economia.

**Ferramentas Essenciais:**
-   **AWS Compute Optimizer:** Uma ferramenta gratuita da AWS que usa machine learning para analisar as métricas do CloudWatch dos seus recursos (EC2, EBS, ECS Fargate, Lambda) e recomenda tamanhos mais adequados. Ele pode dizer, por exemplo: "Você pode mudar esta instância de `m5.2xlarge` para `r6g.xlarge` e economizar 40% com performance melhorada".
-   **CloudWatch Metrics:** Analise diretamente as métricas de `CPUUtilization`, `MemoryUtilization` (com o CloudWatch Agent) e outras para tomar suas próprias decisões. A regra geral é que, se a utilização máxima de um recurso fica consistentemente abaixo de 40%, ele provavelmente está superprovisionado.

---

## 🎯 **Conceito 3: Comprando de Forma Inteligente - Modelos de Precificação**

Depois de ajustar o tamanho, o próximo grande ganho vem de mudar a forma como você paga pelos recursos.

| Modelo | Como Funciona | Ideal Para | Desconto Potencial |
| :--- | :--- | :--- | :--- |
| **On-Demand** | Pague pelo que usar, por segundo, sem compromisso. | Cargas de trabalho imprevisíveis, desenvolvimento e testes. | 0% (Preço base) |
| **Savings Plans** | Comprometa-se a gastar uma certa quantia (ex: $10/hora) em **Compute (EC2/Fargate/Lambda)** por 1 ou 3 anos. Flexível entre tipos de instância e regiões. | Cargas de trabalho estáveis e previsíveis. **A melhor opção para a maioria das empresas.** | **Até 72%** |
| **Reserved Instances (RIs)** | Comprometa-se a usar um tipo de instância específico (ex: `m5.large`) em uma região específica por 1 ou 3 anos. Menos flexível que Savings Plans. | Cargas de trabalho extremamente estáveis onde o tipo de instância não mudará. | Até 72% |
| **Spot Instances** | "Leilão" da capacidade computacional ociosa da AWS. A AWS pode retomar a instância a qualquer momento com um aviso de 2 minutos. | Cargas de trabalho tolerantes a falhas e sem estado (processamento em batch, renderização, CI/CD). | **Até 90%** |

**Estratégia Mista (A Abordagem de Especialista):**
Uma arquitetura madura usa uma combinação de todos os modelos:
-   **Savings Plans** para cobrir a carga de trabalho base e previsível (ex: os 2 servidores que rodam 24/7).
-   **On-Demand** para lidar com picos de tráfego inesperados.
-   **Spot Instances** para os workers do seu pipeline de CI/CD ou para processamento de dados em lote durante a madrugada.

---

## 🎯 **Conceito 4: Otimização de Armazenamento e Dados**

O armazenamento de dados, especialmente no S3, pode se tornar um custo significativo se não for gerenciado.

**1. S3 Lifecycle Policies:**
-   **O que faz:** Automatiza a movimentação de objetos entre diferentes classes de armazenamento do S3 para economizar custos.
-   **Exemplo de Regra:**
    1.  **0-30 dias:** Manter o objeto em `S3 Standard` (acesso frequente).
    2.  **31-90 dias:** Mover para `S3 Standard-IA` (Infrequent Access - mais barato para armazenar, um pouco mais caro para acessar).
    3.  **Após 91 dias:** Mover para `S3 Glacier Flexible Retrieval` (arquivamento de longo prazo, muito barato para armazenar).
    4.  **Após 5 anos:** Expirar (deletar) o objeto permanentemente.

**2. S3 Intelligent-Tiering:**
-   **O que faz:** Uma classe de armazenamento especial que monitora o padrão de acesso de cada objeto e o move automaticamente para a camada mais econômica (frequente ou infrequente) sem nenhum impacto na performance.
-   **Quando usar:** Se você não conhece ou não consegue prever o padrão de acesso dos seus dados, esta é a opção mais simples e eficaz.

**3. EBS Volumes (Discos das EC2):**
-   **Escolha o Tipo Certo:** Não use volumes `gp3` ou `io2` (SSD de alta performance) para armazenamento de logs ou backups. Use `st1` (HDD otimizado para throughput) ou `sc1` (Cold HDD), que são muito mais baratos.
-   **Delete Volumes Não Anexados:** É extremamente comum ter volumes EBS "órfãos" que não estão atachados a nenhuma instância EC2, mas continuam gerando custo.

---

## 🎯 **Conceito 5: Governança e Automação**

**1. AWS Budgets:**
-   **O que faz:** Permite criar orçamentos para seus custos e ser alertado via E-mail ou SNS quando o gasto real ou previsto ultrapassa um limiar (ex: alertar se o custo do projeto X exceder 80% do orçamento mensal).

**2. AWS Cost Explorer:**
-   **O que faz:** A principal ferramenta para **visibilidade**. Permite filtrar e agrupar seus custos por serviço, por tag, por conta, etc. É aqui que você começa suas investigações para entender para onde o dinheiro está indo.

**3. Automação de Limpeza (Cloud Custodian / AWS Lambda):**
-   **O que faz:** Crie scripts ou use ferramentas open-source como o [Cloud Custodian](https://cloudcustodian.io/) para encontrar e remover recursos desperdiçados automaticamente.
-   **Exemplos de Políticas:**
    -   "Encontre todos os volumes EBS não atachados por mais de 7 dias e crie um snapshot antes de deletá-los."
    -   "Encontre todas as instâncias EC2 sem a tag `owner` e as desligue fora do horário comercial."
    -   "Alerte sobre Elastic IPs que não estão associados a nenhuma instância."

Essa automação é a chave para manter a higiene da sua conta AWS e evitar que o desperdício se acumule ao longo do tempo.

--- 