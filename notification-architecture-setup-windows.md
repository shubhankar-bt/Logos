
# Event-Driven Notification System — Architecture & Docker Setup (Windows, Oracle DB)

**Target:** Enterprise-grade event-driven notification system with Outbox + Debezium + Kafka + Schema Registry + Notification Service + WebSocket + React.  
**Platform:** Docker on Windows (WSL2 recommended).  
**Database:** Oracle (production DB). This guide assumes Oracle DB is hosted separately (not containerized in this compose) because Debezium Oracle connector requires special setup and Oracle licensing.

---

## Table of Contents
1. Prerequisites
2. High-level architecture
3. Component responsibilities
4. Docker Compose (Kafka ecosystem + Redis + Zookeeper + Schema Registry + Kafka Connect)
5. Oracle DB configuration for Debezium (LogMiner / XStream notes)
6. Outbox pattern: table schema (Oracle)
7. Debezium connector config (Kafka Connect)
8. Spring Boot snippets
   - Outbox write example
   - Kafka consumer (NotificationService) using Spring Cloud Stream
   - WebSocket STOMP config for realtime pushes
9. React frontend WebSocket snippet
10. Running everything on Windows (WSL2 recommended)
11. Security, monitoring & production hardening notes
12. Rollout / testing checklist
13. Useful commands

---

## 1) Prerequisites (on your Windows dev/host)
- Windows 10/11 with **WSL2** + Ubuntu (strongly recommended) OR Docker Desktop with WSL2 backend enabled.
- Docker Desktop (with Compose V2).
- Java 17+ (for Spring Boot services).
- Maven/Gradle.
- Node.js + npm/yarn (for React).
- Oracle DB accessible (hostname:port/service). You must have a user with privileges for Debezium or you will use Outbox + custom poller.
- Git, IDE (IntelliJ / VS Code).

> **Note:** Running Oracle DB in a Docker container is possible but often not recommended for enterprise production. This guide assumes your Oracle DB instance is external or managed.

---

## 2) High-level architecture

```
[OrderService (Spring Boot: writes order & outbox row to Oracle DB)]
                 |
                 v
        [Oracle DB: outbox_event table]
                 |
        Debezium (Kafka Connect) reads DB changes
                 |
                 v
             Apache Kafka (topics + schema registry)
                 |
    +------------+----------------+-----------------+
    |            |                |                 |
Notification  Analytics       Fraud Detection   Other Consumers
Service (Spring Boot) (Stateless)  (Spring Boot)
    |
    v
Persistence (notifications table) + Redis (unread counts)
    |
    v
WebSocket Cluster (Spring WS) --> React clients (SockJS/STOMP or native WS)
```

---

## 3) Component responsibilities (recap)
- **Oracle DB**: Application data and outbox table (atomic writes).
- **OrderService / Other services**: write domain changes + outbox row in same transaction.
- **Debezium / Kafka Connect**: stream `outbox_event` inserts into Kafka.
- **Kafka**: central event bus.
- **Schema Registry**: manage Avro/Protobuf/JSON schemas.
- **NotificationService**: consume events, persist notifications, publish to realtime channel (Redis/Kafka).
- **WebSocket layer**: push notifications to connected clients.
- **React**: display notifications in UI.

---

## 4) Docker Compose for Kafka ecosystem

Create a `docker-compose.yml` in your project root (this compose file runs Kafka, Zookeeper, Schema Registry, Kafka Connect with Debezium, and Redis). Oracle DB is external.

```yaml
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.4.1
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    volumes:
      - kafka-data:/var/lib/kafka/data

  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.1
    depends_on:
      - zookeeper
      - kafka
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: PLAINTEXT://kafka:9092
    ports:
      - "8081:8081"

  connect:
    image: debezium/connect:2.3
    depends_on:
      - kafka
      - schema-registry
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: my_connect_configs
      OFFSET_STORAGE_TOPIC: my_connect_offsets
      STATUS_STORAGE_TOPIC: my_connect_statuses
      KEY_CONVERTER_SCHEMAS_ENABLE: "false"
      VALUE_CONVERTER_SCHEMAS_ENABLE: "false"
      # Use JSON converter for simple testing; for Avro use io.confluent.connect.avro.AvroConverter
      KEY_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      VALUE_CONVERTER: org.apache.kafka.connect.json.JsonConverter
      PLUGIN_PATH: /kafka/connect
    volumes:
      - ./connect-plugins:/kafka/connect

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  kafka-data:
```

> Save this file. It provides a dev environment. For production use secured/HA Kafka clusters.

---

## 5) Oracle DB configuration for Debezium

Debezium’s Oracle connector requires specific DB features:
- Oracle **LogMiner** or **XStream** based connector. XStream is more robust but requires additional setup and licensing considerations.
- **Supplemental logging** should be enabled.
- A connector user with appropriate privileges and supplemental logging setup.

