
# 🏦 Banking System Design with SQL and Apache Kafka

## **1. System Architecture Overview**

### **Components**

1. **API Gateway** – Handles client requests (REST/GraphQL)  
2. **Authentication Service** – User authentication and authorization  
3. **Core Banking Service** – Account management and transaction handling  
4. **Kafka Event Bus** – Asynchronous communication between services  
5. **Transaction Service** – Manages fund transfers (ACID compliant)  
6. **SQL Database** – PostgreSQL/MySQL for high consistency  
7. **Audit & Notification Service** – Logs and user notifications  

### **Architecture Flow**

```
Client → API Gateway → Core Banking Service → SQL DB
                                   ↓
                             Kafka Broker
                                   ↓
      Transaction Service → SQL DB → Audit/Notification Service
```

---

## **2. Technology Stack**

- **Backend:** Node.js / .NET Core (C#)  
- **Database:** PostgreSQL (ACID-compliant, high consistency)  
- **Message Broker:** Apache Kafka  
- **Containerization:** Docker & Kubernetes  
- **Authentication:** OAuth 2.0 / JWT  
- **Monitoring:** Prometheus & Grafana  

---

## **3. Database Schema (SQL)**

```sql
-- Users Table
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Accounts Table
CREATE TABLE accounts (
    account_id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id),
    balance DECIMAL(15, 2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Transactions Table
CREATE TABLE transactions (
    transaction_id SERIAL PRIMARY KEY,
    from_account INT REFERENCES accounts(account_id),
    to_account INT REFERENCES accounts(account_id),
    amount DECIMAL(15, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Audit Logs Table
CREATE TABLE audit_logs (
    log_id SERIAL PRIMARY KEY,
    event_type VARCHAR(50),
    details JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);
```

---

## **4. Kafka Topics Design**

1. **`transaction_initiated`** – When a transaction starts  
2. **`transaction_completed`** – When a transaction succeeds  
3. **`transaction_failed`** – When a transaction fails  
4. **`audit_log`** – Logs important system events  

---

## **5. Core Logic Implementation**

### **Producer (Node.js) - Initiate Transaction**

```javascript
const { Kafka } = require('kafkajs');
const kafka = new Kafka({ clientId: 'banking-app', brokers: ['localhost:9092'] });
const producer = kafka.producer();

async function initiateTransaction(fromAccount, toAccount, amount) {
    await producer.connect();
    await producer.send({
        topic: 'transaction_initiated',
        messages: [{ value: JSON.stringify({ fromAccount, toAccount, amount }) }],
    });
    await producer.disconnect();
}
```

---

### **Consumer (Transaction Service)**

```javascript
const consumer = kafka.consumer({ groupId: 'transaction-service' });

async function processTransactions() {
    await consumer.connect();
    await consumer.subscribe({ topic: 'transaction_initiated' });

    await consumer.run({
        eachMessage: async ({ message }) => {
            const { fromAccount, toAccount, amount } = JSON.parse(message.value);

            try {
                await db.transaction(async (trx) => {
                    const sender = await trx('accounts').where('account_id', fromAccount).first();
                    if (sender.balance < amount) throw new Error('Insufficient funds');

                    await trx('accounts')
                        .where('account_id', fromAccount)
                        .decrement('balance', amount);

                    await trx('accounts')
                        .where('account_id', toAccount)
                        .increment('balance', amount);

                    await producer.send({
                        topic: 'transaction_completed',
                        messages: [{ value: JSON.stringify({ fromAccount, toAccount, amount }) }],
                    });
                });
            } catch (err) {
                await producer.send({
                    topic: 'transaction_failed',
                    messages: [{ value: JSON.stringify({ fromAccount, toAccount, amount, reason: err.message }) }],
                });
            }
        },
    });
}
```

---

## **6. Consistency & ACID Guarantee**

- **ACID Transactions:** PostgreSQL ensures atomicity, consistency, isolation, and durability.  
- **Exactly-Once Processing:** Kafka ensures no duplication using **idempotent producers**.  
- **Error Handling:** Failed transactions are logged and retried.

---

## **7. High Availability & Fault Tolerance**

- **Kafka Cluster:** Multi-node Kafka cluster for redundancy  
- **Database Replication:** PostgreSQL replication for disaster recovery  
- **Circuit Breaker:** Graceful failure handling between services  
- **Retry Mechanism:** Kafka handles message retries

---

## **8. Deployment with Docker**

### **Docker Compose**

```yaml
version: '3'
services:
  postgres:
    image: postgres
    environment:
      POSTGRES_USER: bankuser
      POSTGRES_PASSWORD: bankpass
      POSTGRES_DB: banking
    ports:
      - "5432:5432"

  kafka:
    image: wurstmeister/kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
```

---

## **9. Monitoring & Logging**

- **Kafka Monitoring:** Kafka Manager, Burrow  
- **Database Monitoring:** PgAdmin, Prometheus  
- **Logging:** Kafka `audit_log` topic, ELK Stack (Elasticsearch, Logstash, Kibana)  

---

## **10. Security Considerations**

- **TLS Encryption** for Kafka communication  
- **Role-Based Access Control (RBAC)** for API and DB  
- **Input Validation** to prevent SQL injection  
- **Audit Logs** for tracking sensitive operations  

---

## **Summary**

- **Strong Consistency** through SQL ACID transactions  
- **Real-Time Processing** with Kafka event-driven architecture  
- **Scalable & Fault-Tolerant** deployment with Docker/Kubernetes  

---
