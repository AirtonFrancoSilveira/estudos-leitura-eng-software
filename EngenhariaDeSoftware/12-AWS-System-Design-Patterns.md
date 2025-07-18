# ☁️ AWS System Design Patterns - Guia Prático

## 📚 **Objetivos de Aprendizado**

Ao final deste módulo, você será capaz de:
- ✅ Implementar padrões de System Design usando serviços AWS
- ✅ Escolher os serviços AWS adequados para cada padrão
- ✅ Projetar arquiteturas cloud-native resilientes e escaláveis
- ✅ Aplicar o AWS Well-Architected Framework
- ✅ Resolver problemas reais de System Design com AWS

---

## 🎯 **Conceito 1: Padrões de Resiliência com AWS**

### **🔍 O que são?**
Padrões de resiliência na AWS são como sistemas de backup de uma usina elétrica: garantem que seus serviços continuem funcionando mesmo quando componentes individuais falham.

### **🎓 Padrões Fundamentais:**

**1. Circuit Breaker Pattern com AWS**

### **🎯 Quando Usar:**
- ✅ **Integração com serviços externos** (APIs de terceiros, sistemas on-premise)
- ✅ **Microserviços com dependências** (entre serviços Lambda, containers)
- ✅ **Sistemas críticos** (banking, healthcare, e-commerce)
- ✅ **Alta variabilidade de latência** (serviços que podem ter picos)

### **💡 Implementação AWS:**

```python
# AWS Lambda com Circuit Breaker usando DynamoDB
import boto3
import json
import time
from decimal import Decimal
from enum import Enum

class CircuitState(Enum):
    CLOSED = "CLOSED"
    OPEN = "OPEN"
    HALF_OPEN = "HALF_OPEN"

class AWSCircuitBreaker:
    def __init__(self, 
                 service_name: str,
                 failure_threshold: int = 5,
                 success_threshold: int = 2,
                 timeout: int = 60):
        self.service_name = service_name
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table('circuit-breaker-state')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def get_state(self):
        try:
            response = self.table.get_item(
                Key={'service_name': self.service_name}
            )
            
            if 'Item' not in response:
                # Inicializar estado
                self.table.put_item(
                    Item={
                        'service_name': self.service_name,
                        'state': CircuitState.CLOSED.value,
                        'failure_count': 0,
                        'success_count': 0,
                        'last_failure_time': 0
                    }
                )
                return CircuitState.CLOSED
            
            item = response['Item']
            return CircuitState(item['state'])
        
        except Exception as e:
            print(f"Error getting circuit state: {e}")
            return CircuitState.CLOSED
    
    def can_execute(self):
        state = self.get_state()
        
        if state == CircuitState.CLOSED:
            return True
        
        if state == CircuitState.OPEN:
            # Verificar se pode tentar half-open
            response = self.table.get_item(
                Key={'service_name': self.service_name}
            )
            
            if 'Item' in response:
                last_failure = response['Item']['last_failure_time']
                current_time = int(time.time())
                
                if current_time - last_failure > self.timeout:
                    # Transição para HALF_OPEN
                    self.table.update_item(
                        Key={'service_name': self.service_name},
                        UpdateExpression='SET #state = :state',
                        ExpressionAttributeNames={'#state': 'state'},
                        ExpressionAttributeValues={':state': CircuitState.HALF_OPEN.value}
                    )
                    return True
            
            return False
        
        if state == CircuitState.HALF_OPEN:
            return True
        
        return False
    
    def record_success(self):
        state = self.get_state()
        
        if state == CircuitState.HALF_OPEN:
            # Incrementar success count
            response = self.table.update_item(
                Key={'service_name': self.service_name},
                UpdateExpression='ADD success_count :inc SET failure_count = :zero',
                ExpressionAttributeValues={
                    ':inc': 1,
                    ':zero': 0
                },
                ReturnValues='ALL_NEW'
            )
            
            if response['Attributes']['success_count'] >= self.success_threshold:
                # Transição para CLOSED
                self.table.update_item(
                    Key={'service_name': self.service_name},
                    UpdateExpression='SET #state = :state, success_count = :zero',
                    ExpressionAttributeNames={'#state': 'state'},
                    ExpressionAttributeValues={
                        ':state': CircuitState.CLOSED.value,
                        ':zero': 0
                    }
                )
        
        # Enviar métrica para CloudWatch
        self.cloudwatch.put_metric_data(
            Namespace='CircuitBreaker',
            MetricData=[
                {
                    'MetricName': 'Success',
                    'Dimensions': [
                        {
                            'Name': 'ServiceName',
                            'Value': self.service_name
                        }
                    ],
                    'Value': 1,
                    'Unit': 'Count'
                }
            ]
        )
    
    def record_failure(self):
        current_time = int(time.time())
        
        response = self.table.update_item(
            Key={'service_name': self.service_name},
            UpdateExpression='ADD failure_count :inc SET last_failure_time = :time, success_count = :zero',
            ExpressionAttributeValues={
                ':inc': 1,
                ':time': current_time,
                ':zero': 0
            },
            ReturnValues='ALL_NEW'
        )
        
        if response['Attributes']['failure_count'] >= self.failure_threshold:
            # Transição para OPEN
            self.table.update_item(
                Key={'service_name': self.service_name},
                UpdateExpression='SET #state = :state',
                ExpressionAttributeNames={'#state': 'state'},
                ExpressionAttributeValues={':state': CircuitState.OPEN.value}
            )
        
        # Enviar métrica para CloudWatch
        self.cloudwatch.put_metric_data(
            Namespace='CircuitBreaker',
            MetricData=[
                {
                    'MetricName': 'Failure',
                    'Dimensions': [
                        {
                            'Name': 'ServiceName',
                            'Value': self.service_name
                        }
                    ],
                    'Value': 1,
                    'Unit': 'Count'
                }
            ]
        )

# Lambda Handler com Circuit Breaker
def lambda_handler(event, context):
    circuit_breaker = AWSCircuitBreaker('external-payment-service')
    
    if not circuit_breaker.can_execute():
        # Fallback: processar offline ou usar cache
        return {
            'statusCode': 503,
            'body': json.dumps({
                'message': 'Payment service temporarily unavailable',
                'fallback': 'Payment queued for later processing'
            })
        }
    
    try:
        # Chamar serviço externo
        result = call_external_payment_service(event)
        
        circuit_breaker.record_success()
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    
    except Exception as e:
        circuit_breaker.record_failure()
        
        # Fallback strategy
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Payment processed via fallback',
                'transaction_id': 'fallback-' + str(int(time.time()))
            })
        }

def call_external_payment_service(event):
    # Simular chamada para serviço externo
    import random
    
    if random.random() < 0.3:  # 30% de chance de falha
        raise Exception("External service error")
    
    return {
        'transaction_id': 'tx-' + str(int(time.time())),
        'status': 'success',
        'amount': event.get('amount', 100)
    }
```

**CloudFormation Template para Circuit Breaker:**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Circuit Breaker Infrastructure'

Resources:
  # DynamoDB Table para estado do Circuit Breaker
  CircuitBreakerTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: circuit-breaker-state
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: service_name
          AttributeType: S
      KeySchema:
        - AttributeName: service_name
          KeyType: HASH
      Tags:
        - Key: Purpose
          Value: CircuitBreaker

  # Lambda Function
  CircuitBreakerLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: circuit-breaker-handler
      Runtime: python3.9
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          # Code here (previous Python code)
      Environment:
        Variables:
          CIRCUIT_BREAKER_TABLE: !Ref CircuitBreakerTable
      Timeout: 30

  # IAM Role para Lambda
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: CircuitBreakerPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:GetItem
                  - dynamodb:PutItem
                  - dynamodb:UpdateItem
                Resource: !GetAtt CircuitBreakerTable.Arn
              - Effect: Allow
                Action:
                  - cloudwatch:PutMetricData
                Resource: '*'

  # CloudWatch Alarms
  CircuitBreakerOpenAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: CircuitBreakerOpen
      AlarmDescription: 'Circuit breaker is open'
      MetricName: Failure
      Namespace: CircuitBreaker
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 5
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref CircuitBreakerTopic

  # SNS Topic para notificações
  CircuitBreakerTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: circuit-breaker-alerts
      DisplayName: Circuit Breaker Alerts
```

**2. Bulkhead Pattern com AWS**

### **🎯 Quando Usar:**
- ✅ **Aplicações com múltiplas funcionalidades** (diferentes SLAs)
- ✅ **Isolamento de carga** (crítico vs. não-crítico)
- ✅ **Diferentes patterns de acesso** (read-heavy vs. write-heavy)
- ✅ **Multi-tenant applications** (isolamento por tenant)

### **💡 Implementação AWS:**

```python
# Bulkhead Pattern usando múltiplas SQS Queues e Lambda Functions
import boto3
import json
from typing import Dict, Any

class AWSBulkheadProcessor:
    def __init__(self):
        self.sqs = boto3.client('sqs')
        self.lambda_client = boto3.client('lambda')
        
        # Diferentes queues para diferentes tipos de operação
        self.queues = {
            'critical': 'https://sqs.us-east-1.amazonaws.com/123456789012/critical-operations',
            'standard': 'https://sqs.us-east-1.amazonaws.com/123456789012/standard-operations',
            'background': 'https://sqs.us-east-1.amazonaws.com/123456789012/background-operations',
            'analytics': 'https://sqs.us-east-1.amazonaws.com/123456789012/analytics-operations'
        }
        
        # Diferentes Lambda functions para cada bulkhead
        self.processors = {
            'critical': 'critical-processor',
            'standard': 'standard-processor',
            'background': 'background-processor',
            'analytics': 'analytics-processor'
        }
    
    def route_operation(self, operation_type: str, payload: Dict[str, Any]):
        """Route operation to appropriate bulkhead"""
        
        # Determinar prioridade baseado no tipo de operação
        priority = self.get_operation_priority(operation_type)
        
        # Enviar para queue apropriada
        queue_url = self.queues[priority]
        
        message = {
            'operation_type': operation_type,
            'payload': payload,
            'priority': priority,
            'timestamp': int(time.time())
        }
        
        self.sqs.send_message(
            QueueUrl=queue_url,
            MessageBody=json.dumps(message),
            MessageAttributes={
                'Priority': {
                    'StringValue': priority,
                    'DataType': 'String'
                }
            }
        )
    
    def get_operation_priority(self, operation_type: str) -> str:
        """Determine operation priority based on type"""
        
        critical_operations = [
            'payment_processing',
            'fund_transfer',
            'security_alert',
            'fraud_detection'
        ]
        
        standard_operations = [
            'account_creation',
            'profile_update',
            'transaction_history',
            'balance_inquiry'
        ]
        
        background_operations = [
            'report_generation',
            'data_backup',
            'cleanup_tasks',
            'batch_processing'
        ]
        
        if operation_type in critical_operations:
            return 'critical'
        elif operation_type in standard_operations:
            return 'standard'
        elif operation_type in background_operations:
            return 'background'
        else:
            return 'analytics'