Steps (high-level):
1. Connect to Oracle as SYSDBA and enable supplemental logging:
   ```sql
   ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
   ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (ALL) COLUMNS;
   ```
2. Create a connector user and grant privileges (example — run as SYSDBA):
   ```sql
   CREATE USER debezium IDENTIFIED BY "dbz";
   GRANT CONNECT, RESOURCE TO debezium;
   -- Additional grants for LogMiner/XStream usage:
   GRANT SELECT ANY TRANSACTION TO debezium;
   GRANT SELECT ANY DICTIONARY TO debezium;
   -- For XStream, additional steps are required; consult Debezium Oracle docs.
   ```
3. Ensure Oracle archive logs/redo logs are accessible and retention is configured.

> **Important:** Oracle CDC setup can be complex. If you cannot use Debezium with Oracle in your environment, use the **Transactional Outbox + Poller** (a small component that reads `outbox_event` and pushes to Kafka). This poller can be containerized and simpler.

---

## 6) Outbox table schema (Oracle)

Create a schema and table in Oracle (example):

```sql
CREATE TABLE outbox_event (
  id VARCHAR2(36) PRIMARY KEY,
  aggregate_id VARCHAR2(128),
  event_type VARCHAR2(100),
  payload CLOB,
  created_at TIMESTAMP DEFAULT SYSTIMESTAMP,
  processed   CHAR(1) DEFAULT 'N',
  processed_at TIMESTAMP,
  dedup_key VARCHAR2(200)
);

-- Index for query by processed flag
CREATE INDEX idx_outbox_processed ON outbox_event (processed, created_at);
```

Application transaction should insert domain row(s) and an `outbox_event` row in the same DB transaction.

---

## 7) Debezium connector config (Kafka Connect)

After Docker Compose is up, register a connector by POSTing to Kafka Connect REST API (`http://localhost:8083/connectors`).

Example JSON for Oracle LogMiner connector (simplified):

```json
{
  "name": "oracle-outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.oracle.OracleConnector",
    "tasks.max": "1",
    "database.server.name": "oracle-server",
    "database.hostname": "ORACLE_HOST",
    "database.port": "1521",
    "database.user": "debezium",
    "database.password": "dbz",
    "database.dbname": "ORCLPDB1", 
    "database.pdb.name": "ORCLPDB1",
    "database.retry.interval.ms": "10000",
    "database.connection.adapter": "logminer", 
    "table.include.list": "YOUR_SCHEMA.OUTBOX_EVENT",
    "schema.include.list": "YOUR_SCHEMA",
    "include.schema.changes": "false",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter"
  }
}
```

Replace `ORACLE_HOST`, `YOUR_SCHEMA`, and connection details appropriately.

After connector runs, you will get Kafka topics like `oracle-server.YOUR_SCHEMA.OUTBOX_EVENT` containing change events. You can then use SMTs or a sink to route/transform into a domain topic like `notifications.events`.

---

## 8) Spring Boot snippets

### a) Outbox write in your transactional service (OrderService)

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
    private final OutboxRepository outboxRepository;

    @Transactional
    public Order placeOrder(OrderDto dto) {
        Order order = new Order(...);
        orderRepository.save(order);

        OutboxEvent out = OutboxEvent.builder()
             .id(UUID.randomUUID().toString())
             .aggregateId(order.getId().toString())
             .eventType("OrderPlaced")
             .payload(toJson(order))
             .createdAt(OffsetDateTime.now())
             .build();

        outboxRepository.save(out);
        return order;
    }
}
```

### b) NotificationService — consume notifications topic (Spring Cloud Stream example)

**pom.xml**: Add `spring-cloud-starter-stream-kafka` (or use `spring-kafka` + `kafka-clients`)

`application.yml`:
```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: localhost:9092
      bindings:
        notifications-in:
          destination: notifications.events
          group: notification-service-group
```

`@StreamListener` / functional style:
```java
@Component
public class NotificationConsumer {

    @Bean
    public Consumer<Message<String>> notificationsIn() {
        return message -> {
            String payload = message.getPayload();
            // parse payload, enrich, persist to notifications table
            // push to Redis or publish to websocket channel
        };
    }
}
```

### c) WebSocket (Spring) config

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // We'll use simple broker for dev; in prod use external broker or Redis
        registry.enableSimpleBroker("/topic");
        registry.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-notifications")
                .setAllowedOriginPatterns("*")
                .withSockJS();
    }
}
```

`NotificationPushService` — uses `SimpMessagingTemplate` to push messages:

