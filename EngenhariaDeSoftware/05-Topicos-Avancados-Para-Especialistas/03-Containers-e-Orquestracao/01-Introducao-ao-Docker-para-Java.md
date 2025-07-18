# 📦 Introdução ao Docker para Desenvolvedores Java

## 📚 **Objetivos de Aprendizagem**

-   ✅ **Compreender o "porquê"** do Docker e como ele resolve o problema do "funciona na minha máquina".
-   ✅ **Dominar os comandos essenciais** do Docker (`build`, `run`, `push`, `pull`).
-   ✅ **Escrever um `Dockerfile` otimizado** para uma aplicação Spring Boot, usando _multi-stage builds_.
-   ✅ **Entender as melhores práticas** para criar imagens Docker pequenas, seguras e eficientes.
-   ✅ **Orquestrar um ambiente de desenvolvimento local** com múltiplos contêineres usando `docker-compose`.

---

## 🎯 **Conceito 1: O Fim do "Funciona na Minha Máquina"**

O Docker é uma plataforma de **containerização** que resolve um problema clássico da engenharia de software. Antes do Docker, um desenvolvedor precisava instalar e configurar manualmente em sua máquina:
-   A versão correta do Java (JDK 8? 11? 17?)
-   O banco de dados (PostgreSQL? MySQL?)
-   O message broker (RabbitMQ? Kafka?)
-   Variáveis de ambiente específicas

Quando o código era enviado para outro ambiente (testes, homologação, produção), qualquer pequena diferença em versões ou configurações causava falhas difíceis de depurar.

**O Docker resolve isso empacotando a aplicação e TODAS as suas dependências em uma única unidade chamada Contêiner.**

-   **Imagem Docker:** Um "molde" ou "template" leve, imutável e portátil que contém tudo que a aplicação precisa para rodar: o código, as bibliotecas, as variáveis de ambiente e a runtime (o JDK).
-   **Contêiner Docker:** Uma instância em execução de uma Imagem Docker. É um processo isolado que roda de forma consistente em qualquer máquina que tenha o Docker instalado, seja o notebook do desenvolvedor ou um servidor na nuvem.

O Docker garante que, se a imagem for construída corretamente, ela rodará de forma idêntica em qualquer lugar.

---

## 🎯 **Conceito 2: Criando uma Imagem Docker para uma Aplicação Spring Boot**

A chave para criar uma imagem é o `Dockerfile`, um arquivo de texto com instruções passo a passo.

### **A Abordagem Ingênua (Não faça isso!)**

Um primeiro `Dockerfile` poderia ser assim:
```dockerfile
# Dockerfile Inocente
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY . .
RUN ./mvnw package
EXPOSE 8080
CMD ["java", "-jar", "target/minha-app-0.0.1-SNAPSHOT.jar"]
```
**Problemas:**
-   **Imagem Gigante:** Copia todo o código-fonte, dependências de build (`.m2`) e o JDK completo para a imagem final. O resultado é uma imagem com mais de 1GB.
-   **Lento para Rebuildar:** Qualquer alteração no código-fonte invalida o cache do `COPY`, forçando o Maven a baixar todas as dependências novamente a cada build.
-   **Inseguro:** A imagem final contém o Maven e o JDK completo, aumentando a superfície de ataque.

### **A Abordagem Otimizada: Multi-Stage Builds**

Esta é a melhor prática para aplicações Java. Usamos múltiplos "estágios" no mesmo `Dockerfile`.

-   **Estágio de Build (`builder`):** Um estágio temporário que usa uma imagem completa (com JDK e Maven) apenas para compilar o código e gerar o `.jar`.
-   **Estágio Final:** Um segundo estágio que começa de uma imagem base mínima (apenas a JRE), copia **apenas o `.jar`** do estágio de build e define como executá-lo.

```dockerfile
# Dockerfile Otimizado com Multi-Stage Build

# --- 1. Estágio de Build ---
# Usamos uma imagem completa com Maven e JDK para compilar nossa aplicação.
# O alias "builder" nos permite referenciar este estágio mais tarde.
FROM maven:3.8.5-openjdk-17 AS builder

# Define o diretório de trabalho dentro do contêiner.
WORKDIR /app

# Copia primeiro o pom.xml. O Docker armazena essa camada em cache.
# Se o pom.xml não mudar, ele reutiliza o cache do próximo passo.
COPY pom.xml .
RUN mvn dependency:go-offline

# Agora copia o resto do código-fonte.
COPY src ./src

# Executa o build. O Maven não vai baixar as dependências de novo se o pom.xml não mudou.
RUN mvn package -DskipTests

# --- 2. Estágio Final ---
# Começamos de uma imagem base limpa e mínima, contendo apenas a Java Runtime.
FROM openjdk:17-jre-slim

WORKDIR /app

# Copia APENAS o artefato .jar gerado do estágio de build.
# Todo o resto (código-fonte, dependências Maven, JDK completo) é descartado.
COPY --from=builder /app/target/*.jar app.jar

# Expõe a porta que a aplicação vai usar.
EXPOSE 8080

# Comando para executar a aplicação quando o contêiner iniciar.
ENTRYPOINT ["java", "-jar", "app.jar"]
```
**Vantagens:**
-   **Imagem Pequena:** A imagem final tem ~200-300MB em vez de >1GB.
-   **Builds Rápidos:** O cache de camadas do Docker é aproveitado ao máximo.
-   **Segurança Melhorada:** A imagem final contém apenas o mínimo necessário para rodar a aplicação.

---

## 🎯 **Conceito 3: Orquestração Local com `docker-compose`**

Nenhuma aplicação real vive sozinha. Geralmente precisamos de um banco de dados, um broker, etc. O `docker-compose` é uma ferramenta para definir e rodar aplicações Docker com múltiplos contêineres.

Você descreve seu ambiente em um único arquivo `docker-compose.yml`.

**Exemplo: Aplicação Spring Boot + Banco de Dados PostgreSQL**

```yaml
# docker-compose.yml
version: '3.8'

services:
  # O serviço da nossa aplicação Java
  minha-app:
    # Diz ao compose para construir a imagem usando o Dockerfile no diretório atual
    build: .
    # Mapeia a porta 8080 do contêiner para a porta 8080 da nossa máquina
    ports:
      - "8080:8080"
    # Define as variáveis de ambiente que a aplicação usará para se conectar ao banco
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres-db:5432/meudb
      - SPRING_DATASOURCE_USERNAME=user
      - SPRING_DATASOURCE_PASSWORD=password
    # Garante que o contêiner do banco de dados inicie antes da nossa aplicação
    depends_on:
      - postgres-db

  # O serviço do nosso banco de dados
  postgres-db:
    # Puxa a imagem oficial do PostgreSQL do Docker Hub
    image: postgres:14-alpine
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=meudb
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    # Monta um volume para persistir os dados do banco mesmo que o contêiner seja removido
    volumes:
      - postgres_data:/var/lib/postgresql/data

# Define o volume nomeado para persistência
volumes:
  postgres_data:
```

Com este arquivo, basta um único comando para subir todo o ambiente de desenvolvimento:
```bash
docker-compose up --build
```
E um único comando para derrubar tudo:
```bash
docker-compose down
```

Este fluxo de trabalho acelera imensamente o onboarding de novos desenvolvedores e garante que todos rodem o mesmo ambiente de forma consistente.

--- 