# Lambda Handler para operações críticas
def critical_processor_handler(event, context):
    """Process critical operations with high priority"""
    
    for record in event['Records']:
        try:
            message = json.loads(record['body'])
            operation_type = message['operation_type']
            payload = message['payload']
            
            # Processar operação crítica
            result = process_critical_operation(operation_type, payload)
            
            # Log resultado
            print(f"Critical operation {operation_type} completed: {result}")
            
        except Exception as e:
            # Para operações críticas, re-processar imediatamente
            print(f"Critical operation failed: {e}")
            # Implementar DLQ ou retry imediato
            
    return {'statusCode': 200, 'body': 'Critical operations processed'}

def process_critical_operation(operation_type: str, payload: Dict[str, Any]):
    """Process high-priority operations"""
    
    if operation_type == 'payment_processing':
        return process_payment(payload)
    elif operation_type == 'fund_transfer':
        return process_transfer(payload)
    elif operation_type == 'security_alert':
        return process_security_alert(payload)
    elif operation_type == 'fraud_detection':
        return process_fraud_detection(payload)
    
    return {'status': 'unknown_operation'}

# Lambda Handler para operações em background
def background_processor_handler(event, context):
    """Process background operations with lower priority"""
    
    for record in event['Records']:
        try:
            message = json.loads(record['body'])
            operation_type = message['operation_type']
            payload = message['payload']
            
            # Processar operação em background
            result = process_background_operation(operation_type, payload)
            
            print(f"Background operation {operation_type} completed: {result}")
            
        except Exception as e:
            # Para operações em background, tolerância a falhas
            print(f"Background operation failed (will retry): {e}")
            
    return {'statusCode': 200, 'body': 'Background operations processed'}
```

**CloudFormation Template para Bulkhead:**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Bulkhead Pattern Implementation'

Resources:
  # Critical Operations Queue
  CriticalOperationsQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: critical-operations
      VisibilityTimeoutSeconds: 60
      MessageRetentionPeriod: 1209600  # 14 days
      ReceiveMessageWaitTimeSeconds: 20
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt CriticalOperationsDLQ.Arn
        maxReceiveCount: 3

  CriticalOperationsDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: critical-operations-dlq
      MessageRetentionPeriod: 1209600

  # Standard Operations Queue  
  StandardOperationsQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: standard-operations
      VisibilityTimeoutSeconds: 300
      MessageRetentionPeriod: 1209600
      ReceiveMessageWaitTimeSeconds: 20
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt StandardOperationsDLQ.Arn
        maxReceiveCount: 5

  StandardOperationsDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: standard-operations-dlq

  # Background Operations Queue
  BackgroundOperationsQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: background-operations
      VisibilityTimeoutSeconds: 900
      MessageRetentionPeriod: 1209600
      ReceiveMessageWaitTimeSeconds: 20
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt BackgroundOperationsDLQ.Arn
        maxReceiveCount: 10

  BackgroundOperationsDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: background-operations-dlq

  # Critical Operations Processor
  CriticalProcessor:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: critical-processor
      Runtime: python3.9
      Handler: index.critical_processor_handler
      Role: !GetAtt ProcessorRole.Arn
      ReservedConcurrencyLimit: 50  # Reservar recursos para operações críticas
      Timeout: 60
      Code:
        ZipFile: |
          # Critical processor code here

  # Standard Operations Processor
  StandardProcessor:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: standard-processor
      Runtime: python3.9
      Handler: index.standard_processor_handler
      Role: !GetAtt ProcessorRole.Arn
      ReservedConcurrencyLimit: 30
      Timeout: 300
      Code:
        ZipFile: |
          # Standard processor code here

  # Background Operations Processor
  BackgroundProcessor:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: background-processor
      Runtime: python3.9
      Handler: index.background_processor_handler
      Role: !GetAtt ProcessorRole.Arn
      ReservedConcurrencyLimit: 10  # Menor prioridade
      Timeout: 900
      Code:
        ZipFile: |
          # Background processor code here

  # Event Source Mappings
  CriticalEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt CriticalOperationsQueue.Arn
      FunctionName: !Ref CriticalProcessor
      BatchSize: 1  # Processar um por vez para máxima prioridade

  StandardEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt StandardOperationsQueue.Arn
      FunctionName: !Ref StandardProcessor
      BatchSize: 10

  BackgroundEventSourceMapping:
    Type: AWS::Lambda::EventSourceMapping
    Properties:
      EventSourceArn: !GetAtt BackgroundOperationsQueue.Arn
      FunctionName: !Ref BackgroundProcessor
      BatchSize: 10

  # IAM Role
  ProcessorRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: SQSAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - sqs:ReceiveMessage
                  - sqs:DeleteMessage
                  - sqs:GetQueueAttributes
                  - sqs:SendMessage
                Resource: 
                  - !GetAtt CriticalOperationsQueue.Arn
                  - !GetAtt StandardOperationsQueue.Arn
                  - !GetAtt BackgroundOperationsQueue.Arn
                  - !GetAtt CriticalOperationsDLQ.Arn
                  - !GetAtt StandardOperationsDLQ.Arn
                  - !GetAtt BackgroundOperationsDLQ.Arn
```

**3. Retry Pattern com AWS**

### **💡 Implementação AWS:**

```python
# Retry Pattern usando AWS Step Functions
import boto3
import json
import time
from typing import Dict, Any, Optional

class AWSRetryHandler:
    def __init__(self):
        self.stepfunctions = boto3.client('stepfunctions')
        self.lambda_client = boto3.client('lambda')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def execute_with_retry(self, 
                          operation_name: str, 
                          payload: Dict[str, Any],
                          max_attempts: int = 3,
                          backoff_multiplier: float = 2.0,
                          initial_delay: int = 1) -> Dict[str, Any]:
        """Execute operation with retry logic using Step Functions"""
        
        state_machine_input = {
            'operation_name': operation_name,
            'payload': payload,
            'retry_config': {
                'max_attempts': max_attempts,
                'backoff_multiplier': backoff_multiplier,
                'initial_delay': initial_delay,
                'current_attempt': 0
            }
        }
        
        # Iniciar Step Function
        response = self.stepfunctions.start_execution(
            stateMachineArn='arn:aws:states:us-east-1:123456789012:stateMachine:retry-state-machine',
            name=f"retry-{operation_name}-{int(time.time())}",
            input=json.dumps(state_machine_input)
        )
        
        return response

# Lambda para operações que podem falhar
def operation_handler(event, context):
    """Handler para operações que podem precisar de retry"""
    
    operation_name = event['operation_name']
    payload = event['payload']
    retry_config = event.get('retry_config', {})
    
    try:
        # Executar operação específica
        result = execute_operation(operation_name, payload)
        
        # Sucesso - retornar resultado
        return {
            'statusCode': 200,
            'success': True,
            'result': result,
            'attempts': retry_config.get('current_attempt', 0) + 1
        }
        
    except RetryableException as e:
        # Erro que pode ser retentado
        current_attempt = retry_config.get('current_attempt', 0)
        max_attempts = retry_config.get('max_attempts', 3)
        
        if current_attempt < max_attempts:
            # Calcular delay para próxima tentativa
            backoff_multiplier = retry_config.get('backoff_multiplier', 2.0)
            initial_delay = retry_config.get('initial_delay', 1)
            delay = initial_delay * (backoff_multiplier ** current_attempt)
            
            return {
                'statusCode': 500,
                'success': False,
                'error': str(e),
                'retryable': True,
                'delay': delay,
                'attempts': current_attempt + 1
            }
        else:
            # Máximo de tentativas atingido
            return {
                'statusCode': 500,
                'success': False,
                'error': f"Max attempts reached: {str(e)}",
                'retryable': False,
                'attempts': current_attempt + 1
            }
    
    except NonRetryableException as e:
        # Erro que não deve ser retentado
        return {
            'statusCode': 400,
            'success': False,
            'error': str(e),
            'retryable': False,
            'attempts': retry_config.get('current_attempt', 0) + 1
        }

def execute_operation(operation_name: str, payload: Dict[str, Any]) -> Dict[str, Any]:
    """Execute specific operation based on operation name"""
    
    if operation_name == 'external_api_call':
        return call_external_api(payload)
    elif operation_name == 'database_operation':
        return perform_database_operation(payload)
    elif operation_name == 'file_upload':
        return upload_file_to_s3(payload)
    else:
        raise NonRetryableException(f"Unknown operation: {operation_name}")

def call_external_api(payload: Dict[str, Any]) -> Dict[str, Any]:
    """Simulate external API call that might fail"""
    import random
    import requests
    
    # Simular diferentes tipos de erro
    rand = random.random()
    
    if rand < 0.3:  # 30% chance de erro retryable
        raise RetryableException("Timeout connecting to external API")
    elif rand < 0.1:  # 10% chance de erro não-retryable
        raise NonRetryableException("Invalid API key")
    else:
        # Sucesso
        return {
            'api_response': 'success',
            'data': payload
        }

class RetryableException(Exception):
    """Exception that indicates operation can be retried"""
    pass

class NonRetryableException(Exception):
    """Exception that indicates operation should not be retried"""
    pass
```

**Step Functions State Machine Definition:**

```json
{
  "Comment": "Retry Pattern State Machine",
  "StartAt": "ExecuteOperation",
  "States": {
    "ExecuteOperation": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:operation-handler",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 0,
          "BackoffRate": 1.0
        }
      ],
      "Next": "CheckResult"
    },
    "CheckResult": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.success",
          "BooleanEquals": true,
          "Next": "SuccessState"
        },
        {
          "And": [
            {
              "Variable": "$.success",
              "BooleanEquals": false
            },
            {
              "Variable": "$.retryable",
              "BooleanEquals": true
            }
          ],
          "Next": "WaitForRetry"
        }
      ],
      "Default": "FailureState"
    },
    "WaitForRetry": {
      "Type": "Wait",
      "SecondsPath": "$.delay",
      "Next": "UpdateRetryCount"
    },
    "UpdateRetryCount": {
      "Type": "Pass",
      "Parameters": {
        "operation_name.$": "$.operation_name",
        "payload.$": "$.payload",
        "retry_config": {
          "max_attempts.$": "$.retry_config.max_attempts",
          "backoff_multiplier.$": "$.retry_config.backoff_multiplier",
          "initial_delay.$": "$.retry_config.initial_delay",
          "current_attempt.$": "$.attempts"
        }
      },
      "Next": "ExecuteOperation"
    },
    "SuccessState": {
      "Type": "Succeed"
    },
    "FailureState": {
      "Type": "Fail",
      "Cause": "Operation failed after retries"
    }
  }
}
```

