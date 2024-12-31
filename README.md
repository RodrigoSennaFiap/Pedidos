# Pedidos
Projeto de Pedidos usando AWS Lambda, Dynamo DB e boas praticas de segurança e desacoplamento com filas SNS e SQS

# Sistema de Gerenciamento de Pedidos em Tempo Real

Este projeto implementa um sistema serverless de gerenciamento de pedidos em tempo real utilizando os principais serviços da AWS. Ele cobre vários aspectos necessários para a certificação **AWS Certified Developer Associate**, incluindo Lambda, DynamoDB, SNS, SQS, Secrets Manager, KMS, AWS SDK, CDK e uma pipeline de CI/CD.

---

## **Arquitetura do Sistema**

### **Fluxo do Sistema**
1. O cliente faz um pedido via API Gateway.
2. A função Lambda processa o pedido e o armazena no DynamoDB.
3. O evento dispara uma notificação para o SNS, que é consumida por um tópico assinado.
4. Mensagens são enviadas para a fila SQS, onde outra função Lambda as consome e realiza ações adicionais.
5. Segredos (como credenciais de integração) são armazenados e gerenciados pelo Secrets Manager.
6. Dados sensíveis são protegidos utilizando KMS para criptografia.

### **Serviços Utilizados**
- **AWS Lambda**: Lógica de negócios e processamento de eventos.
- **Amazon DynamoDB**: Armazenamento dos pedidos.
- **Amazon SNS**: Notificação de novos pedidos.
- **Amazon SQS**: Fila de mensagens para processamento assíncrono.
- **AWS Secrets Manager**: Gerenciamento de segredos.
- **AWS KMS**: Criptografia de dados sensíveis.
- **AWS SDK**: Integração programática entre os serviços.
- **AWS CDK**: Provisionamento da infraestrutura como código.
- **AWS CodePipeline + CodeBuild**: Pipeline CI/CD automatizado.

---

## **Pré-requisitos**
1. **Ferramentas Necessárias**:
   - AWS CLI configurado com credenciais apropriadas.
   - Node.js ou Python instalado.
   - AWS CDK instalado globalmente:
     ```bash
     npm install -g aws-cdk
     ```
   - Git configurado para versionamento.

2. **Configurações da AWS**:
   - Crie um usuário IAM com permissões administrativas para teste.
   - Configure o ambiente local:
     ```bash
     aws configure
     ```

---

## **Passo a Passo para Implementação**

### **1. Criar o Projeto CDK**
1. Inicialize um novo projeto CDK:
   ```bash
   cdk init app --language typescript
   ```

2. Instale as dependências necessárias:
   ```bash
   npm install @aws-cdk/aws-lambda @aws-cdk/aws-dynamodb @aws-cdk/aws-sns @aws-cdk/aws-sqs @aws-cdk/aws-secretsmanager @aws-cdk/aws-apigateway
   ```

3. Crie os recursos no arquivo `lib/order-management-stack.ts`:
   ```typescript
   import * as cdk from 'aws-cdk-lib';
   import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
   import * as sns from 'aws-cdk-lib/aws-sns';
   import * as sqs from 'aws-cdk-lib/aws-sqs';
   import * as lambda from 'aws-cdk-lib/aws-lambda';
   import * as secretsmanager from 'aws-cdk-lib/aws-secretsmanager';
   import * as apigateway from 'aws-cdk-lib/aws-apigateway';

   const app = new cdk.App();
   const stack = new cdk.Stack(app, 'OrderManagementStack');

   // DynamoDB
   const table = new dynamodb.Table(stack, 'OrdersTable', {
       partitionKey: { name: 'orderId', type: dynamodb.AttributeType.STRING },
       encryption: dynamodb.TableEncryption.AWS_MANAGED,
   });

   // SNS Topic
   const topic = new sns.Topic(stack, 'OrderNotifications');

   // SQS Queue
   const queue = new sqs.Queue(stack, 'OrderQueue', {
       encryption: sqs.QueueEncryption.KMS_MANAGED,
   });

   // Lambda Function
   const orderHandler = new lambda.Function(stack, 'OrderHandler', {
       runtime: lambda.Runtime.NODEJS_14_X,
       code: lambda.Code.fromAsset('lambda'),
       handler: 'order.handler',
       environment: {
           TABLE_NAME: table.tableName,
           TOPIC_ARN: topic.topicArn,
           QUEUE_URL: queue.queueUrl,
       },
   });

   // Grant Permissions
   table.grantReadWriteData(orderHandler);
   topic.grantPublish(orderHandler);
   queue.grantSendMessages(orderHandler);

   // API Gateway
   const api = new apigateway.LambdaRestApi(stack, 'OrderAPI', {
       handler: orderHandler,
   });
   ```

4. Realize o deploy:
   ```bash
   cdk deploy
   ```

---

### **2. Implementar as Funções Lambda**

#### **Função Principal**
1. Crie o diretório `lambda` e adicione o arquivo `order.js`:
   ```javascript
   const AWS = require('aws-sdk');
   const dynamodb = new AWS.DynamoDB.DocumentClient();
   const sns = new AWS.SNS();

   exports.handler = async (event) => {
       const order = JSON.parse(event.body);
       const params = {
           TableName: process.env.TABLE_NAME,
           Item: { orderId: order.id, details: order.details },
       };

       await dynamodb.put(params).promise();
       await sns.publish({
           TopicArn: process.env.TOPIC_ARN,
           Message: `New order created: ${order.id}`,
       }).promise();

       return { statusCode: 200, body: JSON.stringify({ message: 'Order created.' }) };
   };
   ```

#### **Consumidor SQS**
2. Adicione outra função `sqs-consumer.js`:
   ```javascript
   const AWS = require('aws-sdk');

   exports.handler = async (event) => {
       for (const record of event.Records) {
           console.log('Processing message:', record.body);
       }
   };
   ```

---

### **3. Configurar o Secrets Manager**
- Crie um segredo fictício no Secrets Manager.
- Atualize as funções Lambda para acessar o segredo:
   ```javascript
   const secretsManager = new AWS.SecretsManager();
   const secret = await secretsManager.getSecretValue({ SecretId: 'my-secret' }).promise();
   console.log('Secret:', secret.SecretString);
   ```

---

### **4. Configurar o Pipeline de CI/CD**
1. Crie um arquivo `buildspec.yml`:
   ```yaml
   version: 0.2
   phases:
     install:
       runtime-versions:
         nodejs: 14
       commands:
         - npm install -g aws-cdk
     build:
       commands:
         - npm install
         - cdk synth
     post_build:
       commands:
         - cdk deploy --require-approval never
   ```

2. Configure um pipeline no CodePipeline para executar o build e o deploy automaticamente.

---

## **Testes e Validação**
1. **Testes Unitários:** Use Jest (Node.js) ou Pytest (Python) para testar as funções Lambda localmente.
2. **Validação de Fluxo:** Crie um pedido via API Gateway e acompanhe o fluxo até a fila SQS.

---

## **Conclusão**
Este projeto integra serviços essenciais da AWS para construir um sistema escalável e serverless, cobrindo os principais tópicos da certificação AWS Developer Associate.