```java
@Service
@RequiredArgsConstructor
public class NotificationPushService {
    private final SimpMessagingTemplate template;

    public void pushToUser(String userId, NotificationDto dto) {
        template.convertAndSend("/topic/notifications."+userId, dto);
    }
}
```

> For a cluster, replace `enableSimpleBroker` with an external messaging broker or use Redis to route messages to specific WebSocket instances.

---

## 9) React frontend (simple STOMP + SockJS)

```javascript
import React, { useEffect, useState } from 'react';
import SockJS from 'sockjs-client';
import { Client } from '@stomp/stompjs';

const Notifications = () => {
  const [notifications, setNotifications] = useState([]);

  useEffect(() => {
    const sock = new SockJS('http://localhost:8081/ws-notifications');
    const client = new Client({
      webSocketFactory: () => sock,
      onConnect: () => {
        client.subscribe('/topic/notifications.user-123', (msg) => {
          const body = JSON.parse(msg.body);
          setNotifications(prev => [body, ...prev]);
        });
      }
    });
    client.activate();

    return () => client.deactivate();
  }, []);

  return (
    <div>
      <h3>Notifications</h3>
      <ul>
        {notifications.map((n, i) => <li key={i}>{n.message}</li>)}
      </ul>
    </div>
  );
};
export default Notifications;
```

Adjust subscription topic to match how your NotificationService pushes messages (per-user topics/channels).

---

## 10) Running on Windows (recommended flow)

1. Install Docker Desktop and enable WSL2 integration. Restart Docker.
2. Put `docker-compose.yml` in a folder.
3. Open WSL2 terminal (Ubuntu) or PowerShell (with Docker) and run:
   ```bash
   docker compose up -d
   ```
4. Wait for services to become healthy:
   - Kafka broker at `localhost:9092`
   - Schema Registry at `http://localhost:8081`
   - Kafka Connect at `http://localhost:8083`
   - Redis at `localhost:6379`
5. Configure Oracle DB (external) per section 5.
6. Register Debezium connector (POST JSON to `http://localhost:8083/connectors`).
7. Start Spring Boot services locally or containerize them and add to docker-compose.
   - For development, run Spring Boot apps via IDE: `mvn spring-boot:run`.
8. Start React app: `npm start` (default port 3000).
9. Test full flow: create an Order via API → check outbox row in Oracle → Debezium publishes change → Kafka topic receives event → NotificationService consumes → pushes via WebSocket → React UI receives.

---

## 11) Security & production hardening (high-level)
- Use **TLS** for Kafka listeners and Kafka Connect.
- Use **SASL** (Kerberos/PLAIN) and RBAC for Kafka.
- Protect Schema Registry with auth.
- Use network segmentation (private subnets) for Kafka and DB.
- Use encryption at rest for Kafka and DB storage.
- Use Secrets Manager or Vault for credentials.
- Harden Docker/Kubernetes nodes; use PodSecurityPolicies or K8s RBAC.

---

## 12) Rollout & testing checklist
- [ ] Implement outbox pattern in a single service (OrderService).
- [ ] Deploy Kafka ecosystem in dev via Docker Compose.
- [ ] Configure Debezium connector for Oracle and validate topic creation.
- [ ] Implement NotificationService consumer & WebSocket push.
- [ ] End-to-end functional tests (create order → notification).
- [ ] Load testing (simulate high volume).
- [ ] Implement DLQ handling and monitoring alerts.
- [ ] Containerize services and prepare Helm charts for K8s.
- [ ] Prepare backups, DR plan, and auditing.

---

## 13) Useful commands

Start compose:
```bash
docker compose up -d
docker compose logs -f kafka
```

List Kafka topics:
```bash
docker exec -it <kafka-container> bash
kafka-topics --bootstrap-server kafka:9092 --list
```

Register connector (curl from host):
```bash
curl -X POST -H "Content-Type: application/json" --data @oracle-outbox-connector.json http://localhost:8083/connectors
```

Check consumer group lags:
```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group notification-service-group
```

---

## Final notes & alternatives
- If Debezium + Oracle is too complex in your environment, start with the **application-written outbox + poller** (a small Spring Boot app that polls `outbox_event` and publishes to Kafka). This gives reliability and is easier to setup.
- For production, use managed Kafka (Confluent Cloud / MSK) if you want to reduce operational overhead.
- I can also generate:
  - Dockerized Spring Boot service templates (`Dockerfile`) for OrderService & NotificationService,
  - Full `docker-compose` with services containerized,
  - Helm charts for Kubernetes,
  - or a ready-to-run Git repo scaffold.

---

**Would you like me to:**
1. Generate Dockerfiles and Docker Compose with the Spring Boot services included?  
2. Create the small Outbox poller app (if you prefer not to use Debezium)?  
3. Generate Helm charts for Kubernetes deployment?

Pick one and I’ll generate the code & files next.