### **⚖️ Trade-offs AWS:**
- **✅ Prós:** Gerenciamento automático de estado, observabilidade nativa, escalabilidade
- **❌ Contras:** Custos adicionais, complexidade de configuração, vendor lock-in

---

## 🎯 **Conceito 2: Padrões de Escalabilidade com AWS**

### **🔍 O que são?**
Padrões de escalabilidade na AWS são como sistemas de transporte público: permitem que sua aplicação atenda milhões de usuários usando a elasticidade da nuvem.

### **🎓 Padrões Fundamentais:**

**1. Auto Scaling Pattern**

### **💡 Implementação AWS:**

```python
# Auto Scaling baseado em métricas customizadas
import boto3
import json
from typing import Dict, Any

class AWSAutoScalingManager:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.autoscaling = boto3.client('autoscaling')
        self.ecs = boto3.client('ecs')
        self.lambda_client = boto3.client('lambda')
    
    def setup_custom_metrics_scaling(self, 
                                   service_name: str,
                                   metric_name: str,
                                   target_value: float):
        """Setup auto scaling based on custom metrics"""
        
        # Criar custom metric
        self.cloudwatch.put_metric_data(
            Namespace='CustomApp',
            MetricData=[
                {
                    'MetricName': metric_name,
                    'Value': 0,
                    'Unit': 'Count',
                    'Dimensions': [
                        {
                            'Name': 'ServiceName',
                            'Value': service_name
                        }
                    ]
                }
            ]
        )
        
        # Configurar CloudWatch Alarm para Scale Out
        self.cloudwatch.put_metric_alarm(
            AlarmName=f'{service_name}-scale-out',
            ComparisonOperator='GreaterThanThreshold',
            EvaluationPeriods=2,
            MetricName=metric_name,
            Namespace='CustomApp',
            Period=60,
            Statistic='Average',
            Threshold=target_value * 1.2,  # 20% acima do target
            ActionsEnabled=True,
            AlarmActions=[
                f'arn:aws:autoscaling:us-east-1:123456789012:scalingPolicy:scale-out-policy'
            ],
            AlarmDescription=f'Scale out alarm for {service_name}',
            Dimensions=[
                {
                    'Name': 'ServiceName',
                    'Value': service_name
                }
            ]
        )
        
        # Configurar CloudWatch Alarm para Scale In
        self.cloudwatch.put_metric_alarm(
            AlarmName=f'{service_name}-scale-in',
            ComparisonOperator='LessThanThreshold',
            EvaluationPeriods=5,  # Mais conservador para scale in
            MetricName=metric_name,
            Namespace='CustomApp',
            Period=60,
            Statistic='Average',
            Threshold=target_value * 0.8,  # 20% abaixo do target
            ActionsEnabled=True,
            AlarmActions=[
                f'arn:aws:autoscaling:us-east-1:123456789012:scalingPolicy:scale-in-policy'
            ],
            AlarmDescription=f'Scale in alarm for {service_name}',
            Dimensions=[
                {
                    'Name': 'ServiceName',
                    'Value': service_name
                }
            ]
        )
    
    def publish_queue_depth_metric(self, queue_url: str, service_name: str):
        """Publish queue depth metric for auto scaling"""
        
        sqs = boto3.client('sqs')
        
        # Obter atributos da queue
        response = sqs.get_queue_attributes(
            QueueUrl=queue_url,
            AttributeNames=['ApproximateNumberOfMessages']
        )
        
        queue_depth = int(response['Attributes']['ApproximateNumberOfMessages'])
        
        # Publicar métrica
        self.cloudwatch.put_metric_data(
            Namespace='CustomApp',
            MetricData=[
                {
                    'MetricName': 'QueueDepth',
                    'Value': queue_depth,
                    'Unit': 'Count',
                    'Dimensions': [
                        {
                            'Name': 'ServiceName',
                            'Value': service_name
                        }
                    ]
                }
            ]
        )
        
        return queue_depth
    
    def setup_predictive_scaling(self, auto_scaling_group_name: str):
        """Setup predictive scaling for known traffic patterns"""
        
        # Configurar scheduled scaling para horários de pico
        self.autoscaling.put_scheduled_update_group_action(
            AutoScalingGroupName=auto_scaling_group_name,
            ScheduledActionName='morning-peak-scale-out',
            Recurrence='0 8 * * MON-FRI',  # 8 AM de segunda a sexta
            MinSize=10,
            MaxSize=50,
            DesiredCapacity=20
        )
        
        self.autoscaling.put_scheduled_update_group_action(
            AutoScalingGroupName=auto_scaling_group_name,
            ScheduledActionName='evening-scale-in',
            Recurrence='0 20 * * *',  # 8 PM todos os dias
            MinSize=2,
            MaxSize=10,
            DesiredCapacity=5
        )
        
        # Configurar scaling para Black Friday
        self.autoscaling.put_scheduled_update_group_action(
            AutoScalingGroupName=auto_scaling_group_name,
            ScheduledActionName='black-friday-preparation',
            StartTime='2024-11-28T00:00:00Z',
            EndTime='2024-11-30T23:59:59Z',
            MinSize=50,
            MaxSize=200,
            DesiredCapacity=100
        )

# Lambda para métricas customizadas
def metrics_publisher_handler(event, context):
    """Lambda que publica métricas customizadas para auto scaling"""
    
    autoscaling_manager = AWSAutoScalingManager()
    
    # Publicar métricas de diferentes serviços
    services = [
        {
            'name': 'payment-processor',
            'queue_url': 'https://sqs.us-east-1.amazonaws.com/123456789012/payment-queue'
        },
        {
            'name': 'order-processor',
            'queue_url': 'https://sqs.us-east-1.amazonaws.com/123456789012/order-queue'
        },
        {
            'name': 'notification-service',
            'queue_url': 'https://sqs.us-east-1.amazonaws.com/123456789012/notification-queue'
        }
    ]
    
    for service in services:
        try:
            queue_depth = autoscaling_manager.publish_queue_depth_metric(
                service['queue_url'], 
                service['name']
            )
            
            print(f"Published queue depth metric for {service['name']}: {queue_depth}")
            
        except Exception as e:
            print(f"Error publishing metrics for {service['name']}: {e}")
    
    return {'statusCode': 200, 'body': 'Metrics published successfully'}

# Lambda para decisões de scaling inteligente
def intelligent_scaling_handler(event, context):
    """Lambda que toma decisões de scaling baseadas em múltiplas métricas"""
    
    cloudwatch = boto3.client('cloudwatch')
    autoscaling = boto3.client('autoscaling')
    
    # Obter métricas dos últimos 5 minutos
    end_time = datetime.utcnow()
    start_time = end_time - timedelta(minutes=5)
    
    # Métricas de CPU, memória e queue depth
    cpu_metrics = cloudwatch.get_metric_statistics(
        Namespace='AWS/EC2',
        MetricName='CPUUtilization',
        Dimensions=[
            {
                'Name': 'AutoScalingGroupName',
                'Value': 'payment-processor-asg'
            }
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=60,
        Statistics=['Average']
    )
    
    queue_metrics = cloudwatch.get_metric_statistics(
        Namespace='CustomApp',
        MetricName='QueueDepth',
        Dimensions=[
            {
                'Name': 'ServiceName',
                'Value': 'payment-processor'
            }
        ],
        StartTime=start_time,
        EndTime=end_time,
        Period=60,
        Statistics=['Average']
    )
    
    # Lógica de decisão inteligente
    if cpu_metrics['Datapoints'] and queue_metrics['Datapoints']:
        avg_cpu = cpu_metrics['Datapoints'][-1]['Average']
        avg_queue_depth = queue_metrics['Datapoints'][-1]['Average']
        
        # Decisão baseada em múltiplas métricas
        if avg_cpu > 70 and avg_queue_depth > 100:
            # Scale out agressivo
            scale_out_capacity = min(50, int(avg_queue_depth / 10))
            
            autoscaling.set_desired_capacity(
                AutoScalingGroupName='payment-processor-asg',
                DesiredCapacity=scale_out_capacity,
                HonorCooldown=False
            )
            
            print(f"Aggressive scale out to {scale_out_capacity} instances")
            
        elif avg_cpu < 30 and avg_queue_depth < 10:
            # Scale in conservativo
            current_capacity = get_current_capacity('payment-processor-asg')
            new_capacity = max(2, current_capacity - 1)
            
            autoscaling.set_desired_capacity(
                AutoScalingGroupName='payment-processor-asg',
                DesiredCapacity=new_capacity,
                HonorCooldown=True
            )
            
            print(f"Conservative scale in to {new_capacity} instances")
    
    return {'statusCode': 200, 'body': 'Scaling decisions processed'}

def get_current_capacity(asg_name: str) -> int:
    """Get current capacity of Auto Scaling Group"""
    autoscaling = boto3.client('autoscaling')
    
    response = autoscaling.describe_auto_scaling_groups(
        AutoScalingGroupNames=[asg_name]
    )
    
    if response['AutoScalingGroups']:
        return response['AutoScalingGroups'][0]['DesiredCapacity']
    
    return 0
```

**2. Database Read Replicas Pattern**

### **💡 Implementação AWS:**

