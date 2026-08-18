# Solace PubSub + with Spring

---

## PART 1: FOUNDATIONS

### 1.1 What is Solace PubSub+?

Solace PubSub+ is an **event broker** (messaging middleware) that lets applications, IoT devices, and microservices publish and consume messages/events. Unlike simple message brokers, Solace supports **multiple messaging patterns in one broker**:

- **Publish/Subscribe (pub-sub)** — topic-based, one-to-many
- **Point-to-Point (P2P)** — queue-based, one-to-one (competing consumers)
- **Request/Reply** — synchronous-style messaging over async transport
- **Streaming** — continuous event streams

It supports multiple protocols on the same broker: **SMF (Solace Message Format - native)**, **JMS (1.1/2.0)**, **AMQP 1.0**, **MQTT**, **REST**, **WebSocket**. This is important to say in interviews — Solace is protocol-agnostic; a JMS publisher can talk to an MQTT subscriber through the same broker.

**Why companies use it (be ready to explain):**
- Single broker for pub-sub AND queues (no need for separate Kafka + RabbitMQ)
- Dynamic topic routing without pre-creating topics (unlike Kafka topics which must exist)
- Hardware appliance, software broker, or cloud (PubSub+ Cloud) — deployment flexibility
- Strong enterprise features: HA, DR, WAN replication, fine-grained ACLs

---

### 1.2 Core Building Blocks — Big Picture

```
Client App ---(Session)---> Message VPN ---> Topics / Queues ---> Other Client Apps
```

| Concept | Analogy | Purpose |
|---|---|---|
| **Event Broker** | The whole post office | The Solace server/cluster itself |
| **Message VPN** | A tenant/department in that post office | Logical isolation (multi-tenancy) |
| **Client / Session** | A person connecting to the post office | Application's connection to broker |
| **Topic** | A radio frequency broadcast | Address for pub-sub messaging, no storage by default |
| **Queue** | A mailbox | Stores messages until consumed (P2P) |
| **Topic Subscription** | Tuning your radio to a frequency (with wildcards) | How consumers receive topic messages |
| **Client Username / ACL Profile / Client Profile** | ID card + permissions + resource limits | Security & governance |

---

## PART 2: MESSAGE VPN (Virtual Private Network)

### 2.1 What is it?

A **Message VPN** is a logical partition within a Solace broker (or cluster) that provides **multi-tenancy**. Each Message VPN has its own:
- Topics and queues (completely isolated namespace)
- Client usernames, ACL profiles, client profiles
- Authentication/authorization settings
- Resource limits (max connections, max queues, max spool size)

**Interview line:** *"Think of a single Solace broker like a bare-metal server, and Message VPNs like separate virtual machines on it — each is isolated, but they share the underlying hardware/broker resources."*

### 2.2 Why it matters
- Multiple environments (dev/test/prod) OR multiple business units can share one physical broker safely.
- A client connecting to VPN "TeamA_VPN" **cannot** see topics/queues in "TeamB_VPN" even if names collide.
- Every client connection specifies exactly **one** Message VPN at connect time.

### 2.3 Key configuration items inside a VPN
- **Max spool usage** — disk space allotted for guaranteed messages (queues)
- **Max connections** — client connection limit
- **Authentication scheme** — internal DB, LDAP, client-certificate, OAuth
- **Default/dynamic queue creation flags**

---

## PART 3: TOPICS (Publish/Subscribe Model)

### 3.1 What is a Topic?

A **Topic** in Solace is **not a physical object you create** (unlike Kafka). It's just a **string label** attached to a message at publish time. Example:

```
orders/us-east/electronics/created
```

Publishers send to a topic string; they don't need to "create" it first. Subscribers subscribe to topic **patterns** using wildcards, and the broker matches published messages to subscriptions in real time.

### 3.2 Topic Hierarchy — Best Practice

Solace recommends structuring topics hierarchically, using `/` as level separator:

