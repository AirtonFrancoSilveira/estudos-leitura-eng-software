# 🚀 DevOps & CI/CD - Guia do Professor

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Criar pipelines CI/CD completas
- ✅ Implementar containerização com Docker
- ✅ Orquestrar aplicações com Kubernetes
- ✅ Configurar monitoramento e observabilidade
- ✅ Aplicar estratégias de deploy avançadas

---

## 🎯 **Conceito 1: Pipeline CI/CD Completa**

### **🔍 O que é?**
CI/CD é como uma linha de produção automatizada: desde a criação do código até a entrega ao cliente final, tudo acontece de forma automática, rápida e confiável.

### **🎓 Conceitos Avançados:**

**1. GitHub Actions - CI/CD Workflow**
```yaml
# .github/workflows/banking-app-ci-cd.yml
name: Banking App CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: banking-app
  ECS_SERVICE: banking-service
  ECS_CLUSTER: banking-cluster
  ECS_TASK_DEFINITION: banking-task-definition

jobs:
  # Etapa 1: Testes e Qualidade
  test:
    name: Test & Quality Check
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_DB: banking_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
          
      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        
    - name: Cache Maven dependencies
      uses: actions/cache@v3
      with:
        path: ~/.m2
        key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
        restore-keys: |
          ${{ runner.os }}-m2-
          
    - name: Run unit tests
      run: mvn clean test
      env:
        SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/banking_test
        SPRING_DATASOURCE_USERNAME: test
        SPRING_DATASOURCE_PASSWORD: test
        SPRING_REDIS_HOST: localhost
        SPRING_REDIS_PORT: 6379
        
    - name: Run integration tests
      run: mvn verify -P integration-tests
      
    - name: Code coverage report
      run: mvn jacoco:report
      
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./target/site/jacoco/jacoco.xml
        
    - name: SonarQube Scan
      uses: sonarqube-quality-gate-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        
    - name: Security scan with Snyk
      uses: snyk/actions/maven@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        
    - name: OWASP Dependency Check
      run: mvn org.owasp:dependency-check-maven:check
      
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: |
          target/surefire-reports/
          target/failsafe-reports/
          target/site/jacoco/

  # Etapa 2: Build e Push para ECR
  build:
    name: Build & Push to ECR
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
        
    - name: Cache Maven dependencies
      uses: actions/cache@v3
      with:
        path: ~/.m2
        key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
        
    - name: Build application
      run: mvn clean package -DskipTests
      
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
        
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
      
    - name: Build, tag, and push image to Amazon ECR
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        # Build Docker image
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:latest .
        
        # Push image to ECR
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
        
        echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
        
    - name: Scan image with Trivy
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ steps.build-image.outputs.image }}
        format: 'sarif'
        output: 'trivy-results.sarif'
        
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  # Etapa 3: Deploy para Staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment: staging
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
        
    - name: Deploy to ECS Staging
      run: |
        # Update ECS task definition
        aws ecs update-service \
          --cluster ${{ env.ECS_CLUSTER }}-staging \
          --service ${{ env.ECS_SERVICE }}-staging \
          --force-new-deployment
          
    - name: Wait for deployment to complete
      run: |
        aws ecs wait services-stable \
          --cluster ${{ env.ECS_CLUSTER }}-staging \
          --services ${{ env.ECS_SERVICE }}-staging
          
    - name: Run smoke tests
      run: |
        # Wait for service to be ready
        sleep 30
        
        # Run basic health check
        curl -f https://staging-api.banking.com/actuator/health
        
        # Run smoke tests
        mvn test -P smoke-tests -Dtest.environment=staging

  # Etapa 4: Deploy para Production
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment: production
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
        
    - name: Blue-Green Deployment
      run: |
        # Script para deploy Blue-Green
        ./scripts/blue-green-deploy.sh
        
    - name: Post-deployment tests
      run: |
        # Testes pós-deploy
        mvn test -P post-deployment-tests
        
    - name: Notify team
      uses: 8398a7/action-slack@v3
      if: always()
      with:
        status: ${{ job.status }}
        text: |
          Banking App deployed to production
          Commit: ${{ github.sha }}
          Status: ${{ job.status }}
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

**2. Dockerfile Multi-stage Otimizado**
```dockerfile
# Multi-stage Dockerfile para aplicação Spring Boot
FROM maven:3.8.6-openjdk-17 AS builder

# Definir diretório de trabalho
WORKDIR /app

# Copiar arquivos de dependências primeiro (para cache)
COPY pom.xml ./
COPY src/main/resources/application.properties ./src/main/resources/

# Baixar dependências (cached se pom.xml não mudou)
RUN mvn dependency:go-offline -B

# Copiar código fonte
COPY src ./src

# Build da aplicação
RUN mvn clean package -DskipTests