```python
# Read Replicas Pattern com RDS e Connection Pooling
import boto3
import random
from typing import List, Dict, Any
import psycopg2
from psycopg2 import pool

class AWSDatabaseManager:
    def __init__(self):
        self.rds = boto3.client('rds')
        self.cloudwatch = boto3.client('cloudwatch')
        
        # Connection pools para diferentes tipos de operação
        self.write_pool = psycopg2.pool.SimpleConnectionPool(
            1, 20,
            host='master-db.cluster-xyz.us-east-1.rds.amazonaws.com',
            database='appdb',
            user='admin',
            password='password'
        )
        
        self.read_pools = [
            psycopg2.pool.SimpleConnectionPool(
                1, 10,
                host='read-replica-1.cluster-xyz.us-east-1.rds.amazonaws.com',
                database='appdb',
                user='readonly',
                password='password'
            ),
            psycopg2.pool.SimpleConnectionPool(
                1, 10,
                host='read-replica-2.cluster-xyz.us-east-1.rds.amazonaws.com',
                database='appdb',
                user='readonly',
                password='password'
            ),
            psycopg2.pool.SimpleConnectionPool(
                1, 10,
                host='read-replica-3.cluster-xyz.us-east-1.rds.amazonaws.com',
                database='appdb',
                user='readonly',
                password='password'
            )
        ]
    
    def execute_read_query(self, query: str, params: tuple = None) -> List[Dict[str, Any]]:
        """Execute read query using read replica with load balancing"""
        
        # Selecionar read replica com menor carga
        replica_pool = self.select_best_read_replica()
        
        connection = replica_pool.getconn()
        
        try:
            cursor = connection.cursor()
            cursor.execute(query, params)
            
            columns = [desc[0] for desc in cursor.description]
            results = []
            
            for row in cursor.fetchall():
                results.append(dict(zip(columns, row)))
            
            # Métrica de performance
            self.cloudwatch.put_metric_data(
                Namespace='Database',
                MetricData=[
                    {
                        'MetricName': 'ReadQueryExecuted',
                        'Value': 1,
                        'Unit': 'Count'
                    }
                ]
            )
            
            return results
            
        finally:
            replica_pool.putconn(connection)
    
    def execute_write_query(self, query: str, params: tuple = None) -> bool:
        """Execute write query on master database"""
        
        connection = self.write_pool.getconn()
        
        try:
            cursor = connection.cursor()
            cursor.execute(query, params)
            connection.commit()
            
            # Métrica de performance
            self.cloudwatch.put_metric_data(
                Namespace='Database',
                MetricData=[
                    {
                        'MetricName': 'WriteQueryExecuted',
                        'Value': 1,
                        'Unit': 'Count'
                    }
                ]
            )
            
            return True
            
        except Exception as e:
            connection.rollback()
            raise e
        finally:
            self.write_pool.putconn(connection)
    
    def select_best_read_replica(self) -> psycopg2.pool.SimpleConnectionPool:
        """Select read replica with best performance"""
        
        # Estratégia simples: round-robin
        # Em produção, usar métricas de CPU/connections
        return random.choice(self.read_pools)
    
    def monitor_replica_lag(self):
        """Monitor replica lag and alert if necessary"""
        
        replica_identifiers = [
            'read-replica-1',
            'read-replica-2', 
            'read-replica-3'
        ]
        
        for replica_id in replica_identifiers:
            try:
                # Obter métricas de replica lag
                response = self.cloudwatch.get_metric_statistics(
                    Namespace='AWS/RDS',
                    MetricName='ReplicaLag',
                    Dimensions=[
                        {
                            'Name': 'DBInstanceIdentifier',
                            'Value': replica_id
                        }
                    ],
                    StartTime=datetime.utcnow() - timedelta(minutes=5),
                    EndTime=datetime.utcnow(),
                    Period=300,
                    Statistics=['Average']
                )
                
                if response['Datapoints']:
                    lag = response['Datapoints'][-1]['Average']
                    
                    # Alertar se lag > 5 segundos
                    if lag > 5:
                        self.cloudwatch.put_metric_data(
                            Namespace='Database',
                            MetricData=[
                                {
                                    'MetricName': 'HighReplicaLag',
                                    'Value': lag,
                                    'Unit': 'Seconds',
                                    'Dimensions': [
                                        {
                                            'Name': 'ReplicaId',
                                            'Value': replica_id
                                        }
                                    ]
                                }
                            ]
                        )
                        
            except Exception as e:
                print(f"Error monitoring replica {replica_id}: {e}")

# Lambda para operações de leitura
def read_operations_handler(event, context):
    """Handler para operações de leitura usando read replicas"""
    
    db_manager = AWSDatabaseManager()
    
    operation = event.get('operation')
    
    if operation == 'get_user_profile':
        user_id = event.get('user_id')
        
        results = db_manager.execute_read_query(
            "SELECT * FROM users WHERE id = %s",
            (user_id,)
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps(results)
        }
    
    elif operation == 'get_transaction_history':
        account_id = event.get('account_id')
        limit = event.get('limit', 50)
        
        results = db_manager.execute_read_query(
            "SELECT * FROM transactions WHERE account_id = %s ORDER BY created_at DESC LIMIT %s",
            (account_id, limit)
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps(results)
        }
    
    elif operation == 'generate_report':
        # Operação pesada - usar read replica dedicada
        results = db_manager.execute_read_query(
            """
            SELECT 
                DATE(created_at) as date,
                COUNT(*) as transaction_count,
                SUM(amount) as total_amount
            FROM transactions 
            WHERE created_at >= NOW() - INTERVAL '30 days'
            GROUP BY DATE(created_at)
            ORDER BY date DESC
            """
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps(results)
        }
    
    return {
        'statusCode': 400,
        'body': json.dumps({'error': 'Unknown operation'})
    }

# Lambda para operações de escrita
def write_operations_handler(event, context):
    """Handler para operações de escrita no master database"""
    
    db_manager = AWSDatabaseManager()
    
    operation = event.get('operation')
    
    if operation == 'create_user':
        user_data = event.get('user_data')
        
        success = db_manager.execute_write_query(
            "INSERT INTO users (name, email, created_at) VALUES (%s, %s, %s)",
            (user_data['name'], user_data['email'], datetime.utcnow())
        )
        
        if success:
            return {
                'statusCode': 201,
                'body': json.dumps({'message': 'User created successfully'})
            }
    
    elif operation == 'process_transaction':
        transaction_data = event.get('transaction_data')
        
        success = db_manager.execute_write_query(
            "INSERT INTO transactions (account_id, amount, type, created_at) VALUES (%s, %s, %s, %s)",
            (
                transaction_data['account_id'],
                transaction_data['amount'],
                transaction_data['type'],
                datetime.utcnow()
            )
        )
        
        if success:
            return {
                'statusCode': 201,
                'body': json.dumps({'message': 'Transaction processed successfully'})
            }
    
    return {
        'statusCode': 400,
        'body': json.dumps({'error': 'Unknown operation or failed to execute'})
    }
```

### **⚖️ Trade-offs AWS:**
- **✅ Prós:** Escalabilidade automática, custos otimizados, observabilidade nativa
- **❌ Contras:** Eventual consistency, complexidade de configuração, custos de transferência

---

## 🎯 **Conceito 3: Padrões de Distribuição com AWS**

### **💡 Cache Pattern com ElastiCache**

```python
# Cache Pattern usando Redis ElastiCache
import boto3
import redis
import json
import hashlib
from typing import Dict, Any, Optional
from datetime import datetime, timedelta

class AWSCacheManager:
    def __init__(self):
        self.redis_client = redis.Redis(
            host='my-cache-cluster.abc123.cache.amazonaws.com',
            port=6379,
            decode_responses=True
        )
        self.cloudwatch = boto3.client('cloudwatch')
        
        # Configurações de cache por tipo de dados
        self.cache_configs = {
            'user_profile': {'ttl': 3600, 'namespace': 'user'},
            'product_catalog': {'ttl': 7200, 'namespace': 'product'},
            'session_data': {'ttl': 1800, 'namespace': 'session'},
            'api_responses': {'ttl': 300, 'namespace': 'api'}
        }
    
    def get_cache_key(self, namespace: str, identifier: str) -> str:
        """Generate cache key with namespace"""
        return f"{namespace}:{identifier}"
    
    def get_from_cache(self, cache_type: str, identifier: str) -> Optional[Dict[str, Any]]:
        """Get data from cache"""
        
        config = self.cache_configs.get(cache_type)
        if not config:
            return None
        
        cache_key = self.get_cache_key(config['namespace'], identifier)
        
        try:
            cached_data = self.redis_client.get(cache_key)
            
            if cached_data:
                # Métrica de cache hit
                self.cloudwatch.put_metric_data(
                    Namespace='Cache',
                    MetricData=[
                        {
                            'MetricName': 'CacheHit',
                            'Value': 1,
                            'Unit': 'Count',
                            'Dimensions': [
                                {
                                    'Name': 'CacheType',
                                    'Value': cache_type
                                }
                            ]
                        }
                    ]
                )
                
                return json.loads(cached_data)
            else:
                # Métrica de cache miss
                self.cloudwatch.put_metric_data(
                    Namespace='Cache',
                    MetricData=[
                        {
                            'MetricName': 'CacheMiss',
                            'Value': 1,
                            'Unit': 'Count',
                            'Dimensions': [
                                {
                                    'Name': 'CacheType',
                                    'Value': cache_type
                                }
                            ]
                        }
                    ]
                )
                
                return None
                
        except Exception as e:
            print(f"Error getting from cache: {e}")
            return None
    
    def set_in_cache(self, cache_type: str, identifier: str, data: Dict[str, Any]) -> bool:
        """Set data in cache"""
        
        config = self.cache_configs.get(cache_type)
        if not config:
            return False
        
        cache_key = self.get_cache_key(config['namespace'], identifier)
        
        try:
            # Serializar dados
            serialized_data = json.dumps(data, default=str)
            
            # Definir TTL
            ttl = config['ttl']
            
            # Armazenar no cache
            self.redis_client.setex(cache_key, ttl, serialized_data)
            
            return True
            
        except Exception as e:
            print(f"Error setting cache: {e}")
            return False
    
    def invalidate_cache(self, cache_type: str, identifier: str) -> bool:
        """Invalidate cache entry"""
        
        config = self.cache_configs.get(cache_type)
        if not config:
            return False
        
        cache_key = self.get_cache_key(config['namespace'], identifier)
        
        try:
            self.redis_client.delete(cache_key)
            return True
        except Exception as e:
            print(f"Error invalidating cache: {e}")
            return False
    
    def cache_aside_pattern(self, cache_type: str, identifier: str, 
                          fetch_function, *args, **kwargs) -> Dict[str, Any]:
        """Implement cache-aside pattern"""
        
        # 1. Tentar buscar no cache
        cached_data = self.get_from_cache(cache_type, identifier)
        
        if cached_data:
            return cached_data
        
        # 2. Buscar da fonte original
        fresh_data = fetch_function(*args, **kwargs)
        
        # 3. Armazenar no cache
        if fresh_data:
            self.set_in_cache(cache_type, identifier, fresh_data)
        
        return fresh_data
    
    def write_through_pattern(self, cache_type: str, identifier: str, 
                            data: Dict[str, Any], persist_function, *args, **kwargs) -> bool:
        """Implement write-through pattern"""
        
        try:
            # 1. Escrever na fonte primária
            success = persist_function(data, *args, **kwargs)
            
            if success:
                # 2. Escrever no cache
                self.set_in_cache(cache_type, identifier, data)
                return True
            
            return False
            
        except Exception as e:
            print(f"Error in write-through pattern: {e}")
            return False
    
    def write_behind_pattern(self, cache_type: str, identifier: str, 
                           data: Dict[str, Any]) -> bool:
        """Implement write-behind pattern"""
        
        try:
            # 1. Escrever no cache imediatamente
            self.set_in_cache(cache_type, identifier, data)
            
            # 2. Marcar para escrita assíncrona
            write_queue_key = f"write_queue:{cache_type}"
            
            write_task = {
                'cache_type': cache_type,
                'identifier': identifier,
                'data': data,
                'timestamp': datetime.utcnow().isoformat()
            }
            
            self.redis_client.lpush(write_queue_key, json.dumps(write_task))
            
            return True
            
        except Exception as e:
            print(f"Error in write-behind pattern: {e}")
            return False

# Lambda para operações com cache
def cached_operations_handler(event, context):
    """Handler para operações que usam cache"""
    
    cache_manager = AWSCacheManager()
    
    operation = event.get('operation')
    
    if operation == 'get_user_profile':
        user_id = event.get('user_id')
        
        # Usar cache-aside pattern
        user_profile = cache_manager.cache_aside_pattern(
            'user_profile',
            user_id,
            fetch_user_from_database,
            user_id
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps(user_profile)
        }
    
    elif operation == 'update_user_profile':
        user_id = event.get('user_id')
        profile_data = event.get('profile_data')
        
        # Usar write-through pattern
        success = cache_manager.write_through_pattern(
            'user_profile',
            user_id,
            profile_data,
            update_user_in_database,
            user_id
        )
        
        if success:
            return {
                'statusCode': 200,
                'body': json.dumps({'message': 'Profile updated successfully'})
            }
        else:
            return {
                'statusCode': 500,
                'body': json.dumps({'error': 'Failed to update profile'})
            }
    
    elif operation == 'get_product_catalog':
        category = event.get('category', 'all')
        
        # Cache com warming para catálogo
        products = cache_manager.cache_aside_pattern(
            'product_catalog',
            category,
            fetch_products_from_database,
            category
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps(products)
        }
    
    return {
        'statusCode': 400,
        'body': json.dumps({'error': 'Unknown operation'})
    }

def fetch_user_from_database(user_id: str) -> Dict[str, Any]:
    """Fetch user profile from database"""
    # Simulação de busca no banco
    return {
        'user_id': user_id,
        'name': f'User {user_id}',
        'email': f'user{user_id}@example.com',
        'created_at': datetime.utcnow().isoformat()
    }

def update_user_in_database(profile_data: Dict[str, Any], user_id: str) -> bool:
    """Update user profile in database"""
    # Simulação de atualização no banco
    print(f"Updating user {user_id} with data: {profile_data}")
    return True

def fetch_products_from_database(category: str) -> List[Dict[str, Any]]:
    """Fetch products from database"""
    # Simulação de busca de produtos
    return [
        {
            'product_id': f'prod_{i}',
            'name': f'Product {i}',
            'category': category,
            'price': 99.99
        }
        for i in range(1, 11)
    ]

# Lambda para processamento de write-behind queue
def write_behind_processor_handler(event, context):
    """Process write-behind queue"""
    
    cache_manager = AWSCacheManager()
    
    write_queue_key = "write_queue:user_profile"
    
    # Processar items da queue
    for _ in range(10):  # Processar até 10 items por execução
        try:
            task_data = cache_manager.redis_client.rpop(write_queue_key)
            
            if not task_data:
                break
            
            task = json.loads(task_data)
            
            # Processar escrita assíncrona
            success = update_user_in_database(task['data'], task['identifier'])
            
            if success:
                print(f"Successfully processed write-behind for {task['identifier']}")
            else:
                # Re-enqueue em caso de falha
                cache_manager.redis_client.lpush(write_queue_key, task_data)
                print(f"Re-enqueued failed write-behind for {task['identifier']}")
                
        except Exception as e:
            print(f"Error processing write-behind task: {e}")
    
    return {
        'statusCode': 200,
        'body': json.dumps({'message': 'Write-behind processing completed'})
    }
```

