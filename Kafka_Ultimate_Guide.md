# Kafka Ultimate Interview Guide
**Complete Reference with Code Examples, Theory, and Real Banking Scenarios**

---

## 🎯 Quick Start: What is Kafka?

**Simple Analogy - Post Office System**

```
Old System (Direct Communication):
Service A → Service B
If B is down: message lost, bad!

New System (Kafka = Post Office):
Service A → Kafka (stores message) → Service B can read when ready
Even if B crashes, message waits safely
```

**Problems Kafka Solves**:
- ✓ Decoupling (services don't know about each other)
- ✓ Resilience (messages stored, never lost)
- ✓ Scale (multiple consumers in parallel)
- ✓ Buffering (handle traffic peaks)
- ✓ Auditability (immutable record of all events)

---

## 📚 Core Concepts - Explained Simply + Code

### 1. PRODUCER: Sending Payment Events

**Concept**: Service that publishes messages to Kafka (like dropping letter in post office)

#### 📌 CODE: Spring Boot Producer

```java
// application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.producer.value-serializer=org.apache.kafka.common.serialization.StringSerializer
spring.kafka.producer.acks=all
spring.kafka.producer.retries=3

// PaymentEvent.java
@Data
public class PaymentEvent {
    private String txnId;
    private String fromAccount;
    private String toAccount;
    private BigDecimal amount;
    private Long timestamp;
}

// PaymentProducer.java
@Service
public class PaymentProducer {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void publishPayment(PaymentEvent event) {
        String json = new ObjectMapper().writeValueAsString(event);
        
        kafkaTemplate.send("payment-events", 
                          event.getTxnId(),  // Partition key
                          json)
            .addCallback(
                success -> logger.info("✓ Published: {}", event.getTxnId()),
                failure -> logger.error("✗ Failed: {}", failure.getMessage())
            );
    }
}

// REST Controller
@PostMapping("/api/payment/transfer")
public ResponseEntity<?> transfer(@RequestBody PaymentEvent event) {
    event.setTimestamp(System.currentTimeMillis());
    paymentProducer.publishPayment(event);
    return ResponseEntity.accepted()
        .body("Payment initiated: " + event.getTxnId());
}
```

**Key Points**:
- `acks=all`: Wait for all replicas to confirm (durability)
- `retries=3`: Retry on failure
- Key-based routing: Same account always goes to same partition (ordering)

---

#### 📌 CODE: Apache Kafka Producer

```java
// PaymentProducer.java (Plain Kafka)
public class PaymentProducer {
    
    private KafkaProducer<String, String> producer;
    
    public PaymentProducer() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("acks", "all");
        props.put("retries", 3);
        props.put("compression.type", "snappy");
        
        this.producer = new KafkaProducer<>(props);
    }
    
    public void publishPaymentSync(PaymentEvent event) {
        String json = new ObjectMapper().writeValueAsString(event);
        
        ProducerRecord<String, String> record = 
            new ProducerRecord<>(
                "payment-events",           // Topic
                event.getTxnId(),           // Key
                json                        // Value
            );
        
        try {
            RecordMetadata metadata = producer.send(record).get();
            System.out.println("✓ Published to partition " + metadata.partition());
        } catch (InterruptedException | ExecutionException e) {
            System.err.println("✗ Failed: " + e.getMessage());
        }
    }
    
    public void publishPaymentAsync(PaymentEvent event) {
        String json = new ObjectMapper().writeValueAsString(event);
        ProducerRecord<String, String> record = 
            new ProducerRecord<>("payment-events", event.getTxnId(), json);
        
        producer.send(record, (metadata, exception) -> {
            if (exception != null) {
                System.err.println("✗ Failed: " + exception.getMessage());
            } else {
                System.out.println("✓ Sent to partition " + metadata.partition());
            }
        });
    }
}
```

**Comparison**:
- Spring Boot: Less code, automatic config, higher-level
- Apache Kafka: Full control, explicit, lower latency

---

### 2. CONSUMER: Processing Payment Events

**Concept**: Service that reads messages from Kafka (worker reading mail)

#### 📌 CODE: Spring Boot Consumer

```java
// application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.consumer.group-id=settlement-service
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit=false

// SettlementService.java
@Service
@Slf4j
public class SettlementService {
    
    @Autowired
    private BalanceRepository balanceRepository;
    
    @Autowired
    private TransactionRepository transactionRepository;
    
    @KafkaListener(topics = "payment-events", groupId = "settlement-service")
    public void processPayment(ConsumerRecord<String, String> record) {
        try {
            PaymentEvent payment = JsonUtils.parse(record.value(), PaymentEvent.class);
            
            log.info("Processing: {}", payment.getTxnId());
            
            // Idempotency check: already processed?
            if (transactionRepository.existsByTxnId(payment.getTxnId())) {
                log.info("✓ Already processed (idempotent)");
                return;
            }
            
            // Update balance
            Balance sender = balanceRepository.findById(payment.getFromAccount());
            Balance receiver = balanceRepository.findById(payment.getToAccount());
            
            sender.setBalance(sender.getBalance().subtract(payment.getAmount()));
            receiver.setBalance(receiver.getBalance().add(payment.getAmount()));
            
            // Record transaction (for idempotency)
            transactionRepository.save(Transaction.builder()
                .txnId(payment.getTxnId())
                .status("PROCESSED")
                .build());
            
            balanceRepository.saveAll(Arrays.asList(sender, receiver));
            
            log.info("✓ Settlement complete");
            
        } catch (Exception e) {
            log.error("✗ Failed: {}", e.getMessage());
            throw new RuntimeException(e);  // Retry or send to DLQ
        }
    }
}
```

---

#### 📌 CODE: Apache Kafka Consumer

```java
// SettlementService.java (Plain Kafka)
public class SettlementService {
    
    private KafkaConsumer<String, String> consumer;
    private volatile boolean running = true;
    
    public SettlementService() {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("group.id", "settlement-service");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("auto.offset.reset", "earliest");
        props.put("enable.auto.commit", false);  // Manual commits
        
        this.consumer = new KafkaConsumer<>(props);
        this.consumer.subscribe(Arrays.asList("payment-events"));
    }
    
    public void start() {
        Thread consumerThread = new Thread(() -> {
            try {
                while (running) {
                    // Poll for messages (wait 1 second)
                    ConsumerRecords<String, String> records = 
                        consumer.poll(Duration.ofMillis(1000));
                    
                    for (ConsumerRecord<String, String> record : records) {
                        try {
                            PaymentEvent payment = JsonUtils.parse(
                                record.value(), 
                                PaymentEvent.class
                            );
                            
                            // Process
                            updateBalance(payment);
                            
                            // Commit offset only on success
                            consumer.commitSync();
                            
                        } catch (Exception e) {
                            System.err.println("✗ Failed: " + e.getMessage());
                            // Don't commit - will retry
                            sendToDLQ(record, e);
                        }
                    }
                }
            } finally {
                consumer.close();
            }
        });
        
        consumerThread.start();
    }
}
```

---

### 3. TOPIC & PARTITION: Data Organization

**Concept**: Topic = channel for message type; Partition = splitting for parallelism

```
Topic: "payment-events" (all payment transactions)
Partitions: 10 (distributed across brokers)

Partition routing by key:
- Account_123 always goes to partition 5
- Account_456 always goes to partition 2
- Same account: same partition (ordering guaranteed)

Consumers:
- Consumer 1: reads partition 0-3
- Consumer 2: reads partition 4-7
- Consumer 3: reads partition 8-9
→ Parallel processing (3x faster)
```

---

### 4. OFFSET: Bookmark for Resuming

**Concept**: Position in topic (like page number in book)

```
Consumer reads offset 0-99
Commits: "I've read up to offset 99"
Later: consumer crashes
Restarts: "Where was I?"
Offset saved: 99
→ Continues from offset 100 (no reprocessing)
```

#### 📌 CODE: Offset Management

```java
// Spring Boot (automatic)
@KafkaListener(topics = "payment-events")
public void processPayment(ConsumerRecord<String, String> record) {
    // Process message
    // Offset committed automatically (if no exception)
    // Next time: starts from next message
}

// Apache Kafka (explicit)
while (running) {
    ConsumerRecords<String, String> records = 
        consumer.poll(Duration.ofMillis(1000));
    
    for (ConsumerRecord<String, String> record : records) {
        try {
            processPayment(record);
            
            // Commit offset (save progress)
            consumer.commitSync();
            
        } catch (Exception e) {
            // Don't commit - will retry same message
        }
    }
}

// Monitor consumer lag
kafka-consumer-groups \
  --bootstrap-server localhost:9092 \
  --group settlement-service \
  --describe

// Output shows:
// TOPIC | PARTITION | CURRENT-OFFSET | LOG-END-OFFSET | LAG
// payment-events | 0 | 5000 | 5100 | 100
//
// Lag = unread messages
// Lag > 0: consumer is behind
// Lag growing: consumer is slow or crashed
```

---

## 🚨 Critical Banking Issues & Solutions

### Issue #1: Message Duplication

**Problem**:
```
Consumer reads message 123
Updates balance
Crashes before committing offset
Restarts, reads message 123 again
Balance updated twice!
```

**Solution**:
```
Idempotency Key Strategy:
1. Each message has txn_id: "123"
2. Database: CREATE TABLE transactions (
              txn_id VARCHAR(255) PRIMARY KEY UNIQUE
           )
3. First processing: INSERT txn_id="123" → success
4. Duplicate arrives: INSERT txn_id="123" → fails (duplicate key)
5. Skip processing (already processed)
6. Result: processed exactly once
```

---

### Issue #2: Message Ordering

**Problem**:
```
Customer sends 2 payments:
1. Debit $600 from account
2. Debit $300 from account
Account has $500

Correct order:
$500 - $600 = FAIL (insufficient funds) ✓

Wrong order (if processed in parallel):
$500 - $300 = $200 ✓
$200 - $600 = FAIL ✓

Different result!
```

**Solution**:
```
Partition by account_id:
- All payments for account 123 → partition 0
- All payments for account 456 → partition 1
- Consumer reads partition 0 sequentially
- Ordering guaranteed within partition
- Parallel across different accounts
```

---

### Issue #3: Consumer Lag

**Problem**:
```
Producer: 100 msg/sec
Consumer: 10 msg/sec
Backlog grows to 1M messages
Customers see 100-second delays
```

**Solution**:
```
Scale horizontally:
- Topic has 10 partitions
- Add 9 more consumers (now 10 total)
- Each consumer handles 1 partition
- Processing speed: 10x faster
- Lag resolves in minutes
```

---

## 🔐 Schema Registry & Data Governance

**Concept**: Everyone agrees on message format

#### 📌 CODE: Spring Boot with Avro + Schema Registry

```java
// application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.schema-registry-url=http://localhost:8081
spring.kafka.producer.value-serializer=io.confluent.kafka.serializers.KafkaAvroSerializer

// PaymentEvent.java
@Data
public class PaymentEvent {
    private String txnId;
    private BigDecimal amount;
    private String accountId;
    private Long timestamp;
    private String status;  // New field (backward compatible)
}

// Producer
@Service
public class AvroPaymentProducer {
    
    @Autowired
    private KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    
    public void publishWithSchema(PaymentEvent event) {
        // Schema Registry validates
        // If mismatch: exception
        // If compatible: publishes with schema ID
        kafkaTemplate.send("payment-events", event.getTxnId(), event);
    }
}

// Consumer
@Service
public class AvroPaymentConsumer {
    
    @KafkaListener(topics = "payment-events")
    public void consumeWithSchema(PaymentEvent event) {
        // Automatically deserialized and validated
        logger.info("Received: {}", event.getTxnId());
    }
}
```

---

#### 📌 CODE: Apache Kafka with Avro

```java
public class AvroPaymentProcessor {
    
    public static void main(String[] args) {
        // Producer with Avro
        Properties producerProps = new Properties();
        producerProps.put("bootstrap.servers", "localhost:9092");
        producerProps.put("value.serializer", 
            "io.confluent.kafka.serializers.KafkaAvroSerializer");
        producerProps.put("schema.registry.url", "http://localhost:8081");
        
        KafkaProducer<String, PaymentEvent> producer = 
            new KafkaProducer<>(producerProps);
        
        // Create Avro record
        PaymentEvent event = PaymentEvent.newBuilder()
            .setTxnId("123")
            .setAmount(500)
            .build();
        
        // Send with schema validation
        producer.send(
            new ProducerRecord<>("payment-events", "123", event),
            (metadata, exception) -> {
                if (exception == null) {
                    System.out.println("✓ Schema validated");
                } else {
                    System.err.println("✗ Schema validation failed");
                }
            }
        );
    }
}
```

**Benefits**:
- Everyone uses same format
- Can evolve schema safely
- No corrupt messages
- Version control for data

---

## 🚨 Dead Letter Queue & Error Handling

**Concept**: Failed messages go to DLQ for investigation

#### 📌 CODE: Spring Boot with DLQ

```java
// application.properties
spring.kafka.listener.type=batch
spring.kafka.listener.concurrency=3

@Service
@Slf4j
public class PaymentServiceWithDLQ {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @KafkaListener(topics = "payment-events")
    public void processPayment(ConsumerRecord<String, String> record) {
        try {
            PaymentEvent payment = JsonUtils.parse(record.value(), PaymentEvent.class);
            
            log.info("Processing: {}", payment.getTxnId());
            
            // Process
            processPayment(payment);
            
            log.info("✓ Success");
            
        } catch (TransientException e) {
            log.warn("⚠ Transient error, will retry: {}", e.getMessage());
            throw new RuntimeException(e);  // Spring retries automatically
            
        } catch (PermanentException e) {
            log.error("✗ Permanent error, sending to DLQ: {}", e.getMessage());
            sendToDLQ(record, e);
            // Don't throw - skip this message
        }
    }
    
    // Spring auto-routes to DLQ when exception thrown
    @KafkaListener(topics = "payment-events-dlt")
    public void handleDLQ(ConsumerRecord<String, String> record) {
        log.error("DLQ: Message needs investigation: {}", record.value());
        // Alert operations team
    }
    
    private void sendToDLQ(ConsumerRecord<String, String> record, Exception e) {
        String dlqPayload = JsonUtils.toJson(DLQRecord.builder()
            .originalMessage(record.value())
            .error(e.getMessage())
            .timestamp(System.currentTimeMillis())
            .build());
        
        kafkaTemplate.send("payment-events-dlq", record.key(), dlqPayload);
    }
}
```

---

#### 📌 CODE: Apache Kafka with Manual Retry & DLQ

```java
public class PaymentServiceWithManualDLQ {
    
    private KafkaConsumer<String, String> consumer;
    private KafkaProducer<String, String> dlqProducer;
    
    private static final int MAX_RETRIES = 3;
    private static final long[] RETRY_DELAYS = {1000, 5000, 30000};
    
    public void processWithDLQ() {
        while (running) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(1000));
            
            for (ConsumerRecord<String, String> record : records) {
                processWithRetryAndDLQ(record);
            }
        }
    }
    
    private void processWithRetryAndDLQ(ConsumerRecord<String, String> record) {
        int retryCount = 0;
        Exception lastException = null;
        
        while (retryCount < MAX_RETRIES) {
            try {
                PaymentEvent payment = JsonUtils.parse(record.value(), PaymentEvent.class);
                
                System.out.println("Attempt " + (retryCount + 1));
                
                // Process
                updateBalance(payment);
                
                // Success
                consumer.commitSync();
                return;
                
            } catch (TemporaryFailureException e) {
                lastException = e;
                retryCount++;
                
                if (retryCount < MAX_RETRIES) {
                    long delayMs = RETRY_DELAYS[Math.min(retryCount - 1, 2)];
                    System.out.println("⚠ Retrying in " + delayMs + "ms");
                    
                    try {
                        Thread.sleep(delayMs);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            } catch (PermanentFailureException e) {
                System.out.println("✗ Permanent error, sending to DLQ");
                sendToDLQ(record, e);
                consumer.commitSync();
                return;
            }
        }
        
        // All retries exhausted
        System.out.println("✗ Max retries exceeded");
        sendToDLQ(record, lastException);
    }
    
    private void sendToDLQ(ConsumerRecord<String, String> record, Exception e) {
        try {
            String dlqPayload = JsonUtils.toJson(DLQRecord.builder()
                .originalMessage(record.value())
                .error(e.getMessage())
                .timestamp(System.currentTimeMillis())
                .build());
            
            dlqProducer.send(
                new ProducerRecord<>("payment-events-dlq", record.key(), dlqPayload)
            ).get();
            
            System.out.println("✓ Sent to DLQ");
        } catch (Exception dlqException) {
            System.err.println("✗ Failed to send to DLQ");
        }
    }
}

// Exception hierarchy
class TemporaryFailureException extends RuntimeException {}
class PermanentFailureException extends RuntimeException {}
```

**Strategy**:
- Transient errors (network, timeout): retry with backoff
- Permanent errors (bad data, account missing): send to DLQ
- Auto-replay for transient, manual review for permanent

---

## 🚀 Exactly-Once Semantics

**Guarantee**: Message processed 1 time, no more, no less

#### 📌 CODE: Spring Boot with Exactly-Once

```java
// application.properties
spring.kafka.producer.acks=all
spring.kafka.producer.enable-idempotence=true
spring.kafka.consumer.enable-auto-commit=false
spring.kafka.consumer.isolation-level=read_committed

@Service
@Slf4j
public class PaymentProcessorExactlyOnce {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private BalanceRepository balanceRepository;
    
    @Autowired
    private TransactionRepository transactionRepository;
    
    @KafkaListener(topics = "payment-events")
    @Transactional  // Database transaction
    public void processExactlyOnce(ConsumerRecord<String, String> record) {
        try {
            PaymentEvent payment = JsonUtils.parse(record.value(), PaymentEvent.class);
            
            // Step 1: Check if already processed (idempotency)
            if (transactionRepository.existsByTxnId(payment.getTxnId())) {
                log.info("✓ Already processed (idempotent)");
                return;
            }
            
            // Step 2: Update balance
            Balance sender = balanceRepository.findById(payment.getFromAccount());
            sender.setBalance(sender.getBalance().subtract(payment.getAmount()));
            balanceRepository.save(sender);
            
            // Step 3: Record transaction (UNIQUE constraint on txn_id)
            transactionRepository.save(Transaction.builder()
                .txnId(payment.getTxnId())  // PRIMARY KEY
                .status("PROCESSED")
                .build());
            
            // Step 4: Publish settlement (within same transaction)
            kafkaTemplate.send("settlement-completed", payment.getTxnId(), "SETTLED");
            
            // All commits here atomically:
            // 1. DB changes committed
            // 2. Offset committed
            // Result: EXACTLY-ONCE
            
        } catch (DuplicateKeyException e) {
            // Already processed (idempotency worked)
            log.info("Duplicate detected");
        } catch (Exception e) {
            log.error("Error: {}", e.getMessage());
            throw e;  // Rollback and retry
        }
    }
}
```

**Database Schema**:
```sql
CREATE TABLE transactions (
    txn_id VARCHAR(255) PRIMARY KEY,  -- UNIQUE constraint
    status VARCHAR(20),
    amount DECIMAL,
    created_at TIMESTAMP
);

-- First attempt: INSERT succeeds
-- Duplicate attempt: INSERT fails (duplicate key)
--   → Exception caught, skip processing
--   → Result: message not processed twice
```

---

#### 📌 CODE: Apache Kafka with Idempotent Producer

```java
public class ExactlyOnceProcessor {
    
    private KafkaProducer<String, String> producer;
    private KafkaConsumer<String, String> consumer;
    
    public ExactlyOnceProcessor() {
        // Idempotent producer
        Properties producerProps = new Properties();
        producerProps.put("bootstrap.servers", "localhost:9092");
        producerProps.put("acks", "all");
        producerProps.put("enable.idempotence", true);  // Auto-dedup
        producerProps.put("max.in.flight.requests.per.connection", 5);
        
        // Consumer with read_committed
        Properties consumerProps = new Properties();
        consumerProps.put("bootstrap.servers", "localhost:9092");
        consumerProps.put("group.id", "settlement-service");
        consumerProps.put("isolation.level", "read_committed");
        consumerProps.put("enable.auto.commit", false);
        
        this.producer = new KafkaProducer<>(producerProps);
        this.consumer = new KafkaConsumer<>(consumerProps);
    }
    
    public void processExactlyOnce() {
        while (running) {
            ConsumerRecords<String, String> records = 
                consumer.poll(Duration.ofMillis(1000));
            
            for (ConsumerRecord<String, String> record : records) {
                processWithIdempotency(record);
            }
        }
    }
    
    private void processWithIdempotency(ConsumerRecord<String, String> record) {
        Connection db = null;
        try {
            db = getConnection();
            db.setAutoCommit(false);
            
            PaymentEvent payment = JsonUtils.parse(record.value(), PaymentEvent.class);
            
            // Check if already processed
            String checkSql = "SELECT COUNT(*) FROM transactions WHERE txn_id = ?";
            PreparedStatement stmt = db.prepareStatement(checkSql);
            stmt.setString(1, payment.getTxnId());
            ResultSet rs = stmt.executeQuery();
            
            if (rs.next() && rs.getInt(1) > 0) {
                System.out.println("✓ Already processed");
                db.commit();
                consumer.commitSync();
                return;
            }
            
            // Insert transaction (UNIQUE constraint)
            String insertSql = "INSERT INTO transactions (txn_id, amount, status) " +
                             "VALUES (?, ?, ?)";
            PreparedStatement insertStmt = db.prepareStatement(insertSql);
            insertStmt.setString(1, payment.getTxnId());
            insertStmt.setBigDecimal(2, payment.getAmount());
            insertStmt.setString(3, "PROCESSED");
            insertStmt.executeUpdate();
            
            db.commit();
            
            // Publish (idempotent producer prevents duplicates)
            ProducerRecord<String, String> settlementRecord = 
                new ProducerRecord<>("settlement-completed", payment.getTxnId(), "SETTLED");
            
            producer.send(settlementRecord).get();  // Wait for confirmation
            
            // Commit offset only after everything succeeds
            consumer.commitSync();
            
            System.out.println("✓ Exactly-once complete");
            
        } catch (SQLException e) {
            if (db != null) {
                try {
                    db.rollback();  // Rollback DB
                } catch (SQLException ignored) {}
            }
            System.err.println("✗ DB Error");
            // Don't commit offset, will retry
        } finally {
            if (db != null) {
                try {
                    db.close();
                } catch (SQLException ignored) {}
            }
        }
    }
}
```

---

## 🏭 Kafka Streams: Real-Time Processing

**Concept**: Transform data in real-time (like factory assembly line)

#### 📌 CODE: Spring Boot Kafka Streams (Fraud Detection)

```java
// application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.application.name=fraud-detection
spring.kafka.streams.application-id=fraud-detection

@Configuration
public class FraudDetectionTopology {
    
    @Bean
    public KStream<String, PaymentEvent> fraudDetectionStream(StreamsBuilder builder) {
        
        // 1. Read transactions
        KStream<String, PaymentEvent> transactions = builder
            .stream("payment-events", Consumed.with(Serdes.String(), getPaymentSerde()));
        
        // 2. Enrich (add customer info)
        KStream<String, EnrichedPayment> enriched = transactions
            .mapValues(payment -> enrichPayment(payment));
        
        // 3. Calculate fraud score
        KStream<String, FraudScore> scored = enriched
            .mapValues(enrichedPayment -> calculateFraudScore(enrichedPayment));
        
        // 4. Filter high-risk
        KStream<String, FraudScore> highRisk = scored
            .filter((key, score) -> score.getRiskScore() > 0.7);
        
        // 5. Output alerts
        highRisk.to("fraud-alerts", Produced.with(Serdes.String(), getFraudSerde()));
        
        // 6. Output approved
        scored.filter((key, score) -> score.getRiskScore() <= 0.7)
            .to("approved-payments", Produced.with(Serdes.String(), getApprovedSerde()));
        
        return transactions;
    }
    
    private FraudScore calculateFraudScore(EnrichedPayment payment) {
        double score = 0.0;
        List<String> factors = new ArrayList<>();
        
        // Rule 1: Velocity (5 txns in 1 minute = suspicious)
        if (payment.getTxnCountPerMinute() > 5) {
            score += 0.3;
            factors.add("High transaction velocity");
        }
        
        // Rule 2: Amount
        if (payment.getAmount().compareTo(payment.getLimit()) > 0) {
            score += 0.4;
            factors.add("Exceeds limit");
        }
        
        // Rule 3: Geolocation jump
        if (isGeolocationJump(payment)) {
            score += 0.2;
            factors.add("Impossible travel");
        }
        
        // Rule 4: Merchant
        if (isHighRiskMerchant(payment)) {
            score += 0.1;
            factors.add("High-risk merchant");
        }
        
        return FraudScore.builder()
            .txnId(payment.getTxnId())
            .riskScore(Math.min(score, 1.0))
            .factors(factors)
            .build();
    }
}

@Configuration
public class KafkaStreamsConfig {
    
    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public StreamsConfig streamsConfig() {
        Map<String, Object> config = new HashMap<>();
        config.put(StreamsConfig.APPLICATION_ID_CONFIG, "fraud-detection");
        config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
        return new StreamsConfig(config);
    }
}
```

---

#### 📌 CODE: Apache Kafka Streams (Plain Java)

```java
public class FraudDetectionApp {
    
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "fraud-detection");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2);
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Read transactions
        KStream<String, String> transactions = builder.stream("payment-events");
        
        // Parse JSON
        KStream<String, PaymentEvent> parsed = transactions
            .mapValues(json -> JsonUtils.parse(json, PaymentEvent.class));
        
        // Enrich
        KStream<String, EnrichedPayment> enriched = parsed
            .mapValues((key, payment) -> enrichWithData(payment));
        
        // Score
        KStream<String, FraudScore> scored = enriched
            .mapValues(payment -> calculateFraudScore(payment));
        
        // Filter and output
        scored.filter((key, score) -> score.getRiskScore() > 0.7)
            .to("fraud-alerts");
        
        scored.filter((key, score) -> score.getRiskScore() <= 0.7)
            .to("approved-payments");
        
        // Run
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
        
        streams.start();
    }
    
    private static FraudScore calculateFraudScore(EnrichedPayment payment) {
        double score = 0.0;
        
        if (payment.getTxnCountPerMinute() > 5) score += 0.3;
        if (payment.getAmount().compareTo(payment.getLimit()) > 0) score += 0.4;
        if (isGeolocationJump(payment)) score += 0.2;
        if (isHighRiskMerchant(payment)) score += 0.1;
        
        return new FraudScore(payment.getTxnId(), Math.min(score, 1.0));
    }
}
```

**Use Cases**:
- Real-time fraud detection (this example)
- Balance updates (enrichment)
- Windowed aggregations (transactions per hour)
- Stream joins (combine multiple topics)

---

## 📊 Kafka Streams: Aggregations & Windowing

**Concept**: Group and combine messages over time windows for analytics

**Real-World Example**: Calculate transaction volume per minute, hourly fraud alerts, daily settlement summaries

### Aggregation Types

```
1. Count: How many transactions?
2. Sum: Total transaction amount?
3. Min/Max: Minimum and maximum amounts?
4. Custom: Complex aggregations (running balance, risk score)

Windowing Types:
- Tumbling: Non-overlapping time windows (5-min chunks)
- Hopping: Overlapping windows (5-min window every 1-min)
- Session: Activity-based windows (close when gap detected)
- Grace Period: Allow late data (data arrived 2 min late)
```

#### 📌 CODE: Spring Boot Kafka Streams Aggregation

```java
// application.properties
spring.kafka.bootstrap-servers=localhost:9092
spring.kafka.streams.application-id=transaction-analytics
spring.kafka.streams.state-dir=/tmp/kafka-streams

// TransactionAnalyticsTopology.java
@Configuration
public class TransactionAnalyticsTopology {
    
    @Bean
    public KStream<String, TransactionStats> analyticsStream(StreamsBuilder builder) {
        
        // 1. Read transactions
        KStream<String, PaymentEvent> transactions = builder
            .stream("payment-events", 
                   Consumed.with(Serdes.String(), getPaymentSerde()));
        
        // 2. Aggregate: Count transactions per account per minute
        KTable<Windowed<String>, Long> txnCountPerMinute = transactions
            .groupByKey()  // Group by account_id (key)
            .windowedBy(TimeWindows.of(Duration.ofMinutes(1)))  // 1-minute window
            .count();  // Count transactions
        
        // 3. Aggregate: Sum amounts per account per hour
        KTable<Windowed<String>, BigDecimal> volumePerHour = transactions
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofHours(1)))
            .aggregate(
                BigDecimal.ZERO,  // Initial value
                (key, payment, accumulator) -> 
                    accumulator.add(payment.getAmount()),  // Adder
                Materialized.as("transaction-volume-store")
            );
        
        // 4. Aggregate: Track max amount per day
        KTable<Windowed<String>, BigDecimal> maxAmountPerDay = transactions
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofDays(1)))
            .aggregate(
                BigDecimal.ZERO,
                (key, payment, max) -> 
                    payment.getAmount().compareTo(max) > 0 
                        ? payment.getAmount() 
                        : max
            );
        
        // 5. Aggregate: Custom - Calculate average
        KTable<Windowed<String>, AverageAmount> avgPerWindow = transactions
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
            .aggregate(
                () -> new AverageAmount(BigDecimal.ZERO, 0),  // Initial
                (key, payment, avg) -> {  // Aggregator
                    BigDecimal newSum = avg.getSum().add(payment.getAmount());
                    int newCount = avg.getCount() + 1;
                    return new AverageAmount(newSum, newCount);
                },
                Materialized.as("average-store")
            );
        
        // 6. Convert to stream and output
        txnCountPerMinute
            .toStream()
            .map((windowedKey, count) -> {
                TransactionStats stats = TransactionStats.builder()
                    .accountId(windowedKey.key())
                    .window(windowedKey.window().startTime())
                    .txnCount(count)
                    .timestamp(System.currentTimeMillis())
                    .build();
                return new KeyValue<>(windowedKey.key(), stats);
            })
            .to("transaction-stats");
        
        return transactions;
    }
}

// Usage: Query stats in real-time
@Service
public class AnalyticsQueryService {
    
    @Autowired
    private KafkaStreams kafkaStreams;
    
    public Long getTxnCountForAccount(String accountId) {
        ReadOnlyWindowStore<String, Long> store = kafkaStreams
            .getQueryableStore("transaction-count-store", 
                              QueryableStoreTypes.windowStore());
        
        // Get all windows for this account
        WindowStoreIterator<Long> iterator = store
            .fetch(accountId, Instant.now().minus(Duration.ofHours(1)), Instant.now());
        
        long totalCount = 0;
        while (iterator.hasNext()) {
            KeyValue<Long, Long> entry = iterator.next();
            totalCount += entry.value;
        }
        return totalCount;
    }
}

// AverageAmount.java
@Data
@AllArgsConstructor
public class AverageAmount {
    private BigDecimal sum;
    private int count;
    
    public BigDecimal getAverage() {
        if (count == 0) return BigDecimal.ZERO;
        return sum.divide(BigDecimal.valueOf(count), 2, RoundingMode.HALF_UP);
    }
}

// TransactionStats.java
@Data
@Builder
public class TransactionStats {
    private String accountId;
    private Instant window;
    private Long txnCount;
    private Long timestamp;
}
```

---

#### 📌 CODE: Apache Kafka Streams Aggregation

```java
public class TransactionAnalyticsApp {
    
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "transaction-analytics");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Read transactions
        KStream<String, String> transactions = builder.stream("payment-events");
        
        // Parse JSON
        KStream<String, PaymentEvent> parsed = transactions
            .mapValues(json -> JsonUtils.parse(json, PaymentEvent.class));
        
        // 1. Count transactions per account per minute
        parsed
            .groupByKey()  // Group by key (account_id)
            .windowedBy(TimeWindows.of(Duration.ofMinutes(1)))
            .count()
            .toStream()
            .map((windowedKey, count) -> {
                String accountId = windowedKey.key();
                long windowStart = windowedKey.window().startTime().toEpochMilli();
                String result = accountId + " - Count: " + count + 
                               " at " + new Date(windowStart);
                return new KeyValue<>(accountId, result);
            })
            .to("txn-count-per-minute");
        
        // 2. Sum amounts per account per hour
        parsed
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofHours(1)))
            .aggregate(
                BigDecimal.ZERO,  // Initializer
                (key, payment, sum) -> sum.add(payment.getAmount()),  // Aggregator
                Materialized.as("volume-hourly")
            )
            .toStream()
            .map((windowedKey, sum) -> {
                String result = windowedKey.key() + " - Volume: " + sum;
                return new KeyValue<>(windowedKey.key(), result);
            })
            .to("txn-volume-per-hour");
        
        // 3. Detect velocity: 5+ transactions in 5 minutes
        parsed
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
            .count()
            .filter((windowedKey, count) -> count >= 5)
            .toStream()
            .to("high-velocity-alerts");
        
        // 4. Track running balance per account
        parsed
            .groupByKey()
            .aggregate(
                BigDecimal.ZERO,  // Starting balance
                (key, payment, balance) -> {
                    // Debit from sender account
                    if (key.equals(payment.getFromAccount())) {
                        return balance.subtract(payment.getAmount());
                    }
                    // Credit to receiver account
                    else if (key.equals(payment.getToAccount())) {
                        return balance.add(payment.getAmount());
                    }
                    return balance;
                },
                Materialized.as("running-balance-store")
            )
            .toStream()
            .to("account-balance-updates");
        
        // 5. Hopping window: 5-minute window every 1 minute
        parsed
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5))
                                   .advanceBy(Duration.ofMinutes(1)))  // Hop by 1 min
            .count()
            .toStream()
            .to("hopping-window-counts");
        
        // 6. Session window: Close when 10-minute gap detected
        parsed
            .groupByKey()
            .windowedBy(SessionWindows.with(Duration.ofMinutes(10)))
            .count()
            .toStream()
            .to("session-analytics");
        
        // Build and run
        Topology topology = builder.build();
        KafkaStreams streams = new KafkaStreams(topology, props);
        
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
        
        streams.start();
    }
}
```

---

### Banking Use Cases for Aggregations

#### Use Case 1: Transaction Velocity Detection

```
Requirement: Alert if account has 5+ transactions in 5 minutes (fraud signal)

Code (Spring Boot):
KTable<Windowed<String>, Long> velocityCheck = transactions
    .groupByKey()  // account_id
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count();

// Send alerts
velocityCheck
    .filter((windowedKey, count) -> count >= 5)
    .toStream()
    .to("velocity-alerts");

Result:
- Account 123: 7 transactions in 5 minutes → ALERT
- Account 456: 2 transactions in 5 minutes → No alert

Real-time decision:
- Alert sent immediately
- Fraud team can investigate
- Block account if needed
```

#### Use Case 2: Daily Settlement Summaries

```
Requirement: Calculate total settled amount per merchant per day

Code:
KTable<Windowed<String>, BigDecimal> dailySettlement = transactions
    .filter((key, payment) -> "SETTLED".equals(payment.getStatus()))
    .groupBy((key, payment) -> payment.getMerchantId())  // Group by merchant
    .windowedBy(TimeWindows.of(Duration.ofDays(1)))
    .aggregate(
        BigDecimal.ZERO,
        (merchantId, payment, total) -> total.add(payment.getAmount())
    );

Result:
- Merchant M1: $1.2M settled on Feb 26
- Merchant M2: $890K settled on Feb 26
- Used for: Finance reconciliation, settlement reports
```

#### Use Case 3: Account Balance Aggregation

```
Requirement: Real-time balance per account (sum of all credit/debit)

Code:
KTable<String, AccountBalance> balances = transactions
    .groupByKey()  // account_id
    .aggregate(
        () -> new AccountBalance(BigDecimal.ZERO),
        (accountId, payment, balance) -> {
            if (accountId.equals(payment.getFromAccount())) {
                balance.debit(payment.getAmount());
            } else if (accountId.equals(payment.getToAccount())) {
                balance.credit(payment.getAmount());
            }
            return balance;
        }
    );

Result:
- Account 1: Balance = +$5000 (running total)
- Account 2: Balance = -$2000 (overdrawn)
- Real-time decision: Can accept more transfers?
```

#### Use Case 4: Hopping Window Analytics

```
Requirement: Monitor metrics constantly (5-min window, refreshed every 1 min)

Code:
KTable<Windowed<String>, Long> hoppingCount = transactions
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5))
                           .advanceBy(Duration.ofMinutes(1)))
    .count();

Windows generated:
- 12:00-12:05 (then 12:01-12:06, 12:02-12:07, ...)
- Dashboard updated every minute
- Smoother trending (not jerky)
- Better for charts

Result:
- More data points
- Trend detection easier
- Early warning for issues
```

#### Use Case 5: Session Windows

```
Requirement: Group transactions by "session" (activity burst)

Code:
KTable<Windowed<String>, SessionStats> sessions = transactions
    .groupByKey()  // account_id
    .windowedBy(SessionWindows.with(Duration.ofMinutes(10)))
    .aggregate(
        () -> new SessionStats(),
        (key, payment, stats) -> {
            stats.addTransaction(payment);
            return stats;
        }
    );

Behavior:
- Customer logs in, does 10 transactions in 8 minutes
- Next transaction after 10+ minute gap → NEW SESSION
- Session closed when no activity for 10 minutes

Result:
- Session 1: 10 txns (Feb 26, 14:00-14:08)
- Session 2: 5 txns (Feb 26, 14:30-14:35)
- Session 3: 3 txns (Feb 27, 09:00-09:05)

Use case:
- User behavior analysis
- Risk scoring (many sessions per day = suspicious)
```

---

### Windowing Deep Dive

```
Window Type       | Use Case                    | Overlap?
──────────────────────────────────────────────────────────
Tumbling          | Hourly reports             | No
Hopping           | Real-time dashboard        | Yes
Session           | User activity tracking     | N/A (dynamic)
Grace Period      | Late data handling         | Configurable

Grace Period Example:
Message delayed 2 minutes arrives to 5-minute window

Without grace:
- Window: 12:00-12:05
- Message arrives: 12:07 (too late)
- Dropped

With grace (5 minutes):
- Window: 12:00-12:05
- Grace period: until 12:10
- Message at 12:07: ACCEPTED
- Message at 12:12: DROPPED

Code (Spring Boot):
.windowedBy(TimeWindows.of(Duration.ofMinutes(5))
                       .grace(Duration.ofMinutes(5)))
```

---

### State Store Queries (Real-time Analytics)

```
Once aggregated, results stored in "state store" (KTable)
Can query for real-time values

Spring Boot Example:
@Service
public class AnalyticsQuery {
    
    @Autowired
    private KafkaStreams kafkaStreams;
    
    public AccountBalance getAccountBalance(String accountId) {
        // Query the state store
        ReadOnlyKeyValueStore<String, AccountBalance> store = 
            kafkaStreams.getQueryableStore(
                "balances-store", 
                QueryableStoreTypes.keyValueStore()
            );
        
        return store.get(accountId);  // Get current balance
    }
    
    public List<VelocityAlert> getHighVelocityAccounts() {
        ReadOnlyWindowStore<String, Long> store = 
            kafkaStreams.getQueryableStore(
                "velocity-store", 
                QueryableStoreTypes.windowStore()
            );
        
        // Get all accounts with count >= 5
        List<VelocityAlert> alerts = new ArrayList<>();
        // ... iterate and filter
        return alerts;
    }
}
```

---

### Common Aggregation Patterns in Banking

```
1. Running Total
   aggregate(0, (k, v, total) -> total + v.amount)
   Use: Account balance, cumulative revenue

2. Running Count
   count()
   Use: Transaction velocity, event counting

3. Min/Max Tracking
   aggregate(ZERO, (k, v, min) -> 
       v.amount < min ? v.amount : min)
   Use: Daily low/high price, min transaction

4. Set Collection
   aggregate(HashSet, (k, v, set) -> {
       set.add(v.id);
       return set;
   })
   Use: Unique customers, unique merchants

5. String Concatenation
   aggregate("", (k, v, str) -> str + v.id + ",")
   Use: Transaction history per account

6. Complex Objects
   aggregate(new Stats(), (k, v, stats) -> {
       stats.add(v);
       return stats;
   })
   Use: Custom aggregations, multiple fields
```

---

## 📊 Auditing & Compliance

**Concept**: Immutable record of all transactions for regulatory compliance

#### 📌 CODE: Audit Logging

```java
@Service
public class AuditService {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void logPaymentInitiated(PaymentEvent payment) {
        AuditEvent event = AuditEvent.builder()
            .txnId(payment.getTxnId())
            .action("PAYMENT_INITIATED")
            .timestamp(System.currentTimeMillis())
            .details(JsonUtils.toJson(payment))
            .initiatorId(getCurrentUserId())
            .ipAddress(getCurrentIPAddress())
            .build();
        
        // Publish to immutable audit topic
        kafkaTemplate.send("audit-log", payment.getTxnId(), 
            JsonUtils.toJson(event));
    }
    
    public void logPaymentSettled(String txnId) {
        AuditEvent event = AuditEvent.builder()
            .txnId(txnId)
            .action("PAYMENT_SETTLED")
            .timestamp(System.currentTimeMillis())
            .build();
        
        kafkaTemplate.send("audit-log", txnId, JsonUtils.toJson(event));
    }
}

// Usage in settlement service
@KafkaListener(topics = "payment-events")
public void processPayment(PaymentEvent payment) {
    // Process
    updateBalance(payment);
    
    // Audit
    auditService.logPaymentSettled(payment.getTxnId());
}
```

**Audit Trail for JPMorgan**:
- Every transaction: recorded
- Every approval: recorded (who, when, why)
- Every failure: recorded (reason, error)
- Never edited: only appended
- 7-year retention: compliance requirement
- Queryable: auditors can verify

---

## 🎯 Interview Questions & Answers

### Q1: "Design Kafka for payment system handling 10M txns/day"

**Answer**:

```
Architecture:
- Brokers: 3-5 (high availability)
- Replication factor: 3 (durability)
- Topics:
  * payment-events (50 partitions)
  * settlement-completed (30 partitions)
  * fraud-alerts (10 partitions)
  * audit-log (100 partitions)

Configuration:
Producer:
  acks = 'all' (wait for all replicas)
  enable.idempotence = true
  retries = 3
  
Consumer:
  isolation.level = 'read_committed'
  enable.auto.commit = false
  max.poll.records = 100

Application:
- Idempotency keys in DB
- Unique constraints prevent duplicates
- DLQ for failed messages
- Retry logic with exponential backoff

Streams:
- Kafka Streams for fraud detection
- Real-time enrichment
- Alerts within 100ms

Audit:
- All events logged to audit-log
- 7-year retention
- Immutable records

Monitoring:
- Consumer lag < 100 messages (alert if higher)
- Producer error rate < 0.1%
- DLQ message count (alert if > 10)
- Broker health (CPU, memory, disk)

Result:
- 10M txns/day (115 txns/sec average)
- 99.99% uptime
- Zero message loss
- Exactly-once processing
- Sub-second fraud detection
- 7-year audit trail
```

---

### Q2: "How do you prevent duplicate payment processing?"

**Answer**:

```
Scenario:
Consumer processes payment, crashes before offset commit
Restarts, processes same payment again

Prevention Strategy:

1. Idempotency Key
   Each message includes: txn_id (unique)
   Database: CREATE TABLE transactions (
              txn_id VARCHAR(255) PRIMARY KEY
           )

2. Check Before Processing
   if (transactionRepository.existsByTxnId(txnId)) {
       return;  // Already processed
   }

3. Process & Record Atomically
   @Transactional
   public void process(PaymentEvent payment) {
       // Update balance
       updateBalance(payment);
       
       // Record transaction (unique constraint)
       transactionRepository.save(
           Transaction.builder()
               .txnId(payment.getTxnId())
               .status("PROCESSED")
               .build()
       );
   }

4. On Duplicate Arrival
   Second attempt: INSERT fails (duplicate key)
   Exception caught: skip processing
   Message never processed twice
   ✓ Exactly-once guaranteed

Configuration:
Producer: enable.idempotence = true
Consumer: isolation.level = read_committed
Database: unique constraint on txn_id
```

---

### Q3: "What's the difference between Kafka Streams and ksqlDB?"

**Answer**:

```
Kafka Streams (Programmatic):
- Use case: Complex transformations
- Language: Java/Python code
- Power: Full access to Kafka internals
- Example: Fraud detection with ML model
- Complexity: Developers needed
- Performance: Lower latency

ksqlDB (SQL-based):
- Use case: SQL queries on streams
- Language: SQL
- Power: Simpler queries
- Example: "Show payments > $1000 last hour"
- Complexity: Analysts can write
- Performance: Good for simple queries

When to use each:

Kafka Streams:
✓ Machine learning models
✓ Complex business logic
✓ Stream joins with lookup tables
✓ Stateful processing
✓ Microservices

ksqlDB:
✓ Real-time dashboards
✓ Ad-hoc SQL queries
✓ Aggregations (count, sum, avg)
✓ Time-windowed analytics
✓ Non-developers can use

In Banking:
- Fraud detection: Kafka Streams
- Real-time reporting dashboard: ksqlDB
- Settlement processing: Kafka Streams
- Monitoring alerts: ksqlDB
```

---

### Q4: "How do you scale a Kafka consumer that's lagging?"

**Answer**:

```
Problem:
Consumer lag = 1M messages
Customers experiencing 100+ second delays

Diagnosis:
1. Check consumer lag
   kafka-consumer-groups --group settlement-service --describe
   
2. Check processing time per message
   avg latency = 50ms
   
3. Calculate:
   Messages per second needed = 1M lag / 100s = 10K msg/sec
   Current capacity = 1000 / 50ms = 20 msg/sec
   Need: 10K / 20 = 500x scaling!

Solutions:

Solution 1: Scale Consumers (Recommended)
- Topic has 10 partitions
- Currently 1 consumer
- Add 9 more consumers (now 10 total)
- Each consumer handles 1 partition
- Capacity: 20 * 10 = 200 msg/sec
- Still not enough!

Solution 2: Optimize Code
- Batch processing (process 100 at once)
- Cache data (avoid DB queries)
- Use async I/O (non-blocking)
- New capacity: 1000 msg/sec
- Lag resolves in 1000 seconds

Solution 3: Combine
- Add 10 more consumers = 2000 msg/sec
- Plus optimize code = 10K+ msg/sec
- Lag resolves in < 100 seconds

Monitoring:
- Add alert: if lag > 100K messages
- Auto-scale: if lag > 100K and CPU < 50%
- Monitor: processing latency (p95, p99)
- Dashboard: consumer lag trend
```

---

### Q5: "Zookeeper vs KRaft - which should we use?"

**Answer**:

```
Zookeeper (Old, External):
- Separate cluster (3-5 servers)
- Coordinates Kafka brokers
- Stores offsets
- Proven, battle-tested
- Works well for <100 brokers

KRaft (New, Built-in):
- Kafka manages itself
- No external coordinator needed
- Faster leader election
- Future direction
- Better for large clusters

Decision Matrix:

For new deployments (2026):
→ Use KRaft
  ✓ Simpler (one less system)
  ✓ Better performance
  ✓ Future-proof
  ✓ No external dependencies

For existing clusters:
→ Stay on Zookeeper (unless migrating)
  ✓ Proven in production
  ✓ Migration is risky
  ✓ Gradual migration plan

Migration Plan:
1. Build KRaft cluster in test
2. Replicate data from Zookeeper
3. Pilot in production (subset of topics)
4. Full migration (planned downtime)
5. Decommission Zookeeper

For JPMorgan:
"New clusters: KRaft
Existing clusters: gradual migration during maintenance windows
Timeline: 2-3 year migration across all clusters"
```

---

## 🌍 Kafka Ecosystem Overview

```
Core Messaging:
- Brokers (storage)
- Producers (publish)
- Consumers (read)

Coordination:
- Zookeeper (old)
- KRaft (new)

Integration:
- Kafka Connect (ETL)
- Source connectors (pull data in)
- Sink connectors (push data out)

Stream Processing:
- Kafka Streams (Java/Python)
- ksqlDB (SQL)
- Apache Flink (advanced)
- Apache Spark (batch+stream)

Schema Management:
- Schema Registry
- Avro / Protobuf / JSON Schema

Data Quality:
- Monitoring (metrics)
- Alerting (thresholds)
- Audit logging

Advanced:
- Tiered storage (cost reduction)
- MirrorMaker (multi-region)
- Exactly-once semantics
- Transactions (ACID)
```

---

## 💡 Key Takeaways for Interview

### What Interviewers Want to Hear

✓ "Kafka decouples services"
✓ "Topics split into partitions for parallelism"  
✓ "Replication ensures durability"
✓ "Idempotency prevents duplicates"
✓ "DLQ handles failures gracefully"
✓ "Exactly-once for financial accuracy"
✓ "Audit trail for compliance"
✓ "Spring Boot for microservices, Apache Kafka for control"

### Common Mistakes to Avoid

❌ "Kafka guarantees exactly-once" (incorrect - app must implement)
❌ "Single partition is fine for all data" (poor scaling)
❌ No mention of replication (data loss risk)
❌ "Just use Spring Boot" without knowing Apache Kafka
❌ Ignoring DLQ strategy (failures unhandled)
❌ No mention of monitoring/alerting

### Real Interview Scenario

Interviewer: "Design payment system with Kafka"

You should cover:
1. ✓ Architecture (brokers, topics, partitions)
2. ✓ Producers (idempotent, acks=all)
3. ✓ Consumers (multiple groups, lag monitoring)
4. ✓ Error handling (DLQ, retries)
5. ✓ Exactly-once (idempotency keys, unique constraints)
6. ✓ Fraud detection (Kafka Streams)
7. ✓ Audit (immutable log)
8. ✓ Scale (10M txns/day)
9. ✓ Monitoring (alerts, dashboards)
10. ✓ Code examples (producer, consumer, streams)

---

## 🚀 Final Checklist Before Interview

**Kafka Core**:
☐ What is Kafka and why use it
☐ Topics, partitions, consumers, producers explained
☐ Offset management and consumer groups
☐ Replication factor and durability

**Banking Patterns**:
☐ Message duplication prevention
☐ Message ordering strategy
☐ Consumer lag resolution
☐ Exactly-once semantics

**Error Handling**:
☐ DLQ strategy and monitoring
☐ Retry logic (transient vs permanent)
☐ Idempotency keys implementation
☐ Database unique constraints

**Advanced Topics**:
☐ Kafka Streams for real-time processing
☐ Schema Registry for data governance
☐ Audit logging for compliance
☐ Multi-region disaster recovery

**Code Skills**:
☐ Spring Boot producer/consumer
☐ Apache Kafka producer/consumer
☐ DLQ implementation
☐ Kafka Streams topology
☐ Error handling patterns

**System Design**:
☐ Design 10M txn/day system
☐ Handle failures gracefully
☐ Monitor and alert appropriately
☐ Scale horizontally (consumers)

---

## 🎯 You're Ready!

With this comprehensive guide, you have:

✅ Theory: Core concepts explained simply
✅ Code: Spring Boot + Apache Kafka examples
✅ Banking: Real payment system patterns
✅ Error Handling: DLQ, retry, exactly-once
✅ Real-time: Kafka Streams for fraud detection
✅ Scale: 10M+ txns/day architecture
✅ Audit: Compliance and immutable logs
✅ Interview: Q&A and talking points

**Time to ace that JPMorgan Chase SVP/VP interview! 🏦🚀**

Good luck on March 4th, 2026!