```
<region>/<domain>/<entity>/<action>
orders/us/retail/created
orders/us/retail/cancelled
orders/eu/retail/created
```

This hierarchy is what makes wildcard subscriptions powerful.

### 3.3 Wildcards — MUST KNOW for interviews

| Wildcard | Meaning | Example |
|---|---|---|
| `*` | Matches exactly ONE topic level (single-level wildcard) | `orders/*/retail/created` matches `orders/us/retail/created` and `orders/eu/retail/created` but NOT `orders/us/east/retail/created` |
| `>` | Matches ONE OR MORE remaining levels (multi-level, must be last) | `orders/us/>` matches `orders/us/retail/created` and `orders/us/retail/created/urgent` |

**Common interview trap question:** *"Can `>` appear in the middle of a topic subscription?"*
**Answer: No.** `>` must always be the **last** element of a subscription string. `*` can appear anywhere, including multiple times.

Example subscription combos:
```
orders/*/retail/>        ✅ valid
orders/>/retail          ❌ invalid (> not at end)
*/us/retail/created      ✅ valid
```

### 3.4 Direct vs Guaranteed messaging over Topics

This is a **critical interview distinction**:

| | Direct Messaging | Guaranteed (Persistent) Messaging |
|---|---|---|
| Delivery guarantee | Best-effort, no ack, no storage | At-least-once, acknowledged, stored to disk |
| Speed | Very fast (µs latency), no broker-side spooling | Slower (writes to disk, sync/async ack) |
| What happens if no subscriber? | Message is **dropped** | Message is only stored if a **queue is bound to that topic** via a subscription |
| Use case | Market data, telemetry, high-frequency updates where occasional loss is OK | Orders, payments, anything that MUST NOT be lost |

**Key insight interviewers look for:** *A topic itself never stores messages. If you want guaranteed delivery for topic-published messages, you must create a QUEUE and add a "Topic Subscription" on that queue, so the queue captures a copy of every matching message.* This bridges pub-sub and point-to-point.

---

## PART 4: QUEUES (Point-to-Point / Guaranteed Messaging)

### 4.1 What is a Queue?

A Queue is a **named, persistent (or non-persistent) storage object** on the broker that holds messages until a consumer acknowledges them. Unlike topics, queues **must be explicitly created/provisioned** (via CLI, SEMP API, PubSub+ Manager GUI, or dynamically by client with permission).

### 4.2 Queue Types

**a) Durability**
- **Durable Queue** — survives broker restart; messages persisted to disk (spool). This is what people mean by "**persistent queue**."
- **Non-Durable / Temporary Queue** — exists only for the life of the client session; auto-deleted on disconnect. Often used for **request-reply** patterns (temporary reply queue).

**b) Access Type**
- **Exclusive Queue** — Only ONE consumer actively receives messages at a time (others are on standby as failover). Preserves strict message order.
- **Non-Exclusive Queue** — Multiple consumers can bind simultaneously; messages are **round-robin distributed** (competing consumers pattern) — like Kafka consumer groups but simpler. Order is NOT guaranteed across consumers.

**Interview tip:** *"If message order matters (e.g., processing bank transactions sequentially per account), use an Exclusive queue. If you want horizontal scaling/throughput, use Non-Exclusive."*

### 4.3 Message Delivery Modes (set by publisher)

| Mode | Description |
|---|---|
| **Direct** | Fire-and-forget, no persistence, no ack |
| **Persistent** | Guaranteed, written to disk on the broker (spooled), acknowledged back to publisher only after commit to disk (or configurable ack mode) |
| **Non-Persistent** | Guaranteed-ish but broker may keep in memory rather than disk depending on config — rarely used; conceptually a middle ground |

In practice, **most production systems use Persistent** delivery mode when interacting with queues.

### 4.4 Queue-to-Topic binding ("Topic Subscription on a Queue")

A queue can subscribe to one or more topics. Any message published to a matching topic gets a **copy delivered into that queue** (if using persistent delivery). This is how Solace unifies pub-sub and P2P:

```
Publisher --topic: orders/us/created--> [Broker]
                                            |
                                 (Queue "OrderProcessingQ" 
                                  has topic-subscription "orders/us/*")
                                            |
                                            v
                                     Consumer reads from Queue
```

### 4.5 Dead Message Queue (DMQ)

When a message on a queue **exceeds max redelivery attempts** or **TTL (time-to-live) expires**, instead of being silently dropped, it can be routed to a **Dead Message Queue** for inspection/reprocessing. Configured per-queue with a "Max Redelivery Count" and "Reject Msg to Sender on Discard" settings.

**Interview answer pattern:** *"We configure a max redelivery count on the queue. After exceeding it, the message is moved to the DMQ instead of being lost, so we can triage poison messages without blocking the main queue."*

### 4.6 Queue vs Topic — the classic interview question

| | Topic | Queue |
|---|---|---|
| Creation | Implicit (just publish/subscribe to a string) | Explicit provisioning required |
| Message storage | None (unless a queue subscribes to it) | Stores messages (spooled to disk if durable) |
| Consumer model | Fan-out — every matching subscriber gets a copy | Competing consumers — one consumer gets each message (non-exclusive) or single active consumer (exclusive) |
| Guarantee | Best-effort by default | Guaranteed (at-least-once) |
| Typical use case | Event broadcast, telemetry | Work queues, order processing, task distribution |

---

## PART 5: CLIENTS, SESSIONS, SECURITY

### 5.1 Client Username
The identity used to authenticate a connection into a specific Message VPN. Analogous to a DB username.

### 5.2 Client Profile
Defines **resource limits and capabilities** for a connecting client: max sessions, max endpoints (queues) it can create, allowed message spool usage, whether it can create durable endpoints dynamically, TCP settings, etc.

### 5.3 ACL Profile (Access Control List)
Defines **what topics/queues** a client is allowed to **publish to** and **subscribe to** — supports allow/deny rules with wildcard exceptions. E.g., allow subscribe to `orders/>` but deny `orders/internal/>`.

### 5.4 Client (Session) connection flow
1. App creates a `JCSMPSession` (native) or `ConnectionFactory` (JMS) pointing at broker host + Message VPN + client-username/password.
2. Session connects — broker authenticates using client-username against the VPN's auth scheme (Basic, client-cert, LDAP, OAuth).
3. Once connected, ACL profile controls what topics/queues that session can touch.

### 5.5 Transport Security
- **TLS/SSL** for encryption in transit.
- **Client Certificate Authentication** — mutual TLS, common in production financial systems.
- Ports: default SMF is 55555 (plain), 55443 (TLS); JMS typically over 55556 or via JNDI-configured connection factory.

---

## PART 6: SOLACE + SPRING INTEGRATION (The part you'll actually code)

There are **three common ways** to integrate Spring apps with Solace. Know all three — interviewers often ask "which one do you use and why."

### 6.1 Option A — Solace JMS (javax.jms / jakarta.jms) with Spring JMS

Solace ships a JMS 1.1/2.0-compliant client. You use it exactly like you'd use ActiveMQ or IBM MQ with `spring-jms`.

**Maven dependency:**
```xml
<dependency>
    <groupId>com.solacesystems</groupId>
    <artifactId>sol-jms</artifactId>
    <version>10.x.x</version>
</dependency>
```

**Connection Factory bean:**
```java
@Configuration
public class SolaceJmsConfig {

    @Bean
    public ConnectionFactory connectionFactory() {
        SolConnectionFactory cf = SolJmsUtility.createConnectionFactory();
        cf.setHost("tcp://localhost:55555");
        cf.setVPN("default");
        cf.setUsername("myClientUser");
        cf.setPassword("myPassword");
        return cf;
    }

    @Bean
    public JmsTemplate jmsTemplate(ConnectionFactory cf) {
        return new JmsTemplate(cf);
    }
}
```