### **⚖️ Trade-offs AWS:**
- **✅ Prós:** Performance excelente, managed service, multi-AZ
- **❌ Contras:** Eventual consistency, custos adicionais, complexidade de invalidação

---

## 🎯 **Checklist de Implementação AWS**

### **🔍 Pré-Implementação**
- [ ] Definir requisitos de performance e SLA
- [ ] Calcular custos estimados dos serviços
- [ ] Planejar estratégia de monitoramento
- [ ] Definir estratégia de backup e disaster recovery

### **🚀 Implementação**
- [ ] Configurar IAM roles e policies
- [ ] Implementar CloudFormation templates
- [ ] Configurar CloudWatch alarms
- [ ] Implementar testes de carga
- [ ] Configurar CI/CD pipeline

### **📊 Pós-Implementação**
- [ ] Monitorar métricas de performance
- [ ] Revisar custos mensalmente
- [ ] Implementar otimizações baseadas em dados
- [ ] Documentar lições aprendidas
- [ ] Treinar equipe em troubleshooting

---

## 🎯 **Próximos Passos**

1. **Escolha um padrão** para implementar primeiro
2. **Crie um ambiente de teste** na AWS
3. **Implemente o padrão** seguindo os exemplos
4. **Monitore e meça** resultados
5. **Otimize** baseado nas métricas
6. **Documente** para a equipe

**Lembre-se:** A AWS oferece building blocks. O design patterns é como você os combina para resolver problemas reais! 🚀

---

## 🎯 **Conceito 4: Arquiteturas de Referência AWS**

### **🏦 Arquitetura Banking na AWS**

```yaml
# CloudFormation Template para Sistema Bancário
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Banking System Architecture'

Parameters:
  Environment:
    Type: String
    Default: 'dev'
    AllowedValues: ['dev', 'staging', 'prod']

Resources:
  # VPC com múltiplas AZs
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-banking-vpc'

  # Subnets privadas para databases
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [0, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-1'

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [1, !GetAZs '']
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-2'

  # RDS Aurora Cluster para transações
  DatabaseCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      Engine: aurora-postgresql
      MasterUsername: admin
      MasterUserPassword: !Ref DatabasePassword
      DatabaseName: bankingdb
      VpcSecurityGroupIds:
        - !Ref DatabaseSecurityGroup
      DBSubnetGroupName: !Ref DatabaseSubnetGroup
      BackupRetentionPeriod: 35
      PreferredBackupWindow: "03:00-04:00"
      PreferredMaintenanceWindow: "sun:04:00-sun:05:00"
      StorageEncrypted: true
      DeletionProtection: true
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # ElastiCache para sessões e cache
  ElastiCacheCluster:
    Type: AWS::ElastiCache::ReplicationGroup
    Properties:
      ReplicationGroupDescription: Banking cache cluster
      NumCacheClusters: 2
      Engine: redis
      CacheNodeType: cache.r6g.large
      VpcSecurityGroupIds:
        - !Ref CacheSecurityGroup
      SubnetGroupName: !Ref CacheSubnetGroup
      AtRestEncryptionEnabled: true
      TransitEncryptionEnabled: true
      MultiAZEnabled: true
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # API Gateway para APIs bancárias
  BankingApiGateway:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: !Sub '${Environment}-banking-api'
      Description: Banking API Gateway
      EndpointConfiguration:
        Types:
          - REGIONAL
      Policy:
        Statement:
          - Effect: Allow
            Principal: '*'
            Action: 'execute-api:Invoke'
            Resource: '*'
            Condition:
              StringEquals:
                'aws:SourceIp': '10.0.0.0/16'

  # Lambda Functions para operações bancárias
  AccountOperationsFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${Environment}-account-operations'
      Runtime: python3.9
      Handler: index.handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Code:
        ZipFile: |
          import json
          import boto3
          import logging
          
          logger = logging.getLogger()
          logger.setLevel(logging.INFO)
          
          def handler(event, context):
              logger.info(f"Processing account operation: {event}")
              
              # Implementar operações de conta
              operation = event.get('operation')
              
              if operation == 'get_balance':
                  return get_account_balance(event)
              elif operation == 'transfer':
                  return process_transfer(event)
              elif operation == 'transaction_history':
                  return get_transaction_history(event)
              
              return {
                  'statusCode': 400,
                  'body': json.dumps({'error': 'Unknown operation'})
              }
          
          def get_account_balance(event):
              # Implementar lógica de saldo
              return {
                  'statusCode': 200,
                  'body': json.dumps({'balance': 1000.00})
              }
          
          def process_transfer(event):
              # Implementar lógica de transferência
              return {
                  'statusCode': 200,
                  'body': json.dumps({'transaction_id': 'tx123'})
              }
          
          def get_transaction_history(event):
              # Implementar histórico de transações
              return {
                  'statusCode': 200,
                  'body': json.dumps({'transactions': []})
              }
      Environment:
        Variables:
          ENVIRONMENT: !Ref Environment
      ReservedConcurrencyLimit: 100
      Timeout: 30

  # SQS Queues para processamento assíncrono
  TransactionQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub '${Environment}-transaction-queue'
      VisibilityTimeoutSeconds: 300
      MessageRetentionPeriod: 1209600
      RedrivePolicy:
        deadLetterTargetArn: !GetAtt TransactionDLQ.Arn
        maxReceiveCount: 3
      KmsMasterKeyId: alias/aws/sqs
      Tags:
        - Key: Environment
          Value: !Ref Environment

  TransactionDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: !Sub '${Environment}-transaction-dlq'
      MessageRetentionPeriod: 1209600
      KmsMasterKeyId: alias/aws/sqs

  # SNS Topics para notificações
  TransactionNotificationTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${Environment}-transaction-notifications'
      KmsMasterKeyId: alias/aws/sns
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # CloudWatch Alarms
  HighErrorRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${Environment}-high-error-rate'
      AlarmDescription: 'High error rate in banking operations'
      MetricName: Errors
      Namespace: AWS/Lambda
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 10
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: FunctionName
          Value: !Ref AccountOperationsFunction
      AlarmActions:
        - !Ref TransactionNotificationTopic

  # Security Groups
  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for banking database
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref LambdaSecurityGroup
      Tags:
        - Key: Environment
          Value: !Ref Environment

  LambdaSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for Lambda functions
      VpcId: !Ref VPC
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          DestinationSecurityGroupId: !Ref DatabaseSecurityGroup
        - IpProtocol: tcp
          FromPort: 6379
          ToPort: 6379
          DestinationSecurityGroupId: !Ref CacheSecurityGroup
      Tags:
        - Key: Environment
          Value: !Ref Environment

Parameters:
  DatabasePassword:
    Type: String
    NoEcho: true
    Description: Password for the database
    MinLength: 8

Outputs:
  ApiGatewayUrl:
    Description: API Gateway URL
    Value: !Sub 'https://${BankingApiGateway}.execute-api.${AWS::Region}.amazonaws.com/prod'
    Export:
      Name: !Sub '${Environment}-banking-api-url'

  DatabaseEndpoint:
    Description: Database cluster endpoint
    Value: !GetAtt DatabaseCluster.Endpoint.Address
    Export:
      Name: !Sub '${Environment}-database-endpoint'
```

### **🛒 Arquitetura E-commerce na AWS**

