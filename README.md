# 🧩 OrderFlow — Estudo de Caso: Arquitetura de Microsserviços com Mensageria

> **Sistema distribuído de processamento de pedidos** com comunicação assíncrona via RabbitMQ, construído com Java 25, Spring Boot 4.0.6 e PostgreSQL.

---

## 📑 Índice

1. [Visão Geral do Projeto](#-visão-geral-do-projeto)
2. [Problema que o Projeto Resolve](#-problema-que-o-projeto-resolve)
3. [O que você irá aprender](#-o-que-você-irá-aprender)
4. [Stack Tecnológica](#-stack-tecnológica)
5. [Arquitetura Geral](#-arquitetura-geral)
6. [Mapa dos Microsserviços](#-mapa-dos-microsserviços)
7. [Topologia do RabbitMQ](#-topologia-do-rabbitmq)
8. [Schemas dos Bancos de Dados](#-schemas-dos-bancos-de-dados)
9. [Contratos de API (Endpoints)](#-contratos-de-api-endpoints)
10. [Fluxos de Negócio Detalhados](#-fluxos-de-negócio-detalhados)
11. [Estrutura de Diretórios](#-estrutura-de-diretórios)
12. [Pré-requisitos](#-pré-requisitos)
13. [Configuração do Ambiente](#️-configuração-do-ambiente)
14. [Configuração do docker-compose](#-configuração-do-docker-compose)
15. [Implementação Passo a Passo](#-implementação-passo-a-passo)
16. [Propriedades de Configuração](#-propriedades-de-configuração)
17. [Testando o Sistema](#-testando-o-sistema)
18. [Monitoramento e Observabilidade](#-monitoramento-e-observabilidade)
19. [Glossário](#-glossário)

---

## 🎯 Visão Geral do Projeto

**OrderFlow** é um sistema de processamento de pedidos de e-commerce construído como estudo de caso de **arquitetura distribuída de microsserviços**. O sistema simula o ciclo de vida completo de um pedido: desde a criação pelo cliente, passando pela reserva de estoque, processamento de pagamento, até o envio da notificação de confirmação.

O projeto é intencionalmente **simples no domínio de negócio** para que o foco total esteja na **infraestrutura distribuída**, nos **padrões de mensageria** e na **independência entre serviços**.

### Resumo Executivo

| Característica        | Detalhe                                      |
|-----------------------|----------------------------------------------|
| Estilo Arquitetural   | Microsserviços com Event-Driven Architecture |
| Comunicação Síncrona  | REST (HTTP/JSON entre cliente e serviços)    |
| Comunicação Assíncrona| RabbitMQ (AMQP entre os serviços)            |
| Persistência          | PostgreSQL (banco dedicado por serviço)      |
| Número de Serviços    | 4 microsserviços + 1 API Gateway             |
| Padrão de Mensagens   | Publish/Subscribe + Work Queue               |
| Linguagem             | Java 25                                      |
| Framework             | Spring Boot 4.0.6                            |

---

## 🔥 Problema que o Projeto Resolve

### Cenário: A Aplicação Monolítica com Gargalos

Imagine um sistema de e-commerce monolítico onde, ao realizar um pedido, **tudo acontece de forma síncrona e acoplada** em uma única transação:

```
Cliente → [Monólito: valida pedido → reserva estoque → processa pagamento → envia e-mail] → Resposta
```

**Problemas desse modelo:**

| Problema                    | Consequência                                                                        |
|-----------------------------|-------------------------------------------------------------------------------------|
| 🐢 Lentidão na resposta     | O cliente aguarda todas as etapas (pagamento + e-mail) antes de receber confirmação |
| 🔗 Acoplamento forte        | Uma falha no serviço de e-mail derruba TODO o processo de pedido                   |
| 📈 Escalabilidade limitada  | Só é possível escalar o sistema inteiro, mesmo que apenas o pagamento esteja lento  |
| 🚫 Ponto único de falha     | Se o módulo de estoque cair, nenhum pedido pode ser criado                         |
| 🔄 Deploy arriscado         | Qualquer alteração (em qualquer módulo) exige deploy de tudo                       |

### A Solução: Arquitetura de Microsserviços + Mensageria

Com **OrderFlow**, cada responsabilidade é isolada em um serviço independente. A comunicação entre eles é **assíncrona via RabbitMQ**:

```
Cliente → Order Service → [publica evento: OrderCreated]
                                    ↓ (assíncrono, paralelo)
               ┌────────────────────┴──────────────────┐
          Inventory Service                     Payment Service
          (reserva estoque)                  (processa pagamento)
               └────────────────────┬──────────────────┘
                                    ↓
                          Notification Service
                          (envia e-mail/SMS)
```

O cliente recebe uma resposta **imediata** (`202 Accepted`) e o restante do processamento ocorre de forma **resiliente e paralela** em background.

---

## 🎓 O que você irá aprender

Ao implementar este projeto, você terá experiência prática com:

- ✅ Decomposição de um domínio em microsserviços independentes
- ✅ Comunicação assíncrona com RabbitMQ (Exchanges, Queues, Bindings, Routing Keys)
- ✅ Padrão **Database per Service** (cada serviço tem seu próprio PostgreSQL schema/banco)
- ✅ Padrão **Outbox Pattern** para garantir consistência eventual sem transações distribuídas
- ✅ Padrão **Dead Letter Queue (DLQ)** para tratamento de mensagens com falha
- ✅ Idempotência no consumo de mensagens
- ✅ Configuração de projetos multi-módulo com Spring Boot
- ✅ Containerização com Docker e orquestração com Docker Compose
- ✅ Observabilidade básica com Spring Actuator + RabbitMQ Management UI

---

## 🛠 Stack Tecnológica

### Backend

| Tecnologia          | Versão   | Papel no Projeto                                         |
|---------------------|----------|----------------------------------------------------------|
| Java                | 25       | Linguagem principal, utilizando Virtual Threads (Loom)   |
| Spring Boot         | 4.0.6    | Framework base de todos os serviços                      |
| Spring Web          | -        | Exposição de APIs REST                                   |
| Spring AMQP         | -        | Integração com RabbitMQ (producer e consumer)            |
| Spring Data JPA     | -        | Persistência com Hibernate                               |
| Spring Validation   | -        | Validação de DTOs de entrada                             |
| Spring Actuator     | -        | Health checks e métricas                                 |
| Lombok              | -        | Redução de boilerplate (getters, builders, etc.)         |
| MapStruct           | -        | Mapeamento entre entidades e DTOs                        |

### Infraestrutura

| Tecnologia     | Versão | Papel no Projeto                                             |
|----------------|--------|--------------------------------------------------------------|
| PostgreSQL      | 16     | Banco de dados relacional (instância dedicada por serviço)   |
| RabbitMQ        | 3.13   | Message Broker para comunicação assíncrona entre serviços    |
| Docker          | -      | Containerização dos serviços e infraestrutura                |
| Docker Compose  | -      | Orquestração local do ambiente completo                      |

### Ferramentas de Build

| Ferramenta | Versão | Uso                       |
|------------|--------|---------------------------|
| Maven      | 3.9+   | Gerenciamento de projeto  |

---

## 🏗 Arquitetura Geral

### Diagrama de Componentes

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENTE (HTTP)                              │
└────────────────────────────┬────────────────────────────────────────┘
                             │ REST
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Porta 8080)                         │
│              (Spring Cloud Gateway / Roteamento)                    │
└──────┬───────────────────────┬──────────────────────────────────────┘
       │ /orders/**            │ /products/**
       ▼                       ▼
┌──────────────┐       ┌──────────────────┐
│ ORDER SERVICE│       │INVENTORY SERVICE │
│  Porta 8081  │       │   Porta 8082     │
│              │       │                  │
│  [DB: orders]│       │ [DB: inventory]  │
└──────┬───────┘       └────────┬─────────┘
       │                        │
       │  ┌─────────────────────┘
       │  │
       ▼  ▼
┌────────────────────────────────────────────────────────────────────┐
│                    RABBITMQ (Porta 5672 / UI: 15672)               │
│                                                                     │
│  Exchanges:                                                         │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────┐  │
│  │  order.exchange  │  │ payment.exchange  │  │inventory.exchange │  │
│  │  (topic)         │  │ (topic)           │  │ (topic)           │  │
│  └────────┬────────┘  └────────┬─────────┘  └─────────┬─────────┘  │
│           │                    │                        │            │
│  Queues:  ▼                    ▼                        ▼            │
│  ┌──────────────┐  ┌────────────────┐  ┌──────────────────────┐     │
│  │payment.queue │  │notification    │  │notification          │     │
│  │inventory     │  │.payment.queue  │  │.inventory.queue      │     │
│  │.queue        │  └────────────────┘  └──────────────────────┘     │
│  └──────────────┘                                                    │
└─────────────┬─────────────────┬────────────────────────────────────┘
              │                 │
              ▼                 ▼
┌─────────────────┐     ┌───────────────────────┐
│ PAYMENT SERVICE │     │  NOTIFICATION SERVICE  │
│   Porta 8083    │     │      Porta 8084        │
│                 │     │                        │
│ [DB: payments]  │     │  [DB: notifications]   │
└─────────────────┘     └───────────────────────┘
```

### Diagrama de Sequência: Fluxo Completo de um Pedido

```
Cliente     Gateway    OrderSvc   RabbitMQ    InventorySvc  PaymentSvc  NotificationSvc
   │            │          │          │             │             │              │
   │──POST /orders─────────►          │             │             │              │
   │            │──route──►│          │             │             │              │
   │            │          │──save──► DB            │             │              │
   │            │          │          │             │             │              │
   │            │          │──publish OrderCreated──►             │              │
   │            │          │          │─────────────────────────► │              │
   │            │          │          │             │             │              │
   │◄──202 Accepted────────┤          │             │             │              │
   │            │          │          │             │             │              │
   │            │          │          │  consume ◄──│             │              │
   │            │          │          │  reserve stock            │              │
   │            │          │          │  publish InventoryReserved►              │
   │            │          │          │             │             │              │
   │            │          │          │  consume ◄──────────────── │              │
   │            │          │          │  process payment           │              │
   │            │          │          │  publish PaymentProcessed──────────────► │
   │            │          │          │             │             │              │
   │            │          │          │             │             │   send email │
   │            │          │          │             │             │   save record│
   │            │          │          │             │             │              │
```

---

## 🧩 Mapa dos Microsserviços

### Tabela de Serviços

| Serviço               | Porta | Responsabilidade Principal                                            | Produz Eventos                          | Consome Eventos                                    |
|-----------------------|-------|-----------------------------------------------------------------------|-----------------------------------------|----------------------------------------------------|
| **api-gateway**       | 8080  | Roteamento de requisições, ponto único de entrada                     | —                                       | —                                                  |
| **order-service**     | 8081  | Criação e gerenciamento do ciclo de vida dos pedidos                  | `OrderCreated`, `OrderCancelled`        | `PaymentProcessed`, `InventoryReservationFailed`   |
| **inventory-service** | 8082  | Gerenciamento de produtos e reserva/liberação de estoque              | `InventoryReserved`, `InventoryFailed`  | `OrderCreated`, `OrderCancelled`                   |
| **payment-service**   | 8083  | Processamento de pagamentos (simulado)                                | `PaymentProcessed`, `PaymentFailed`     | `InventoryReserved`                                |
| **notification-service** | 8084 | Envio de notificações (e-mail simulado com log)                   | —                                       | `PaymentProcessed`, `PaymentFailed`, `OrderCancelled` |

### Detalhamento por Serviço

#### 1. `api-gateway`

- **Tipo:** Spring Cloud Gateway
- **Função:** Ponto único de entrada para os clientes. Roteia requisições para os serviços corretos com base no path da URL.
- **Não possui banco de dados próprio.**
- **Não produz nem consome mensagens RabbitMQ.**

**Regras de Roteamento:**

| Path de Entrada            | Serviço de Destino     | Reescrita de Path         |
|----------------------------|------------------------|---------------------------|
| `/api/orders/**`           | `order-service:8081`   | `/orders/**`              |
| `/api/products/**`         | `inventory-service:8082`| `/products/**`           |
| `/api/payments/**`         | `payment-service:8083` | `/payments/**`            |
| `/api/notifications/**`    | `notification-service:8084` | `/notifications/**`  |

---

#### 2. `order-service`

- **Função:** É o **ponto de entrada do domínio de pedidos**. Recebe o pedido do cliente via REST, persiste no banco, e publica o evento `OrderCreated` no RabbitMQ. Aguarda eventos dos outros serviços para atualizar o status do pedido.
- **Banco de dados:** `db_orders` (PostgreSQL)

**Estados de um Pedido (Order Status):**

```
PENDING ──────────────────────────────────────────► CONFIRMED
   │        (InventoryReserved + PaymentProcessed)
   │
   ├──► INVENTORY_FAILED ──► CANCELLED
   │
   └──► PAYMENT_FAILED ──► CANCELLED
```

| Status                 | Descrição                                                         |
|------------------------|-------------------------------------------------------------------|
| `PENDING`              | Pedido criado, aguardando reserva de estoque e pagamento          |
| `INVENTORY_RESERVED`   | Estoque reservado com sucesso, aguardando pagamento               |
| `CONFIRMED`            | Pagamento processado e estoque reservado — pedido confirmado      |
| `INVENTORY_FAILED`     | Falha na reserva de estoque                                       |
| `PAYMENT_FAILED`       | Falha no processamento do pagamento                               |
| `CANCELLED`            | Pedido cancelado após falha em alguma etapa                       |

---

#### 3. `inventory-service`

- **Função:** Gerencia o cadastro de produtos e o controle de estoque. Consome o evento `OrderCreated` para tentar reservar os itens solicitados, e publica o resultado.
- **Banco de dados:** `db_inventory` (PostgreSQL)

**Estados de um Produto:**

| Campo        | Tipo    | Descrição                              |
|--------------|---------|----------------------------------------|
| `quantity`   | Integer | Quantidade total disponível em estoque |
| `reserved`   | Integer | Quantidade reservada por pedidos       |
| `available`  | Computed| `quantity - reserved`                  |

---

#### 4. `payment-service`

- **Função:** Simula o processamento de um pagamento. Consome o evento `InventoryReserved` e gera um resultado de pagamento (`PaymentProcessed` ou `PaymentFailed`) com base em uma lógica aleatória configurável (para fins de teste).
- **Banco de dados:** `db_payments` (PostgreSQL)

> **Nota de Implementação:** Em produção, este serviço se integraria com um gateway de pagamento real (ex: Stripe, PagSeguro). No estudo de caso, a aprovação será simulada.

---

#### 5. `notification-service`

- **Função:** Consome eventos de outros serviços e simula o envio de notificações ao cliente (via log estruturado). É o serviço mais simples e serve para demonstrar o padrão **fan-out** (múltiplos eventos de diferentes origens chegando ao mesmo serviço).
- **Banco de dados:** `db_notifications` (PostgreSQL — apenas para registro de histórico)

---

## 📨 Topologia do RabbitMQ

### Conceitos Utilizados

| Conceito        | Tipo utilizado | Descrição                                                                      |
|-----------------|----------------|--------------------------------------------------------------------------------|
| **Exchange**    | `topic`        | Roteador de mensagens. Direciona para filas com base na routing key            |
| **Queue**       | `durable`      | Fila persistente que sobrevive a reinicializações do broker                    |
| **Binding**     | —              | Ligação entre uma exchange e uma fila com uma routing key                      |
| **Routing Key** | `*.evento`     | Chave usada pelo exchange para decidir para qual fila enviar a mensagem        |
| **DLQ**         | Dead Letter     | Fila que recebe mensagens que não puderam ser processadas após N tentativas    |

### Tabela Completa de Exchanges, Filas e Bindings

| Exchange                | Tipo  | Routing Key                    | Fila de Destino                    | Consumidor              |
|-------------------------|-------|--------------------------------|------------------------------------|-------------------------|
| `order.exchange`        | topic | `order.created`                | `inventory.order.queue`            | `inventory-service`     |
| `order.exchange`        | topic | `order.created`                | `payment.order.queue`              | *(aguarda estoque)*     |
| `order.exchange`        | topic | `order.cancelled`              | `inventory.cancel.queue`           | `inventory-service`     |
| `order.exchange`        | topic | `order.cancelled`              | `notification.cancel.queue`        | `notification-service`  |
| `inventory.exchange`    | topic | `inventory.reserved`           | `payment.inventory.queue`          | `payment-service`       |
| `inventory.exchange`    | topic | `inventory.failed`             | `order.inventory.failed.queue`     | `order-service`         |
| `inventory.exchange`    | topic | `inventory.failed`             | `notification.inventory.queue`     | `notification-service`  |
| `payment.exchange`      | topic | `payment.processed`            | `order.payment.queue`              | `order-service`         |
| `payment.exchange`      | topic | `payment.processed`            | `notification.payment.queue`       | `notification-service`  |
| `payment.exchange`      | topic | `payment.failed`               | `order.payment.failed.queue`       | `order-service`         |
| `payment.exchange`      | topic | `payment.failed`               | `notification.payment.failed.queue`| `notification-service`  |

### Dead Letter Queues (DLQs)

Cada fila principal possui uma DLQ correspondente. Mensagens são enviadas à DLQ após **3 tentativas** de processamento com falha.

| Fila Principal               | DLQ                                 | TTL de Retry |
|------------------------------|-------------------------------------|--------------|
| `inventory.order.queue`      | `inventory.order.queue.dlq`         | 5000ms       |
| `payment.inventory.queue`    | `payment.inventory.queue.dlq`       | 5000ms       |
| `order.payment.queue`        | `order.payment.queue.dlq`           | 5000ms       |
| `notification.payment.queue` | `notification.payment.queue.dlq`    | 5000ms       |

### Diagrama Visual das Filas

```
                    ┌─────────────────────────────────────────────────────┐
                    │               RABBITMQ TOPOLOGY                     │
                    └─────────────────────────────────────────────────────┘

 [order-service]
      │
      ├──publish──► [order.exchange] (topic)
      │                  │
      │          routing key: order.created ──────────────────────────────────────────────►[inventory.order.queue]──► inventory-service
      │                  │
      │          routing key: order.cancelled ────────────────────────────────────────────►[inventory.cancel.queue]──► inventory-service
      │                  │                   └──────────────────────────────────────────►[notification.cancel.queue]──► notification-service
      │
      ◄──consume──[order.payment.queue] ◄── [payment.exchange] ◄── routing key: payment.processed ◄── payment-service
      ◄──consume──[order.inventory.failed.queue] ◄── [inventory.exchange] ◄── routing key: inventory.failed ◄── inventory-service

 [inventory-service]
      │
      ├──publish──► [inventory.exchange] (topic)
      │                  │
      │          routing key: inventory.reserved ────────────────────────────────────────►[payment.inventory.queue]──► payment-service
      │                  │
      │          routing key: inventory.failed ──────────────────────────────────────────►[order.inventory.failed.queue]──► order-service
      │                                       └──────────────────────────────────────────►[notification.inventory.queue]──► notification-service

 [payment-service]
      │
      ├──publish──► [payment.exchange] (topic)
                         │
                 routing key: payment.processed ──────────────────────────────────────────►[order.payment.queue]──► order-service
                         │                    └──────────────────────────────────────────►[notification.payment.queue]──► notification-service
                         │
                 routing key: payment.failed ────────────────────────────────────────────►[order.payment.failed.queue]──► order-service
                                             └──────────────────────────────────────────►[notification.payment.failed.queue]──► notification-service
```

---

## 🗄 Schemas dos Bancos de Dados

### Banco: `db_orders` (order-service)

#### Tabela: `orders`

| Coluna            | Tipo           | Restrições          | Descrição                              |
|-------------------|----------------|---------------------|----------------------------------------|
| `id`              | UUID           | PK, NOT NULL        | Identificador único do pedido          |
| `customer_id`     | UUID           | NOT NULL            | ID do cliente que realizou o pedido    |
| `customer_email`  | VARCHAR(255)   | NOT NULL            | E-mail do cliente para notificações    |
| `status`          | VARCHAR(50)    | NOT NULL            | Status atual do pedido (enum)          |
| `total_amount`    | DECIMAL(10,2)  | NOT NULL            | Valor total do pedido                  |
| `created_at`      | TIMESTAMP      | NOT NULL, DEFAULT NOW | Data/hora de criação                 |
| `updated_at`      | TIMESTAMP      | NOT NULL            | Data/hora da última atualização        |

#### Tabela: `order_items`

| Coluna       | Tipo          | Restrições          | Descrição                            |
|--------------|---------------|---------------------|--------------------------------------|
| `id`         | UUID          | PK, NOT NULL        | Identificador único do item          |
| `order_id`   | UUID          | FK → orders.id      | Referência ao pedido pai             |
| `product_id` | UUID          | NOT NULL            | ID do produto (referência externa)   |
| `quantity`   | INTEGER       | NOT NULL, > 0       | Quantidade do produto solicitada     |
| `unit_price` | DECIMAL(10,2) | NOT NULL            | Preço unitário no momento do pedido  |

#### Tabela: `outbox_events` *(Outbox Pattern)*

| Coluna          | Tipo         | Restrições          | Descrição                                      |
|-----------------|--------------|---------------------|------------------------------------------------|
| `id`            | UUID         | PK, NOT NULL        | Identificador do evento                        |
| `aggregate_id`  | UUID         | NOT NULL            | ID da entidade que gerou o evento (order.id)   |
| `event_type`    | VARCHAR(100) | NOT NULL            | Tipo do evento (ex: `OrderCreated`)            |
| `payload`       | JSONB        | NOT NULL            | Corpo serializado do evento                    |
| `published`     | BOOLEAN      | DEFAULT FALSE       | Se o evento já foi publicado no RabbitMQ       |
| `created_at`    | TIMESTAMP    | NOT NULL            | Data/hora de criação do evento                 |

---

### Banco: `db_inventory` (inventory-service)

#### Tabela: `products`

| Coluna          | Tipo           | Restrições          | Descrição                                |
|-----------------|----------------|---------------------|------------------------------------------|
| `id`            | UUID           | PK, NOT NULL        | Identificador único do produto           |
| `name`          | VARCHAR(255)   | NOT NULL            | Nome do produto                          |
| `description`   | TEXT           | —                   | Descrição detalhada                      |
| `price`         | DECIMAL(10,2)  | NOT NULL            | Preço unitário do produto                |
| `quantity`      | INTEGER        | NOT NULL, >= 0      | Quantidade total em estoque              |
| `reserved`      | INTEGER        | NOT NULL, DEFAULT 0 | Quantidade reservada por pedidos ativos  |
| `created_at`    | TIMESTAMP      | NOT NULL            | Data de cadastro                         |

#### Tabela: `stock_reservations`

| Coluna       | Tipo        | Restrições          | Descrição                               |
|--------------|-------------|---------------------|-----------------------------------------|
| `id`         | UUID        | PK, NOT NULL        | Identificador da reserva                |
| `order_id`   | UUID        | NOT NULL, UNIQUE    | ID do pedido que originou a reserva     |
| `product_id` | UUID        | FK → products.id    | Produto reservado                       |
| `quantity`   | INTEGER     | NOT NULL            | Quantidade reservada                    |
| `status`     | VARCHAR(50) | NOT NULL            | `RESERVED`, `RELEASED`, `CONFIRMED`     |
| `created_at` | TIMESTAMP   | NOT NULL            | Data da reserva                         |

---

### Banco: `db_payments` (payment-service)

#### Tabela: `payments`

| Coluna            | Tipo           | Restrições          | Descrição                                    |
|-------------------|----------------|---------------------|----------------------------------------------|
| `id`              | UUID           | PK, NOT NULL        | Identificador único do pagamento             |
| `order_id`        | UUID           | NOT NULL, UNIQUE    | ID do pedido referenciado                    |
| `customer_id`     | UUID           | NOT NULL            | ID do cliente                                |
| `amount`          | DECIMAL(10,2)  | NOT NULL            | Valor do pagamento                           |
| `status`          | VARCHAR(50)    | NOT NULL            | `PENDING`, `APPROVED`, `REJECTED`, `REFUNDED`|
| `payment_method`  | VARCHAR(50)    | NOT NULL            | `CREDIT_CARD`, `PIX`, `BOLETO`               |
| `transaction_id`  | VARCHAR(100)   | —                   | ID externo da transação (simulado)           |
| `processed_at`    | TIMESTAMP      | —                   | Data/hora do processamento                   |
| `created_at`      | TIMESTAMP      | NOT NULL            | Data de criação                              |

---

### Banco: `db_notifications` (notification-service)

#### Tabela: `notifications`

| Coluna         | Tipo           | Restrições          | Descrição                                       |
|----------------|----------------|---------------------|-------------------------------------------------|
| `id`           | UUID           | PK, NOT NULL        | Identificador único da notificação              |
| `order_id`     | UUID           | NOT NULL            | Pedido relacionado                              |
| `customer_email`| VARCHAR(255)  | NOT NULL            | Destinatário                                    |
| `type`         | VARCHAR(100)   | NOT NULL            | Tipo da notificação (ex: `ORDER_CONFIRMED`)     |
| `message`      | TEXT           | NOT NULL            | Conteúdo da notificação                         |
| `sent_at`      | TIMESTAMP      | NOT NULL            | Data/hora do envio (ou tentativa)               |
| `success`      | BOOLEAN        | NOT NULL            | Se o envio foi bem-sucedido                     |

---

## 🌐 Contratos de API (Endpoints)

### Order Service (`/orders`)

| Método | Endpoint              | Descrição                                | Request Body                | Response                          |
|--------|-----------------------|------------------------------------------|-----------------------------|-----------------------------------|
| POST   | `/orders`             | Cria um novo pedido                      | `CreateOrderRequest`        | `202 Accepted` + `OrderResponse`  |
| GET    | `/orders/{id}`        | Busca um pedido por ID                   | —                           | `200 OK` + `OrderResponse`        |
| GET    | `/orders/customer/{customerId}` | Lista pedidos de um cliente   | —                           | `200 OK` + `List<OrderResponse>`  |
| DELETE | `/orders/{id}/cancel` | Cancela um pedido (se PENDING)           | —                           | `200 OK`                          |

**CreateOrderRequest:**
```json
{
  "customerId": "uuid-do-cliente",
  "customerEmail": "cliente@email.com",
  "paymentMethod": "CREDIT_CARD",
  "items": [
    {
      "productId": "uuid-do-produto",
      "quantity": 2
    }
  ]
}
```

**OrderResponse:**
```json
{
  "id": "uuid-do-pedido",
  "customerId": "uuid-do-cliente",
  "status": "PENDING",
  "totalAmount": 199.90,
  "items": [
    {
      "productId": "uuid-do-produto",
      "quantity": 2,
      "unitPrice": 99.95
    }
  ],
  "createdAt": "2025-01-15T10:30:00Z"
}
```

---

### Inventory Service (`/products`)

| Método | Endpoint                 | Descrição                                 | Request Body              | Response                         |
|--------|--------------------------|-------------------------------------------|---------------------------|----------------------------------|
| POST   | `/products`              | Cadastra um novo produto                  | `CreateProductRequest`    | `201 Created` + `ProductResponse`|
| GET    | `/products`              | Lista todos os produtos                   | —                         | `200 OK` + `List<ProductResponse>`|
| GET    | `/products/{id}`         | Busca produto por ID                      | —                         | `200 OK` + `ProductResponse`     |
| PUT    | `/products/{id}`         | Atualiza dados de um produto              | `UpdateProductRequest`    | `200 OK` + `ProductResponse`     |
| POST   | `/products/{id}/stock`   | Adiciona estoque ao produto               | `AddStockRequest`         | `200 OK`                         |

---

### Payment Service (`/payments`)

| Método | Endpoint           | Descrição                        | Response                          |
|--------|--------------------|----------------------------------|-----------------------------------|
| GET    | `/payments/{orderId}` | Busca pagamento por pedido    | `200 OK` + `PaymentResponse`      |
| GET    | `/payments`        | Lista todos os pagamentos        | `200 OK` + `List<PaymentResponse>`|

---

### Notification Service (`/notifications`)

| Método | Endpoint                       | Descrição                            | Response                              |
|--------|--------------------------------|--------------------------------------|---------------------------------------|
| GET    | `/notifications/order/{orderId}` | Busca notificações de um pedido    | `200 OK` + `List<NotificationResponse>`|

---

## 🔄 Fluxos de Negócio Detalhados

### Fluxo 1: Pedido com Sucesso

```
1. Cliente chama POST /api/orders com os itens do pedido

2. order-service:
   - Valida o request (campos obrigatórios, itens não vazios)
   - Consulta inventory-service via REST para verificar preço dos produtos
   - Cria o pedido com status = PENDING
   - Persiste o evento OrderCreated na tabela outbox_events
   - Um @Scheduled publica o evento no RabbitMQ via order.exchange
   - Retorna 202 Accepted com o ID do pedido

3. inventory-service (consome inventory.order.queue):
   - Recebe OrderCreated
   - Verifica se há estoque disponível para todos os itens
   - Cria registros em stock_reservations com status = RESERVED
   - Decrementa o campo "reserved" nos produtos
   - Publica InventoryReserved em inventory.exchange com routing key inventory.reserved

4. payment-service (consome payment.inventory.queue):
   - Recebe InventoryReserved
   - Cria um registro em payments com status = PENDING
   - Simula o processamento (aprovação aleatória ou baseada em configuração)
   - Atualiza status do pagamento para APPROVED ou REJECTED
   - Publica PaymentProcessed ou PaymentFailed em payment.exchange

5. order-service (consome order.payment.queue):
   - Recebe PaymentProcessed
   - Atualiza status do pedido para CONFIRMED

6. notification-service (consome notification.payment.queue):
   - Recebe PaymentProcessed
   - Registra notificação no banco
   - Loga a notificação: "Seu pedido {orderId} foi confirmado!"
```

### Fluxo 2: Falha no Estoque

```
1–2. Igual ao Fluxo 1.

3. inventory-service:
   - Recebe OrderCreated
   - Verifica que NÃO há estoque suficiente
   - Publica InventoryFailed em inventory.exchange com routing key inventory.failed

4. order-service (consome order.inventory.failed.queue):
   - Recebe InventoryFailed
   - Atualiza status do pedido para CANCELLED

5. notification-service (consome notification.inventory.queue):
   - Recebe InventoryFailed
   - Notifica cliente: "Infelizmente seu pedido {orderId} foi cancelado por falta de estoque."
```

### Fluxo 3: Falha no Pagamento

```
1–4. Igual ao Fluxo 1 (estoque reservado com sucesso).

5. payment-service:
   - Processa e rejeita o pagamento
   - Publica PaymentFailed em payment.exchange

6. order-service (consome order.payment.failed.queue):
   - Atualiza status do pedido para CANCELLED

7. inventory-service (precisa liberar a reserva):
   - Order-service publica OrderCancelled
   - inventory-service consome e libera a reserva (status = RELEASED)

8. notification-service:
   - Notifica cliente sobre a falha no pagamento.
```

### Fluxo 4: Mensagem com Falha (DLQ)

```
1. Uma mensagem é consumida por um serviço.
2. O processamento lança uma exceção (ex: falha de banco de dados).
3. Spring AMQP faz o nack da mensagem.
4. RabbitMQ reenvia a mensagem até 3 vezes (configurado via x-delivery-limit).
5. Após 3 falhas, a mensagem é movida para a DLQ correspondente.
6. Um alerta pode ser configurado no RabbitMQ Management para notificar sobre mensagens na DLQ.
7. Um desenvolvedor pode inspecionar e reprocessar manualmente via Management UI.
```

---

## 📁 Estrutura de Diretórios

```
orderflow/
├── pom.xml                          ← POM pai (multi-module Maven)
├── docker-compose.yml               ← Orquestração completa do ambiente
├── docker-compose.infra.yml         ← Apenas infraestrutura (RabbitMQ + PostgreSQL)
├── README.md
│
├── api-gateway/
│   ├── pom.xml
│   └── src/main/
│       ├── java/com/orderflow/gateway/
│       │   └── GatewayApplication.java
│       └── resources/
│           └── application.yml      ← Rotas do gateway
│
├── order-service/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/orderflow/order/
│       │   ├── OrderServiceApplication.java
│       │   ├── config/
│       │   │   ├── RabbitMQConfig.java       ← Declaração de exchanges/filas/bindings
│       │   │   └── JpaConfig.java
│       │   ├── controller/
│       │   │   └── OrderController.java
│       │   ├── dto/
│       │   │   ├── request/CreateOrderRequest.java
│       │   │   └── response/OrderResponse.java
│       │   ├── entity/
│       │   │   ├── Order.java
│       │   │   ├── OrderItem.java
│       │   │   └── OutboxEvent.java
│       │   ├── event/                        ← Classes de eventos (mensagens)
│       │   │   ├── OrderCreatedEvent.java
│       │   │   └── OrderCancelledEvent.java
│       │   ├── listener/                     ← Consumers do RabbitMQ
│       │   │   └── PaymentEventListener.java
│       │   ├── publisher/                    ← Producers do RabbitMQ
│       │   │   └── OrderEventPublisher.java
│       │   ├── repository/
│       │   │   ├── OrderRepository.java
│       │   │   └── OutboxEventRepository.java
│       │   └── service/
│       │       └── OrderService.java
│       └── resources/
│           ├── application.yml
│           └── db/migration/                 ← Flyway migrations
│               ├── V1__create_orders_table.sql
│               ├── V2__create_order_items_table.sql
│               └── V3__create_outbox_events_table.sql
│
├── inventory-service/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/orderflow/inventory/
│       │   ├── InventoryServiceApplication.java
│       │   ├── config/RabbitMQConfig.java
│       │   ├── controller/ProductController.java
│       │   ├── dto/...
│       │   ├── entity/
│       │   │   ├── Product.java
│       │   │   └── StockReservation.java
│       │   ├── event/
│       │   │   ├── InventoryReservedEvent.java
│       │   │   └── InventoryFailedEvent.java
│       │   ├── listener/OrderEventListener.java
│       │   ├── publisher/InventoryEventPublisher.java
│       │   ├── repository/...
│       │   └── service/InventoryService.java
│       └── resources/
│           ├── application.yml
│           └── db/migration/...
│
├── payment-service/
│   ├── pom.xml
│   └── src/
│       ├── main/java/com/orderflow/payment/
│       │   ├── PaymentServiceApplication.java
│       │   ├── config/RabbitMQConfig.java
│       │   ├── controller/PaymentController.java
│       │   ├── entity/Payment.java
│       │   ├── event/
│       │   │   ├── PaymentProcessedEvent.java
│       │   │   └── PaymentFailedEvent.java
│       │   ├── listener/InventoryEventListener.java
│       │   ├── publisher/PaymentEventPublisher.java
│       │   ├── repository/PaymentRepository.java
│       │   └── service/PaymentService.java
│       └── resources/application.yml
│
└── notification-service/
    ├── pom.xml
    └── src/
        ├── main/java/com/orderflow/notification/
        │   ├── NotificationServiceApplication.java
        │   ├── config/RabbitMQConfig.java
        │   ├── entity/Notification.java
        │   ├── listener/
        │   │   ├── PaymentNotificationListener.java
        │   │   └── InventoryNotificationListener.java
        │   ├── repository/NotificationRepository.java
        │   └── service/NotificationService.java
        └── resources/application.yml
```

---

## ✅ Pré-requisitos

Certifique-se de ter as seguintes ferramentas instaladas antes de iniciar:

| Ferramenta     | Versão Mínima | Verificar com           |
|----------------|---------------|-------------------------|
| Java JDK       | 25            | `java -version`         |
| Maven          | 3.9+          | `mvn -version`          |
| Docker         | 24+           | `docker --version`      |
| Docker Compose | 2.x           | `docker compose version`|
| Git            | 2.x           | `git --version`         |

---

## ⚙️ Configuração do Ambiente

### 1. Clonar o repositório

```bash
git clone https://github.com/seu-usuario/orderflow.git
cd orderflow
```

### 2. Subir apenas a infraestrutura (desenvolvimento local)

Para desenvolvimento, suba apenas RabbitMQ e os bancos PostgreSQL:

```bash
docker compose -f docker-compose.infra.yml up -d
```

Isso inicializará:
- RabbitMQ na porta `5672` (AMQP) e `15672` (Management UI)
- 4 instâncias PostgreSQL nas portas `5433`, `5434`, `5435`, `5436`

### 3. Subir o ambiente completo com todos os serviços

```bash
docker compose up -d --build
```

### 4. Verificar os serviços

```bash
docker compose ps
```

Todos os serviços devem estar com status `healthy`.

### 5. Acessar os serviços

| Serviço                  | URL                                                         | Credenciais        |
|--------------------------|-------------------------------------------------------------|--------------------|
| API Gateway              | http://localhost:8080                                       | —                  |
| RabbitMQ Management UI   | http://localhost:15672                                      | guest / guest      |
| Order Service Actuator   | http://localhost:8081/actuator/health                       | —                  |
| Inventory Service Actuator | http://localhost:8082/actuator/health                     | —                  |
| Payment Service Actuator | http://localhost:8083/actuator/health                       | —                  |
| Notification Service Actuator | http://localhost:8084/actuator/health                  | —                  |

---

## 🐳 Configuração do docker-compose

### `docker-compose.infra.yml`

```yaml
version: '3.9'

services:
  rabbitmq:
    image: rabbitmq:3.13-management
    container_name: orderflow-rabbitmq
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 10s
      timeout: 5s
      retries: 5

  db-orders:
    image: postgres:16
    container_name: orderflow-db-orders
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: db_orders
      POSTGRES_USER: orderflow
      POSTGRES_PASSWORD: orderflow123
    volumes:
      - db_orders_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orderflow -d db_orders"]
      interval: 10s
      timeout: 5s
      retries: 5

  db-inventory:
    image: postgres:16
    container_name: orderflow-db-inventory
    ports:
      - "5434:5432"
    environment:
      POSTGRES_DB: db_inventory
      POSTGRES_USER: orderflow
      POSTGRES_PASSWORD: orderflow123
    volumes:
      - db_inventory_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orderflow -d db_inventory"]
      interval: 10s
      timeout: 5s
      retries: 5

  db-payments:
    image: postgres:16
    container_name: orderflow-db-payments
    ports:
      - "5435:5432"
    environment:
      POSTGRES_DB: db_payments
      POSTGRES_USER: orderflow
      POSTGRES_PASSWORD: orderflow123
    volumes:
      - db_payments_data:/var/lib/postgresql/data

  db-notifications:
    image: postgres:16
    container_name: orderflow-db-notifications
    ports:
      - "5436:5432"
    environment:
      POSTGRES_DB: db_notifications
      POSTGRES_USER: orderflow
      POSTGRES_PASSWORD: orderflow123
    volumes:
      - db_notifications_data:/var/lib/postgresql/data

volumes:
  rabbitmq_data:
  db_orders_data:
  db_inventory_data:
  db_payments_data:
  db_notifications_data:
```

### `docker-compose.yml` (ambiente completo)

```yaml
version: '3.9'

include:
  - docker-compose.infra.yml

services:
  api-gateway:
    build: ./api-gateway
    container_name: orderflow-gateway
    ports:
      - "8080:8080"
    depends_on:
      - order-service
      - inventory-service
    environment:
      ORDER_SERVICE_URL: http://order-service:8081
      INVENTORY_SERVICE_URL: http://inventory-service:8082
      PAYMENT_SERVICE_URL: http://payment-service:8083
      NOTIFICATION_SERVICE_URL: http://notification-service:8084

  order-service:
    build: ./order-service
    container_name: orderflow-order
    ports:
      - "8081:8081"
    depends_on:
      db-orders:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db-orders:5432/db_orders
      SPRING_DATASOURCE_USERNAME: orderflow
      SPRING_DATASOURCE_PASSWORD: orderflow123
      SPRING_RABBITMQ_HOST: rabbitmq
      INVENTORY_SERVICE_URL: http://inventory-service:8082

  inventory-service:
    build: ./inventory-service
    container_name: orderflow-inventory
    ports:
      - "8082:8082"
    depends_on:
      db-inventory:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db-inventory:5432/db_inventory
      SPRING_DATASOURCE_USERNAME: orderflow
      SPRING_DATASOURCE_PASSWORD: orderflow123
      SPRING_RABBITMQ_HOST: rabbitmq

  payment-service:
    build: ./payment-service
    container_name: orderflow-payment
    ports:
      - "8083:8083"
    depends_on:
      db-payments:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db-payments:5432/db_payments
      SPRING_DATASOURCE_USERNAME: orderflow
      SPRING_DATASOURCE_PASSWORD: orderflow123
      SPRING_RABBITMQ_HOST: rabbitmq
      PAYMENT_APPROVAL_RATE: "0.8"  # 80% de aprovação nos testes

  notification-service:
    build: ./notification-service
    container_name: orderflow-notification
    ports:
      - "8084:8084"
    depends_on:
      db-notifications:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db-notifications:5432/db_notifications
      SPRING_DATASOURCE_USERNAME: orderflow
      SPRING_DATASOURCE_PASSWORD: orderflow123
      SPRING_RABBITMQ_HOST: rabbitmq
```

---

## 🚀 Implementação Passo a Passo

### Etapa 1: POM Pai (Multi-Module)

Crie o `pom.xml` raiz com as dependências comuns a todos os módulos:

```xml
<project>
  <groupId>com.orderflow</groupId>
  <artifactId>orderflow-parent</artifactId>
  <version>1.0.0-SNAPSHOT</version>
  <packaging>pom</packaging>

  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.6</version>
  </parent>

  <modules>
    <module>api-gateway</module>
    <module>order-service</module>
    <module>inventory-service</module>
    <module>payment-service</module>
    <module>notification-service</module>
  </modules>

  <properties>
    <java.version>25</java.version>
    <maven.compiler.source>25</maven.compiler.source>
    <maven.compiler.target>25</maven.compiler.target>
  </properties>

  <dependencyManagement>
    <dependencies>
      <!-- Spring Cloud BOM -->
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>2025.0.0</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
</project>
```

---

### Etapa 2: Dependências de cada serviço (pom.xml individual)

Dependências comuns para `order-service`, `inventory-service`, `payment-service` e `notification-service`:

```xml
<dependencies>
  <!-- Web -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>

  <!-- JPA + PostgreSQL -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>

  <!-- RabbitMQ -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
  </dependency>

  <!-- Validação -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>

  <!-- Actuator -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>

  <!-- Flyway (migrations) -->
  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
  </dependency>
  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
  </dependency>

  <!-- Lombok -->
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>

  <!-- Testes -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit-test</artifactId>
    <scope>test</scope>
  </dependency>
</dependencies>
```

---

### Etapa 3: Configuração do RabbitMQ no order-service

```java
// order-service: src/main/java/com/orderflow/order/config/RabbitMQConfig.java

@Configuration
public class RabbitMQConfig {

    // ===================== EXCHANGES =====================

    @Bean
    public TopicExchange orderExchange() {
        return ExchangeBuilder
            .topicExchange("order.exchange")
            .durable(true)
            .build();
    }

    // ===================== QUEUES (com DLQ) =====================

    @Bean
    public Queue orderPaymentQueue() {
        return QueueBuilder
            .durable("order.payment.queue")
            .withArgument("x-dead-letter-exchange", "order.payment.queue.dlq")
            .withArgument("x-delivery-limit", 3)
            .build();
    }

    @Bean
    public Queue orderPaymentQueueDlq() {
        return QueueBuilder.durable("order.payment.queue.dlq").build();
    }

    @Bean
    public Queue orderInventoryFailedQueue() {
        return QueueBuilder
            .durable("order.inventory.failed.queue")
            .withArgument("x-dead-letter-exchange", "order.inventory.failed.queue.dlq")
            .withArgument("x-delivery-limit", 3)
            .build();
    }

    @Bean
    public Queue orderInventoryFailedQueueDlq() {
        return QueueBuilder.durable("order.inventory.failed.queue.dlq").build();
    }

    // ===================== BINDINGS =====================

    // Essa exchange é declarada por inventory-service, mas fazemos o binding aqui
    // ALTERNATIVA: cada serviço declara apenas suas próprias exchanges/filas
    @Bean
    public TopicExchange paymentExchange() {
        return ExchangeBuilder.topicExchange("payment.exchange").durable(true).build();
    }

    @Bean
    public TopicExchange inventoryExchange() {
        return ExchangeBuilder.topicExchange("inventory.exchange").durable(true).build();
    }

    @Bean
    public Binding bindingPaymentProcessed() {
        return BindingBuilder
            .bind(orderPaymentQueue())
            .to(paymentExchange())
            .with("payment.processed");
    }

    @Bean
    public Binding bindingInventoryFailed() {
        return BindingBuilder
            .bind(orderInventoryFailedQueue())
            .to(inventoryExchange())
            .with("inventory.failed");
    }

    // ===================== SERIALIZAÇÃO JSON =====================

    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public AmqpTemplate amqpTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate template = new RabbitTemplate(connectionFactory);
        template.setMessageConverter(jsonMessageConverter());
        return template;
    }
}
```

---

### Etapa 4: Publisher de Eventos (order-service)

```java
// order-service: src/main/java/com/orderflow/order/publisher/OrderEventPublisher.java

@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventPublisher {

    private final AmqpTemplate amqpTemplate;

    public void publishOrderCreated(OrderCreatedEvent event) {
        log.info("Publicando OrderCreated para orderId: {}", event.getOrderId());
        amqpTemplate.convertAndSend("order.exchange", "order.created", event);
    }

    public void publishOrderCancelled(OrderCancelledEvent event) {
        log.info("Publicando OrderCancelled para orderId: {}", event.getOrderId());
        amqpTemplate.convertAndSend("order.exchange", "order.cancelled", event);
    }
}
```

---

### Etapa 5: Consumer de Eventos (order-service)

```java
// order-service: src/main/java/com/orderflow/order/listener/PaymentEventListener.java

@Component
@RequiredArgsConstructor
@Slf4j
public class PaymentEventListener {

    private final OrderService orderService;

    @RabbitListener(queues = "order.payment.queue")
    public void onPaymentProcessed(PaymentProcessedEvent event) {
        log.info("Recebido PaymentProcessed para orderId: {}", event.getOrderId());
        orderService.confirmOrder(event.getOrderId());
    }

    @RabbitListener(queues = "order.inventory.failed.queue")
    public void onInventoryFailed(InventoryFailedEvent event) {
        log.warn("Recebido InventoryFailed para orderId: {}", event.getOrderId());
        orderService.cancelOrder(event.getOrderId(), "Estoque insuficiente");
    }
}
```

---

### Etapa 6: Outbox Pattern (garantia de entrega)

O Outbox Pattern garante que o evento seja publicado **mesmo se o serviço cair** após salvar no banco mas antes de publicar no RabbitMQ:

```java
// order-service: src/main/java/com/orderflow/order/service/OrderService.java

@Service
@RequiredArgsConstructor
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxEventRepository outboxEventRepository;

    public OrderResponse createOrder(CreateOrderRequest request) {
        // 1. Salva o pedido
        Order order = buildOrder(request);
        orderRepository.save(order);

        // 2. Salva o evento na tabela outbox (mesma transação!)
        OrderCreatedEvent event = buildOrderCreatedEvent(order);
        OutboxEvent outboxEvent = OutboxEvent.builder()
            .aggregateId(order.getId())
            .eventType("OrderCreated")
            .payload(toJson(event))
            .published(false)
            .build();
        outboxEventRepository.save(outboxEvent);

        // A transação do banco confirma AMBOS atomicamente.
        // O @Scheduled abaixo publica no RabbitMQ separadamente.
        return toResponse(order);
    }
}

// Publicador agendado
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxEventScheduler {

    private final OutboxEventRepository outboxEventRepository;
    private final OrderEventPublisher publisher;

    @Scheduled(fixedDelay = 1000) // a cada 1 segundo
    @Transactional
    public void processOutboxEvents() {
        List<OutboxEvent> unpublished = outboxEventRepository.findByPublishedFalse();
        for (OutboxEvent event : unpublished) {
            try {
                publisher.publishOrderCreated(fromJson(event.getPayload()));
                event.setPublished(true);
                outboxEventRepository.save(event);
            } catch (Exception e) {
                log.error("Falha ao publicar outbox event {}: {}", event.getId(), e.getMessage());
            }
        }
    }
}
```

---

### Etapa 7: application.yml de cada serviço

**order-service:**
```yaml
server:
  port: 8081

spring:
  application:
    name: order-service

  datasource:
    url: jdbc:postgresql://localhost:5433/db_orders
    username: orderflow
    password: orderflow123
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: validate   # flyway gerencia o schema
    show-sql: false
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true

  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        acknowledge-mode: manual     # ACK manual para controle de erros
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 2000
          multiplier: 2.0
          max-interval: 10000

  flyway:
    enabled: true
    locations: classpath:db/migration

management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
  endpoint:
    health:
      show-details: always

inventory:
  service:
    url: ${INVENTORY_SERVICE_URL:http://localhost:8082}

logging:
  level:
    com.orderflow: DEBUG
    org.springframework.amqp: INFO
```

> **Nota:** Os outros serviços seguem a mesma estrutura, ajustando `server.port`, `spring.datasource.url` e `spring.application.name`.

---

## 🧪 Testando o Sistema

### 1. Cadastrar um produto no Inventory Service

```bash
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Notebook Gamer X Pro",
    "description": "16GB RAM, RTX 4060, 512GB SSD",
    "price": 5999.99,
    "quantity": 50
  }'
```

**Resposta esperada:** `201 Created`
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "name": "Notebook Gamer X Pro",
  "price": 5999.99,
  "quantity": 50,
  "reserved": 0,
  "available": 50
}
```

---

### 2. Criar um pedido

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "11111111-2222-3333-4444-555555555555",
    "customerEmail": "joao@email.com",
    "paymentMethod": "CREDIT_CARD",
    "items": [
      {
        "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "quantity": 1
      }
    ]
  }'
```

**Resposta esperada:** `202 Accepted`
```json
{
  "id": "aabbccdd-1234-5678-abcd-ef0123456789",
  "status": "PENDING",
  "totalAmount": 5999.99,
  "createdAt": "2025-01-15T10:30:00Z"
}
```

---

### 3. Consultar o status do pedido (após alguns segundos)

```bash
curl http://localhost:8080/api/orders/aabbccdd-1234-5678-abcd-ef0123456789
```

**Resposta esperada (pedido confirmado):**
```json
{
  "id": "aabbccdd-1234-5678-abcd-ef0123456789",
  "status": "CONFIRMED",
  "totalAmount": 5999.99
}
```

---

### 4. Testar falha por falta de estoque

Crie um pedido com quantidade maior que o estoque disponível:

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "11111111-2222-3333-4444-555555555555",
    "customerEmail": "joao@email.com",
    "paymentMethod": "PIX",
    "items": [
      {
        "productId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "quantity": 9999
      }
    ]
  }'
```

Consulte o status após alguns segundos:
```json
{
  "status": "CANCELLED"
}
```

---

### 5. Verificar as filas no RabbitMQ Management UI

Acesse http://localhost:15672 (guest/guest) e verifique:

- **Exchanges:** Confirme que `order.exchange`, `inventory.exchange` e `payment.exchange` foram criados.
- **Queues:** Confirme todas as filas e suas respectivas DLQs.
- **Message Rates:** Observe o throughput de mensagens em tempo real durante os testes.

---

### 6. Testes de Unidade e Integração

```bash
# Executar todos os testes
mvn test

# Executar testes de apenas um módulo
mvn test -pl order-service

# Executar apenas testes de integração
mvn test -Dtest="*IntegrationTest"
```

---

## 📊 Monitoramento e Observabilidade

### Health Checks via Actuator

```bash
# Status geral de todos os serviços
curl http://localhost:8081/actuator/health
curl http://localhost:8082/actuator/health
curl http://localhost:8083/actuator/health
curl http://localhost:8084/actuator/health
```

O Spring Boot Actuator verifica automaticamente a conectividade com **PostgreSQL** e **RabbitMQ** e reporta `UP` ou `DOWN`.

### Métricas de Filas RabbitMQ

Acesse http://localhost:15672 para monitorar:

| Métrica                | O que indica                                              |
|------------------------|-----------------------------------------------------------|
| Messages Ready         | Mensagens aguardando consumo (idealmente próximo de 0)    |
| Messages Unacked       | Mensagens sendo processadas no momento                    |
| Messages in DLQ        | ⚠️ Mensagens com falha — requerem atenção imediata        |
| Publish Rate           | Throughput de publicação (msg/s)                          |
| Deliver Rate           | Throughput de consumo (msg/s)                             |

### Logs Estruturados

Todos os serviços utilizam logging estruturado. Principais eventos registrados:

```
[order-service]     INFO  Pedido criado: orderId=xxx, customerId=yyy, total=R$5999.99
[order-service]     INFO  Publicando OrderCreated para orderId: xxx
[inventory-service] INFO  Recebido OrderCreated para orderId: xxx
[inventory-service] INFO  Estoque reservado com sucesso para orderId: xxx
[payment-service]   INFO  Recebido InventoryReserved para orderId: xxx
[payment-service]   INFO  Pagamento APROVADO para orderId: xxx, txId: abc123
[notification-svc]  INFO  Notificação enviada: ORDER_CONFIRMED para joao@email.com
[order-service]     INFO  Pedido CONFIRMADO: orderId=xxx
```

---

## 📖 Glossário

| Termo                         | Definição                                                                                                   |
|-------------------------------|-------------------------------------------------------------------------------------------------------------|
| **Microsserviço**             | Aplicação independente com responsabilidade única, deployável e escalável individualmente                    |
| **Message Broker**            | Intermediário que recebe, armazena e entrega mensagens entre produtores e consumidores                       |
| **Exchange (RabbitMQ)**       | Componente que recebe mensagens dos produtores e as roteia para filas baseado em regras (bindings)           |
| **Queue (RabbitMQ)**          | Fila onde mensagens aguardam para ser consumidas pelos serviços                                              |
| **Routing Key**               | Identificador usado pelo exchange para decidir para qual fila enviar a mensagem                              |
| **Binding**                   | Regra que liga um exchange a uma fila com uma routing key específica                                         |
| **Topic Exchange**            | Exchange que roteia por padrões com wildcards `*` (uma palavra) e `#` (zero ou mais palavras)               |
| **Dead Letter Queue (DLQ)**   | Fila de destino para mensagens que falharam após todas as tentativas de retry                                |
| **Outbox Pattern**            | Padrão que garante atomicidade entre persistência de dados e publicação de eventos usando a mesma transação  |
| **Idempotência**              | Propriedade de uma operação que pode ser executada múltiplas vezes com o mesmo resultado                     |
| **Consistência Eventual**     | Modelo onde o sistema garante que todos os serviços convergirão para um estado consistente, mas não imediatamente |
| **Database per Service**      | Padrão onde cada microsserviço possui seu próprio banco de dados isolado                                     |
| **Event-Driven Architecture** | Estilo arquitetural onde os serviços comunicam-se através de eventos assíncronos                             |
| **ACK / NACK**                | Acknowledge (confirmação) / Negative Acknowledge (rejeição) de uma mensagem pelo consumidor                  |
| **Virtual Threads (Loom)**    | Threads leves do Java 25 que permitem alta concorrência sem overhead de threads do SO                        |
| **Flyway**                    | Ferramenta de versionamento e migração de schema de banco de dados                                          |

---

## 🧭 Próximos Passos (Evolução do Estudo de Caso)

Após implementar a versão base, considere evoluir o projeto com:

| Evolução                          | Ferramenta Sugerida            | O que aprenderá                           |
|-----------------------------------|--------------------------------|-------------------------------------------|
| Service Discovery                 | Spring Cloud Eureka            | Registro e descoberta dinâmica de serviços|
| Distributed Tracing               | OpenTelemetry + Jaeger         | Rastreamento de requisições entre serviços|
| Circuit Breaker                   | Resilience4j                   | Tolerância a falhas com fallback          |
| Centralized Config                | Spring Cloud Config Server     | Configuração externalizada                |
| API Documentation                 | OpenAPI 3 / Springdoc          | Documentação automática de APIs REST      |
| Containerized Deployment          | Kubernetes (Kind/Minikube)     | Orquestração de containers em produção    |
| Event Sourcing                    | —                              | Histórico imutável de eventos como estado |

---

*Feito com ❤️ como estudo de caso educacional de arquitetura distribuída.*
*Versão do documento: 1.0.0 | Stack: Java 25 · Spring Boot 4.0.6 · PostgreSQL 16 · RabbitMQ 3.13*