**Publishing to a queue:**
```java
@Service
public class OrderProducer {

    @Autowired
    private JmsTemplate jmsTemplate;

    public void sendOrder(String orderJson) {
        jmsTemplate.convertAndSend("OrderProcessingQ", orderJson);
    }
}
```

**Consuming with `@JmsListener`:**
```java
@Component
public class OrderConsumer {

    @JmsListener(destination = "OrderProcessingQ")
    public void receiveOrder(String orderJson) {
        System.out.println("Received order: " + orderJson);
        // process...
    }
}
```

**Publishing/Subscribing to a Topic (JMS style):**
```java
Topic topic = session.createTopic("orders/us/retail/created");
MessageProducer producer = session.createProducer(topic);
```

To make topic-published messages durable, you'd create a **Durable Topic Subscriber** (JMS concept) which under the hood maps to a Solace queue with a topic subscription.

---

### 6.2 Option B — Native Solace API (JCSMP) — most feature-rich, most common in real Solace shops

JCSMP (Java Client Solace Message Passing) is Solace's **native** API — gives you full control (topics, queues, guaranteed messaging, flow control, transactions, request-reply).

**Maven dependency:**
```xml
<dependency>
    <groupId>com.solacesystems</groupId>
    <artifactId>sol-jcsmp</artifactId>
    <version>10.x.x</version>
</dependency>
```

**Creating a session:**
```java
@Configuration
public class JcsmpConfig {

    @Bean
    public JCSMPSession jcsmpSession() throws Exception {
        JCSMPProperties properties = new JCSMPProperties();
        properties.setProperty(JCSMPProperties.HOST, "tcp://localhost:55555");
        properties.setProperty(JCSMPProperties.VPN_NAME, "default");
        properties.setProperty(JCSMPProperties.USERNAME, "myClientUser");
        properties.setProperty(JCSMPProperties.PASSWORD, "myPassword");

        JCSMPSession session = JCSMPFactory.onlyInstance().createSession(properties);
        session.connect();
        return session;
    }
}
```

**Publishing a Direct message to a Topic:**
```java
Topic topic = JCSMPFactory.onlyInstance().createTopic("orders/us/retail/created");
XMLMessageProducer producer = session.getMessageProducer(new JCSMPStreamingPublishCorrelatingEventHandler() {
    @Override
    public void responseReceivedEx(Object key) { /* ack callback */ }

    @Override
    public void handleErrorEx(Object key, JCSMPException cause, long timestamp) { /* nack callback */ }
});

TextMessage msg = JCSMPFactory.onlyInstance().createMessage(TextMessage.class);
msg.setText("{\"orderId\":123}");
producer.send(msg, topic);
```

**Publishing a Guaranteed (Persistent) message:**
```java
TextMessage msg = JCSMPFactory.onlyInstance().createMessage(TextMessage.class);
msg.setText(orderJson);
msg.setDeliveryMode(DeliveryMode.PERSISTENT);   // <-- key line
producer.send(msg, topic);
```

**Binding a consumer to a Queue (guaranteed messaging):**
```java
Queue queue = JCSMPFactory.onlyInstance().createQueue("OrderProcessingQ");

ConsumerFlowProperties flowProps = new ConsumerFlowProperties();
flowProps.setEndpoint(queue);
flowProps.setAckMode(JCSMPProperties.SUPPORTED_MESSAGE_ACK_CLIENT); // client acks manually

FlowReceiver consumer = session.createFlow(new XMLMessageListener() {
    @Override
    public void onReceive(BytesXMLMessage msg) {
        System.out.println("Received: " + msg.dump());
        msg.ackMessage();   // manual acknowledgment
    }
    @Override
    public void onException(JCSMPException e) { e.printStackTrace(); }
}, flowProps);

consumer.start();
```