```python
# Terraform para E-commerce Architecture
provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "ecommerce_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.environment}-ecommerce-vpc"
    Environment = var.environment
  }
}

# Application Load Balancer
resource "aws_lb" "ecommerce_alb" {
  name               = "${var.environment}-ecommerce-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [aws_subnet.public_subnet_1.id, aws_subnet.public_subnet_2.id]

  enable_deletion_protection = false

  tags = {
    Environment = var.environment
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "ecommerce_cluster" {
  name = "${var.environment}-ecommerce-cluster"

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 1
  }

  tags = {
    Environment = var.environment
  }
}

# ECS Service for Product Service
resource "aws_ecs_service" "product_service" {
  name            = "${var.environment}-product-service"
  cluster         = aws_ecs_cluster.ecommerce_cluster.id
  task_definition = aws_ecs_task_definition.product_service.arn
  desired_count   = 2

  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight            = 100
  }

  network_configuration {
    subnets          = [aws_subnet.private_subnet_1.id, aws_subnet.private_subnet_2.id]
    security_groups  = [aws_security_group.ecs_sg.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.product_service_tg.arn
    container_name   = "product-service"
    container_port   = 8080
  }

  depends_on = [aws_lb_listener.ecommerce_listener]
}

# DynamoDB Tables
resource "aws_dynamodb_table" "products" {
  name           = "${var.environment}-products"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "product_id"
  stream_enabled = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  attribute {
    name = "product_id"
    type = "S"
  }

  attribute {
    name = "category"
    type = "S"
  }

  global_secondary_index {
    name     = "category-index"
    hash_key = "category"
  }

  tags = {
    Environment = var.environment
  }
}

resource "aws_dynamodb_table" "orders" {
  name           = "${var.environment}-orders"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "order_id"
  stream_enabled = true
  stream_view_type = "NEW_AND_OLD_IMAGES"

  attribute {
    name = "order_id"
    type = "S"
  }

  attribute {
    name = "customer_id"
    type = "S"
  }

  global_secondary_index {
    name     = "customer-index"
    hash_key = "customer_id"
  }

  tags = {
    Environment = var.environment
  }
}

# ElastiCache for Session Management
resource "aws_elasticache_replication_group" "session_cache" {
  replication_group_id       = "${var.environment}-session-cache"
  description                = "Session cache for e-commerce"
  
  node_type                  = "cache.r6g.large"
  port                       = 6379
  parameter_group_name       = "default.redis6.x"
  
  num_cache_clusters         = 2
  
  subnet_group_name          = aws_elasticache_subnet_group.cache_subnet_group.name
  security_group_ids         = [aws_security_group.cache_sg.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  
  tags = {
    Environment = var.environment
  }
}

# Lambda Functions for Order Processing
resource "aws_lambda_function" "order_processor" {
  filename         = "order_processor.zip"
  function_name    = "${var.environment}-order-processor"
  role            = aws_iam_role.lambda_role.arn
  handler         = "index.handler"
  runtime         = "python3.9"
  timeout         = 30

  environment {
    variables = {
      ENVIRONMENT = var.environment
      ORDERS_TABLE = aws_dynamodb_table.orders.name
      PRODUCTS_TABLE = aws_dynamodb_table.products.name
    }
  }

  reserved_concurrent_executions = 100

  tags = {
    Environment = var.environment
  }
}

# SQS Queue for Order Processing
resource "aws_sqs_queue" "order_queue" {
  name                      = "${var.environment}-order-queue"
  delay_seconds             = 0
  max_message_size          = 262144
  message_retention_seconds = 1209600
  receive_wait_time_seconds = 10
  visibility_timeout_seconds = 300

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.order_dlq.arn
    maxReceiveCount     = 3
  })

  kms_master_key_id = "alias/aws/sqs"

  tags = {
    Environment = var.environment
  }
}

resource "aws_sqs_queue" "order_dlq" {
  name                      = "${var.environment}-order-dlq"
  message_retention_seconds = 1209600
  kms_master_key_id        = "alias/aws/sqs"

  tags = {
    Environment = var.environment
  }
}

# CloudFront Distribution for Static Assets
resource "aws_cloudfront_distribution" "ecommerce_cdn" {
  origin {
    domain_name = aws_s3_bucket.static_assets.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.static_assets.bucket}"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-${aws_s3_bucket.static_assets.bucket}"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 3600
    max_ttl                = 86400
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }

  tags = {
    Environment = var.environment
  }
}

# S3 Bucket for Static Assets
resource "aws_s3_bucket" "static_assets" {
  bucket = "${var.environment}-ecommerce-static-assets"

  tags = {
    Environment = var.environment
  }
}

# Auto Scaling Configuration
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = 10
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.ecommerce_cluster.name}/${aws_ecs_service.product_service.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_policy_cpu" {
  name               = "${var.environment}-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

# Outputs
output "load_balancer_dns" {
  value = aws_lb.ecommerce_alb.dns_name
}

output "cloudfront_distribution_id" {
  value = aws_cloudfront_distribution.ecommerce_cdn.id
}

output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.ecommerce_cdn.domain_name
}
```

---

## 🎯 **Conceito 5: AWS Well-Architected Framework**

### **🏗️ Os 6 Pilares Aplicados**

#### **1. Operational Excellence**

```python
# Implementação de Observabilidade
import boto3
import json
import logging
from datetime import datetime

class AWSObservabilityManager:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.xray = boto3.client('xray')
        self.logs = boto3.client('logs')
        
        # Configurar logging estruturado
        self.logger = logging.getLogger()
        self.logger.setLevel(logging.INFO)
        
        # Configurar X-Ray tracing
        from aws_xray_sdk.core import xray_recorder
        self.xray_recorder = xray_recorder
    
    def log_business_event(self, event_type: str, event_data: dict):
        """Log business events with structured format"""
        
        log_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': event_type,
            'event_data': event_data,
            'service': 'banking-service',
            'version': '1.0.0'
        }
        
        self.logger.info(json.dumps(log_entry))
        
        # Também enviar como métrica customizada
        self.cloudwatch.put_metric_data(
            Namespace='Business/Events',
            MetricData=[
                {
                    'MetricName': event_type,
                    'Value': 1,
                    'Unit': 'Count',
                    'Timestamp': datetime.utcnow()
                }
            ]
        )
    
    def trace_operation(self, operation_name: str):
        """Decorator para tracing de operações"""
        def decorator(func):
            def wrapper(*args, **kwargs):
                with self.xray_recorder.capture(operation_name) as segment:
                    try:
                        result = func(*args, **kwargs)
                        
                        # Adicionar metadados ao trace
                        segment.put_metadata('operation_result', 'success')
                        segment.put_annotation('operation_type', operation_name)
                        
                        return result
                        
                    except Exception as e:
                        segment.put_metadata('operation_result', 'error')
                        segment.put_metadata('error_message', str(e))
                        raise
            return wrapper
        return decorator
    
    def create_custom_dashboard(self, dashboard_name: str):
        """Create CloudWatch dashboard for monitoring"""
        
        dashboard_body = {
            "widgets": [
                {
                    "type": "metric",
                    "properties": {
                        "metrics": [
                            ["AWS/Lambda", "Duration", "FunctionName", "account-operations"],
                            ["AWS/Lambda", "Errors", "FunctionName", "account-operations"],
                            ["AWS/Lambda", "Invocations", "FunctionName", "account-operations"]
                        ],
                        "period": 300,
                        "stat": "Average",
                        "region": "us-east-1",
                        "title": "Lambda Performance"
                    }
                },
                {
                    "type": "metric",
                    "properties": {
                        "metrics": [
                            ["AWS/RDS", "CPUUtilization", "DBClusterIdentifier", "banking-cluster"],
                            ["AWS/RDS", "DatabaseConnections", "DBClusterIdentifier", "banking-cluster"]
                        ],
                        "period": 300,
                        "stat": "Average",
                        "region": "us-east-1",
                        "title": "Database Performance"
                    }
                },
                {
                    "type": "metric",
                    "properties": {
                        "metrics": [
                            ["Business/Events", "payment_processed"],
                            ["Business/Events", "transfer_completed"],
                            ["Business/Events", "account_created"]
                        ],
                        "period": 300,
                        "stat": "Sum",
                        "region": "us-east-1",
                        "title": "Business Metrics"
                    }
                }
            ]
        }
        
        self.cloudwatch.put_dashboard(
            DashboardName=dashboard_name,
            DashboardBody=json.dumps(dashboard_body)
        )

# Exemplo de uso
observability = AWSObservabilityManager()

@observability.trace_operation('process_payment')
def process_payment(payment_data):
    """Process payment with full observability"""
    
    # Log início da operação
    observability.log_business_event('payment_started', {
        'payment_id': payment_data['payment_id'],
        'amount': payment_data['amount'],
        'currency': payment_data['currency']
    })
    
    try:
        # Processar pagamento
        result = execute_payment_processing(payment_data)
        
        # Log sucesso
        observability.log_business_event('payment_processed', {
            'payment_id': payment_data['payment_id'],
            'transaction_id': result['transaction_id'],
            'status': 'success'
        })
        
        return result
        
    except Exception as e:
        # Log erro
        observability.log_business_event('payment_failed', {
            'payment_id': payment_data['payment_id'],
            'error': str(e),
            'status': 'failed'
        })
        
        raise

def execute_payment_processing(payment_data):
    """Simulate payment processing"""
    import uuid
    import time
    
    # Simulate processing time
    time.sleep(0.5)
    
    return {
        'transaction_id': str(uuid.uuid4()),
        'status': 'completed',
        'processed_at': datetime.utcnow().isoformat()
    }
```

#### **2. Security**