# Extrair JAR layers para otimizar cache do Docker
RUN java -Djarmode=layertools -jar target/*.jar extract

# Runtime stage
FROM openjdk:17-jre-slim

# Instalar utilitários necessários
RUN apt-get update && apt-get install -y \
    curl \
    jq \
    && rm -rf /var/lib/apt/lists/*

# Criar usuário não-root para segurança
RUN addgroup --system banking && adduser --system banking --ingroup banking

# Definir diretório de trabalho
WORKDIR /app

# Copiar layers do builder (otimiza cache)
COPY --from=builder app/dependencies/ ./
COPY --from=builder app/spring-boot-loader/ ./
COPY --from=builder app/snapshot-dependencies/ ./
COPY --from=builder app/application/ ./

# Copiar script de entrada
COPY docker-entrypoint.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/docker-entrypoint.sh

# Definir proprietário dos arquivos
RUN chown -R banking:banking /app

# Mudar para usuário não-root
USER banking

# Configurar health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# Expor porta
EXPOSE 8080

# Configurar JVM para containerização
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+UseG1GC \
               -XX:MaxGCPauseMillis=200 \
               -XX:+UseStringDeduplication \
               -XX:+OptimizeStringConcat \
               -Djava.security.egd=file:/dev/./urandom"

# Configurar Spring Boot
ENV SPRING_PROFILES_ACTIVE=docker

# Entrypoint
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["java", "-cp", "BOOT-INF/classes:BOOT-INF/lib/*", "com.bank.BankingApplication"]
```

**3. Script de Entrada Inteligente**
```bash
#!/bin/bash
# docker-entrypoint.sh

set -e

# Cores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

warn() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] WARNING:${NC} $1"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR:${NC} $1"
}

# Função para aguardar dependências
wait_for_dependencies() {
    log "Checking dependencies..."
    
    # Aguardar banco de dados
    if [ -n "$DATABASE_HOST" ]; then
        log "Waiting for database at $DATABASE_HOST:${DATABASE_PORT:-5432}"
        until nc -z $DATABASE_HOST ${DATABASE_PORT:-5432}; do
            warn "Database not ready, waiting 5 seconds..."
            sleep 5
        done
        log "Database is ready!"
    fi
    
    # Aguardar Redis
    if [ -n "$REDIS_HOST" ]; then
        log "Waiting for Redis at $REDIS_HOST:${REDIS_PORT:-6379}"
        until nc -z $REDIS_HOST ${REDIS_PORT:-6379}; do
            warn "Redis not ready, waiting 5 seconds..."
            sleep 5
        done
        log "Redis is ready!"
    fi
}

# Função para configurar JVM baseado no ambiente
configure_jvm() {
    log "Configuring JVM settings..."
    
    # Detectar memória disponível
    MEMORY_LIMIT=$(cat /sys/fs/cgroup/memory/memory.limit_in_bytes)
    if [ $MEMORY_LIMIT -gt 0 ]; then
        # Configurar heap baseado na memória do container
        MAX_HEAP_SIZE=$((MEMORY_LIMIT * 75 / 100))
        export JAVA_OPTS="$JAVA_OPTS -Xmx${MAX_HEAP_SIZE}"
        log "Set max heap size to ${MAX_HEAP_SIZE} bytes"
    fi
    
    # Configurações específicas por ambiente
    case "$ENVIRONMENT" in
        "development")
            export JAVA_OPTS="$JAVA_OPTS -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=5005"
            log "Development mode: Remote debugging enabled on port 5005"
            ;;
        "production")
            export JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof"
            log "Production mode: Heap dump enabled"
            ;;
    esac
}

# Função para pre-flight checks
preflight_checks() {
    log "Running preflight checks..."
    
    # Verificar variáveis obrigatórias
    if [ -z "$DATABASE_URL" ]; then
        error "DATABASE_URL is required"
        exit 1
    fi
    
    # Verificar conectividade externa se necessário
    if [ "$ENVIRONMENT" = "production" ]; then
        if ! curl -s --max-time 10 https://api.example.com/health > /dev/null; then
            warn "External API is not reachable"
        fi
    fi
    
    log "Preflight checks completed"
}

# Função para configurar monitoramento
setup_monitoring() {
    log "Setting up monitoring..."
    
    # Configurar APM se especificado
    if [ -n "$APM_SERVER_URL" ]; then
        export JAVA_OPTS="$JAVA_OPTS -javaagent:/app/elastic-apm-agent.jar"
        export ELASTIC_APM_SERVER_URL=$APM_SERVER_URL
        export ELASTIC_APM_APPLICATION_NAME="banking-app"
        export ELASTIC_APM_ENVIRONMENT=$ENVIRONMENT
        log "APM monitoring enabled"
    fi
    
    # Configurar métricas JVM
    export JAVA_OPTS="$JAVA_OPTS -Dmanagement.endpoints.web.exposure.include=health,info,metrics,prometheus"
    
    log "Monitoring setup completed"
}

# Função para graceful shutdown
graceful_shutdown() {
    log "Received shutdown signal, starting graceful shutdown..."
    
    # Enviar SIGTERM para o processo Java
    if [ -n "$JAVA_PID" ]; then
        kill -TERM $JAVA_PID
        
        # Aguardar por até 30 segundos
        for i in {1..30}; do
            if ! kill -0 $JAVA_PID 2>/dev/null; then
                log "Application stopped gracefully"
                exit 0
            fi
            sleep 1
        done
        
        warn "Graceful shutdown timed out, forcing shutdown"
        kill -KILL $JAVA_PID
    fi
    
    exit 1
}

# Configurar signal handlers
trap graceful_shutdown SIGTERM SIGINT

# Função principal
main() {
    log "Starting Banking Application..."
    
    # Executar checks e configurações
    preflight_checks
    wait_for_dependencies
    configure_jvm
    setup_monitoring
    
    log "Starting Java application with options: $JAVA_OPTS"
    
    # Iniciar aplicação em background para permitir signal handling
    exec "$@" &
    JAVA_PID=$!
    
    # Aguardar processo
    wait $JAVA_PID
}

# Executar função principal
main "$@"
```

### **💡 Caso de Uso Prático: Sistema Bancário com Deploy Automatizado**

```yaml
# docker-compose.yml para ambiente de desenvolvimento
version: '3.8'

services:
  banking-app:
    build: .
    ports:
      - "8080:8080"
      - "5005:5005"  # Debug port
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - DATABASE_URL=jdbc:postgresql://postgres:5432/banking
      - DATABASE_USERNAME=banking
      - DATABASE_PASSWORD=banking123
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - ENVIRONMENT=development
    depends_on:
      - postgres
      - redis
      - localstack
    volumes:
      - ./logs:/app/logs
    networks:
      - banking-network

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=banking
      - POSTGRES_USER=banking
      - POSTGRES_PASSWORD=banking123
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    ports:
      - "5432:5432"
    networks:
      - banking-network

  redis:
    image: redis:6-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    networks:
      - banking-network

  localstack:
    image: localstack/localstack:latest
    ports:
      - "4566:4566"
    environment:
      - SERVICES=s3,sqs,sns,dynamodb,secretsmanager
      - DEBUG=1
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - localstack_data:/tmp/localstack
      - ./localstack/init-aws.sh:/etc/localstack/init/ready.d/init-aws.sh
    networks:
      - banking-network

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    networks:
      - banking-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    depends_on:
      - prometheus
    networks:
      - banking-network

volumes:
  postgres_data:
  redis_data:
  localstack_data:
  prometheus_data:
  grafana_data:

networks:
  banking-network:
    driver: bridge
```

---

## 🎯 **Conceito 2: Kubernetes e Orquestração**

### **🔍 O que é?**
Kubernetes é como um maestro de orquestra que coordena centenas de músicos (containers) para tocar uma sinfonia harmoniosa (aplicação distribuída).

### **🎓 Conceitos Avançados:**

**1. Deployment e Service**
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: banking-app
  namespace: banking
  labels:
    app: banking-app
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: banking-app
  template:
    metadata:
      labels:
        app: banking-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: banking-app
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: banking-app
        image: banking-app:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 8081
          name: management
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: banking-secrets
              key: database-url
        - name: DATABASE_USERNAME
          valueFrom:
            secretKeyRef:
              name: banking-secrets
              key: database-username
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: banking-secrets
              key: database-password
        - name: REDIS_HOST
          value: "redis-service"
        - name: REDIS_PORT
          value: "6379"
        - name: JVM_OPTS
          value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8081
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8081
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /actuator/health/startup
            port: 8081
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        - name: logs-volume
          mountPath: /app/logs
      volumes:
      - name: config-volume
        configMap:
          name: banking-config
      - name: logs-volume
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: banking-app-service
  namespace: banking
  labels:
    app: banking-app
spec:
  selector:
    app: banking-app
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: management
    port: 8081
    targetPort: 8081
    protocol: TCP
  type: ClusterIP
```

**2. ConfigMap e Secrets**
```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: banking-config
  namespace: banking
data:
  application.yaml: |
    spring:
      datasource:
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5
          connection-timeout: 30000
      redis:
        timeout: 2000
        lettuce:
          pool:
            max-active: 8
            max-idle: 8
            min-idle: 2
      kafka:
        bootstrap-servers: kafka-service:9092
        producer:
          retries: 3
          batch-size: 16384
          linger-ms: 1
        consumer:
          group-id: banking-app
          auto-offset-reset: earliest
    
    management:
      endpoints:
        web:
          exposure:
            include: health,info,metrics,prometheus
      endpoint:
        health:
          show-details: always
      metrics:
        export:
          prometheus:
            enabled: true
    
    logging:
      level:
        com.bank: INFO
        org.springframework.security: DEBUG
      pattern:
        console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
        file: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n"
      file:
        name: /app/logs/banking-app.log
        max-size: 100MB
        max-history: 30

---
apiVersion: v1
kind: Secret
metadata:
  name: banking-secrets
  namespace: banking
type: Opaque
data:
  database-url: amRiYzpwb3N0Z3Jlc3FsOi8vcG9zdGdyZXM6NTQzMi9iYW5raW5n
  database-username: YmFua2luZw==
  database-password: YmFua2luZzEyMw==
  jwt-secret: c3VwZXJfc2VjcmV0X2tleV9mb3JfandUX3NpZ25pbmc=
  encryption-key: YWVzMjU2X2VuY3J5cHRpb25fa2V5X2Zvcl9kYXRh
```

**3. Ingress e Service Mesh**
```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: banking-app-ingress
  namespace: banking
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - api.banking.com
    secretName: banking-app-tls
  rules:
  - host: api.banking.com
    http:
      paths:
      - path: /api/(.*)
        pathType: Prefix
        backend:
          service:
            name: banking-app-service
            port:
              number: 80
      - path: /actuator/(.*)
        pathType: Prefix
        backend:
          service:
            name: banking-app-service
            port:
              number: 8081
```

**4. Horizontal Pod Autoscaler (HPA)**
```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: banking-app-hpa
  namespace: banking
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: banking-app
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

### **💡 Caso de Uso Prático: Deploy Blue-Green no Kubernetes**

```yaml
# k8s/blue-green-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: banking-app-rollout
  namespace: banking
spec:
  replicas: 5
  strategy:
    blueGreen:
      activeService: banking-app-active
      previewService: banking-app-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: banking-app-preview
      postPromotionAnalysis:
        templates:
        - templateName: success-rate
        args:
        - name: service-name
          value: banking-app-active
  selector:
    matchLabels:
      app: banking-app
  template:
    metadata:
      labels:
        app: banking-app
    spec:
      containers:
      - name: banking-app
        image: banking-app:latest
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: banking-app-active
  namespace: banking
spec:
  selector:
    app: banking-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: banking-app-preview
  namespace: banking
spec:
  selector:
    app: banking-app
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
  namespace: banking
spec:
  args:
  - name: service-name
  metrics:
  - name: success-rate
    successCondition: result[0] >= 0.95
    interval: 60s
    count: 5
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",code!~"5.."}[5m])) / 
          sum(rate(http_requests_total{service="{{args.service-name}}"}[5m]))
```

---

## 🎯 **Conceito 3: Monitoramento e Observabilidade**

### **🔍 O que é?**
Observabilidade é como ter um painel de controle completo de um avião: você vê tudo o que está acontecendo, pode antecipar problemas e tomar decisões informadas.

### **🎓 Conceitos Avançados:**

**1. Prometheus e Grafana**
```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'banking-app'
    static_configs:
      - targets: ['banking-app:8080']
    metrics_path: /actuator/prometheus
    scrape_interval: 10s
    scrape_timeout: 5s

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
```

**2. Alertas Customizados**
```yaml
# monitoring/alert_rules.yml
groups:
- name: banking-app-alerts
  rules:
  - alert: HighErrorRate
    expr: |
      (
        sum(rate(http_requests_total{job="banking-app",code=~"5.."}[5m])) /
        sum(rate(http_requests_total{job="banking-app"}[5m]))
      ) > 0.05
    for: 5m
    labels:
      severity: critical
      service: banking-app
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

  - alert: HighResponseTime
    expr: |
      histogram_quantile(0.95, 
        sum(rate(http_request_duration_seconds_bucket{job="banking-app"}[5m])) by (le)
      ) > 0.5
    for: 10m
    labels:
      severity: warning
      service: banking-app
    annotations:
      summary: "High response time detected"
      description: "95th percentile response time is {{ $value }}s"

  - alert: DatabaseConnectionPoolHigh
    expr: |
      hikaricp_connections_active{job="banking-app"} / 
      hikaricp_connections_max{job="banking-app"} > 0.8
    for: 5m
    labels:
      severity: warning
      service: banking-app
    annotations:
      summary: "Database connection pool usage high"
      description: "Connection pool usage is {{ $value | humanizePercentage }}"

  - alert: MemoryUsageHigh
    expr: |
      (
        process_resident_memory_bytes{job="banking-app"} / 
        container_spec_memory_limit_bytes{job="banking-app"}
      ) > 0.9
    for: 10m
    labels:
      severity: critical
      service: banking-app
    annotations:
      summary: "Memory usage is very high"
      description: "Memory usage is {{ $value | humanizePercentage }}"

  - alert: PodCrashLooping
    expr: |
      increase(kube_pod_container_status_restarts_total{job="kubernetes-pods"}[15m]) > 0
    for: 0m
    labels:
      severity: critical
      service: banking-app
    annotations:
      summary: "Pod is crash looping"
      description: "Pod {{ $labels.pod }} is crash looping"
```

**3. Distributed Tracing com Jaeger**
```java
// Configuração de tracing
@Configuration
public class TracingConfiguration {
    
    @Bean
    public Tracer jaegerTracer() {
        return Configuration.fromEnv("banking-app")
            .withSampler(Configuration.SamplerConfiguration.fromEnv()
                .withType(ConstSampler.TYPE)
                .withParam(1))
            .withReporter(Configuration.ReporterConfiguration.fromEnv()
                .withLogSpans(true)
                .withFlushInterval(1000)
                .withMaxQueueSize(10000)
                .withSender(Configuration.SenderConfiguration.fromEnv()
                    .withAgentHost("jaeger-agent")
                    .withAgentPort(6831)))
            .getTracer();
    }
    
    @Bean
    public GlobalTracer globalTracer(Tracer tracer) {
        GlobalTracer.register(tracer);
        return GlobalTracer.get();
    }
}

// Service com tracing
@Service
public class TracedTransactionService {
    
    private final Tracer tracer;
    private final TransactionRepository transactionRepository;
    
    public TracedTransactionService(Tracer tracer, TransactionRepository transactionRepository) {
        this.tracer = tracer;
        this.transactionRepository = transactionRepository;
    }
    
    @Traced
    public TransactionResult processTransaction(TransferRequest request) {
        Span span = tracer.nextSpan()
            .name("process-transaction")
            .tag("transaction.type", request.getType())
            .tag("transaction.amount", request.getAmount().toString())
            .start();
        
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            
            // Validar request
            Span validationSpan = tracer.nextSpan()
                .name("validate-request")
                .start();
            
            try (Tracer.SpanInScope validationScope = tracer.withSpanInScope(validationSpan)) {
                validateRequest(request);
            } catch (Exception e) {
                validationSpan.tag("error", true);
                validationSpan.tag("error.message", e.getMessage());
                throw e;
            } finally {
                validationSpan.end();
            }
            
            // Processar transação
            Span processingSpan = tracer.nextSpan()
                .name("process-transfer")
                .start();
            
            try (Tracer.SpanInScope processingScope = tracer.withSpanInScope(processingSpan)) {
                return doProcessTransaction(request);
            } catch (Exception e) {
                processingSpan.tag("error", true);
                processingSpan.tag("error.message", e.getMessage());
                throw e;
            } finally {
                processingSpan.end();
            }
            
        } catch (Exception e) {
            span.tag("error", true);
            span.tag("error.message", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### **💡 Caso de Uso Prático: Monitoramento Bancário Completo**

```java
// Métricas customizadas para sistema bancário
@Component
public class BankingMetrics {
    
    private final Counter transactionCounter;
    private final Timer transactionTimer;
    private final Gauge activeTransactions;
    private final Counter fraudDetectionCounter;
    private final Summary transactionAmount;
    
    public BankingMetrics(MeterRegistry meterRegistry) {
        this.transactionCounter = Counter.builder("banking_transactions_total")
            .description("Total number of transactions processed")
            .register(meterRegistry);
            
        this.transactionTimer = Timer.builder("banking_transaction_duration")
            .description("Time taken to process transactions")
            .register(meterRegistry);
            
        this.activeTransactions = Gauge.builder("banking_active_transactions")
            .description("Number of active transactions")
            .register(meterRegistry, this, BankingMetrics::getActiveTransactionCount);
            
        this.fraudDetectionCounter = Counter.builder("banking_fraud_detections_total")
            .description("Number of fraud detections")
            .register(meterRegistry);
            
        this.transactionAmount = Summary.builder("banking_transaction_amount")
            .description("Distribution of transaction amounts")
            .register(meterRegistry);
    }
    
    public void recordTransaction(String type, String status, BigDecimal amount, Duration duration) {
        transactionCounter.increment(
            Tags.of("type", type, "status", status)
        );
        
        transactionTimer.record(duration);
        
        transactionAmount.record(amount.doubleValue());
    }
    
    public void recordFraudDetection(String reason) {
        fraudDetectionCounter.increment(Tags.of("reason", reason));
    }
    
    private double getActiveTransactionCount() {
        // Implementar lógica para contar transações ativas
        return transactionService.getActiveTransactionCount();
    }
}
```

---

## 🎯 **Conceito 4: Infrastructure as Code (IaC)**

### **🔍 O que é?**
Infrastructure as Code é como ter uma receita de bolo para construir infraestrutura: você escreve código que define exatamente como criar, configurar e gerenciar seus recursos de infraestrutura.

### **🎓 Conceitos Avançados:**

**1. Terraform - Infraestrutura Multi-Cloud**
```hcl
# main.tf - Infraestrutura principal
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "banking-terraform-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
    
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "Banking System"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

# Variables
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "production"
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# VPC Module
module "vpc" {
  source = "./modules/vpc"
  
  environment        = var.environment
  vpc_cidr          = var.vpc_cidr
  availability_zones = var.availability_zones
}

# ECS Cluster Module
module "ecs_cluster" {
  source = "./modules/ecs"
  
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnet_ids
  
  cluster_name = "banking-cluster"
  
  services = [
    {
      name         = "banking-api"
      image        = "banking-api:latest"
      cpu          = 512
      memory       = 1024
      port         = 8080
      desired_count = 3
      health_check_path = "/actuator/health"
    },
    {
      name         = "notification-service"
      image        = "notification-service:latest"
      cpu          = 256
      memory       = 512
      port         = 8081
      desired_count = 2
      health_check_path = "/health"
    }
  ]
}

# RDS Module
module "database" {
  source = "./modules/rds"
  
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.database_subnet_ids
  
  engine         = "postgres"
  engine_version = "14.9"
  instance_class = "db.r5.xlarge"
  
  allocated_storage     = 100
  max_allocated_storage = 1000
  storage_encrypted     = true
  
  database_name = "banking"
  username      = "banking_user"
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  multi_az               = true
  publicly_accessible    = false
  
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  performance_insights_enabled = true
}

# ElastiCache Module
module "cache" {
  source = "./modules/elasticache"
  
  environment = var.environment
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.cache_subnet_ids
  
  engine               = "redis"
  node_type           = "cache.r6g.large"
  num_cache_nodes     = 3
  parameter_group_name = "default.redis7"
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  
  automatic_failover_enabled = true
  multi_az_enabled          = true
}

# Outputs
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "ecs_cluster_id" {
  value = module.ecs_cluster.cluster_id
}

output "database_endpoint" {
  value     = module.database.endpoint
  sensitive = true
}

output "cache_endpoint" {
  value = module.cache.endpoint
}
```

**2. Módulo VPC Reutilizável**
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.environment}-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.environment}-igw"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.environment}-public-${count.index + 1}"
    Type = "Public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "${var.environment}-private-${count.index + 1}"
    Type = "Private"
  }
}

# Database Subnets
resource "aws_subnet" "database" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 20)
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "${var.environment}-database-${count.index + 1}"
    Type = "Database"
  }
}

# NAT Gateways
resource "aws_eip" "nat" {
  count = length(var.availability_zones)
  
  domain = "vpc"
  
  tags = {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count = length(var.availability_zones)
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = {
    Name = "${var.environment}-nat-${count.index + 1}"
  }
  
  depends_on = [aws_internet_gateway.main]
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${var.environment}-public-rt"
  }
}

resource "aws_route_table" "private" {
  count = length(var.availability_zones)
  
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = {
    Name = "${var.environment}-private-rt-${count.index + 1}"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Security Groups
resource "aws_security_group" "alb" {
  name_prefix = "${var.environment}-alb-"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.environment}-alb-sg"
  }
}

resource "aws_security_group" "ecs" {
  name_prefix = "${var.environment}-ecs-"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 0
    to_port         = 65535
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.environment}-ecs-sg"
  }
}

resource "aws_security_group" "database" {
  name_prefix = "${var.environment}-db-"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.ecs.id]
  }
  
  tags = {
    Name = "${var.environment}-database-sg"
  }
}

# modules/vpc/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  value = aws_subnet.database[*].id
}

output "cache_subnet_ids" {
  value = aws_subnet.private[*].id
}

output "security_group_alb_id" {
  value = aws_security_group.alb.id
}

output "security_group_ecs_id" {
  value = aws_security_group.ecs.id
}

output "security_group_database_id" {
  value = aws_security_group.database.id
}
```

**3. Módulo ECS com Auto Scaling**
```hcl
# modules/ecs/main.tf
resource "aws_ecs_cluster" "main" {
  name = var.cluster_name
  
  configuration {
    execute_command_configuration {
      logging = "OVERRIDE"
      
      log_configuration {
        cloud_watch_encryption_enabled = true
        cloud_watch_log_group_name     = aws_cloudwatch_log_group.ecs.name
      }
    }
  }
  
  tags = {
    Name = var.cluster_name
  }
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${var.environment}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [var.security_group_alb_id]
  subnets            = var.public_subnet_ids
  
  enable_deletion_protection = var.environment == "production"
  
  tags = {
    Name = "${var.environment}-alb"
  }
}

resource "aws_lb_target_group" "app" {
  count = length(var.services)
  
  name     = "${var.environment}-${var.services[count.index].name}-tg"
  port     = var.services[count.index].port
  protocol = "HTTP"
  vpc_id   = var.vpc_id
  
  target_type = "ip"
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = var.services[count.index].health_check_path
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  
  tags = {
    Name = "${var.environment}-${var.services[count.index].name}-tg"
  }
}

resource "aws_lb_listener" "app" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app[0].arn
  }
}

# ECS Task Definitions
resource "aws_ecs_task_definition" "app" {
  count = length(var.services)
  
  family                   = "${var.environment}-${var.services[count.index].name}"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.services[count.index].cpu
  memory                   = var.services[count.index].memory
  
  execution_role_arn = aws_iam_role.ecs_execution.arn
  task_role_arn      = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name  = var.services[count.index].name
      image = var.services[count.index].image
      
      portMappings = [
        {
          containerPort = var.services[count.index].port
          protocol      = "tcp"
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          awslogs-group         = aws_cloudwatch_log_group.ecs.name
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = var.services[count.index].name
        }
      }
      
      environment = [
        {
          name  = "ENVIRONMENT"
          value = var.environment
        },
        {
          name  = "SERVICE_NAME"
          value = var.services[count.index].name
        }
      ]
      
      secrets = [
        {
          name      = "DATABASE_PASSWORD"
          valueFrom = aws_secretsmanager_secret.database_password.arn
        }
      ]
      
      essential = true
    }
  ])
  
  tags = {
    Name = "${var.environment}-${var.services[count.index].name}"
  }
}

# ECS Services
resource "aws_ecs_service" "app" {
  count = length(var.services)
  
  name            = "${var.environment}-${var.services[count.index].name}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app[count.index].arn
  desired_count   = var.services[count.index].desired_count
  
  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }
  
  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [var.security_group_ecs_id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.app[count.index].arn
    container_name   = var.services[count.index].name
    container_port   = var.services[count.index].port
  }
  
  depends_on = [aws_lb_listener.app]
  
  tags = {
    Name = "${var.environment}-${var.services[count.index].name}"
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "ecs" {
  count = length(var.services)
  
  max_capacity       = var.services[count.index].max_capacity
  min_capacity       = var.services[count.index].min_capacity
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app[count.index].name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_cpu" {
  count = length(var.services)
  
  name               = "${var.environment}-${var.services[count.index].name}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs[count.index].resource_id
  scalable_dimension = aws_appautoscaling_target.ecs[count.index].scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs[count.index].service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    
    target_value = 70.0
  }
}

resource "aws_appautoscaling_policy" "ecs_memory" {
  count = length(var.services)
  
  name               = "${var.environment}-${var.services[count.index].name}-memory-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs[count.index].resource_id
  scalable_dimension = aws_appautoscaling_target.ecs[count.index].scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs[count.index].service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
    
    target_value = 80.0
  }
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "ecs" {
  name              = "/ecs/${var.environment}"
  retention_in_days = 7
  
  tags = {
    Name = "/ecs/${var.environment}"
  }
}

# Secrets Manager
resource "aws_secretsmanager_secret" "database_password" {
  name                    = "${var.environment}-database-password"
  recovery_window_in_days = 7
  
  tags = {
    Name = "${var.environment}-database-password"
  }
}

# IAM Roles
resource "aws_iam_role" "ecs_execution" {
  name = "${var.environment}-ecs-execution-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution" {
  role       = aws_iam_role.ecs_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_iam_role_policy" "ecs_execution_secrets" {
  name = "${var.environment}-ecs-execution-secrets"
  role = aws_iam_role.ecs_execution.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          aws_secretsmanager_secret.database_password.arn
        ]
      }
    ]
  })
}

resource "aws_iam_role" "ecs_task" {
  name = "${var.environment}-ecs-task-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}
```

**4. Terraform com CI/CD**
```yaml
# .github/workflows/terraform.yml
name: Terraform Infrastructure

on:
  push:
    branches: [ main ]
    paths: [ 'terraform/**' ]
  pull_request:
    branches: [ main ]
    paths: [ 'terraform/**' ]

env:
  TF_VERSION: 1.6.0
  AWS_REGION: us-east-1

jobs:
  terraform:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: ${{ env.TF_VERSION }}
        
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
        
    - name: Terraform Format Check
      run: terraform fmt -check -recursive
      working-directory: ./terraform
      
    - name: Terraform Init
      run: terraform init
      working-directory: ./terraform
      
    - name: Terraform Validate
      run: terraform validate
      working-directory: ./terraform
      
    - name: Terraform Plan
      run: terraform plan -out=tfplan
      working-directory: ./terraform
      
    - name: Upload Terraform Plan
      uses: actions/upload-artifact@v3
      with:
        name: terraform-plan
        path: terraform/tfplan
        
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply tfplan
      working-directory: ./terraform
      
    - name: Terraform Output
      if: github.ref == 'refs/heads/main'
      run: terraform output -json > terraform-outputs.json
      working-directory: ./terraform
      
    - name: Upload Terraform Outputs
      if: github.ref == 'refs/heads/main'
      uses: actions/upload-artifact@v3
      with:
        name: terraform-outputs
        path: terraform/terraform-outputs.json
```

**5. CloudFormation para Comparação**
```yaml
# cloudformation/banking-infrastructure.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Banking System Infrastructure'

Parameters:
  Environment:
    Type: String
    Default: production
    AllowedValues: [development, staging, production]
    
  VpcCidr:
    Type: String
    Default: 10.0.0.0/16
    Description: CIDR block for VPC

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-igw'

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  # Public Subnets
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Sub '10.0.1.0/24'
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-1'

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Sub '10.0.2.0/24'
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-2'

  # ECS Cluster
  ECSCluster:
    Type: AWS::ECS::Cluster
    Properties:
      ClusterName: !Sub '${Environment}-cluster'
      CapacityProviders:
        - FARGATE
        - FARGATE_SPOT
      DefaultCapacityProviderStrategy:
        - CapacityProvider: FARGATE
          Weight: 1
          Base: 0

  # Task Definition
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: !Sub '${Environment}-banking-api'
      TaskRoleArn: !GetAtt TaskRole.Arn
      ExecutionRoleArn: !GetAtt ExecutionRole.Arn
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE
      Cpu: 512
      Memory: 1024
      ContainerDefinitions:
        - Name: banking-api
          Image: banking-api:latest
          Essential: true
          PortMappings:
            - ContainerPort: 8080
              Protocol: tcp
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-group: !Ref LogGroup
              awslogs-region: !Ref AWS::Region
              awslogs-stream-prefix: banking-api

  # IAM Roles
  TaskRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole

  ExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ecs-tasks.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

  # CloudWatch Log Group
  LogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/ecs/${Environment}'
      RetentionInDays: 7

Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-vpc-id'

  ClusterId:
    Description: ECS Cluster ID
    Value: !Ref ECSCluster
    Export:
      Name: !Sub '${Environment}-cluster-id'
```

### **💡 Caso de Uso Prático: Banco com Múltiplos Ambientes**

```hcl
# environments/production/main.tf
module "banking_infrastructure" {
  source = "../../modules/banking-complete"
  
  environment = "production"
  aws_region  = "us-east-1"
  
  # VPC Configuration
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  
  # ECS Configuration
  ecs_services = [
    {
      name          = "banking-api"
      image         = "banking-api:v1.2.3"
      cpu           = 1024
      memory        = 2048
      port          = 8080
      desired_count = 5
      min_capacity  = 3
      max_capacity  = 10
      health_check_path = "/actuator/health"
    },
    {
      name          = "payment-service"
      image         = "payment-service:v1.1.0"
      cpu           = 512
      memory        = 1024
      port          = 8081
      desired_count = 3
      min_capacity  = 2
      max_capacity  = 8
      health_check_path = "/health"
    }
  ]
  
  # Database Configuration
  database_config = {
    engine         = "postgres"
    engine_version = "14.9"
    instance_class = "db.r5.2xlarge"
    allocated_storage = 500
    max_allocated_storage = 2000
    multi_az = true
    backup_retention_period = 30
    backup_window = "03:00-04:00"
    maintenance_window = "sun:04:00-sun:05:00"
  }
  
  # Cache Configuration
  cache_config = {
    engine = "redis"
    node_type = "cache.r6g.xlarge"
    num_cache_nodes = 3
    automatic_failover_enabled = true
    multi_az_enabled = true
  }
  
  # Monitoring Configuration
  monitoring_config = {
    enable_detailed_monitoring = true
    log_retention_days = 90
    enable_performance_insights = true
    enable_enhanced_monitoring = true
  }
  
  # Security Configuration
  security_config = {
    enable_encryption_at_rest = true
    enable_encryption_in_transit = true
    enable_waf = true
    enable_shield = true
  }
  
  # Compliance Configuration
  compliance_config = {
    enable_cloudtrail = true
    enable_config = true
    enable_guardduty = true
    enable_security_hub = true
  }
  
  # Backup Configuration
  backup_config = {
    enable_automated_backups = true
    backup_retention_period = 30
    enable_point_in_time_recovery = true
    enable_cross_region_backup = true
  }
  
  # Tags
  common_tags = {
    Project = "Banking System"
    Environment = "production"
    Owner = "DevOps Team"
    CostCenter = "IT"
    Compliance = "SOX"
  }
}
```

### **6. Melhores Práticas de IaC**

**Estrutura de Projeto:**
```
terraform/
├── environments/
│   ├── development/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   └── production/
├── modules/
│   ├── vpc/
│   ├── ecs/
│   ├── rds/
│   └── monitoring/
├── scripts/
│   ├── init.sh
│   ├── plan.sh
│   └── apply.sh
└── policies/
    ├── security.json
    └── compliance.json
```

**Código de Qualidade:**
```hcl
# terraform/scripts/validate.sh
#!/bin/bash

echo "🔍 Validando código Terraform..."

# Formatter
echo "Formatando código..."
terraform fmt -recursive

# Validação
echo "Validando configuração..."
terraform validate

# Security scan
echo "Verificando segurança..."
tfsec .

# Compliance check
echo "Verificando compliance..."
terragrunt hclfmt

# Policy validation
echo "Validando políticas..."
opa test policies/

echo "✅ Validação concluída!"
```

---

## 🎯 **Exercícios Práticos**

### **Exercício 1: Pipeline Completa**
**Cenário:** Criar pipeline que vai do commit ao deploy em produção.

**Sua tarefa:**
- Configure GitHub Actions com todos os stages
- Implemente testes automatizados
- Configure deploy blue-green
- Adicione notificações

### **Exercício 2: Kubernetes Production-Ready**
**Cenário:** Deploy de aplicação bancária em Kubernetes.

**Sua tarefa:**
- Configure deployment com HPA
- Implemente health checks
- Configure ingress com SSL
- Adicione monitoramento

### **Exercício 3: Observabilidade Completa**
**Cenário:** Sistema de monitoramento para detectar problemas.

**Sua tarefa:**
- Configure Prometheus e Grafana
- Crie dashboards customizados
- Implemente alertas inteligentes
- Configure distributed tracing

### **Exercício 4: Infrastructure as Code**
**Cenário:** Provisionar infraestrutura completa para sistema bancário.

**Sua tarefa:**
- Crie módulos Terraform reutilizáveis
- Configure múltiplos ambientes (dev/staging/prod)
- Implemente pipeline de IaC
- Configure state management e locking
- Implemente validação e testes de infraestrutura

---

## 🚀 **Próximos Passos**

1. **Pratique com minikube** - Ambiente Kubernetes local
2. **Domine Terraform** - Crie módulos reutilizáveis
3. **Implemente observabilidade** - Métricas e logs
4. **Estude qualidade de código** - Próximo: 05-Qualidade-Codigo.md
5. **Automatize tudo** - Infrastructure as Code

**Lembre-se:** DevOps é cultura, não apenas ferramenta! 