**Interview talking point on Ack Modes:**
- `AUTO` — broker/client library acks automatically on receipt (risk: message lost if app crashes mid-processing)
- `CLIENT` — application explicitly calls `msg.ackMessage()` after successful processing (safer, "at-least-once" processing)

---

### 6.3 Option C — Spring Cloud Stream with Solace PubSub+ Binder (modern, most "Spring-native" approach)

This is the **cleanest integration** if the shop is fully invested in Spring Cloud Stream — you write plain functional interfaces (`Supplier`, `Function`, `Consumer`) and the binder handles Solace wiring.

**Maven dependency:**
```xml
<dependency>
    <groupId>com.solace.spring.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-solace</artifactId>
    <version>4.x.x</version>
</dependency>
```

**application.yml:**
```yaml
spring:
  cloud:
    stream:
      bindings:
        orderProcessor-in-0:
          destination: orders/us/retail/created
          group: order-processing-group
        orderProcessor-out-0:
          destination: orders/us/retail/validated
      solace:
        bindings:
          orderProcessor-in-0:
            consumer:
              queue-additional-subscriptions: orders/us/retail/*
  solace:
    java:
      host: tcp://localhost:55555
      msg-vpn: default
      client-username: myClientUser
      client-password: myPassword
```

**Functional processor:**
```java
@Component
public class OrderProcessor {

    @Bean
    public Function<Order, ValidatedOrder> orderProcessor() {
        return order -> {
            // business logic
            return new ValidatedOrder(order.getId(), true);
        };
    }
}
```

The binder automatically:
- Provisions a durable queue for the `group` (`order-processing-group`) if it doesn't exist
- Adds the topic subscription to that queue
- Handles ack/nack based on function success/failure (auto DMQ routing on repeated failure)

**When to mention this in interview:** *"Spring Cloud Stream abstracts the messaging middleware behind Supplier/Function/Consumer beans, so we could theoretically swap Solace for Kafka by changing the binder dependency, without touching business logic."*

---

### 6.4 Solace Spring Boot Auto-Configuration (starter)

Solace also provides `solace-spring-boot-starter`, which auto-configures a `SpringJCSMPFactory` bean from `application.yml`, removing manual `JCSMPSession` boilerplate:

```xml
<dependency>
    <groupId>com.solace.spring.boot</groupId>
    <artifactId>solace-spring-boot-starter</artifactId>
</dependency>
```

```yaml
solace:
  java:
    host: tcp://localhost:55555
    msg-vpn: default
    client-username: myClientUser
    client-password: myPassword
```

```java
@Autowired
private SpringJCSMPFactory springJCSMPFactory;

public void connect() throws JCSMPException {
    JCSMPSession session = springJCSMPFactory.createSession();
    session.connect();
}
```

---

## PART 7: ADVANCED CONCEPTS

### 7.1 Transactions
Solace supports **local transactions** on a session grouping multiple sends/acks into one atomic commit — either all messages are sent/acked, or none are, on rollback. Useful for "receive from Queue A, process, publish to Queue B" atomically (similar to Kafka's exactly-once transactions, though Solace's model is simpler, XA-transaction support also exists via JMS).

```java
session.commit();   // or session.rollback();
```

### 7.2 Request-Reply Pattern
Solace has built-in support for sync-style request/reply over async messaging:
- Publisher creates a **temporary queue/topic** as the reply-to destination.
- Sends request message with `ReplyTo` header set.
- Consumer processes and publishes reply to that `ReplyTo` destination.
- Solace JCSMP even has a helper: `session.createRequestor()` for blocking-style request-reply calls.

### 7.3 High Availability & Disaster Recovery
- **HA Pair** — Two broker nodes (active/standby) within a datacenter sharing a virtual IP; automatic failover on node failure, no message loss for guaranteed messages (synchronously mirrored).
- **DR (Disaster Recovery) / Replication** — Asynchronous replication of Message VPNs to a broker in a different datacenter/region.
- **Bridges (VPN Bridges)** — Connect two Message VPNs (same or different brokers) so messages published in one are relayed to the other — used for hybrid cloud or datacenter interconnect.
- **DMR (Dynamic Message Routing)** — Mesh routing between brokers in different "cluster" nodes; broker-to-broker routing of topics/messages without explicit static bridges — used in large multi-broker "event mesh" topologies.