```python
# Implementação de Segurança
import boto3
import json
import hashlib
import hmac
from datetime import datetime, timedelta
import jwt

class AWSSecurityManager:
    def __init__(self):
        self.kms = boto3.client('kms')
        self.secretsmanager = boto3.client('secretsmanager')
        self.iam = boto3.client('iam')
        self.cloudtrail = boto3.client('cloudtrail')
        
        # Key Management
        self.encryption_key_id = 'alias/banking-encryption-key'
    
    def encrypt_sensitive_data(self, data: str) -> str:
        """Encrypt sensitive data using KMS"""
        
        response = self.kms.encrypt(
            KeyId=self.encryption_key_id,
            Plaintext=data.encode('utf-8')
        )
        
        # Encode to base64 for storage
        import base64
        return base64.b64encode(response['CiphertextBlob']).decode('utf-8')
    
    def decrypt_sensitive_data(self, encrypted_data: str) -> str:
        """Decrypt sensitive data using KMS"""
        
        import base64
        ciphertext_blob = base64.b64decode(encrypted_data.encode('utf-8'))
        
        response = self.kms.decrypt(
            CiphertextBlob=ciphertext_blob
        )
        
        return response['Plaintext'].decode('utf-8')
    
    def generate_secure_token(self, user_id: str, permissions: list) -> str:
        """Generate secure JWT token"""
        
        # Get signing key from Secrets Manager
        secret_response = self.secretsmanager.get_secret_value(
            SecretId='banking-jwt-secret'
        )
        
        signing_key = secret_response['SecretString']
        
        # Create JWT payload
        payload = {
            'user_id': user_id,
            'permissions': permissions,
            'iat': datetime.utcnow(),
            'exp': datetime.utcnow() + timedelta(hours=1),
            'iss': 'banking-system'
        }
        
        # Sign token
        token = jwt.encode(payload, signing_key, algorithm='HS256')
        
        return token
    
    def validate_token(self, token: str) -> dict:
        """Validate JWT token"""
        
        try:
            # Get signing key from Secrets Manager
            secret_response = self.secretsmanager.get_secret_value(
                SecretId='banking-jwt-secret'
            )
            
            signing_key = secret_response['SecretString']
            
            # Decode and validate token
            payload = jwt.decode(token, signing_key, algorithms=['HS256'])
            
            return {
                'valid': True,
                'user_id': payload['user_id'],
                'permissions': payload['permissions']
            }
            
        except jwt.ExpiredSignatureError:
            return {'valid': False, 'error': 'Token expired'}
        except jwt.InvalidTokenError:
            return {'valid': False, 'error': 'Invalid token'}
    
    def check_permissions(self, user_permissions: list, required_permission: str) -> bool:
        """Check if user has required permission"""
        
        return required_permission in user_permissions
    
    def log_security_event(self, event_type: str, event_data: dict):
        """Log security events for audit"""
        
        security_event = {
            'timestamp': datetime.utcnow().isoformat(),
            'event_type': event_type,
            'event_data': event_data,
            'service': 'banking-security',
            'severity': 'HIGH' if event_type in ['authentication_failed', 'unauthorized_access'] else 'INFO'
        }
        
        # Log to CloudWatch
        logs_client = boto3.client('logs')
        logs_client.put_log_events(
            logGroupName='/aws/lambda/security-events',
            logStreamName=datetime.utcnow().strftime('%Y/%m/%d'),
            logEvents=[
                {
                    'timestamp': int(datetime.utcnow().timestamp() * 1000),
                    'message': json.dumps(security_event)
                }
            ]
        )
    
    def create_least_privilege_policy(self, service_name: str, resources: list) -> dict:
        """Create IAM policy with least privilege"""
        
        policy_document = {
            "Version": "2012-10-17",
            "Statement": [
                {
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
        
        # Add specific permissions based on service
        if service_name == 'account-service':
            policy_document["Statement"].append({
                "Effect": "Allow",
                "Action": [
                    "dynamodb:GetItem",
                    "dynamodb:PutItem",
                    "dynamodb:UpdateItem",
                    "dynamodb:Query"
                ],
                "Resource": resources
            })
        
        elif service_name == 'payment-service':
            policy_document["Statement"].extend([
                {
                    "Effect": "Allow",
                    "Action": [
                        "dynamodb:GetItem",
                        "dynamodb:PutItem",
                        "dynamodb:UpdateItem"
                    ],
                    "Resource": resources
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "kms:Encrypt",
                        "kms:Decrypt",
                        "kms:GenerateDataKey"
                    ],
                    "Resource": "arn:aws:kms:*:*:key/*"
                }
            ])
        
        return policy_document

# Decorator para autenticação e autorização
def require_permission(permission: str):
    def decorator(func):
        def wrapper(event, context):
            security_manager = AWSSecurityManager()
            
            # Extrair token do header
            token = event.get('headers', {}).get('Authorization', '').replace('Bearer ', '')
            
            if not token:
                security_manager.log_security_event('authentication_failed', {
                    'reason': 'missing_token',
                    'source_ip': event.get('requestContext', {}).get('identity', {}).get('sourceIp')
                })
                
                return {
                    'statusCode': 401,
                    'body': json.dumps({'error': 'Authentication required'})
                }
            
            # Validar token
            token_validation = security_manager.validate_token(token)
            
            if not token_validation['valid']:
                security_manager.log_security_event('authentication_failed', {
                    'reason': token_validation['error'],
                    'source_ip': event.get('requestContext', {}).get('identity', {}).get('sourceIp')
                })
                
                return {
                    'statusCode': 401,
                    'body': json.dumps({'error': 'Invalid token'})
                }
            
            # Verificar permissões
            if not security_manager.check_permissions(token_validation['permissions'], permission):
                security_manager.log_security_event('unauthorized_access', {
                    'user_id': token_validation['user_id'],
                    'required_permission': permission,
                    'user_permissions': token_validation['permissions']
                })
                
                return {
                    'statusCode': 403,
                    'body': json.dumps({'error': 'Insufficient permissions'})
                }
            
            # Adicionar user_id ao evento
            event['user_id'] = token_validation['user_id']
            event['user_permissions'] = token_validation['permissions']
            
            return func(event, context)
        
        return wrapper
    return decorator

# Exemplo de uso
@require_permission('transfer:execute')
def lambda_handler(event, context):
    """Lambda handler with security"""
    
    user_id = event['user_id']
    
    # Processar transferência
    transfer_data = json.loads(event['body'])
    
    # Log da operação
    security_manager = AWSSecurityManager()
    security_manager.log_security_event('transfer_initiated', {
        'user_id': user_id,
        'amount': transfer_data['amount'],
        'destination_account': transfer_data['destination_account']
    })
    
    # Executar transferência
    result = process_transfer(transfer_data)
    
    return {
        'statusCode': 200,
        'body': json.dumps(result)
    }

def process_transfer(transfer_data):
    """Process transfer with encryption"""
    security_manager = AWSSecurityManager()
    
    # Encrypt sensitive data
    encrypted_account = security_manager.encrypt_sensitive_data(
        transfer_data['destination_account']
    )
    
    # Store in database with encrypted account number
    # Implementation here...
    
    return {
        'transfer_id': 'txn_123456',
        'status': 'completed'
    }
```

#### **3. Reliability**

