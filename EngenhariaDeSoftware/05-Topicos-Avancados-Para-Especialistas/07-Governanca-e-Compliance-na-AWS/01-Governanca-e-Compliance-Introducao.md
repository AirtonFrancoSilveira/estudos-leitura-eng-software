# 🎯 Governança e Compliance na AWS

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Internalizar o Princípio do Menor Privilégio (Principle of Least Privilege)** e aplicá-lo com AWS IAM.
-   ✅ **Diferenciar o uso de IAM Users, Groups e Roles**, entendendo por que Roles são a melhor prática para aplicações.
-   ✅ **Auditar continuamente as configurações dos seus recursos** com o AWS Config para garantir a conformidade com as melhores práticas.
-   ✅ **Rastrear todas as ações e chamadas de API** na sua conta AWS usando o CloudTrail para auditoria e investigações de segurança.
-   ✅ **Entender como a Infraestrutura como Código (IaC)** é uma ferramenta fundamental para a governança.
-   ✅ **Compreender como esses serviços ajudam a atender a regulações** como LGPD, GDPR e PCI-DSS.

---

## 🎯 **Conceito 1: Por que Governança é Crucial?**

Em um ambiente pequeno, é fácil gerenciar permissões e configurações manualmente. Em uma grande empresa com dezenas de times, centenas de desenvolvedores e milhares de recursos na nuvem, essa abordagem leva ao caos, a brechas de segurança e a custos descontrolados.

**Governança na Nuvem** é o ato de estabelecer políticas, processos e controles para gerenciar sua presença na AWS de forma segura, eficiente e em conformidade. Não se trata de burocracia, mas de criar "guard-rails" (grades de proteção) que permitem aos times de desenvolvimento inovar com velocidade e **segurança**.

Os três serviços que formam a espinha dorsal da governança técnica na AWS são: **IAM, AWS Config e CloudTrail.**

---

## 🎯 **Conceito 2: IAM - Quem Pode Fazer O Quê?**

O **AWS Identity and Access Management (IAM)** é o serviço que controla o acesso a todos os outros recursos da AWS. A regra fundamental aqui é o **Princípio do Menor Privilégio**: uma identidade (seja um usuário ou uma aplicação) deve ter **apenas** as permissões estritamente necessárias para realizar sua função, e nada mais.

**Componentes Chave:**

-   **IAM Users:** Representam pessoas. Devem ser usados apenas para acesso ao Console AWS ou CLI por humanos, sempre com **Multi-Factor Authentication (MFA)** ativado.
-   **IAM Groups:** Uma coleção de usuários. É mais fácil aplicar uma política a um grupo do que a múltiplos usuários individuais.
-   **IAM Policies:** O documento JSON que define as permissões. Ele especifica o `Effect` (Allow/Deny), `Action` (ex: `s3:GetObject`), e `Resource` (o ARN do recurso).
-   **IAM Roles:** **A forma correta de dar permissões para aplicações e serviços AWS.** Uma Role é uma identidade que pode ser "assumida" por uma entidade confiável (como uma instância EC2, uma função Lambda ou outro serviço AWS). A aplicação assume a Role e obtém credenciais temporárias para realizar as ações permitidas. Isso elimina a necessidade de armazenar credenciais de longo prazo (chaves de acesso) no código, o que é uma enorme falha de segurança.

**Exemplo de Política de Menor Privilégio para uma Lambda:**
Esta política permite que uma função Lambda leia objetos de um bucket S3 específico, e nada mais.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowLambdaToReadFromSpecificBucket",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::meu-bucket-de-dados-especifico/*"
        },
        {
            "Sid": "AllowLogging",
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        }
    ]
}
```

---

## 🎯 **Conceito 3: AWS Config - O Auditor de Configurações**

Enquanto o IAM controla *quem pode fazer o quê*, o **AWS Config** responde à pergunta: **"O estado dos meus recursos está em conformidade com as minhas regras?"**.

O AWS Config é um serviço que **descobre, registra e avalia continuamente as configurações dos seus recursos AWS**.

**Como Funciona:**
1.  **Gravação:** O Config registra toda e qualquer mudança na configuração de um recurso (ex: uma regra de Security Group foi alterada, a criptografia de um bucket S3 foi desabilitada). Ele mantém um histórico detalhado de cada recurso.
2.  **Regras (Rules):** Você define regras que representam a sua política de segurança e conformidade. A AWS fornece dezenas de regras gerenciadas prontas para uso, e você pode criar as suas.
3.  **Avaliação:** O Config avalia continuamente as configurações dos seus recursos contra as regras definidas.
4.  **Remediação (Opcional):** Se um recurso é marcado como "não-conforme", o Config pode acionar uma ação automática (via AWS Systems Manager) para corrigir a configuração.

**Exemplos de Regras do AWS Config:**
-   `s3-bucket-public-read-prohibited`: Verifica se algum bucket S3 permite acesso público de leitura.
-   `encrypted-volumes`: Garante que todos os volumes EBS atachados a instâncias EC2 sejam criptografados.
-   `mfa-enabled-for-iam-console-access`: Garante que todos os usuários IAM com acesso ao console tenham o MFA ativado.

O AWS Config é a sua principal ferramenta para garantir a conformidade contínua e automatizada.

---

## 🎯 **Conceito 4: AWS CloudTrail - O Log de Auditoria Definitivo**

Se o IAM é o "porteiro" e o Config é o "auditor", o **CloudTrail** é a "câmera de segurança que grava absolutamente tudo".

O CloudTrail registra **toda chamada de API** feita na sua conta AWS, seja ela via Console, CLI, SDK ou por outro serviço AWS. Para cada chamada, ele registra:
-   **Quem:** A identidade (usuário, role) que fez a chamada.
-   **Quando:** O timestamp do evento.
-   **De onde:** O endereço IP de origem.
-   **O quê:** A ação que foi realizada (ex: `ec2:RunInstances`, `s3:DeleteBucket`).
-   **Em qual recurso:** O recurso que foi afetado.

**Por que o CloudTrail é Indispensável?**
-   **Investigações de Segurança:** Se um recurso foi deletado ou uma permissão foi alterada indevidamente, o CloudTrail é o primeiro lugar que você vai olhar para saber exatamente quem fez e quando.
-   **Auditoria de Compliance (LGPD, PCI-DSS):** Auditores exigirão os logs do CloudTrail para provar que você tem um registro imutável de todas as atividades na sua conta.
-   **Análise Operacional:** Ajuda a depurar problemas operacionais, entendendo a sequência de ações que levou a um estado indesejado.

**Melhor Prática:** Habilite o CloudTrail em **todas as regiões** e configure-o para enviar os logs para um bucket S3 centralizado em uma conta AWS separada (uma "conta de Log Archive"), com políticas de proteção contra exclusão.

---

## 🎯 **IaC como Ferramenta de Governança**

Usar **Infraestrutura como Código (IaC)** com CloudFormation ou Terraform é, em si, uma poderosa prática de governança.
-   **Estado Desejado:** O código define o estado exato que a infraestrutura deve ter, servindo como uma documentação viva.
-   **Revisão de Pares (Peer Review):** Todas as mudanças na infraestrutura passam pelo mesmo processo de Pull Request e revisão que o código da aplicação, permitindo que a equipe valide as alterações antes que elas sejam aplicadas.
-   **Trilha de Auditoria:** O histórico do Git fornece um log claro de quem mudou o quê na infraestrutura e por quê.

Ao combinar IaC com AWS Config e CloudTrail, você cria um sistema de governança robusto e em múltiplas camadas.

--- 