### 7.4 Event Mesh
A **marketing/architecture term** Solace popularized: multiple interconnected brokers (via DMR) act as a single distributed nervous system, so an event published in one datacenter can flow automatically to a subscriber in another datacenter/cloud, without app-level routing logic. Good to mention if asked "have you heard of event mesh?"

### 7.5 Flow Control / Guaranteed Messaging Window
Consumer flows have a **configurable window size** (like TCP flow control) — number of unacknowledged messages the broker will send before pausing, preventing consumer overload. Tunable via `ConsumerFlowProperties`.

### 7.6 Message Eliding / Content Filtering (advanced, mention only if asked)
- **Message Eliding** — broker can coalesce/drop intermediate updates for a topic if consumer is slow and only cares about the latest value (useful for market-data style feeds).
- Solace also supports **Selectors** (JMS message selectors) — SQL-92-like filtering on message headers/properties at the queue level.

### 7.7 SEMP API
**SEMP (Solace Element Management Protocol)** — REST API to configure and monitor the broker programmatically (create queues, VPNs, check queue depth, etc.) — used for infra-as-code / automation (Terraform Solace provider uses SEMP under the hood).

### 7.8 PubSub+ Manager / CLI
GUI and CLI tools for administering the broker manually — creating queues, VPNs, client-usernames, ACL profiles, monitoring queue depth and consumer flow, viewing DMQ contents.

---

## PART 8: SOLACE VS KAFKA VS RABBITMQ — Very common interview question