```python
# Implementação de Reliability
import boto3
import json
import time
from typing import Dict, Any, List

class AWSReliabilityManager:
    def __init__(self):
        self.backup = boto3.client('backup')
        self.rds = boto3.client('rds')
        self.s3 = boto3.client('s3')
        self.cloudformation = boto3.client('cloudformation')
        
    def setup_automated_backups(self):
        """Setup automated backups for critical resources"""
        
        # Create backup plan
        backup_plan = {
            "BackupPlanName": "banking-backup-plan",
            "Rules": [
                {
                    "RuleName": "daily-backups",
                    "TargetBackupVault": "banking-backup-vault",
                    "ScheduleExpression": "cron(0 2 ? * * *)",  # Daily at 2 AM
                    "StartWindowMinutes": 60,
                    "CompletionWindowMinutes": 120,
                    "Lifecycle": {
                        "DeleteAfterDays": 35,
                        "MoveToColdStorageAfterDays": 7
                    },
                    "RecoveryPointTags": {
                        "Environment": "production",
                        "BackupType": "automated"
                    }
                },
                {
                    "RuleName": "weekly-backups",
                    "TargetBackupVault": "banking-backup-vault",
                    "ScheduleExpression": "cron(0 1 ? * SUN *)",  # Weekly on Sunday
                    "StartWindowMinutes": 60,
                    "CompletionWindowMinutes": 240,
                    "Lifecycle": {
                        "DeleteAfterDays": 365,
                        "MoveToColdStorageAfterDays": 30
                    },
                    "RecoveryPointTags": {
                        "Environment": "production",
                        "BackupType": "weekly"
                    }
                }
            ]
        }
        
        response = self.backup.create_backup_plan(BackupPlan=backup_plan)
        backup_plan_id = response['BackupPlanId']
        
        # Create backup selection
        backup_selection = {
            "SelectionName": "banking-resources",
            "IamRoleArn": "arn:aws:iam::123456789012:role/aws-backup-service-role",
            "Resources": [
                "arn:aws:rds:us-east-1:123456789012:cluster:banking-cluster",
                "arn:aws:dynamodb:us-east-1:123456789012:table/accounts",
                "arn:aws:dynamodb:us-east-1:123456789012:table/transactions"
            ],
            "ListOfTags": [
                {
                    "ConditionType": "STRINGEQUALS",
                    "ConditionKey": "Environment",
                    "ConditionValue": "production"
                }
            ]
        }
        
        self.backup.create_backup_selection(
            BackupPlanId=backup_plan_id,
            BackupSelection=backup_selection
        )
    
    def setup_cross_region_replication(self, primary_region: str, replica_regions: List[str]):
        """Setup cross-region replication for disaster recovery"""
        
        # S3 Cross-Region Replication
        s3_replication_config = {
            "Role": "arn:aws:iam::123456789012:role/replication-role",
            "Rules": [
                {
                    "ID": "banking-data-replication",
                    "Status": "Enabled",
                    "Priority": 1,
                    "Filter": {
                        "Prefix": "banking-data/"
                    },
                    "Destination": {
                        "Bucket": f"arn:aws:s3:::banking-backup-{replica_regions[0]}",
                        "StorageClass": "STANDARD_IA"
                    }
                }
            ]
        }
        
        self.s3.put_bucket_replication(
            Bucket='banking-primary-data',
            ReplicationConfiguration=s3_replication_config
        )
        
        # RDS Cross-Region Read Replica
        for region in replica_regions:
            self.rds.create_db_cluster(
                DBClusterIdentifier=f'banking-cluster-replica-{region}',
                Engine='aurora-postgresql',
                ReplicationSourceIdentifier='arn:aws:rds:us-east-1:123456789012:cluster:banking-cluster',
                VpcSecurityGroupIds=['sg-0123456789abcdef0'],
                DBSubnetGroupName=f'banking-subnet-group-{region}',
                BackupRetentionPeriod=7,
                StorageEncrypted=True
            )
    
    def setup_health_checks(self):
        """Setup comprehensive health checks"""
        
        route53 = boto3.client('route53')
        
        # Application health check
        health_check_config = {
            "Type": "HTTPS",
            "ResourcePath": "/health",
            "FullyQualifiedDomainName": "api.banking.com",
            "Port": 443,
            "RequestInterval": 30,
            "FailureThreshold": 3
        }
        
        response = route53.create_health_check(
            CallerReference=str(int(time.time())),
            HealthCheckConfig=health_check_config,
            Tags=[
                {
                    "Key": "Name",
                    "Value": "Banking API Health Check"
                },
                {
                    "Key": "Environment",
                    "Value": "production"
                }
            ]
        )
        
        health_check_id = response['HealthCheck']['Id']
        
        # Create CloudWatch alarm for health check
        cloudwatch = boto3.client('cloudwatch')
        
        cloudwatch.put_metric_alarm(
            AlarmName='banking-api-health-check-failed',
            ComparisonOperator='LessThanThreshold',
            EvaluationPeriods=2,
            MetricName='HealthCheckStatus',
            Namespace='AWS/Route53',
            Period=60,
            Statistic='Minimum',
            Threshold=1.0,
            ActionsEnabled=True,
            AlarmActions=[
                'arn:aws:sns:us-east-1:123456789012:banking-alerts'
            ],
            AlarmDescription='Banking API health check failed',
            Dimensions=[
                {
                    'Name': 'HealthCheckId',
                    'Value': health_check_id
                }
            ]
        )
    
    def create_disaster_recovery_plan(self):
        """Create comprehensive disaster recovery plan"""
        
        dr_plan = {
            "rpo_target": 60,  # Recovery Point Objective: 1 hour
            "rto_target": 240,  # Recovery Time Objective: 4 hours
            "procedures": [
                {
                    "step": 1,
                    "action": "Assess Impact",
                    "description": "Determine scope and impact of the disaster",
                    "responsible_team": "SRE",
                    "estimated_time": 15
                },
                {
                    "step": 2,
                    "action": "Activate DR Site",
                    "description": "Failover to secondary region",
                    "responsible_team": "SRE",
                    "estimated_time": 30,
                    "automation": {
                        "lambda_function": "activate-dr-site",
                        "parameters": {
                            "primary_region": "us-east-1",
                            "dr_region": "us-west-2"
                        }
                    }
                },
                {
                    "step": 3,
                    "action": "Restore Database",
                    "description": "Restore from latest backup",
                    "responsible_team": "DBA",
                    "estimated_time": 120,
                    "automation": {
                        "lambda_function": "restore-database",
                        "parameters": {
                            "backup_identifier": "latest"
                        }
                    }
                },
                {
                    "step": 4,
                    "action": "Update DNS",
                    "description": "Point traffic to DR site",
                    "responsible_team": "SRE",
                    "estimated_time": 5,
                    "automation": {
                        "lambda_function": "update-dns-records",
                        "parameters": {
                            "dr_endpoint": "dr-api.banking.com"
                        }
                    }
                },
                {
                    "step": 5,
                    "action": "Verify Services",
                    "description": "Run smoke tests to verify all services",
                    "responsible_team": "QA",
                    "estimated_time": 30,
                    "automation": {
                        "lambda_function": "run-smoke-tests",
                        "parameters": {
                            "test_suite": "critical-path"
                        }
                    }
                }
            ]
        }
        
        # Store DR plan in S3
        self.s3.put_object(
            Bucket='banking-dr-plans',
            Key='disaster-recovery-plan.json',
            Body=json.dumps(dr_plan, indent=2),
            ContentType='application/json'
        )
        
        return dr_plan
    
    def simulate_disaster_recovery(self):
        """Simulate disaster recovery to test procedures"""
        
        # Create test environment
        test_stack_name = f"banking-dr-test-{int(time.time())}"
        
        # Deploy test stack
        with open('disaster-recovery-test.yaml', 'r') as template_file:
            template_body = template_file.read()
        
        self.cloudformation.create_stack(
            StackName=test_stack_name,
            TemplateBody=template_body,
            Parameters=[
                {
                    'ParameterKey': 'Environment',
                    'ParameterValue': 'dr-test'
                }
            ],
            Capabilities=['CAPABILITY_IAM'],
            Tags=[
                {
                    'Key': 'Purpose',
                    'Value': 'DisasterRecoveryTest'
                }
            ]
        )
        
        # Wait for stack creation
        waiter = self.cloudformation.get_waiter('stack_create_complete')
        waiter.wait(StackName=test_stack_name)
        
        # Run DR tests
        test_results = self.run_dr_tests(test_stack_name)
        
        # Cleanup test environment
        self.cloudformation.delete_stack(StackName=test_stack_name)
        
        return test_results
    
    def run_dr_tests(self, test_stack_name: str) -> Dict[str, Any]:
        """Run disaster recovery tests"""
        
        test_results = {
            "test_execution_time": datetime.utcnow().isoformat(),
            "stack_name": test_stack_name,
            "tests": []
        }
        
        # Test 1: Database failover
        test_results["tests"].append({
            "test_name": "Database Failover",
            "status": "PASSED",
            "duration": 45,
            "details": "Database failover completed within RTO"
        })
        
        # Test 2: Application deployment
        test_results["tests"].append({
            "test_name": "Application Deployment",
            "status": "PASSED",
            "duration": 120,
            "details": "All services deployed successfully"
        })
        
        # Test 3: Data integrity
        test_results["tests"].append({
            "test_name": "Data Integrity",
            "status": "PASSED",
            "duration": 30,
            "details": "No data corruption detected"
        })
        
        # Store test results
        self.s3.put_object(
            Bucket='banking-dr-test-results',
            Key=f'dr-test-{int(time.time())}.json',
            Body=json.dumps(test_results, indent=2),
            ContentType='application/json'
        )
        
        return test_results

# Lambda function for DR automation
def activate_dr_site_handler(event, context):
    """Lambda handler for DR site activation"""
    
    reliability_manager = AWSReliabilityManager()
    
    primary_region = event['primary_region']
    dr_region = event['dr_region']
    
    # Activate DR procedures
    try:
        # Update Route 53 health checks
        route53 = boto3.client('route53')
        
        # Fail over to DR region
        # Implementation depends on your specific setup
        
        # Update DNS records
        # Implementation here...
        
        # Notify teams
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:banking-alerts',
            Message=f'DR site activated for region {dr_region}',
            Subject='Disaster Recovery Activated'
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({'message': 'DR site activated successfully'})
        }
        
    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

### **⚖️ Trade-offs do Well-Architected Framework:**
- **✅ Prós:** Estrutura abrangente, best practices comprovadas, auditoria sistemática
- **❌ Contras:** Pode ser complexo para sistemas simples, requer investimento em ferramentas, cultura organizacional

---

## 🎯 **Resumo de Serviços AWS por Padrão**

### **📊 Mapeamento Padrão → Serviços AWS**

| Padrão | Serviços AWS Principais | Serviços de Apoio | Exemplo de Uso |
|--------|------------------------|-------------------|----------------|
| **Circuit Breaker** | Lambda, DynamoDB, CloudWatch | SNS, Step Functions | API Gateway → Lambda → External API |
| **Bulkhead** | SQS, Lambda, ECS | CloudWatch, Auto Scaling | Separar processamento crítico vs. batch |
| **Retry** | Step Functions, Lambda, SQS | CloudWatch, DLQ | Retry de operações com APIs externas |
| **Rate Limiting** | API Gateway, Lambda, ElastiCache | CloudWatch, WAF | Proteger APIs contra abuso |
| **Auto Scaling** | Auto Scaling Groups, ECS, Lambda | CloudWatch, Target Groups | Escalar baseado em métricas |
| **Database Sharding** | RDS, DynamoDB, ElastiCache | Route 53, Lambda | Distribuir dados por região |
| **CQRS** | DynamoDB, RDS, Lambda | Kinesis, EventBridge | Separar leitura de escrita |
| **Cache-Aside** | ElastiCache, Lambda, DynamoDB | CloudWatch | Cache de dados frequentes |
| **API Gateway** | API Gateway, Lambda, ALB | Route 53, CloudFront | Agregação de microserviços |
| **Event-Driven** | EventBridge, SNS, SQS | Lambda, Step Functions | Comunicação assíncrona |
| **Eventual Consistency** | DynamoDB Streams, Kinesis | Lambda, SQS | Sincronização cross-region |
| **Saga Pattern** | Step Functions, Lambda, DynamoDB | SQS, EventBridge | Transações distribuídas |

### **💰 Considerações de Custo**

#### **💸 Padrões Mais Caros:**
1. **Database Sharding** (múltiplas instâncias RDS)
2. **Cross-Region Replication** (transferência de dados)
3. **Always-On Resources** (RDS, ElastiCache)

#### **💚 Padrões Mais Econômicos:**
1. **Lambda-based patterns** (pay-per-use)
2. **SQS/SNS patterns** (muito baixo custo)
3. **DynamoDB on-demand** (pay-per-request)

#### **🎯 Otimizações de Custo:**
- Use **Reserved Instances** para recursos constantes
- Implemente **Auto Scaling** para otimizar uso
- Use **Spot Instances** para workloads tolerantes a falhas
- Configure **S3 Lifecycle Policies** para dados antigos

---

## 🚀 **Próximos Passos Práticos**

### **📋 Lista de Implementação**

1. **Semana 1-2: Fundamentos**
   - [ ] Configurar conta AWS com boas práticas
   - [ ] Implementar Circuit Breaker básico
   - [ ] Configurar monitoramento com CloudWatch

2. **Semana 3-4: Escalabilidade**
   - [ ] Implementar Auto Scaling
   - [ ] Configurar Load Balancer
   - [ ] Implementar Cache com ElastiCache

3. **Semana 5-6: Resiliência**
   - [ ] Configurar Multi-AZ deployment
   - [ ] Implementar backup automatizado
   - [ ] Criar disaster recovery plan

4. **Semana 7-8: Observabilidade**
   - [ ] Implementar logging estruturado
   - [ ] Configurar dashboards
   - [ ] Criar alertas inteligentes

### **🎯 Checklist de Produção**

#### **Antes do Deploy:**
- [ ] Revisão de segurança completa
- [ ] Testes de carga realizados
- [ ] Disaster recovery testado
- [ ] Documentação atualizada
- [ ] Equipe treinada

#### **Durante o Deploy:**
- [ ] Monitoramento em tempo real
- [ ] Rollback plan pronto
- [ ] Comunicação com stakeholders
- [ ] Verificação de health checks

#### **Após o Deploy:**
- [ ] Monitoramento de métricas
- [ ] Revisão de custos
- [ ] Feedback da equipe
- [ ] Lessons learned documentadas

---

## 🎓 **Recursos Adicionais**

### **📚 Documentação AWS:**
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [AWS Whitepapers](https://aws.amazon.com/whitepapers/)

### **🛠️ Ferramentas Úteis:**
- [AWS CLI](https://aws.amazon.com/cli/)
- [AWS CDK](https://aws.amazon.com/cdk/)
- [Terraform](https://terraform.io/)
- [Ansible](https://ansible.com/)

### **📊 Monitoramento:**
- [CloudWatch](https://aws.amazon.com/cloudwatch/)
- [X-Ray](https://aws.amazon.com/xray/)
- [AWS Config](https://aws.amazon.com/config/)

**Lembre-se:** System Design na AWS é uma jornada contínua. Comece simples, meça sempre, e evolua baseado em dados reais! 🚀 