| Feature | Solace | Kafka | RabbitMQ |
|---|---|---|---|
| Messaging model | Topics (dynamic) + Queues, both native | Topic-partition log (pub-sub only, consumer groups simulate queues) | Queues (exchanges route to queues) |
| Topic creation | Implicit/dynamic, hierarchical with wildcards | Must be explicitly created, flat namespace (no wildcard hierarchy) | N/A (uses exchanges + routing keys) |
| Message retention | Queue-based (deleted after ack); no long-term log by default | Log-based retention (replay by offset, configurable retention) | Queue-based (deleted after ack) |
| Protocol support | SMF, JMS, AMQP, MQTT, REST, WebSocket — all on one broker | Kafka protocol (some proxies for others) | AMQP 0-9-1 natively, plugins for MQTT/STOMP |
| Ordering | Exclusive queue = strict order; topic = per-publisher order | Strict order within a partition | Per-queue order (mostly) |
| Best for | Hybrid pub-sub + P2P, event mesh across hybrid cloud, low-latency direct messaging | High-throughput event streaming, replay, log-based processing | Traditional task/work queues, RPC-style messaging |
| Replay from history | Not a core feature (some via Solace's "Replay" add-on) | Core feature — read from any offset | Not typical |

**Good soundbite:** *"Kafka is optimized as a distributed log for high-throughput stream processing and replay. RabbitMQ is optimized as a flexible task/work queue broker. Solace tries to give you both pub-sub AND queueing semantics from a single broker, with very rich topic hierarchies and multi-protocol support, which made it attractive for our hybrid on-prem + cloud integration use case."*

---

## PART 9: COMMON INTERVIEW Q&A CHEAT SHEET

**Q: What's the difference between Direct and Persistent messaging in Solace?**
A: Direct is fire-and-forget with no broker-side storage or acknowledgment — fastest, but a message is lost if there's no active subscriber. Persistent (guaranteed) messaging is written to the broker's message spool (disk) and only removed after the consumer acknowledges it — survives broker restarts and consumer downtime.

**Q: How does a topic differ from a queue?**
A: A topic is just a label used for pub-sub with no built-in storage; any number of subscribers can receive a copy. A queue is a stored, named endpoint that must be provisioned, and messages in it are consumed by ONE consumer per message (or one active consumer if exclusive) — point-to-point.

**Q: How do you get guaranteed delivery for messages published to a topic?**
A: Create a durable queue and add a topic subscription on that queue matching the topic string(s) you care about. The broker will then persist a copy of every matching topic message into that queue.

**Q: What is a Message VPN?**
A: A logical, isolated namespace/tenant within a Solace broker — has its own topics, queues, clients, ACLs, resource limits. Enables multi-tenancy on shared broker infrastructure.

**Q: Exclusive vs Non-exclusive queue?**
A: Exclusive allows only one active consumer at a time (others act as hot standby) — preserves order. Non-exclusive allows multiple simultaneous consumers with round-robin distribution — higher throughput but no strict ordering guarantee.

**Q: What happens to a message that fails processing repeatedly?**
A: Once it exceeds the queue's configured max redelivery count, it's moved to the Dead Message Queue (DMQ) instead of being retried forever or silently dropped.

**Q: What are the wildcard characters in Solace topic subscriptions?**
A: `*` matches exactly one topic level; `>` matches one or more remaining levels and must be the last character in the subscription.

**Q: How did you integrate Spring Boot with Solace?**
A: (Tailor to your project — mention whichever of: Spring JMS + `sol-jms`, native JCSMP with `sol-jcsmp`, or Spring Cloud Stream Solace binder, plus `solace-spring-boot-starter` for auto-configuration.)

**Q: How do you ensure no message loss end-to-end?**
A: Publish with `DeliveryMode.PERSISTENT`, use publisher ack callbacks (`responseReceivedEx`/`handleErrorEx`) to confirm broker receipt, consume with `CLIENT` ack mode and only `ackMessage()` after successful business processing, and configure a DMQ for poison messages.

**Q: What is an Event Mesh?**
A: A network of interconnected Solace brokers (via Dynamic Message Routing) that transparently routes events between distributed applications across data centers/clouds/regions, so publishers and subscribers don't need to know each other's physical location.

**Q: Client Profile vs ACL Profile?**
A: Client Profile = resource limits and capabilities (max connections, can it create endpoints, message spool quota). ACL Profile = permission rules (what topics/queues it may publish/subscribe to).

---

## PART 10: QUICK-REFERENCE SUMMARY (read this the morning of the interview)

1. **Broker** hosts one or more **Message VPNs** (multi-tenant isolation).
2. **Topics** = dynamic, hierarchical pub-sub addresses; no storage by default; wildcards `*` (one level) and `>` (rest of levels, must be last).
3. **Queues** = explicitly provisioned, persistent storage endpoints for point-to-point; Exclusive (ordered, 1 active consumer) vs Non-Exclusive (competing consumers, round robin).
4. Bind a **queue's topic subscription** to capture guaranteed copies of pub-sub traffic.
5. **Delivery modes**: Direct (fast, no guarantee) vs Persistent (guaranteed, disk-spooled, acked).
6. **Ack modes**: AUTO vs CLIENT (manual ack — safer for at-least-once processing).
7. Failed messages beyond max redelivery → **Dead Message Queue (DMQ)**.
8. Spring integration options: **Spring JMS (`sol-jms`)**, **native JCSMP (`sol-jcsmp`)**, **Spring Cloud Stream Solace binder**, or **`solace-spring-boot-starter`** for auto-config.
9. **Security**: Client Username (identity) + Client Profile (limits) + ACL Profile (permissions) + TLS/mTLS.
10. **HA** = active/standby pair, sync, no data loss. **DR** = cross-datacenter, async replication.
11. **Event Mesh** = brokers interconnected via Dynamic Message Routing (DMR) for global, location-transparent event distribution.
12. Versus Kafka: Solace = flexible pub-sub + queueing + multi-protocol; Kafka = distributed log with replay, best for high-throughput stream processing.